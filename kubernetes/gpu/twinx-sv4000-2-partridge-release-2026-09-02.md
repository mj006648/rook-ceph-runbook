# TwinX sv4000-2 Partridge 전용 해제와 공유 스케줄링 전환 2026-09-02

> 작성일: 2026-09-02
>
> 상태: 완료. sv4000-2의 Partridge taint/label과 namespace scheduling annotation을 제거했고, 일반 Pod smoke test가 sv4000-2에서 Ready를 통과했다.
>
> 중요 결정: 노드 reboot는 수행하지 않았다. A100은 full GPU로 전환하지 않고 기존 MIG 1g.5gb 7개 구성을 유지했다.

## Current status

최종 상태:

| 항목 | 결과 |
| --- | --- |
| Kubernetes node | sv4000-2 Ready, schedulable |
| 전용 taint | 제거 |
| 전용 label | 제거 |
| GPU node label | NodeGroup=gpu 유지 |
| 일반 GPU | nvidia.com/gpu=2, NVIDIA L40 2장 |
| A100 | nvidia.com/mig-1g.5gb=7 유지 |
| MIG config | all-1g.5gb / state=success |
| Rook operator | l40s에서 1/1 Running |
| MetalLB | 기존 partridge-pool과 할당 IP 유지, 전용 nodeSelector만 제거 |
| Partridge PVC | 삭제 |
| Partridge PV | Released / Retain |
| NFS backend | 삭제하지 않음 |
| Partridge RBAC | OIDC, ServiceAccount, 장기 token Secret 제거 |
| 일반 Pod smoke test | sv4000-2에서 Running / Ready |
| node reboot | 수행하지 않음 |

운영 저장소:

~~~text
https://github.com/SmartX-Team/TwinX-Ops
~~~

관련 PR:

- 수동 Sync gate: https://github.com/SmartX-Team/TwinX-Ops/pull/237
- mok-infra YAML boundary hotfix: https://github.com/SmartX-Team/TwinX-Ops/pull/238
- sv4000-2 사전 정리: https://github.com/SmartX-Team/TwinX-Ops/pull/239

접속 credential, token 값, kubeconfig, 인증서 본문은 이 문서에 기록하지 않는다.

## Symptom

sv4000-2는 다음 설정으로 Partridge 전용 노드처럼 동작하고 있었다.

~~~text
twinx.dreamai.kr/dedicated-node=partridge
twinx.dreamai.kr/dedicated-node=partridge:NoSchedule
nvidia.com/mig.config=all-1g.5gb
~~~

Partridge namespace Pod에는 Kyverno가 nodeSelector와 toleration을 자동 주입했다. 별도 ResourceQuota는 일반 GPU를 0으로 막고 A100 MIG 1g.5gb를 7개까지 허용했다.

운영 목표는 다음과 같았다.

1. sv4000-2를 Partridge 전용에서 해제한다.
2. 일반 Pod가 sv4000-2에 스케줄될 수 있게 한다.
3. 기존 MetalLB 외부 IP와 Ceph 서비스를 중단하지 않는다.
4. Partridge RBAC, ServiceAccount, token, quota를 제거한다.
5. NFS 실제 데이터는 삭제하지 않는다.
6. 노드 reboot는 하지 않는다.

## Diagnosis

### 노드와 GPU

작업 전 확인:

~~~bash
kubectl get node sv4000-2 -o 'custom-columns=NAME:.metadata.name,READY:.status.conditions[?(@.type=="Ready")].status,UNSCHEDULABLE:.spec.unschedulable,MIG_CONFIG:.metadata.labels.nvidia\.com/mig\.config,MIG_STATE:.metadata.labels.nvidia\.com/mig\.config\.state,GPU:.status.capacity.nvidia\.com/gpu,MIG_1G:.status.capacity.nvidia\.com/mig-1g\.5gb'
~~~

초기 결과:

~~~text
Ready       = True
GPU         = 2
MIG_1G      = 7
MIG_CONFIG  = all-1g.5gb
MIG_STATE   = success
~~~

sv4000-2 하드웨어:

