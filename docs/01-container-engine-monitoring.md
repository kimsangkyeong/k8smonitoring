# 01. 컨테이너 엔진 안정성 · 자원 부족 모니터링

> 설계요구사항 1 대응 · 대상: OCP 4.20/4.22 (컨테이너 엔진 = CRI-O), Kubernetes 1.35+

## 변경이력

| 버전 | 날짜 | 작성/변경자 | 변경 내용 |
|------|------|-------------|-----------|
| 0.1 | 2026-09-27 | Kiro | 최초 작성 |

---

## 1. 무엇을 "컨테이너 엔진 안정성"으로 보는가

OCP의 컨테이너 엔진은 **CRI-O**이며, 각 노드에서 kubelet과 함께 동작합니다. 안정성/자원부족은 다음 3개 축으로 감시합니다.

| 축 | 관찰 대상 | 대표 신호 |
|----|-----------|-----------|
| 런타임 건전성 | CRI-O / kubelet 프로세스, 컨테이너 생성/삭제 지연 | 이미지 pull 실패율, 런타임 작업 지연, OOM kill |
| 노드 자원부족 | 노드 CPU/메모리/디스크/PID, system-reserved | system-reserved 초과, MemoryPressure/DiskPressure |
| 지표 파이프라인 | kubelet/cAdvisor, node-exporter | 지표 수집 자체가 끊겼는지 |

지표 출처: kubelet은 cAdvisor를 통해 pod/container 지표를 수집합니다. Kubernetes 1.35 기준으로 CRI 경유 통계(`PodAndContainerStatsFromCRI`)는 beta 기능이며 CRI-O >= 1.31에서 호환됩니다. 기본은 cAdvisor 경로를 사용합니다. (근거: [R11], [R8])

---

## 2. 가장 빠른 시작: OCP 내장 Node 대시보드 (요구 8과 연계)

추가 설치 없이 웹 콘솔에서 확인 가능합니다.

- 경로: **Administrator 관점 → Observe → Dashboards → Dashboard 필터에서 `Node cluster` 선택**
- Critical 카테고리 위젯(엔진/노드 안정성 핵심): (근거: [R2])
  - 지난 1일간 OOM kill이 가장 많은 컨테이너 Top 3
  - 지난 1시간 이미지 pull 실패율
  - system-reserved 메모리 사용률 > 80% 노드
  - kubelet system-reserved 메모리 사용률 > 50% 노드
  - CRI-O system-reserved 메모리 사용률 > 50% 노드
  - system-reserved CPU 사용률 > 80% 노드
  - kubelet / CRI-O system-reserved CPU 사용률 > 50% 노드
- 그 외 Outliers(95퍼센타일 초과 작업), Average durations, Number of operations로 런타임 처리 지연 추이 확인.

> Critical 위젯에 데이터가 없으면 이상 없음(정상)을 의미합니다. (근거: [R2])

---

## 3. 핵심 PromQL (CMO Prometheus에서 그대로 사용)

### 3.1 컨테이너 OOM kill (엔진 안정성 핵심)
Node 대시보드의 기본 쿼리와 동일한 CRI-O 지표를 사용합니다. (근거: [R2])

```promql
# 지난 1일 OOM kill Top 3 컨테이너
topk(3, sum(increase(container_runtime_crio_containers_oom_count_total[1d])) by (name))
```

### 3.2 컨테이너 재시작 급증 (KSM)
```promql
# 최근 1시간 재시작 증가량이 큰 컨테이너
topk(10, increase(kube_pod_container_status_restarts_total[1h]))
```

### 3.3 노드 자원 부족 압박 상태 (KSM node condition)
```promql
# 메모리/디스크/PID 압박 중인 노드
kube_node_status_condition{condition=~"MemoryPressure|DiskPressure|PIDPressure",status="true"} == 1
```

### 3.4 노드 실제 가용 메모리/디스크 (node-exporter)
```promql
# 사용 가능 메모리 비율(%) - 15% 미만이면 위험
100 * (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)

# 루트/이미지 파일시스템 사용률(%) - CRI-O 이미지 저장 공간 고갈 감시
100 * (1 - (node_filesystem_avail_bytes{mountpoint="/"} / node_filesystem_size_bytes{mountpoint="/"}))
```

