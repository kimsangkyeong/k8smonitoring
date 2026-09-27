# k8smonitoring

OCP 4.20+ / Kubernetes 1.35+ 컨테이너 모니터링 아키텍처 문서 세트입니다.

폐쇄망(disconnected) 환경에서 bastion 서버의 `oc` 기반으로 복수 클러스터(OCP 4.20 · 4.22)를 모니터링하고,
컨테이너 엔진 안정성 / Pod 리소스 적정성 / 클러스터 용량 추이 / 이상징후 / 위험도 기반 알림(onTune 연동)을 다룹니다.

## 문서 위치

문서는 [`docs/`](docs/README.md) 폴더에 요구사항별로 분리되어 있습니다.

- 시작 지점: [docs/README.md](docs/README.md) (문서 인덱스 · 요구사항 커버리지 매트릭스)
- 권고안/개요: [docs/00-overview-recommendation.md](docs/00-overview-recommendation.md)
- 참고자료(근거 링크): [docs/09-references.md](docs/09-references.md)