~~~text
GPU 0/1 = NVIDIA L40
GPU 2   = NVIDIA A100-PCIE-40GB
~~~

Kubernetes Pod에는 GPU request가 없었지만 호스트 Docker와 Xorg가 NVIDIA driver를 사용하고 있었다.

~~~text
ttsum-vllm       = L40 사용, 약 43GiB
isaac-sim-SAMG   = 실행 중
lightdm/Xorg     = NVIDIA 공용 device handle 사용
~~~

### MetalLB 의존성

partridge-pool은 이름과 달리 Partridge만 사용하지 않았다.

~~~text
10.38.38.180 = Isaac streaming Service
10.38.38.181 = Ceph RGW trident-store
10.38.38.182 = Ceph RGW trident-kci-store
~~~

따라서 IPAddressPool을 삭제하지 않고 L2Advertisement의 전용 nodeSelector만 제거했다.

### Rook operator 의존성

Rook operator Helm values도 같은 Partridge label에 고정돼 있었다. taint/label을 먼저 제거하면 operator 재생성 시 스케줄링 문제가 생길 수 있으므로 Rook을 먼저 분리했다.

### NFS

Partridge NFS는 외부 정적 NFS PV였다.

~~~text
server = 10.38.36.221
path   = /exports/e481f53d-1e06-44fb-825d-b95cf2f797aa/partridge
policy = Retain
~~~

작업 전 Partridge workload는 없었다. PVC와 Kubernetes 접근 리소스는 제거했지만 외부 NFS server/export/file은 삭제하거나 mount해 읽지 않았다.

## Root cause

### Partridge 전용성이 여러 계층에 흩어져 있었음

전용성은 하나의 manifest가 아니라 다음 계층에 나뉘어 있었다.

- Node taint/label
- Kyverno Pod mutation
- Namespace scheduling annotation
- ResourceQuota와 LimitRange
- OIDC/ServiceAccount RBAC
- MetalLB L2Advertisement nodeSelector
- Rook operator nodeSelector/toleration
- GPU Operator, DRA, OTel toleration

Application 삭제 동작은 child finalizer 상태에 따라 cascade 또는 orphan으로 달라진다. 작업 당시 Partridge child Application에는 finalizer가 없음을 확인했으므로, enabled=false에 의존하지 않고 child app을 수동 Sync로 전환해 실제 desired resource를 단계적으로 정리했다.

### ServerSideApply는 다른 manager의 필드를 지우지 않음

Rook Deployment의 기존 nodeSelector는 kubectl-patch manager가 소유했다.

Git/Helm에서 nodeSelector를 생략한 뒤 Argo Sync는 성공했지만 live Deployment에는 필드가 남았다. Partridge toleration만 제거돼 operator Pod가 Pending이 됐다.

확인:

~~~bash
kubectl get deployment rook-ceph-operator -n rook-ceph --show-managed-fields=true -o yaml
~~~

복구는 전체 Rook Application Replace가 아니라 Deployment 한 개만 선택한 Replace Sync로 수행했다.

### MIG Manager가 GPU 3장을 함께 reset하려 했음

all-disabled 적용 시 MIG Manager는 GPU Operator client와 kubelet을 중지한 뒤 GPU reset을 시도했다. A100만 MIG-capable이지만 reset 단계에서는 L40 두 장도 함께 포함됐다.

로그:

~~~text
GPU 00000000:01:00.0: In use by another client
GPU 00000000:C1:00.0: In use by another client
GPU 00000000:C2:00.0: In use by another client
~~~

L40에서는 VLLM과 Xorg가 실행 중이었다. A100만 지정한 reset도 공용 nvidiactl/uvm handle 때문에 거부됐다.

운영자가 node reboot를 금지했고 VLLM/Xorg 중단도 승인하지 않았으므로 full A100 전환을 중단했다.

## Fix

### 1. Argo child app 수동 Sync gate

다음 앱을 수동 Sync로 전환했다.

~~~text
metallb-resources
gpu-operator
nvidia-dra-driver-gpu
rook-ceph-operator
otelcol
partridge-policies
partridge-infra
~~~

