# 02. Pod 리소스 적정성 분석 · 튜닝 가이드

> 설계요구사항 3 대응 · requests/limits 적정성 분석과 튜닝 절차

## 변경이력

| 버전 | 날짜 | 작성/변경자 | 변경 내용 |
|------|------|-------------|-----------|
| 0.1 | 2026-09-27 | Kiro | 최초 작성 |

---

## 1. 왜 적정성이 중요한가

- **너무 작게 요청(request)** 하면 스케줄링은 쉬우나 CPU 스로틀링/OOM kill 위험. (근거: [R14])
- **너무 크게 요청** 하면 노드 용량을 선점하여 낭비. requests는 스케줄러가 배치를 결정하는 가장 중요한 속성입니다. (근거: [R15])
- 목표: **실사용에 근접한 request + 여유 있는 limit**으로 안정성과 밀집도를 동시에 확보.

---

## 2. 적정성 판정 기준 (분석 규칙)

컨테이너별로 아래를 비교합니다. (PromQL은 CMO Prometheus 기준)

### 2.1 실사용 대비 request 비율

```promql
# CPU: 실사용(초당 코어) / request. 지속적으로 낮으면 과다요청(낭비)
sum(rate(container_cpu_usage_seconds_total{container!="",container!="POD"}[5m])) by (namespace,pod,container)
  / on(namespace,pod,container)
  kube_pod_container_resource_requests{resource="cpu"}

# Memory: 실사용 working set / request. 1에 근접/초과하면 과소요청(OOM 위험)
container_memory_working_set_bytes{container!="",container!="POD"}
  / on(namespace,pod,container)
  kube_pod_container_resource_requests{resource="memory"}
```
> `container_memory_working_set_bytes`는 OOM 판정과 가장 관련 깊은 메모리 지표입니다. (근거: [R12], [R13])

### 2.2 CPU 스로틀링 비율 (limit 과소 신호)

```promql
# 스로틀링된 기간 비율. 높으면 CPU limit 상향 검토
sum(rate(container_cpu_cfs_throttled_periods_total{container!=""}[5m])) by (namespace,pod,container)
  / sum(rate(container_cpu_cfs_periods_total{container!=""}[5m])) by (namespace,pod,container)
```

### 2.3 판정표

| 신호 | 해석 | 조치 |
|------|------|------|
| CPU 사용/request 지속 < 30% | 과다요청 | request 하향 (실사용 p95 기준) |
| Mem working_set/request 지속 > 90% | 과소요청 | request 상향, 누수 여부는 [04](04-anomaly-detection-alerting.md) |
| CPU throttled 비율 > 25% | limit 과소 | CPU limit 상향 또는 제거 검토 |
| OOMKilled 발생 | limit 과소/누수 | memory limit 상향 또는 누수 조사 |
| request 미설정 | 스케줄 품질 저하 | request 명시 (BestEffort 지양) |

---

## 3. VPA 권고값 활용 (수치 근거 확보)

수동 계산 대신 **Vertical Pod Autoscaler(VPA)의 recommender**로 컨테이너별 권고 request/limit을 산출할 수 있습니다. VPA는 Deployment/StatefulSet 등 워크로드의 실제 사용량을 관찰하여 rightsizing 값을 제안합니다. (근거: [R16])

### 3.1 권고: "권고 전용(Off) 모드" 사용

- `updateMode: "Off"` 로 두면 VPA가 **Pod를 재기동하지 않고 권고값만 산출**합니다. 실무에서 가장 널리 쓰는 안전한 방식입니다. (근거: [R16], [R17])
- 산출된 값을 검토 후 배포 시점에 수동으로 requests에 반영 → 예측 가능하고 안전.

```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: sample-app-vpa
  namespace: team-a
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment          # StatefulSet도 지원
    name: sample-app
  updatePolicy:
    updateMode: "Off"         # 권고만 산출, 자동 적용 안 함
```

권고값 조회:
```bash
oc get vpa sample-app-vpa -n team-a -o jsonpath='{.status.recommendation.containerRecommendations}' | jq
```

### 3.2 자동 적용(Auto/Recreate) 주의

- Auto 모드는 Pod를 재생성하여 값을 적용하므로 **StatefulSet/무중단 요구 워크로드에는 신중**해야 합니다. 다수 팀이 권고 전용으로만 쓰는 이유입니다. (근거: [R17])
- HPA(수평 확장)와 VPA를 같은 CPU/메모리 지표로 동시에 자동 적용하면 충돌하므로, VPA는 권고 전용으로만 사용 권고. (근거: [R16])

### 3.3 폐쇄망 도입 주의 (비용/제약)

- OCP에서 VPA는 별도 Operator(Vertical Pod Autoscaler Operator)로 제공되며, 폐쇄망에서는 이미지 미러링이 선행되어야 합니다. → [06](06-disconnected-bastion-operations.md)
- **차선안(0원)**: VPA 도입 전이라도 2장의 PromQL(실사용 p95, 스로틀링)만으로 수동 rightsizing이 가능합니다. VPA는 정밀도를 높이는 보강 수단입니다.

---

## 4. 튜닝 절차 (표준 워크플로우)

1. **관찰 기간 확보**: 최소 1~2주(주기적 부하 포함) 지표 축적.
2. **기준값 산출**:
   - CPU request = 최근 2주 사용량 **p95**
   - Memory request = 최근 2주 working_set **최대치 + 20% 여유**
3. **VPA 권고값과 교차 검증**(선택): 큰 차이가 나면 부하 패턴 재검토.
4. **점진 적용**: 한 번에 크게 바꾸지 말고 단계적으로. 변경 후 스로틀링/OOM 재관찰.
5. **가드레일 설정**: 네임스페이스에 `LimitRange`/`ResourceQuota`로 상·하한과 총량 통제.
6. **재평가 주기화**: 분기별 또는 트래픽 변동 시 재산출.

### 4.1 p95 기반 CPU request 산출 쿼리
```promql
quantile_over_time(0.95,
  sum(rate(container_cpu_usage_seconds_total{namespace="team-a",container!="",container!="POD"}[5m])) by (pod,container)[14d:5m]
)
```

### 4.2 Memory request 산출(피크+여유)
```promql
1.2 * max_over_time(
  container_memory_working_set_bytes{namespace="team-a",container!="",container!="POD"}[14d:5m]
)
```

---

## 5. 가드레일 예시 (LimitRange)

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: team-a-defaults
  namespace: team-a
spec:
  limits:
    - type: Container
      default:            # limit 미지정 시 기본값
        cpu: "500m"
        memory: "512Mi"
      defaultRequest:     # request 미지정 시 기본값
        cpu: "100m"
        memory: "128Mi"
      max:
        cpu: "2"
        memory: "2Gi"
```

---

## 6. 리포트 아이디어 (운영 편의, 요구 8)

- **주간 rightsizing 리포트**: 네임스페이스별 "과다요청 Top N", "스로틀링 Top N", "OOM 발생 컨테이너"를 대시보드/CSV로 정리.
- bastion 스크립트로 `oc`+PromQL API를 호출해 클러스터별 동일 포맷 산출 → [06](06-disconnected-bastion-operations.md).

---

## 7. 요약

- 실사용 대비 request/limit을 PromQL로 정량 분석 → 과다/과소 판정.
- VPA는 **권고 전용(Off) 모드**로 안전하게 수치 근거 확보.
- p95 CPU / 피크+20% 메모리 기준으로 점진 적용, LimitRange로 가드레일.

> 근거 링크는 [09-references.md](09-references.md).
