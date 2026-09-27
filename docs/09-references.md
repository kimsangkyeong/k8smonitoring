# 09. 참고자료 (근거 링크 통합)

> 이 문서는 전체 문서 세트의 근거 출처를 통합 관리합니다(기본요구 5). 각 문서 본문은 `[R번호]`로 여기를 참조합니다.
> 링크는 작성 시점(2026-09-27)에 정상 호출을 확인했습니다. 폐쇄망에서는 접근이 제한될 수 있으니, 사내 미러/캐시 문서와 교차 확인하세요.

## 변경이력

| 버전 | 날짜 | 작성/변경자 | 변경 내용 |
|------|------|-------------|-----------|
| 0.1 | 2026-09-27 | Kiro | 최초 작성. R1~R28 정리 |

---

## Red Hat OpenShift (OCP 4.20 / 4.22)

| ID | 주제 | 링크 |
|----|------|------|
| R1 | 모니터링 스택 아키텍처(Prometheus/Alertmanager 기반) | https://docs.redhat.com/en/documentation/monitoring_stack_for_red_hat_openshift/4.20/html-single/about_monitoring/index |
| R2 | Node metrics 대시보드(CRI-O/Kubelet, OOM Top3, system-reserved 임계) | https://docs.redhat.com/en/documentation/OpenShift_container_platform/4.20/html/nodes/nodes-dashboard-using |
| R3 | Managing alerts(알림/Silence/규칙 개요) | https://docs.redhat.com/en/documentation/monitoring_stack_for_red_hat_openshift/4.20/html-single/managing_alerts/ |
| R5 | 지원되는 모니터링 구성 방식(CMO ConfigMap만 지원) | https://docs.redhat.com/en/documentation/monitoring_stack_for_red_hat_openshift/4.20/html-single/support_for_monitoring |
| R6 | Monitoring APIs / AlertingRule(사용자 정의 알림 규칙) | https://docs.redhat.com/en/documentation/OpenShift_container_platform/4.20/html/monitoring_apis/monitoring-apis |
| R7 | 핵심 개념 / 대시보드·구성 | https://docs.redhat.com/en/documentation/monitoring_stack_for_red_hat_openshift/4.20/html/monitoring_key_concepts/key-concepts |
| R10 | Node Observability Operator(CRI-O/Kubelet 프로파일링) | https://docs.redhat.com/en/documentation/openshift_container_platform/4.21/html/scalability_and_performance/using-node-observability-operator |
| R20 | 핵심 플랫폼 모니터링 알림/알림전송 구성 | https://docs.redhat.com/en/documentation/monitoring_stack_for_red_hat_openshift/4.20/html/configuring_core_platform_monitoring/configuring-alerts-and-notifications |
| R21 | 핵심 플랫폼 모니터링 구성(Prometheus/Alertmanager) | https://docs.redhat.com/en/documentation/monitoring_stack_for_red_hat_openshift/4.20/html-single/configuring_core_platform_monitoring/ |
| R22 | 알림 알림전송 구성 / Watchdog 알림 | https://docs.redhat.com/en/documentation/OpenShift_container_platform/4.14/html/postinstallation_configuration/configuring-alert-notifications |
| R23 | AlertmanagerConfig CRD(v1beta1) | https://docs.redhat.com/it/documentation/openshift_container_platform/4.20/html/monitoring_apis/alertmanagerconfig-monitoring-coreos-com-v1beta1 |
| R24 | 사용자 정의 프로젝트 알림 라우팅 활성화 | https://docs.redhat.com/en/documentation/openshift_container_platform/4.12/html/monitoring/enabling-alert-routing-for-user-defined-projects |
| R25 | 폐쇄망 OLM(Operator) 사용 | https://docs.redhat.com/fr/documentation/openshift_container_platform/4.20/html/disconnected_environments/olm-restricted-networks |
| R26 | 폐쇄망 이미지 미러링(oc-mirror v2) 4.22 | https://docs.redhat.com/en/documentation/openshift_container_platform/4.22/html/disconnected_environments/about-installing-oc-mirror-v2 |
| R27 | 폐쇄망 미러링 개요(4.22 PDF) | https://docs.redhat.com/en/documentation/openshift_container_platform/4.22/pdf/disconnected_environments/installing-mirroring-disconnected |
| R28 | Alerting UI(Observe → Alerting) | https://docs.redhat.com/en/documentation/monitoring_stack_for_red_hat_openshift/4.20/html-single/managing_alerts/ |