root app을 Sync한 뒤 child app의 automated/prune/selfHeal 필드가 제거됐는지 확인했다.

### 2. GitOps 정리

PR 239에서 다음을 반영했다.

- partridge-advertisement nodeSelector 제거
- Rook operator Partridge nodeSelector/toleration 제거
- Kyverno node 강제 배치와 MetalLB annotation mutation 제거
- PVC, ResourceQuota, LimitRange 제거
- OIDC RBAC, ServiceAccount, token Secret 제거
- Namespace와 Retain PV 유지

수동 Sync 순서:

~~~text
metallb-resources
rook-ceph-operator
partridge-policies
partridge-infra
~~~

### 3. Rook selective Replace 복구

전체 Application이 아닌 Deployment 하나만 Replace했다.

~~~bash
kubectl patch application.argoproj.io rook-ceph-operator -n argocd --type=merge --patch '{
    "operation": {
      "sync": {
        "prune": false,
        "resources": [{
          "group": "apps",
          "kind": "Deployment",
          "namespace": "rook-ceph",
          "name": "rook-ceph-operator"
        }],
        "syncOptions": ["Replace=true"]
      }
    }
  }'
~~~

Force와 Prune은 사용하지 않았다. 결과적으로 operator가 l40s에서 1/1 Running으로 복구됐다.

### 4. MIG 해제 시도와 rollback

안전 장치:

~~~bash
kubectl cordon sv4000-2
kubectl label node sv4000-2 nvidia.com/mig.config=all-disabled --overwrite
~~~

MIG Manager가 failed를 보고해 taint/label 제거와 uncordon을 중단했다.

reboot 없이 원래 구성으로 rollback:

~~~bash
kubectl label node sv4000-2 nvidia.com/mig.config=all-1g.5gb --overwrite
~~~

MIG state는 success로 돌아왔지만 kubelet 재시작 뒤 containerd의 NVIDIA runtime handler가 잠시 복구되지 않아 GPU capacity가 0으로 보였다.

toolkit Pod 하나만 재생성했다.

~~~bash
kubectl get pods -n gpu-operator --field-selector spec.nodeName=sv4000-2 -o name
kubectl delete pod -n gpu-operator TOOLKIT_POD_NAME_FROM_PREVIOUS_COMMAND
~~~

두 번째 명령의 placeholder에는 첫 번째 명령에서 확인한 toolkit Pod 이름 하나만 넣는다. 실제 작업에서도 정확한 Pod 이름을 확인한 뒤 한 개만 삭제했다.

복구 성공 기준:

~~~text
MIG_CONFIG = all-1g.5gb
MIG_STATE  = success
GPU        = 2
MIG_1G     = 7
~~~

### 5. Partridge 전용 metadata 제거

MIG rollback 완료 뒤 전용 metadata만 제거했다.

~~~bash
kubectl taint node sv4000-2 twinx.dreamai.kr/dedicated-node-

kubectl label node sv4000-2 twinx.dreamai.kr/dedicated-node-

kubectl annotate namespace partridge scheduler.alpha.kubernetes.io/defaultTolerations- scheduler.alpha.kubernetes.io/tolerationsWhitelist-
kubectl uncordon sv4000-2
~~~

NodeGroup=gpu와 nvidia.com/mig.config=all-1g.5gb는 유지했다.

## Verification

### Node와 GPU

~~~bash
kubectl get node sv4000-2 -o 'custom-columns=NAME:.metadata.name,READY:.status.conditions[?(@.type=="Ready")].status,UNSCHEDULABLE:.spec.unschedulable,GPU:.status.capacity.nvidia\.com/gpu,MIG_1G:.status.capacity.nvidia\.com/mig-1g\.5gb,MIG_CONFIG:.metadata.labels.nvidia\.com/mig\.config,MIG_STATE:.metadata.labels.nvidia\.com/mig\.config\.state'
~~~

성공 기준:

~~~text
Ready=True
Unschedulable=<none>
GPU=2
MIG_1G=7
MIG_CONFIG=all-1g.5gb
MIG_STATE=success
Partridge taint 없음
Partridge dedicated label 없음
~~~

