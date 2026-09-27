# 04. 이상징후 모니터링 · 알림

> 설계요구사항 5 대응 · StatefulSet 정상기동 실패, 메모리 릭 의심, CPU 과다 사용 Pod

## 변경이력

| 버전 | 날짜 | 작성/변경자 | 변경 내용 |
|------|------|-------------|-----------|
| 0.1 | 2026-09-27 | Kiro | 최초 작성 |

---

## 1. 다루는 이상징후

| 유형 | 대표 신호 | 데이터 출처 |
|------|-----------|-------------|
| A. StatefulSet 기동 실패 (worker node 이상 등) | replica 미충족, Pod Pending/스케줄 실패 | kube-state-metrics(KSM) |
| B. 메모리 릭 의심 | working_set이 재시작 없이 우상향, OOMKilled 반복 | cAdvisor + KSM |
| C. CPU 과다 사용 Pod | CPU 사용 급증, 지속 스로틀링 | cAdvisor |

KSM은 Deployment/Pod 등 오브젝트 상태를 Prometheus가 쓰기 쉬운 지표로 노출합니다. (근거: [R18], [R19])

---

## 2. A. StatefulSet 정상기동 실패

### 2.1 배경
StatefulSet은 순서 보장·안정 네트워크/스토리지를 갖는 Pod 집합입니다. worker node 이상, PV 결합 실패, 스케줄 불가 등으로 원하는 replica 수를 채우지 못하면 서비스 영향이 큽니다.

### 2.2 PromQL
```promql
# 준비된 replica < 원하는 replica (기동 미충족)
kube_statefulset_status_replicas_ready < kube_statefulset_replicas

# 오래 Pending 상태인 Pod (스케줄 실패 포함)
kube_pod_status_phase{phase="Pending"} == 1

# 스케줄 불가 상태(Unschedulable) 노드
kube_node_spec_unschedulable == 1

# NotReady 노드 (worker node 이상)
kube_node_status_condition{condition="Ready",status="true"} == 0
```
> `kube_pod_container_status_last_terminated_reason{reason="OOMKilled"}`는 다른 이유로 재종료되기 전까지 값이 유지되므로, 최근성 판단은 재시작 증가량과 함께 봅니다. (근거: [R12])

### 2.3 알림 규칙
```yaml
apiVersion: monitoring.openshift.io/v1
kind: AlertingRule
metadata:
  name: statefulset-health
  namespace: openshift-monitoring
spec:
  groups:
    - name: statefulset.rules
      rules:
        - alert: StatefulSetReplicasUnavailable
          expr: (kube_statefulset_status_replicas_ready < kube_statefulset_replicas)
          for: 10m
          labels: { severity: critical, risk: high }
          annotations:
            summary: "StatefulSet {{ $labels.namespace }}/{{ $labels.statefulset }} replica 미충족"
            description: "worker node 이상/PV 결합/스케줄 실패 여부 점검."
        - alert: PodPendingTooLong
          expr: kube_pod_status_phase{phase="Pending"} == 1
          for: 15m
          labels: { severity: warning, risk: medium }
          annotations:
            summary: "Pod {{ $labels.namespace }}/{{ $labels.pod }} 15분 이상 Pending"
        - alert: WorkerNodeNotReady
          expr: kube_node_status_condition{condition="Ready",status="true"} == 0
          for: 5m
          labels: { severity: critical, risk: high }
          annotations:
            summary: "노드 {{ $labels.node }} NotReady"
```

---

## 3. B. 메모리 릭 의심

### 3.1 판정 아이디어
메모리 릭은 **재시작 없이 working_set이 장시간 단조 증가**하거나, **OOMKilled가 주기적으로 반복**되는 패턴입니다. `container_memory_working_set_bytes`가 핵심 지표입니다. (근거: [R13], [R12])

### 3.2 PromQL
```promql
# (1) 최근 6시간 메모리 증가 기울기가 양수이고 큰 컨테이너 (릭 의심)
topk(10,
  deriv(container_memory_working_set_bytes{container!="",container!="POD"}[6h])
) > 0

# (2) 재시작이 없는데도 working_set이 우상향하는지 병행 확인
increase(kube_pod_container_status_restarts_total[6h]) == 0

# (3) OOMKilled 반복 (마지막 종료 사유)
kube_pod_container_status_last_terminated_reason{reason="OOMKilled"} == 1
```

