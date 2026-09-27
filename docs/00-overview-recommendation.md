# 00. 권고안 · 아키텍처 개요

> 설계요구사항 9(도구 역할별 통일성 있는 권고), 8(운영 편의) 대응 · 다른 요구사항 문서의 상위 요약

## 변경이력

| 버전 | 날짜 | 작성/변경자 | 변경 내용 |
|------|------|-------------|-----------|
| 0.1 | 2026-09-27 | Kiro | 최초 작성 |

---

## 1. 설계 원칙

폐쇄망 + 복수 클러스터(OCP 4.20 / 4.22) + 회사 표준 onTune 환경을 전제로, 다음 4가지 원칙을 따릅니다.

1. **기본 내장 우선 (No extra license)** — OCP에 기본 포함된 Cluster Monitoring Operator(CMO) 기반 Prometheus/Alertmanager를 1차 데이터 소스로 사용합니다. CMO 옵션 외 방식은 지원되지 않으므로, 설정은 반드시 CMO ConfigMap을 통해서만 수행합니다. (근거: [R1], [R5])
2. **표준 지표 파이프라인 통일** — 모든 클러스터가 동일한 지표 출처(kubelet/cAdvisor, node-exporter, kube-state-metrics, CRI-O 지표)를 쓰도록 하여 클러스터 간 비교와 튜닝 기준을 통일합니다. (근거: [R8], [R9], [R11])
3. **단일 알림 관문** — 모든 알림은 Alertmanager를 단일 관문으로 통과시키고, 위험도 '상'만 사내 알림서버/onTune로 라우팅합니다. (근거: [R3], [R6])
4. **운영자 중심** — 신규 UI 학습을 최소화하기 위해 OCP 웹 콘솔의 Observe(Dashboards/Metrics/Alerting)와 내장 Node 대시보드를 기본 운영 창구로 사용합니다. (근거: [R2], [R7])

---

## 2. 도구 역할 분담 (요구 9)

| 계층 | 도구 (역할) | 수집 대상 | 문서 |
|------|-------------|-----------|------|
| 컨테이너 엔진/노드 | CRI-O 지표 + kubelet/cAdvisor + node-exporter | 런타임 안정성, OOM, 노드 자원부족 | [01](01-container-engine-monitoring.md) |
| 오브젝트 상태 | kube-state-metrics (KSM) | Pod/StatefulSet 상태, 재시작, 스케줄 실패 | [04](04-anomaly-detection-alerting.md) |
| 지표 저장/평가 | Prometheus (CMO 관리) | 시계열 저장 + rule 평가 | [03](03-cluster-capacity-trend-alerting.md) |
| 알림 라우팅 | Alertmanager (CMO 관리) | 알림 그룹핑/억제/라우팅 | [05](05-risk-grading-ontune-integration.md) |
| 리소스 적정성 | Vertical Pod Autoscaler(VPA) recommender (권고 전용 모드) | requests/limits 권고값 산출 | [02](02-pod-rightsizing-tuning.md) |
| 시각화/운영 | OCP 웹 콘솔 Observe + (선택) Grafana | 대시보드, 추이 | [06](06-disconnected-bastion-operations.md) |
| 전사 통합/알림 | onTune (회사 표준) | '상' 등급 알림 수신, 인프라 통합 뷰 | [05](05-risk-grading-ontune-integration.md) |

> 원칙: **"수집·평가·알림은 클러스터 내장 스택으로 통일, 전사 통합/최종 알림은 onTune로 집약"**. 도구를 늘리지 않고 역할 경계를 명확히 하여 운영 편의성을 확보합니다.

---

## 3. 권고 스택 (통일안)

```
[각 클러스터 (OCP 4.20 / 4.22)]
  CRI-O metrics ─┐
  kubelet/cAdvisor ─┤
  node-exporter ─┤──▶ Prometheus (CMO) ──▶ Alertmanager (CMO)
  kube-state-metrics ─┘         │                    │
                                │ (recording/alerting rules)
                                ▼                    ▼ (위험도 '상'만)
                        OCP 콘솔 Observe        사내 알림서버 / onTune
                                ▲
                        VPA recommender (권고값 조회)

[bastion 서버] ── oc 명령 ──▶ 각 클러스터 API (수집/점검 스크립트)
```

- **다중 클러스터 통합 뷰(선택)**: 클러스터별 콘솔로 충분하면 추가 비용 0. 통합 단일 화면이 필요하면 아래 4장 옵션 참고.

---

## 4. 다중 클러스터 통합 옵션 (비용 주의 · 승인 필요)

복수 클러스터를 하나의 화면에서 보고 싶을 때의 선택지입니다. 폐쇄망에서는 모든 이미지가 미러링 가능해야 합니다.

| 옵션 | 비용 | 장점 | 단점 | 권고 |
|------|------|------|------|------|
| A. 클러스터별 내장 콘솔만 사용 | 0 | 추가 설치 없음, 지원 범위 | 통합 뷰 없음 | **1차 기본 (권고)** |
| B. onTune로 핵심 지표 집약 | 라이선스(기존 보유) | 회사 표준, 인프라와 통합 | 컨테이너 상세는 제한적일 수 있음 | 전사 알림/통합 뷰 |
| C. 중앙 Grafana(오픈소스) + 클러스터별 Prometheus datasource | 0(오픈소스)+운영 | 상세 대시보드 통합 | 폐쇄망 미러링/운영 부담 | 상세 통합 필요 시 |
| D. 상용 통합 관측 SaaS/온프렘 | 높음 | 기능 풍부 | 고비용, 폐쇄망 제약 | 비권고(비용) |

> **차선의 비용 절감안(기본요구 4)**: 우선 A(0원)로 시작하고, 통합 알림은 이미 보유한 onTune(B)를 활용합니다. 상세 통합 대시보드가 꼭 필요할 때만 C(오픈소스 Grafana)를 승인 후 도입합니다. D는 비용 문제로 권고하지 않습니다.

---

## 5. 운영 편의성 요약 (요구 8)

- **단일 진입점**: OCP 웹 콘솔 → Observe. 대시보드/메트릭/알림을 한 곳에서 확인.
- **내장 Node 대시보드**: Observe → Dashboards → "Node cluster"에서 OOM Top3, 노드 system-reserved 초과 등 Critical 위젯 즉시 확인. (근거: [R2])
- **bastion 점검 스크립트**: 반복 점검을 `oc` 스크립트로 표준화하여 클러스터 간 동일 절차 적용. → [06](06-disconnected-bastion-operations.md)
- **알림 피로 저감**: 위험도 등급으로 '상'만 사람에게 push, '중/하'는 대시보드/리포트로 확인. → [05](05-risk-grading-ontune-integration.md)

---

## 6. 다음 단계

1. [01 컨테이너 엔진 모니터링](01-container-engine-monitoring.md)부터 순서대로 적용.
2. 각 문서의 PromQL/알림 규칙을 CMO 사용자 정의 규칙으로 등록.
3. [05](05-risk-grading-ontune-integration.md)에서 onTune 연동 인터페이스 확정 후 '상' 등급 라우팅 활성화.

> 근거 링크는 [09-references.md](09-references.md)에서 관리합니다.
