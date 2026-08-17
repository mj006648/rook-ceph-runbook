# Kubernetes Security

SPIFFE/SPIRE workload identity, OpenBao, External Secrets Operator, Kyverno, cert-manager, RBAC/OIDC, and webhook lifecycle runbooks.

## Quick map

| Last update | Topic | Document | Contents |
| --- | --- | --- | --- |
| 2026-08-17 | SPIFFE/SPIRE / OpenBao / ESO | [Workload Identity와 Secret 관리 학습 경로](identity-secrets/) | 개념, 위협 모델, Kubernetes 실습, 통합 설계, 운영·장애 대응, 복습 문제를 포함한 순서형 학습 자료 |
| 2026-06-28 | OpenBao / ESO | [TwinX OpenBao Sealed Recovery](twinx-openbao-sealed-recovery-2026-06-28.md) | OpenBao sealed 상태로 ESO/ExternalSecret이 Degraded 된 장애 복구 |
| 2026-06-25 | Kyverno / cert-manager | [Kyverno + cert-manager](kyverno-cert-manager.md) | Kyverno chart v3.7.x 인증서 ping-pong 문제 |
| 2026-06-25 | Webhook TLS | [Webhook Cert SIGTERM](webhook-cert-sigterm.md) | webhook controller가 인증서 owner 충돌로 주기적 SIGTERM 재시작되는 문제 |


## Study guides

- **[identity-secrets/](identity-secrets/)** — SPIFFE/SPIRE와 OpenBao/External Secrets Operator를 처음부터 운영 수준까지 학습하는 한국어 교재·실습서
- **[TwinX OpenBao Sealed 복구 기록](twinx-openbao-sealed-recovery-2026-06-28.md)** — 학습 내용을 실제 TwinX 장애에 연결하는 사례