### 일반 Pod smoke test

전용 toleration 없이 hostname selector만 사용했다.

~~~bash
kubectl run sv4000-2-scheduling-check -n default --image=registry.k8s.io/pause:3.10 --restart=Never --overrides='{"spec":{"nodeSelector":{"kubernetes.io/hostname":"sv4000-2"}}}'

kubectl wait pod/sv4000-2-scheduling-check -n default --for=condition=Ready --timeout=120s

kubectl get pod sv4000-2-scheduling-check -n default -o wide

kubectl delete pod sv4000-2-scheduling-check -n default
~~~

관측 결과:

~~~text
Phase = Running
Ready = true
Node  = sv4000-2
Partridge dedicated toleration 없음
테스트 후 Pod 삭제 확인
~~~

### 주변 서비스

검증 결과:

- GPU Operator component: Running
- Rook operator: l40s에서 Running
- MetalLB Ceph RGW IP 10.38.38.181/182 유지
- ttsum-vllm: 실행 유지
- isaac-sim-SAMG: healthy 유지
- 관련 Argo Application: Synced / Healthy
- 로컬 TwinX-Ops main: clean

## Prevention

### MIG 변경 전 Kubernetes 외 GPU client도 감사

Kubernetes Pod의 resource request만 봐서는 부족하다.

다음 명령은 이번 장애에서 사용한 확인 경로이자 향후 MIG 변경 전 preflight다. 실행에는 대상 노드 SSH와 sudo 권한이 필요하다.

~~~bash
ssh sv4000-2 nvidia-smi
ssh sv4000-2 "sudo fuser -v /dev/nvidia* /dev/nvidia-caps/*"
ssh sv4000-2 "sudo docker ps"
ssh sv4000-2 "systemctl status display-manager --no-pager"
~~~

다음이 있으면 MIG mode 변경 전에 서비스 중단 승인을 받아야 한다.

- Docker/Podman GPU container
- Xorg/LightDM
- host CUDA process
- DCGM 또는 별도 GPU monitoring client
- NVIDIA DRA/plugin 또는 외부 NVML client

### SSA 필드 소유권 확인

Git에서 필드를 지웠는데 live resource에 남으면 managedFields를 먼저 본다.

~~~bash
kubectl get <resource> --show-managed-fields=true -o yaml
~~~

다른 manager가 소유한 필드를 지우기 위해 전체 Application에 Force/Replace를 적용하지 않는다. 필요한 resource 하나만 선택하거나 별도 안전한 migration을 사용한다.

### 실패 시 fail closed

MIG state가 failed이거나 GPU capacity가 기대값과 다르면 다음을 수행하지 않는다.

- Partridge taint/label 제거
- uncordon
- 일반 workload 투입

먼저 기존 MIG 구성과 NVIDIA runtime/device-plugin을 복구한다.

## Remaining risks

- A100은 full GPU가 아니라 MIG 1g.5gb 7개로 유지된다.
- full A100 전환에는 VLLM/Xorg 등 NVIDIA driver client 중단이 필요하다.
- node reboot는 운영 정책상 금지됐다.
- partridge-static-nfs-pv는 Released / Retain이며 이전 PVC claimRef를 기억한다.
- NFS 파일 내용은 이번 작업에서 mount하거나 검증하지 않았다.
- inject-partridge-node-selector라는 기존 ClusterPolicy 이름은 남지만 실제 rule은 NFS compatibility 2개뿐이다.
- GPU Operator, NVIDIA DRA, OTel의 Partridge toleration은 taint 제거 후 무해하지만 후속 GitOps cleanup 대상이다.
- partridge-pool 이름과 IP는 기존 Ceph/Isaac consumer 때문에 유지한다.
- Ceph HEALTH_WARN은 이번 작업 이전부터 존재한 별도 이슈다.

## Related

- [TwinX NVIDIA DRA Driver Rollout](twinx-nvidia-dra-driver-rollout-2026-06-28.md)
- [TwinX KISS GPU Node Onboarding](twinx-kiss-gpu-node-onboarding-2026-08-24.md)
- TwinX-Ops PR 237, 238, 239