### 3.5 이미지 pull 실패 (런타임 건전성)
```promql
# 이미지 pull 실패율 (증가분 기준)
sum(rate(kubelet_image_pull_duration_seconds_count[5m])) by (node)
```
> 참고: OCP Node 대시보드는 "지난 1시간 이미지 pull 실패율"을 Critical 위젯으로 이미 제공합니다. 상세 지표명은 클러스터 버전에서 콘솔의 위젯 쿼리로 교차 확인하세요. (근거: [R2])

---

## 4. 권고 알림 규칙 (예시)

CMO에서 지원되는 `AlertingRule`(namespace: `openshift-monitoring`)로 등록합니다. 이는 플랫폼 모니터링 스택에서 알림을 생성하는 지원되는 방법입니다. (근거: [R6])

```yaml
apiVersion: monitoring.openshift.io/v1
kind: AlertingRule
metadata:
  name: container-engine-stability
  namespace: openshift-monitoring
spec:
  groups:
    - name: container-engine.rules
      rules:
        - alert: ContainerFrequentOOMKill
          expr: sum(increase(container_runtime_crio_containers_oom_count_total[15m])) by (name) > 3
          for: 5m
          labels:
            severity: warning
            risk: high        # 위험도 등급(문서 05)에서 사용
          annotations:
            summary: "컨테이너 {{ $labels.name }} 15분 내 OOM kill 다발"
            description: "메모리 한도/누수 점검 필요. 문서 02/04 참고."
        - alert: NodeMemoryPressure
          expr: kube_node_status_condition{condition="MemoryPressure",status="true"} == 1
          for: 5m
          labels:
            severity: warning
            risk: high
          annotations:
            summary: "노드 {{ $labels.node }} MemoryPressure"
        - alert: NodeRootFsAlmostFull
          expr: 100 * (1 - (node_filesystem_avail_bytes{mountpoint="/"} / node_filesystem_size_bytes{mountpoint="/"})) > 85
          for: 10m
          labels:
            severity: warning
            risk: medium
          annotations:
            summary: "노드 {{ $labels.instance }} 루트 파일시스템 85% 초과"
```

> `risk` 라벨은 위험도 등급(상=high/중=medium/하=low)을 알림에 부착하여, [05 문서](05-risk-grading-ontune-integration.md)의 라우팅에서 '상'만 사내 알림서버로 보내는 데 사용합니다.

---

## 5. 심화: CRI-O/kubelet 프로파일링 (선택)

간헐적 런타임 지연/성능 이슈의 근본 원인 분석이 필요하면 **Node Observability Operator**로 CRI-O·kubelet 프로파일링 데이터를 수집·분석할 수 있습니다. 성능 추세 분석과 성능 관련 이슈 디버깅에 사용합니다. (근거: [R10])

- 폐쇄망에서는 해당 Operator 이미지의 미러링이 선행되어야 합니다. → [06 문서](06-disconnected-bastion-operations.md)
- 상시 상주가 아닌 **문제 조사 시점에 한시적으로** 사용 권고(자원/부하 절감).

---

## 6. bastion에서의 빠른 점검 (요구 2 연계)

```bash
# 노드 상태/압박 조건 확인
oc get nodes -o wide
oc adm top nodes

# 특정 노드의 CRI-O/kubelet 상태
oc debug node/<node> -- chroot /host systemctl status crio kubelet

# 최근 OOM/재시작 컨테이너 개괄
oc get pods -A --field-selector=status.phase!=Running
```
> 상세 수집 스크립트는 [06 문서](06-disconnected-bastion-operations.md) 참고.

---

## 7. 요약

- **먼저** OCP 내장 Node 대시보드로 엔진/노드 Critical 신호를 즉시 확인.
- **다음** CRI-O OOM, KSM 재시작, node-exporter 자원부족 PromQL로 알림 규칙화.
- **필요 시** Node Observability Operator로 심화 프로파일링(한시적).
- 알림은 `risk` 라벨을 붙여 [05 문서](05-risk-grading-ontune-integration.md)의 단일 관문으로 보냅니다.

> 근거 링크는 [09-references.md](09-references.md).
