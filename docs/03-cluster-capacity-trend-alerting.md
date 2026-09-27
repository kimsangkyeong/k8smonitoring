# 03. 클러스터 용량/사용량 추이 · 임계 알림

> 설계요구사항 4 대응 · 클러스터 자원 사용량, 여유 용량 추이, 임계점 초과 알림

## 변경이력

| 버전 | 날짜 | 작성/변경자 | 변경 내용 |
|------|------|-------------|-----------|
| 0.1 | 2026-09-27 | Kiro | 최초 작성 |

---

## 1. 목표

- 클러스터/노드 단위 **현재 사용량**과 **여유(available) 용량**을 동시에 본다.
- **추이(trend)** 를 통해 용량 소진 시점을 예측한다.
- **임계점 초과** 시 Alertmanager로 알림한다. (근거: [R1], [R3])

용량은 두 관점으로 나눕니다.

| 관점 | 의미 | 왜 중요한가 |
|------|------|-------------|
| Requests(할당) | 스케줄러가 예약한 양 | 이게 꽉 차면 신규 Pod 스케줄 불가(실사용 낮아도) |
| Usage(실사용) | 실제 소비량 | 노드 과부하/스로틀링/OOM 위험 |

> 두 값을 함께 봐야 "예약은 꽉 찼는데 실사용은 낮음(과다요청)" 또는 "예약 여유 있는데 실사용 급증(핫스팟)"을 구분할 수 있습니다. → 튜닝은 [02](02-pod-rightsizing-tuning.md).

---

## 2. 핵심 PromQL

### 2.1 클러스터 CPU 할당률 / 사용률
```promql
# CPU requests 할당률(%) = 예약된 CPU / 할당가능(allocatable) CPU
100 * sum(kube_pod_container_resource_requests{resource="cpu"})
      / sum(kube_node_status_allocatable{resource="cpu"})

# CPU 실사용률(%)
100 * sum(rate(container_cpu_usage_seconds_total{container!="",container!="POD"}[5m]))
      / sum(kube_node_status_allocatable{resource="cpu"})
```

### 2.2 클러스터 메모리 할당률 / 사용률
```promql
# Memory requests 할당률(%)
100 * sum(kube_pod_container_resource_requests{resource="memory"})
      / sum(kube_node_status_allocatable{resource="memory"})

# Memory 실사용률(%) (working set 기준)
100 * sum(container_memory_working_set_bytes{container!="",container!="POD"})
      / sum(kube_node_status_allocatable{resource="memory"})
```

### 2.3 여유 용량 (남은 스케줄 가능량)
```promql
# 남은 할당 가능 CPU(코어)
sum(kube_node_status_allocatable{resource="cpu"})
  - sum(kube_pod_container_resource_requests{resource="cpu"})

# 남은 할당 가능 Memory(bytes)
sum(kube_node_status_allocatable{resource="memory"})
  - sum(kube_pod_container_resource_requests{resource="memory"})
```

### 2.4 노드별 여유(가장 빡빡한 노드 찾기)
```promql
# 노드별 CPU request 할당률 상위
topk(5, 100 *
  sum(kube_pod_container_resource_requests{resource="cpu"}) by (node)
  / sum(kube_node_status_allocatable{resource="cpu"}) by (node))
```

---

## 3. 추이 · 소진 예측

### 3.1 추이 보기 (대시보드)
- OCP 콘솔 **Observe → Metrics**에 위 쿼리를 입력하고 시간 범위를 6h/1d/1w로 조절해 추이 확인. (근거: [R7])
- **Observe → Dashboards**의 "Kubernetes / Compute Resources / Cluster" 계열 대시보드로 클러스터/네임스페이스 사용량 추이 확인.

### 3.2 용량 소진 예측 (predict_linear)
```promql
# 현재 추세로 4시간 뒤 메모리 할당률이 100% 도달할지 예측
predict_linear(
  ( sum(kube_pod_container_resource_requests{resource="memory"})
    / sum(kube_node_status_allocatable{resource="memory"}) )[6h:], 4*3600
) > 1
```
> 디스크/용량류 예측에 유용합니다. 노이즈가 있으면 관찰창(`[6h:]`)과 예측 구간을 조정하세요.

---

## 4. 임계 알림 규칙 (요구 4)

CMO `AlertingRule`로 등록합니다. (근거: [R6]) `risk` 라벨로 위험도 등급을 부여합니다(문서 05).

```yaml
apiVersion: monitoring.openshift.io/v1
kind: AlertingRule
metadata:
  name: cluster-capacity-thresholds
  namespace: openshift-monitoring
spec:
  groups:
    - name: capacity.rules
      rules:
        - alert: ClusterCpuRequestsHigh
          expr: 100 * sum(kube_pod_container_resource_requests{resource="cpu"}) / sum(kube_node_status_allocatable{resource="cpu"}) > 85
          for: 15m
          labels: { severity: warning, risk: medium }
          annotations:
            summary: "클러스터 CPU 예약률 85% 초과 (신규 스케줄 여유 부족)"
        - alert: ClusterCpuRequestsCritical
          expr: 100 * sum(kube_pod_container_resource_requests{resource="cpu"}) / sum(kube_node_status_allocatable{resource="cpu"}) > 95
          for: 10m
          labels: { severity: critical, risk: high }
          annotations:
            summary: "클러스터 CPU 예약률 95% 초과 (신규 Pod 스케줄 실패 임박)"
        - alert: ClusterMemoryUsageCritical
          expr: 100 * sum(container_memory_working_set_bytes{container!="",container!="POD"}) / sum(kube_node_status_allocatable{resource="memory"}) > 90
          for: 10m
          labels: { severity: critical, risk: high }
          annotations:
            summary: "클러스터 메모리 실사용률 90% 초과 (OOM/노드 압박 위험)"
        - alert: ClusterMemoryExhaustionPredicted
          expr: |
            predict_linear(
              (sum(kube_pod_container_resource_requests{resource="memory"})
               / sum(kube_node_status_allocatable{resource="memory"}))[6h:], 4*3600) > 1
          for: 30m
          labels: { severity: warning, risk: medium }
          annotations:
            summary: "현재 추세로 4시간 내 메모리 예약 소진 예측"
```

### 4.1 권고 임계값 (시작점)

| 지표 | 주의(중) | 위험(상) |
|------|----------|----------|
| CPU/Memory 예약률 | 85% | 95% |
| Memory 실사용률 | 80% | 90% |
| 노드 루트 FS 사용률 | 80% | 90% |
| PV 사용률(StatefulSet) | 80% | 90% |

> 임계값은 환경별로 조정하세요. 처음엔 보수적으로 시작해 알림 피로를 관찰하며 튜닝합니다.

---

## 5. 다중 클러스터(4.20/4.22) 관점

- 각 클러스터의 CMO Prometheus가 독립적으로 평가/알림합니다(권고 기본안, 비용 0).
- 여러 클러스터를 한 화면에서 비교하려면 [00 문서 4장](00-overview-recommendation.md)의 통합 옵션(onTune 집약 또는 오픈소스 Grafana) 참고.
- bastion에서 클러스터별 용량 스냅샷을 동일 포맷으로 뽑는 스크립트 → [06](06-disconnected-bastion-operations.md).

---

## 6. 요약

- 예약률(requests)과 실사용률을 **함께** 감시하고, 여유 용량을 수치화.
- `predict_linear`로 소진 시점 예측.
- 85%/95% 등 임계 초과를 `AlertingRule`로 알림하고 `risk` 라벨로 등급 부여.

> 근거 링크는 [09-references.md](09-references.md).
