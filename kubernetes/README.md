# Kubernetes Operations Runbook

NetAI TwinX/MiniX 클러스터의 Kubernetes 기반 운영 기록을 모아둔 중심 디렉터리입니다.
대부분의 런북은 Kubernetes 구성 요소, Kubernetes 위에서 동작하는 플랫폼 컴포넌트, 또는 Kubernetes 작업 중 발생한 장애를 기준으로 분류합니다.

## Quick map

| Last update | Area | Topic | Document |
| --- | --- | --- | --- |
| 2026-08-17 | Security | SPIFFE/SPIRE workload identity와 OpenBao/ESO secret 관리의 개념, 실습, 통합 설계, 운영 진단 | [Workload Identity와 Secret 관리 학습 경로](security/identity-secrets/) |
| 2026-07-24 | Cluster lifecycle | EdgeX/TwinX Control Plane bootstrap 복구, localhost API proxy seed, kubeadm preflight 충돌 제거, Compute 8대 KISS Join | [EdgeX/TwinX KISS Cluster Join Recovery](cluster-lifecycle/edgex-twinx-kiss-cluster-join-recovery-2026-07-24.md) |
| 2026-07-24 | Cluster lifecycle | DataX worker 이전 kubelet CA 제거, KISS 재조인, Wi-Fi bond 불안정과 유선 전환, stale CNI 정리 | [DataX Worker Rejoin Recovery](cluster-lifecycle/datax-worker-rejoin-recovery-2026-07-24.md) |
| 2026-07-01 | Networking | Cilium GitOps ownership 전환, Hubble relay/metrics 활성화, hostNetwork 복구 | [TwinX Cilium Hubble GitOps](networking/twinx-cilium-hubble-gitops-2026-07-01.md) |
| 2026-06-29 | Cluster lifecycle | TwinX control1 단일 control-plane 전환 완료, control2/control3 제거, Kubespray inventory 정리 | [TwinX Single Control Plane Transition](cluster-lifecycle/twinx-single-control-plane-transition-2026-06-29.md) |
| 2026-06-29 | Registry | Harbor registry/jobservice RWO PVC rollout deadlock, rm352-1 node pinning fix | [TwinX Harbor RWO PVC Rollout Deadlock](registry/twinx-harbor-rwo-pvc-rollout-deadlock-2026-06-29.md) |
| 2026-06-28 | Security | OpenBao sealed 상태로 ESO/ExternalSecret이 Degraded 된 장애 복구 | [TwinX OpenBao Sealed Recovery](security/twinx-openbao-sealed-recovery-2026-06-28.md) |
| 2026-06-28 | GPU | GPU Operator v25.3.4, NVIDIA DRA driver 25.3.2, GPU/MIG/ComputeDomain ResourceSlice 검증 | [TwinX NVIDIA DRA Driver Rollout](gpu/twinx-nvidia-dra-driver-rollout-2026-06-28.md) |
| 2026-06-28 | Virtualization | 사용자별 KubeVirt VM 제공, v1.8.4 GitOps 준비, preflight, smoke test, rollback 기준 | [TwinX KubeVirt User VM Plan](virtualization/twinx-kubevirt-user-vm-plan-2026-06-28.md) |
| 2026-06-26 | Multicluster | ScaleX-POD Karmada/ArgoCD/Kueue/Resource Pool 실험 | [Karmada ScaleX-POD Lab](multicluster/karmada/) |
| 2026-06-27 | Cluster lifecycle | TwinX v1.35.4 업그레이드 완료 노드, edgebox3/4 보류, Kubespray no-drain 패치 | [TwinX Kubernetes Upgrade Run](cluster-lifecycle/twinx-kubernetes-upgrade-run-2026-06-26.md) |
| 2026-06-25 | Incidents | MiniX Kubespray upgrade 중 Cilium/kubeadm/Rook/CoreDNS/GPU API 문제 | [MiniX Kubespray Upgrade Troubleshooting](incidents/minix-kubespray-upgrade-2026-06-25.md) |

## Directories

- **[cluster-lifecycle/](cluster-lifecycle/)** — Kubernetes version upgrade, control-plane/etcd topology 변경, node 제거, maintenance window 계획
- **[networking/](networking/)** — Cilium, Hubble, MTU, Pod communication, Kubernetes 관련 node routing
- **[storage/](storage/)** — Rook-Ceph, OSD, PVC, LV, local disk 운영
- **[security/](security/)** — SPIFFE/SPIRE workload identity, OpenBao, External Secrets, Kyverno, cert-manager, OIDC/RBAC, webhook lifecycle
- **[gpu/](gpu/)** — GPU Operator, NVIDIA DRA, MIG, GPU node 운영
- **[registry/](registry/)** — Harbor registry, image storage, RWO PVC rollout 운영
- **[virtualization/](virtualization/)** — KubeVirt 기반 사용자별 VM 계획과 운영 runbook
- **[observability/](observability/)** — Prometheus, Grafana, OpenTelemetry, Hubble metrics, alerting
- **[apps/](apps/)** — Kubernetes 위에서 운영되는 앱과 플랫폼 서비스 runbook
- **[multicluster/](multicluster/)** — Karmada, ScaleX-POD, multi-cluster placement, Kueue 연동 실험/운영
- **[incidents/](incidents/)** — Kubernetes 장애 기록과 postmortem
