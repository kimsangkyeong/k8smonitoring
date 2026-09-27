# 05. 위험도 등급 · onTune 알림 연동

> 설계요구사항 6(상/중/하 평가 + '상'을 사내 알림서버 연동), 7(onTune 사용) 대응

## 변경이력

| 버전 | 날짜 | 작성/변경자 | 변경 내용 |
|------|------|-------------|-----------|
| 0.1 | 2026-09-27 | Kiro | 최초 작성 |

---

## 1. 설계 개요

```
Prometheus (rule 평가, risk 라벨 부여)
        │
        ▼
Alertmanager (단일 관문: 그룹핑/억제/라우팅)
        ├── risk="high"(상) ──▶ 사내 알림서버 / onTune  (즉시 push)
        ├── risk="medium"(중) ─▶ 티켓/메일/대시보드      (검토용)
        └── risk="low"(하) ────▶ 대시보드/주간 리포트만
```

핵심: **모든 알림은 Alertmanager를 단일 관문으로 통과**하고, `risk` 라벨로 라우팅을 분기합니다. OCP 4.20에서 Alertmanager를 구성해 Prometheus 알림을 엔드포인트 수신자(receiver)로 라우팅하는 것은 지원되는 방식입니다. (근거: [R20], [R21])

---

## 2. 위험도 평가 기준 (상/중/하)

두 축(영향도 × 긴급도)으로 평가하고 표준 라벨로 고정합니다.

| 등급 | `risk` 라벨 | 정의 | 예시 |
|------|-------------|------|------|
| 상 | `high` | 서비스 가용성/데이터에 즉시·직접 영향, 즉시 대응 | 노드 NotReady, StatefulSet replica 미충족, OOMKilled 반복, 클러스터 자원 95% 초과 |
| 중 | `medium` | 성능 저하/잠재 위험, 당일 내 검토 | CPU 스로틀링 과다, 메모리 릭 의심, 자원 85% 초과, 소진 예측 |
| 하 | `low` | 경미/일시적, 추세 관찰 | 단발 Pending, FS 80% 초과 |

> 각 알림 규칙(`AlertingRule`)에 `labels.risk`를 부여합니다. 규칙 예시는 [01](01-container-engine-monitoring.md)/[03](03-cluster-capacity-trend-alerting.md)/[04](04-anomaly-detection-alerting.md) 참고.

### 2.1 등급 산정 매트릭스(참고)

| 영향도 \ 긴급도 | 즉시 | 당일 | 추세 |
|-----------------|------|------|------|
| 가용성/데이터 | 상 | 상 | 중 |
| 성능 | 상 | 중 | 중 |
| 경미 | 중 | 하 | 하 |

---

## 3. onTune 연동 방식

> onTune(TeemStone)은 회사 표준 서버/인프라 성능 모니터링 시스템입니다. 컨테이너 알림을 onTune 또는 사내 알림서버로 보내는 방법은 환경에 따라 아래 3가지 중 선택합니다. **구체 엔드포인트/포맷은 사내 onTune 담당자 확인이 필요**하며, 본 문서는 인터페이스 관점의 표준안을 제시합니다.

| 방식 | 개념 | 장점 | 단점/확인필요 |
|------|------|------|----------------|
| A. Alertmanager Webhook → 사내 수신 게이트웨이 → onTune | Alertmanager가 '상' 알림을 사내 HTTP 엔드포인트로 push, 게이트웨이가 onTune API/포맷으로 변환 | 표준적·유연, onTune 스키마 변경에 게이트웨이만 수정 | 사내 수신 URL/인증, 변환 규격 확정 필요 |
| B. Alertmanager → onTune 지원 채널(email 등) | onTune이 수신 가능한 채널로 직접 전송 | 게이트웨이 불필요 | onTune 수신 채널 지원 여부 확인 |
| C. onTune 에이전트/이벤트 연동 | 노드의 onTune 에이전트가 이벤트/트랩 수신 | 기존 인프라 알림과 통합 | onTune의 외부 이벤트 수신 규격 확인 |

> **권고**: **A(웹훅 + 사내 게이트웨이)**. Alertmanager는 표준 webhook을 지원하고, 게이트웨이에서 onTune 포맷으로 변환하면 결합도가 낮아 유지보수가 쉽습니다. 폐쇄망에서도 사내 네트워크 내 HTTP 호출이므로 외부 연결이 불필요합니다.

---

