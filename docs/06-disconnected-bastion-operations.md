# 06. 폐쇄망 bastion 운영 가이드

> 설계요구사항 2(폐쇄망 bastion `oc` 수집, OCP 4.20/4.22 복수 클러스터), 8(운영 편의) 대응

## 변경이력

| 버전 | 날짜 | 작성/변경자 | 변경 내용 |
|------|------|-------------|-----------|
| 0.1 | 2026-09-27 | Kiro | 최초 작성 |

---

## 1. 전제 환경

- 인터넷 직접 연결 불가(air-gapped/disconnected). bastion 서버에서 `oc`로 각 클러스터 API 접근.
- OCP 4.20, 4.22 클러스터가 복수 존재. 클러스터별 kubeconfig/컨텍스트로 구분.
- 추가 컴포넌트(VPA Operator, Node Observability Operator, Grafana 등)는 **이미지 미러링**이 선행되어야 설치 가능. (근거: [R25], [R26])

---

## 2. 폐쇄망에서 추가 도구 설치 (미러링)

내장 CMO 스택은 기본 설치되어 있어 추가 미러링이 불필요합니다(권고 기본안, 비용 0). 아래는 **선택 도구**를 도입할 때만 필요합니다.

- 폐쇄망 클러스터는 사설 레지스트리에 미러링된 이미지에서 동작하며, 이 레지스트리는 클러스터 구동 내내 가동되어야 합니다. (근거: [R26])
- OLM 기반 Operator(VPA, Node Observability 등)는 폐쇄망에서 기본 OperatorHub 소스에 접근할 수 없으므로, 카탈로그/이미지를 미러링해야 합니다. (근거: [R25])
- 미러링은 `oc-mirror` 플러그인(v2 권장)으로 수행합니다. 인터넷 연결된 시스템에서 이미지를 받아 사설 레지스트리로 옮깁니다. (근거: [R26], [R27])

```bash
# (인터넷 가능 구간) 이미지 세트 미러링 → 사설 레지스트리
oc mirror --config=imageset-config.yaml docker://registry.internal.corp:5000

# (폐쇄망) 미러된 카탈로그를 CatalogSource로 등록 후 Operator 설치
oc apply -f mirror-catalogsource.yaml
```

> **비용/부담 주의(기본요구 4)**: 미러링·운영 부담이 크므로, **1차는 내장 CMO만으로 요구 1~6을 충족**하도록 설계했습니다. 선택 도구는 필요성이 확인될 때 승인 후 미러링합니다.

---

## 3. bastion 기본 셋업

```bash
# 클러스터별 컨텍스트 등록 (예)
oc login https://api.cluster420.corp:6443 --token=... 
oc config rename-context <current> ocp420
oc login https://api.cluster422.corp:6443 --token=...
oc config rename-context <current> ocp422

# 컨텍스트 전환
oc config use-context ocp420
```

권장: 읽기 전용 서비스어카운트 + 최소 권한(RBAC)으로 수집 전용 토큰을 발급해 bastion 스크립트에 사용(보안).

---

## 4. 표준 수집 스크립트 (다중 클러스터 동일 포맷)

여러 클러스터에 동일 점검을 반복 적용하는 예시입니다. `oc` 출력만 사용하므로 폐쇄망에서 동작합니다.

```bash
#!/usr/bin/env bash
# collect.sh - 클러스터별 스냅샷 수집 (읽기 전용)
set -euo pipefail
CONTEXTS=("ocp420" "ocp422")
OUT="./snapshots/$(date +%Y%m%d-%H%M)"
mkdir -p "$OUT"

for ctx in "${CONTEXTS[@]}"; do
  echo "== $ctx =="
  d="$OUT/$ctx"; mkdir -p "$d"
  oc --context "$ctx" get nodes -o wide                > "$d/nodes.txt"
  oc --context "$ctx" adm top nodes                    > "$d/top-nodes.txt"  || true
  oc --context "$ctx" adm top pods -A --sum            > "$d/top-pods.txt"   || true
  oc --context "$ctx" get statefulset -A               > "$d/statefulsets.txt"
  oc --context "$ctx" get pods -A --field-selector=status.phase=Pending > "$d/pending.txt"
  oc --context "$ctx" get pods -A -o json \
    | jq -r '.items[] | select(any(.status.containerStatuses[]?; .restartCount>3))
             | "\(.metadata.namespace)/\(.metadata.name) restarts"' > "$d/high-restarts.txt" || true
  oc --context "$ctx" get events -A --field-selector type=Warning     > "$d/warnings.txt"    || true
done
echo "saved to $OUT"
```

