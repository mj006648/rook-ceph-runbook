# NetAI Operations Runbook

[GIST NetAI Lab](https://netai.smartx.kr/)에서 **TwinX**와 **MiniX** 클러스터를 운영하며 쌓은 운영 런북입니다.
네트워킹, 스토리지, 정책, 관측, 애플리케이션, Kubernetes 운영, 장애 대응 기록을 한 곳에 정리합니다.

> 주 대상은 랩 내부 운영자입니다.
> 완성된 매뉴얼이라기보다 운영 중 계속 갱신하는 작업 노트에 가깝습니다.
> 공개 저장소로 두는 이유는 비슷한 문제를 만난 사람이 그대로 참고할 수 있게 하기 위해서입니다.

## Quick map

최근 또는 중요한 운영 이슈를 바로 찾기 위한 표입니다. `Last update`는 해당 문서의 최근 반영일입니다.

| Last update | Area | Issue / note | Contents |
| --- | --- | --- | --- |
| 2026-08-12 | Hardware / Provisioning + Kubernetes / Cluster Lifecycle | [KISS 노드 프로비저닝·클러스터 Join 한눈에 보기](hardware/provisioning/kiss-node-lifecycle-at-a-glance.md) | Box 등록·수정, PXE/Ubuntu 설치, Commissioning, Control Plane·Compute Join, 상태 판정, 주요 장애와 복구 |
| 2026-07-24 | Kubernetes / Cluster Lifecycle | [EdgeX/TwinX KISS 클러스터 구성 및 Compute 재조인 복구](kubernetes/cluster-lifecycle/edgex-twinx-kiss-cluster-join-recovery-2026-07-24.md) | Control Plane bootstrap ConfigMap/RBAC 복구, localhost:6443 proxy seed, kubeadm CA/10250 preflight 충돌 해소, Compute 8대 조인 |
| 2026-07-24 | Kubernetes / Cluster Lifecycle | [DataX Worker 재조인 복구 및 네트워크 안정화](kubernetes/cluster-lifecycle/datax-worker-rejoin-recovery-2026-07-24.md) | 이전 kubelet CA 백업·분리, KISS 재조인, Wi-Fi bond 불안정과 임시 유선 전환, stale CNI 정리, CNI 미배포 시 정상 판정 |
| 2026-07-13 | Hardware / Provisioning | [Supermicro E300 Ubuntu 24.04 PXE/UEFI 복구](hardware/provisioning/supermicro-e300-ubuntu-24-04-pxe-uefi-recovery-2026-07-13.md) | KISS PXE network/HWE package 실패, stale UEFI NVRAM entry와 GRUB 등록 오류, Subiquity 전체 재시작 및 disk boot 검증 |
| 2026-07-01 | Hardware / Power | [SYS-210P DC PSU 장애 및 AC 전환 문의](hardware/power/supermicro-sys-210p-dc-psu-2026-07-01.md) | edgebox3/4 power-off, S-1200-48 external AC-to-48V DC PSU 불안정, Supermicro 공식 AC conversion 확인 항목 |
| 2026-07-01 | Kubernetes / Networking | [TwinX Cilium GitOps 및 Hubble relay 복구](kubernetes/networking/twinx-cilium-hubble-gitops-2026-07-01.md) | Cilium GitOps ownership 전환, Hubble relay/metrics 활성화, certgen Job, relay hostNetwork 복구, edgebox3/4 known exception |
| 2026-06-29 | Kubernetes / Cluster Lifecycle | [TwinX 단일 Control Plane 전환 계획](kubernetes/cluster-lifecycle/twinx-single-control-plane-transition-2026-06-29.md) | control1만 남기고 control2/control3를 제거하기 위한 preflight, etcd snapshot, 단계별 제거, 중단 기준 |
| 2026-06-29 | Kubernetes / Registry | [TwinX Harbor RWO PVC Rollout Deadlock](kubernetes/registry/twinx-harbor-rwo-pvc-rollout-deadlock-2026-06-29.md) | Harbor registry/jobservice가 RWO PVC와 RollingUpdate 조합으로 ContainerCreating에 걸린 원인, PR #209 node pinning fix와 검증 |
| 2026-06-28 | Kubernetes / Security | [TwinX OpenBao Sealed Recovery](kubernetes/security/twinx-openbao-sealed-recovery-2026-06-28.md) | OpenBao sealed 상태로 ESO와 Trident ExternalSecret이 Degraded 된 장애 복구, unseal 및 검증 절차 |
| 2026-06-28 | Kubernetes / GPU | [TwinX NVIDIA DRA Driver Rollout](kubernetes/gpu/twinx-nvidia-dra-driver-rollout-2026-06-28.md) | GPU Operator v25.3.4 환경에서 NVIDIA DRA driver chart 25.3.2를 GitOps로 추가하고 DeviceClass/ResourceSlice 검증 |
| 2026-06-28 | Kubernetes / Virtualization | [TwinX KubeVirt User VM Plan](kubernetes/virtualization/twinx-kubevirt-user-vm-plan-2026-06-28.md) | 사용자별 KubeVirt VM 제공 설계, v1.8.4 선택 이유, GitOps 준비 결과, 배포 전 preflight와 중단 기준 |
| 2026-06-26 | Kubernetes / Multicluster | [Karmada ScaleX-POD Lab](kubernetes/multicluster/karmada/) | MiniX kind 기반 Karmada, ArgoCD, Kueue, Resource Pool, WorkloadRebalancer 실험 |
| 2026-06-26 | Kubernetes / Cluster Lifecycle | [TwinX Kubernetes Upgrade Run 2026-06-26](kubernetes/cluster-lifecycle/twinx-kubernetes-upgrade-run-2026-06-26.md) | 당일 preflight 결과, OTP 임시 해제, Harbor/Ceph/Partridge/Kubespray blocker, 축소된 1차 wave |
| 2026-06-25 | Kubernetes / Incidents | [MiniX Kubespray Upgrade Troubleshooting](kubernetes/incidents/minix-kubespray-upgrade-2026-06-25.md) | Cilium 권한, kubeadm health-check timeout, Rook-Ceph PDB drain block, CoreDNS loop, GPU node API timeout |
| 2026-06-25 | Kubernetes / Storage | [Rook-Ceph Reinstall](kubernetes/storage/rook-ceph-reinstall.md) | Rook-Ceph 전체 재설치와 단계별 bring-up 절차 |
| 2026-06-25 | Kubernetes / Security | [Kyverno + cert-manager](kubernetes/security/kyverno-cert-manager.md) | Kyverno chart v3.7.x에서 인증서 ping-pong이 반복되는 문제 |

## Sections

- **[kubernetes/](kubernetes/)** — Kubernetes 기반 운영 런북의 중심 디렉터리
  - **[kubernetes/cluster-lifecycle/](kubernetes/cluster-lifecycle/)** — Kubernetes upgrade, Kubespray 작업, control-plane/etcd topology 변경, node 제거
  - **[kubernetes/networking/](kubernetes/networking/)** — Cilium, Hubble, MTU, Pod 통신, Kubernetes 관련 node routing
  - **[kubernetes/storage/](kubernetes/storage/)** — Rook-Ceph, OSD, PVC, LV, local disk 운영
  - **[kubernetes/security/](kubernetes/security/)** — OpenBao, External Secrets, Kyverno, cert-manager, OIDC/RBAC, webhook lifecycle
  - **[kubernetes/gpu/](kubernetes/gpu/)** — GPU Operator, NVIDIA DRA, MIG, GPU node 운영
  - **[kubernetes/registry/](kubernetes/registry/)** — Harbor registry, image storage, RWO PVC rollout 운영
  - **[kubernetes/virtualization/](kubernetes/virtualization/)** — KubeVirt 기반 사용자별 VM 제공 계획과 운영 절차
  - **[kubernetes/observability/](kubernetes/observability/)** — Prometheus, Grafana, OpenTelemetry, Hubble metrics
  - **[kubernetes/apps/](kubernetes/apps/)** — Kubernetes 위에서 운영되는 포털, 카탈로그, 쿼리 엔진 등 앱 운영
  - **[kubernetes/multicluster/](kubernetes/multicluster/)** — Karmada, ScaleX-POD, multi-cluster placement 실험과 운영
  - **[kubernetes/incidents/](kubernetes/incidents/)** — Kubernetes 장애 기록, postmortem, cross-component failure
- **[hardware/](hardware/)** — PSU, BMC/IPMI, NIC cabling, rack 등 Kubernetes 바깥 물리 인프라 이슈
- **[templates/](templates/)** — 새 runbook/incident 작성 템플릿
- **[archive/](archive/)** — active 구조에서 빠진 오래된 자료

## Note format

각 노트는 아래 형식을 기준으로 작성합니다.

- **Symptom** — 실제로 관측되는 현상
- **Diagnosis** — 확인에 사용한 정확한 명령
- **Root cause** — 왜 발생했는지
- **Fix** — 복구 절차
- **Prevention** — 다음에 같은 문제를 피하는 방법

## Discussions

정식 문서로 정리하기 전의 질문이나 메모는 [Discussions](https://github.com/mj006648/netai-devsecops-runbook/discussions)에 남깁니다.

## License

This repository is licensed under the [MIT License](LICENSE).