### 3.3 알림 규칙
```yaml
        - alert: SuspectedMemoryLeak
          expr: |
            (deriv(container_memory_working_set_bytes{container!="",container!="POD"}[6h]) > 1048576)
            and (increase(kube_pod_container_status_restarts_total[6h]) == 0)
          for: 30m
          labels: { severity: warning, risk: medium }
          annotations:
            summary: "메모리 릭 의심: {{ $labels.namespace }}/{{ $labels.pod }}/{{ $labels.container }}"
            description: "재시작 없이 working_set 지속 증가(>1MiB/s 추세). 힙덤프/프로파일 점검."
        - alert: RepeatedOOMKilled
          expr: increase(kube_pod_container_status_restarts_total[1h]) >= 3
                and on(namespace,pod,container) kube_pod_container_status_last_terminated_reason{reason="OOMKilled"} == 1
          for: 5m
          labels: { severity: critical, risk: high }
          annotations:
            summary: "OOMKilled 반복: {{ $labels.namespace }}/{{ $labels.pod }}"
```
> 릭 "확진"은 애플리케이션 프로파일링이 필요합니다. 모니터링은 **의심 대상 선별**까지 수행하고, 확인은 개발팀과 협업합니다.

---

## 4. C. CPU 과다 사용 Pod

### 4.1 PromQL
```promql
# CPU 사용 Top 10 (코어)
topk(10, sum(rate(container_cpu_usage_seconds_total{container!="",container!="POD"}[5m])) by (namespace,pod))

# limit 대비 사용률 100%에 근접(과다) - limit 있는 컨테이너
sum(rate(container_cpu_usage_seconds_total{container!="",container!="POD"}[5m])) by (namespace,pod,container)
  / on(namespace,pod,container)
  kube_pod_container_resource_limits{resource="cpu"} > 0.9

# 지속 스로틀링(위 [02]와 연계)
sum(rate(container_cpu_cfs_throttled_periods_total[5m])) by (namespace,pod,container)
  / sum(rate(container_cpu_cfs_periods_total[5m])) by (namespace,pod,container) > 0.25
```

### 4.2 알림 규칙
```yaml
        - alert: PodCpuUsageHigh
          expr: |
            sum(rate(container_cpu_usage_seconds_total{container!="",container!="POD"}[5m])) by (namespace,pod,container)
            / on(namespace,pod,container) kube_pod_container_resource_limits{resource="cpu"} > 0.9
          for: 15m
          labels: { severity: warning, risk: medium }
          annotations:
            summary: "CPU 과다: {{ $labels.namespace }}/{{ $labels.pod }} limit의 90% 초과 지속"
        - alert: PodCpuThrottlingHigh
          expr: |
            sum(rate(container_cpu_cfs_throttled_periods_total[5m])) by (namespace,pod,container)
            / sum(rate(container_cpu_cfs_periods_total[5m])) by (namespace,pod,container) > 0.25
          for: 15m
          labels: { severity: warning, risk: medium }
          annotations:
            summary: "CPU 스로틀링 과다: {{ $labels.namespace }}/{{ $labels.pod }} (limit 상향 검토)"
```

> 주의: CPU 사용이 낮아 보여도 스로틀링으로 애플리케이션이 느려질 수 있습니다. 사용률과 스로틀링을 함께 봐야 합니다. (근거: [R14])

---

## 5. 이상징후 → 위험도 매핑 (요구 6 연계)

| 이상징후 | 기본 위험도 | 근거 |
|----------|-------------|------|
| worker node NotReady / StatefulSet replica 미충족 | 상(high) | 서비스 가용성 직접 영향 |
| OOMKilled 반복 | 상(high) | 데이터 손실/재시작 루프 |
| 메모리 릭 의심(재시작 전) | 중(medium) | 확진 전, 추적 필요 |
| CPU 과다/스로틀링 | 중(medium) | 성능 저하, 즉시 장애는 아님 |
| 단발성 Pending | 하(low) | 일시적일 수 있음 |

> 위험도 라벨(`risk`)로 [05 문서](05-risk-grading-ontune-integration.md)에서 '상'만 사내 알림서버/onTune로 라우팅합니다.

---

## 6. bastion 즉시 점검 (요구 2)
```bash
# StatefulSet 상태
oc get statefulset -A
oc get pods -A --field-selector=status.phase=Pending

# 노드 이상
oc get nodes | grep -v " Ready"

# 특정 Pod 자원/이벤트
oc describe pod <pod> -n <ns>
oc adm top pod <pod> -n <ns> --containers
```

---

## 7. 요약

- KSM으로 StatefulSet/노드 상태를, cAdvisor로 메모리·CPU 추세를 감시.
- 메모리 릭은 "재시작 없는 working_set 우상향 + OOM 반복"으로 의심 선별.
- CPU는 사용률과 스로틀링을 함께 평가.
- 모든 알림에 `risk` 라벨 부여 → [05](05-risk-grading-ontune-integration.md).

> 근거 링크는 [09-references.md](09-references.md).