> `oc adm top`은 metrics-server/monitoring 지표에 의존합니다. 값이 비면 클러스터 모니터링 상태를 먼저 확인하세요.

---

## 5. PromQL을 bastion에서 조회 (Thanos Querier 경유)

OCP는 인증이 필요한 `thanos-querier` 경로로 PromQL API를 제공합니다. bastion에서 토큰으로 질의할 수 있습니다.

```bash
CTX=ocp420
TOKEN=$(oc --context $CTX whoami -t)
HOST=$(oc --context $CTX -n openshift-monitoring get route thanos-querier -o jsonpath='{.spec.host}')

# 예: 클러스터 CPU 예약률(%) 조회 (문서 03의 쿼리)
curl -sk -H "Authorization: Bearer $TOKEN" \
  --data-urlencode 'query=100 * sum(kube_pod_container_resource_requests{resource="cpu"}) / sum(kube_node_status_allocatable{resource="cpu"})' \
  "https://$HOST/api/v1/query" | jq '.data.result'
```

> 이 방식으로 [01]~[04]의 PromQL을 스크립트화해 클러스터별 동일 리포트를 만들 수 있습니다. (요구 8: 운영 편의)

---

## 6. 운영자 편의 가이드 (요구 8)

### 6.1 1차 창구: OCP 웹 콘솔 Observe
- **Observe → Dashboards**: "Node cluster"(엔진/노드 Critical), "Compute Resources / Cluster·Namespace"(용량 추이). (근거: [R2], [R7])
- **Observe → Metrics**: 문서의 PromQL을 붙여넣어 즉석 조회.
- **Observe → Alerting**: 발생 알림/Silence/규칙 확인. (근거: [R28])

### 6.2 운영 루틴 제안
| 주기 | 활동 | 도구 |
|------|------|------|
| 상시 | '상' 알림 대응 | onTune/사내 알림서버([05](05-risk-grading-ontune-integration.md)) |
| 일일 | Node 대시보드 Critical, Pending/재시작 확인 | 콘솔 + collect.sh |
| 주간 | rightsizing 리포트, 용량 추이 검토 | [02](02-pod-rightsizing-tuning.md)/[03](03-cluster-capacity-trend-alerting.md) PromQL |
| 분기 | 임계값/요청값 재튜닝 | [02](02-pod-rightsizing-tuning.md) |

### 6.3 통합 화면이 필요하면 (선택)
- 이미 보유한 **onTune로 핵심 지표 집약**(추가 비용 최소) → [00 문서 4장](00-overview-recommendation.md).
- 상세 통합 대시보드가 꼭 필요하면 **오픈소스 Grafana**(미러링 후) 승인 도입.

---

## 7. 체크리스트

- [ ] 클러스터별 컨텍스트(ocp420/ocp422) 등록 및 읽기전용 토큰
- [ ] `collect.sh` 스케줄링(cron)으로 일일 스냅샷
- [ ] Thanos Querier 경유 PromQL 리포트 스크립트
- [ ] (선택) VPA/Node Observability Operator 미러링 여부 결정
- [ ] Alertmanager '상' 라우팅 → 사내 게이트웨이 연결([05](05-risk-grading-ontune-integration.md))

---

## 8. 요약

- 폐쇄망에서는 내장 CMO만으로 요구 1~6을 충족(비용 0)하고, 선택 도구만 미러링.
- bastion에서 `oc` + Thanos Querier PromQL로 다중 클러스터를 동일 포맷 수집.
- 운영자는 OCP 콘솔 Observe를 1차 창구로, '상' 알림은 onTune로 받는다.

> 근거 링크는 [09-references.md](09-references.md).