> R2는 본 세트에서 특히 중요한 근거입니다. OOM Top3 기본 쿼리 `topk(3, sum(increase(container_runtime_crio_containers_oom_count_total[1d])) by (name))`와 system-reserved 임계(메모리 80%/50%, CPU 80%/50%) 위젯을 제공합니다.

---

## Kubernetes (1.35+) / cAdvisor / kube-state-metrics

| ID | 주제 | 링크 |
|----|------|------|
| R8 | CRI Pod & Container Metrics(cAdvisor/CRI, beta, CRI-O ≥1.31) | https://kubernetes.io/docs/reference/instrumentation/cri-pod-container-metrics/ |
| R9 | (해설) kubelet·cAdvisor·CRI 지표 수집 원리 | https://learnkube.com/kubernetes-metrics-cadvisor-kubelet-cri |
| R11 | (해설) kubelet metrics: cAdvisor/CRI/node-exporter | https://medium.com/itnext/kubelet-metrics-how-cadvisor-and-cri-collect-kubernetes-stats-3c12b7e512ce |
| R12 | KSM Pod 지표(restarts_total, resource_requests/limits, last_terminated_reason) | https://github.com/kubernetes/kube-state-metrics/blob/main/docs/metrics/workload/pod-metrics.md |
| R13 | (해설) OOM/스로틀링과 working_set 해석 | https://thebackroomtech.com/kubernetes-performance-troubleshooting-cpu-throttling-oomkilled/ |
| R14 | (Q&A) OOMKilled 메트릭·해석 사례 | https://stackoverflow.com/questions/76333545/how-to-get-metric-for-a-spark-pod-oomkilled-using-prometheus |
| R15 | (해설) requests가 스케줄링·용량에 미치는 영향 | https://medium.com/@ymb002/vertical-pod-autoscaling-right-sizing-your-pods-18af3a0d5184 |
| R18 | KSM 개요(README) | https://github.com/kubernetes/kube-state-metrics/blob/main/docs/README.md |
| R19 | (해설) KSM 기반 알림 시작하기 | https://zenn.dev/nissy_dev/articles/kube-state-metrics-alert?locale=en |

---

## Vertical Pod Autoscaler (VPA)

| ID | 주제 | 링크 |
|----|------|------|
| R16 | VPA 공식 개념(k8s 1.35, rightsizing/updateMode) | https://v1-35.docs.kubernetes.io/docs/concepts/workloads/autoscaling/vertical-pod-autoscale/ |
| R17 | (해설) VPA 한계·모범사례(권고 전용 모드 관행) | https://scaleops.com/blog/kubernetes-vpa/ |

---

## onTune (회사 표준 모니터링) — 확인 필요 항목

onTune(TeemStone)의 외부 알림/이벤트 수신 규격(엔드포인트, 인증, 페이로드 포맷)은 **사내 onTune 담당자/제품 문서로 확정**해야 합니다. 본 문서 세트의 [05](05-risk-grading-ontune-integration.md)는 Alertmanager webhook → 사내 게이트웨이 → onTune 변환이라는 표준 인터페이스 패턴을 제시합니다.

| 확인 항목 | 담당 |
|-----------|------|
| onTune 외부 이벤트 수신 방식(API/Webhook/메일/트랩) | 사내 onTune 운영팀 |
| 인증 방식 및 사내 게이트웨이 URL | 사내 인프라/보안팀 |
| 이벤트 필드 매핑(risk/severity/namespace/alertname → onTune) | 공동 |

---

## 출처 이용 안내

- 상기 링크의 서술은 라이선스 준수를 위해 요약·재구성했습니다. (Content was rephrased for compliance with licensing restrictions)
- 해설(블로그/Q&A) 링크는 보조 참고이며, 공식 문서(Red Hat / kubernetes.io / GitHub 공식 저장소)를 우선 신뢰하세요.
- 링크 상태는 시간이 지나면 변경될 수 있습니다. 폐쇄망 반입 전 사내에서 재검증을 권고합니다.
