# Kubernetes Cluster Lifecycle Runbooks

Kubernetes 클러스터의 생애주기 작업을 모아둔다.  
여기에는 Kubespray 기반 Kubernetes version upgrade, control-plane/etcd topology 변경, node 제거, maintenance window 계획이 포함된다.

`kubespray`라는 이름을 쓰지 않는 이유는 이 디렉터리가 Kubespray만을 위한 공간이 아니기 때문이다. 예를 들어 TwinX 단일 control-plane 전환은 worker/GPU 노드의 직접 Docker container 영향을 피하기 위해 Kubespray 전체 playbook을 기본 경로로 쓰지 않는다.

## Quick map

| Last update | Target | Document | Contents |
| --- | --- | --- | --- |
| 2026-08-12 | EdgeX/TwinX/DataX Compute 10대 및 기존 E300 Control Plane 2대 | [KISS EdgeX/TwinX/DataX 노드 default 클러스터 복귀](kiss-default-cluster-return-2026-08-12.md) | source 인증정보를 root-only backup으로 격리하고 default kubelet/CA/CNI 복원, E300 etcd snapshot과 control-plane 비활성화, localhost/nginx proxy 인계 복구, 12대 Node Ready 검증 |
| 2026-07-24 | EdgeX/TwinX Control Plane 및 Compute 8대 | [EdgeX/TwinX KISS 클러스터 구성 및 Compute 재조인 복구](edgex-twinx-kiss-cluster-join-recovery-2026-07-24.md) | bootstrap ConfigMap/RBAC 복구, localhost:6443 proxy seed, CA/10250 preflight 충돌 해소, CNI 미배포 상태의 Node 등록 검증 |
| 2026-07-24 | DataX worker | [DataX Worker 재조인 복구 및 네트워크 안정화](datax-worker-rejoin-recovery-2026-07-24.md) | 이전 kubelet CA 백업·분리, KISS 재조인, Wi-Fi bond 경로 장애와 임시 유선 전환, stale Cilium/Multus 설정 정리, CNI 미배포 시 검증·rollback |
| 2026-06-29 | TwinX control-plane | [TwinX 단일 Control Plane 전환 계획 및 실행 로그](twinx-single-control-plane-transition-2026-06-29.md) | control1 단일 control-plane 전환 완료, control2/control3 제거 결과, GPU Operator/NFD/DRA controller sv4000-1 재배치, Kubespray inventory 정리, 남은 known issue |
| 2026-06-27 | TwinX run log | [TwinX Kubernetes Upgrade Run 2026-06-26](twinx-kubernetes-upgrade-run-2026-06-26.md) | v1.35.4 업그레이드 완료 노드, edgebox1/2 완료, edgebox3/4 보류, Kubespray no-drain 패치, Ceph/Harbor/OTP 남은 작업 |
| 2026-06-25 | TwinX `v1.33.3 -> v1.35.4` | [TwinX Kubernetes 1.35 Upgrade Plan](twinx-kubernetes-1-35-upgrade-plan.md) | Kubespray 경로, Ceph/Harbor/Partridge/l40s blocker, Hubble/DRA 순서, 중단 기준 |