## 4. 구성 예시 (방식 A: Webhook 라우팅)

### 4.1 핵심 플랫폼 알림 → Alertmanager 라우팅

OCP 4.20에서 Alertmanager 설정은 CMO가 관리하는 `alertmanager-main` 시크릿(또는 콘솔의 Administration → Cluster Settings → Configuration → Alertmanager)에서 구성합니다. (근거: [R21], [R22])

```yaml
# alertmanager.yaml (개념 예시) - '상(risk=high)'만 사내 게이트웨이로 라우팅
route:
  receiver: default
  group_by: ['alertname', 'namespace']
  routes:
    - matchers:
        - risk = "high"
      receiver: company-alert-gateway
      continue: false        # 상 등급은 게이트웨이로만
    - matchers:
        - risk = "medium"
      receiver: ticket-mail
receivers:
  - name: default
  - name: company-alert-gateway
    webhook_configs:
      - url: "http://alert-gateway.internal.corp/api/ocp-alerts"   # 사내 수신 엔드포인트(예시)
        send_resolved: true
  - name: ticket-mail
    # email_configs: [...] 사내 SMTP
```

> 라벨 기반 매처(`risk = "high"`)로 '상' 등급만 사내 알림서버로 라우팅합니다. `send_resolved: true`로 해제 알림도 전달해 상태 동기화.

### 4.2 사용자 정의 프로젝트 알림(선택)

개발팀이 자기 프로젝트 알림 라우팅을 직접 관리해야 하면, `AlertmanagerConfig` CRD로 네임스페이스 범위 라우팅/수신자를 정의할 수 있습니다. 이 CRD는 그룹핑·라우팅·수신자·억제 규칙을 지정합니다. (근거: [R23], [R24])

```yaml
apiVersion: monitoring.coreos.com/v1beta1
kind: AlertmanagerConfig
metadata:
  name: team-a-routing
  namespace: team-a
spec:
  route:
    receiver: team-a-webhook
    matchers:
      - name: risk
        value: high
        matchType: "="
  receivers:
    - name: team-a-webhook
      webhookConfigs:
        - url: "http://alert-gateway.internal.corp/api/team-a"
```

---

## 5. 사내 게이트웨이(변환기) 개념

Alertmanager webhook 페이로드(JSON: `alerts[].labels`, `annotations`, `status` 등)를 받아 onTune가 이해하는 포맷으로 변환하는 얇은 사내 서비스입니다.

- 입력: Alertmanager webhook JSON
- 처리: `risk`, `severity`, `namespace`, `alertname` 매핑 → onTune 이벤트 필드로 변환, 중복 억제
- 출력: onTune API/채널로 전송
- 이점: onTune 스키마가 바뀌어도 클러스터 설정은 불변, 게이트웨이만 수정. 다중 클러스터(4.20/4.22)가 동일 게이트웨이를 공유해 **포맷 통일**.

> 게이트웨이가 과하면(비용/운영) **차선안**: Alertmanager가 onTune이 직접 수신 가능한 채널(email 등, 방식 B)로 바로 전송. 단 필드 매핑 유연성은 낮아집니다.

---

## 6. 검증 · 운영

- **Watchdog 알림**: OCP는 상시 발생하는 watchdog 알림을 제공합니다. 수신 측(게이트웨이/onTune)에서 watchdog이 끊기면 Alertmanager↔수신자 경로 장애로 판단할 수 있습니다. (근거: [R22])
- **연동 테스트 절차**:
  1. 테스트용 `risk=high` 알림을 임시로 발생시켜 게이트웨이 수신 확인.
  2. 게이트웨이 로그에서 onTune 전송 성공 확인.
  3. `send_resolved`로 해제까지 반영되는지 확인.
- **다중 클러스터**: 각 클러스터 Alertmanager는 동일한 라우팅 규칙/게이트웨이 URL을 사용하도록 표준화(형상 일치).

---

## 7. 요약

- 위험도는 `risk` 라벨(high/medium/low)로 규칙에 고정.
- Alertmanager 단일 관문에서 `risk=high`만 사내 게이트웨이(webhook)로 라우팅 → onTune 전달.
- 게이트웨이로 결합도를 낮춰 다중 클러스터 포맷 통일·유지보수 용이.
- 구체 onTune 엔드포인트/포맷은 사내 담당자 확인 필요(본 문서는 표준 인터페이스 안).

> 근거 링크는 [09-references.md](09-references.md).
