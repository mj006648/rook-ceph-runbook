# TwinX KISS GPU 노드 편입과 GPU Operator/DRA 복구 2026-08-24

> 작성일: 2026-08-24
>
> 상태: TwinX RTX 3070 Compute 3대의 GPU Operator와 NVIDIA DRA 검증 완료. VINE Greeter 비활성화, 재부팅 검증, 일반 GPU와 DRA CUDA smoke test 통과
>
> 선행 기록: [DataX Gen12 Compute의 default 경유 TwinX 재조인](../cluster-lifecycle/datax-default-twinx-worker-migration-2026-08-24.md)

## 결과 요약

이번 작업은 `SmartX-Team/twinx-k8s` preset으로 관리되는 KISS TwinX 클러스터에 GPU Operator와 NVIDIA DRA를 적용하고, GPU Compute 3대를 Kubernetes GPU workload용으로 안정화한 기록이다.

최종 상태:

| 항목 | 결과 |
| --- | --- |
| Kubernetes | `v1.34.3` |
| GPU Operator | `v25.10.1`, Argo CD `Synced / Healthy` |
| NVIDIA driver | `580.82.07` |
| NVIDIA DRA driver | `25.8.1`, Argo CD `Synced / Healthy` |
| GPU | RTX 3070 3대, 노드별 `nvidia.com/gpu=1` |
| DRA | 노드별 `gpu.nvidia.com` ResourceSlice와 `gpu-0` 확인 |
| 일반 GPU smoke test | `nvidia.com/gpu: 1`로 CUDA `vectorAdd` 통과 |
| DRA smoke test | `ResourceClaim`만 사용해 CUDA `vectorAdd` 통과 |
| VINE Greeter | GPU 노드 3대에서 비활성화 |
| Cilium | 전체 노드 Ready |

이번 문서의 TwinX는 다음 경로를 뜻한다.

```text
SmartX-Team/twinx-k8s
cluster.name = twinx
Kubernetes context = kubernetes-admin@ops.twinx.openark
Argo CD control cluster = mobilex-argocd
GPU namespace = gpu-nvidia
```

기존 `TwinX-Ops`, `twinx.dreamai.kr`, Kubernetes 1.35 기반 과거 TwinX 문서와는 다른 클러스터다. 과거 DRA values와 driver root를 이번 환경에 그대로 복사하지 않는다.

접속 endpoint, 비밀번호, private key, kubeconfig·인증서 본문은 이 문서나 Git에 기록하지 않는다. 관리 호스트 작업은 지정된 tmux 세션 `chang`에서만 수행했다.

## 대상 노드

| 역할 | Alias | UUID / Kubernetes Node | Internal IP | GPU |
| --- | --- | --- | --- | --- |
| Control Plane | `twinx-control-plane-e300-01` | `00000000-0000-0000-0000-ac1f6be5de08` | `10.32.47.196` | 없음 |
| Compute | `twinx-desktop-gen12-02` | `9bc3697d-072e-dc48-29fb-1c697ad99e51` | `10.38.35.104` | RTX 3070 |
| Compute | `twinx-desktop-gen12-03` | `9bcc45d1-c926-ab80-b3c0-1c697ad99d94` | `10.38.35.105` | RTX 3070 |
| Compute | `twinx-desktop-gen12-05` | `3adacb41-faad-9484-7477-1c697ad8c17d` | `10.38.35.110` | RTX 3070 |

`gen12-05`는 같은 날 `datax -> default -> twinx` 순서로 이동한 노드다. kubelet/CA/CNI 전환 과정은 선행 cluster-lifecycle 문서에 분리해 기록했다.

## GitOps 적용 기록

### GPU Operator

`twinx-k8s` PR:

- <https://github.com/SmartX-Team/twinx-k8s/pull/1>
- feature commit: `c150fef`
- merge commit: `39aca02`

### NVIDIA DRA

`twinx-k8s` PR:

- <https://github.com/SmartX-Team/twinx-k8s/pull/2>
- feature commit: `0702db9`
- merge commit: `d44f400`

최종 `twinx-k8s/values.yaml`의 관련 부분:

```yaml
driver:
  nvidia:
    gpu:
      eula: true

features:
  - nvidia.com/gpu
  - nvidia.com/gpu/dynamic-resource-allocation
  - org.ulagbulag.io/cni
```

이 변경은 `twinx-k8s`만 수정했다. `smartx-k8s`는 기존 공용 Application 정의를 제공했으며 이번 작업에서 수정하지 않았다.

렌더링된 주요 Application:

```text
twinx-nvidia-gpu-operator
twinx-nvidia-gpu-dra
```

DRA 공용 values에서 중요한 값:

```yaml
nvidiaDriverRoot: /run/nvidia/driver
gpuResourcesEnabledOverride: true
```

GPU driver가 GPU Operator driver container로 관리되므로 host root `/`가 아니라 `/run/nvidia/driver`를 사용한다.

## 실제 장애와 복구 결과

### gen12-02

초기 상태:

```text
GPU PCI driver       = vfio-pci
driver_override      = vfio-pci
VINE Greeter         = enabled
Kubernetes GPU       = 미노출
```

GPU Operator가 GPU를 NVIDIA driver로 전환한 뒤에도 Greeter가 다음 부팅에서 다시 PCI binding을 바꿀 수 있었다. 사용자 workload가 없음을 확인하고 cordon/drain, Greeter 비활성화, reboot 순서로 처리했다.

최종 상태:

```text
GPU PCI driver       = nvidia
driver_override      = (null)
VINE Greeter         = disabled / inactive
nvidia.com/gpu       = 1
CUDA                 = PASS
```

### gen12-03

초기에는 NVIDIA open kernel module `580.82.07`이 이미 로드돼 있었다. GPU Operator driver 초기화 과정에서 자동 drain이 DaemonSet Pod 삭제를 시도하며 실패했다.

관측 오류:

```text
failed to drain node: cannot delete DaemonSet-managed Pods
```

`--force`를 쓰지 않고 다음 순서로 복구했다.

1. 사용자 workload와 VolumeAttachment가 없음을 확인
2. node cordon
3. `--ignore-daemonsets --delete-emptydir-data`로 drain
4. VINE Greeter 비활성화
5. reboot
6. 새 Boot ID와 Node Ready 확인
7. GPU Operator와 CUDA Validator 확인

### gen12-05

노드 Join 직후 GPU Operator 구성요소는 모두 생성됐고 일반 `nvidia.com/gpu` CUDA Validator도 통과했다. DRA Kubelet Plugin과 ResourceSlice도 겉으로는 정상처럼 보였다.

그러나 DRA `ResourceClaim` Pod에서는 다음 현상이 발생했다.

```text
nvidia-smi                                      = 성공
DRA claim -> gpu-0                              = 성공
CUDA vectorAdd                                  = 실패
Failed to allocate device vector A (unknown)   = 재현
일반 nvidia.com/gpu Pod                         = 성공
```

즉 GPU hardware, NVIDIA driver, legacy device plugin 문제가 아니라 DRA CDI device injection 문제였다.

DRA container 내부 장치:

```text
/dev/nvidia0          존재
/dev/nvidiactl        존재
/dev/nvidia-modeset   존재
/dev/nvidia-uvm       누락
/dev/nvidia-uvm-tools 누락
```

기존 정상 노드의 CDI에는 UVM 장치 두 개가 모두 있었다. 새 노드는 DRA Plugin이 NVIDIA driver 준비와 겹쳐 불완전한 CDI 명세를 생성했다.

직접 원인:

```text
DRA Kubelet Plugin 시작
  -> /run/nvidia/driver의 UVM device node 준비 전 CDI 생성
  -> Plugin Pod와 ResourceSlice는 Running
  -> NVML 기반 nvidia-smi는 성공
  -> CUDA memory allocation은 실패
```

드라이버 준비 후 새 노드의 DRA Plugin Pod 한 개만 재생성하자 CDI에 UVM 장치가 추가됐고 DRA CUDA가 통과했다.

Greeter도 별도 위험이었다.

```text
openark-vine-greeter.service = enabled / failed
GPU driver                   = nvidia
GPU driver_override          = (null)
/etc/modules-load.d/vfio-pci.conf 존재
```

`failed` 상태라고 해서 아무 동작도 하지 않은 것은 아니었다. 같은 boot의 journal에는 Greeter가 NVIDIA audio와 USB PCI 장치를 unbind하고 `vfio-pci`로 바인딩하려 한 기록이 있었다. Kubernetes container GPU 노드에서는 Greeter를 비활성화해야 한다.

Greeter 비활성화 후 통제된 reboot를 수행했다. 재부팅 직후 DRA CDI에는 `/dev/nvidia-uvm`은 있었지만 `/dev/nvidia-uvm-tools`가 빠지는 race가 한 번 더 관측됐다. 드라이버가 완전히 준비된 뒤 DRA Plugin을 재생성해 정상 노드와 같은 CDI 명세로 복구하고 DRA smoke test를 다시 통과시켰다.

## 표준 GPU 노드 편입 절차

이 절차는 GPU Operator와 DRA Application이 이미 배포된 TwinX에 새 NVIDIA GPU 노드를 추가할 때 사용한다.

### 1. 변수와 대상 확인

```bash
NODE='<kubernetes-node-uuid>'
ALIAS='<box-alias>'
GPU_NS='gpu-nvidia'

kubectl get node "$NODE" -o wide
kubectl get node "$NODE" \
  -o custom-columns='NAME:.metadata.name,READY:.status.conditions[?(@.type=="Ready")].status,UNSCHEDULABLE:.spec.unschedulable,BOOT_ID:.status.nodeInfo.bootID,GPU:.status.allocatable.nvidia\.com/gpu'
```

worker SSH 뒤에는 반드시 hostname guard를 사용한다.

```bash
test "$(hostname)" = "$NODE"
```

### 2. workload와 DRA claim 사전 감사

```bash
kubectl get pods -A \
  --field-selector "spec.nodeName=$NODE" \
  -o custom-columns='NAMESPACE:.metadata.namespace,NAME:.metadata.name,STATUS:.status.phase,OWNER:.metadata.ownerReferences[0].kind,GPU:.spec.containers[*].resources.limits.nvidia\.com/gpu,CLAIM:.spec.resourceClaims[*].resourceClaimName'

kubectl get volumeattachments.storage.k8s.io \
  -o custom-columns='NAME:.metadata.name,NODE:.spec.nodeName,ATTACHED:.status.attached,PV:.spec.source.persistentVolumeName' |
  grep -E "NAME|$NODE"
```

다음 중 하나라도 있으면 reboot나 DRA Plugin 재시작을 중단한다.

- 소유자를 모르는 일반 Pod
- 실행 중인 GPU job
- 대상 노드에서 사용하는 `ResourceClaim`
- attached VolumeAttachment
- 중단 영향이 확인되지 않은 stateful workload

### 3. host GPU와 VINE 상태 확인

```bash
systemctl is-enabled openark-vine-greeter.service 2>&1 || true
systemctl is-active openark-vine-greeter.service 2>&1 || true
systemctl status openark-vine-greeter.service --no-pager -l

lspci -nnk -s 01:00.0
readlink /sys/bus/pci/devices/0000:01:00.0/driver
cat /sys/bus/pci/devices/0000:01:00.0/driver_override

lsmod | grep -E '^(nvidia|vfio)' || true
```

판단 기준:

| 상태 | 조치 |
| --- | --- |
| `driver=nvidia`, override `(null)`, Greeter disabled | GPU binding 변경 불필요 |
| `driver=nvidia`, Greeter enabled/failed | Greeter 비활성화 필요 |
| `driver=vfio-pci` 또는 override `vfio-pci` | workload 감사 후 통제된 GPU 복구와 reboot 필요 |
| `vfio_pci` module만 loaded | module 존재만으로 장애로 판단하지 않음. 실제 PCI binding과 override 확인 |

### 4. cordon과 drain

먼저 server dry-run을 수행한다.

```bash
kubectl cordon "$NODE"

kubectl drain "$NODE" \
  --ignore-daemonsets \
  --delete-emptydir-data \
  --timeout=10m \
  --dry-run=server
```

영향이 예상 범위인지 확인한 뒤 실제 drain한다.

```bash
kubectl drain "$NODE" \
  --ignore-daemonsets \
  --delete-emptydir-data \
  --timeout=10m
```

`--force`로 미확인 Pod를 지우지 않는다. DaemonSet Pod를 직접 대량 삭제하지 않는다.

### 5. VINE Greeter 비활성화

대상 worker에서 실행한다.

```bash
test "$(hostname)" = "$NODE"

sudo systemctl disable --now openark-vine-greeter.service
sudo systemctl reset-failed openark-vine-greeter.service || true

systemctl is-enabled openark-vine-greeter.service 2>&1 || true
systemctl is-active openark-vine-greeter.service 2>&1 || true
systemctl show openark-vine-greeter.service \
  -p UnitFileState -p ActiveState -p SubState
```

성공 기준:

```text
UnitFileState=disabled
ActiveState=inactive
SubState=dead
```

### 6. reboot와 복귀 확인

reboot 전 Boot ID를 기록한다.

```bash
kubectl get node "$NODE" -o jsonpath='{.status.nodeInfo.bootID}{"\n"}'
```

worker에서 hostname guard 뒤 reboot한다.

```bash
test "$(hostname)" = "$NODE" && sudo systemctl reboot
```

reboot 뒤 다음을 확인한다.

```bash
kubectl get node "$NODE" \
  -o custom-columns='NAME:.metadata.name,READY:.status.conditions[?(@.type=="Ready")].status,UNSCHEDULABLE:.spec.unschedulable,BOOT_ID:.status.nodeInfo.bootID,GPU:.status.allocatable.nvidia\.com/gpu'

kubectl -n kube-system get pods \
  --field-selector "spec.nodeName=$NODE" -o wide |
  grep -E 'NAME|cilium'

kubectl -n "$GPU_NS" get pods \
  --field-selector "spec.nodeName=$NODE" -o wide

kubectl get clusterpolicy
```

Join 또는 reboot 직후에는 잠시 다음 오류가 나타날 수 있다.

```text
no runtime for "nvidia" is configured
```

NVIDIA Container Toolkit이 containerd runtime을 구성하는 초기 수렴 구간이다. Toolkit Pod가 Ready가 된 뒤에도 계속 반복되는지 확인한다. 일정 시간 뒤 GPU allocatable `1`, ClusterPolicy `ready`, GPU Pod `Running` 또는 Validator `Succeeded`가 되어야 한다.

### 7. DRA CDI UVM 확인

```bash
DRA_POD=$(kubectl -n "$GPU_NS" get pods \
  --field-selector "spec.nodeName=$NODE" \
  -l nvidia-dra-driver-gpu-component=kubelet-plugin \
  -o jsonpath='{.items[0].metadata.name}')

kubectl -n "$GPU_NS" exec "$DRA_POD" -c gpus -- \
  grep -n -E 'nvidia-uvm|nvidia-modeset|nvidiactl' \
  /var/run/cdi/k8s.gpu.nvidia.com-device_base.yaml
```

필수 항목:

```text
/dev/nvidia-modeset
/dev/nvidia-uvm
/dev/nvidia-uvm-tools
/dev/nvidiactl
```

같은 driver version과 GPU Operator 구성을 사용하는 정상 노드와 CDI hash를 비교할 수 있다.

```bash
kubectl -n "$GPU_NS" exec "$DRA_POD" -c gpus -- \
  sha256sum /var/run/cdi/k8s.gpu.nvidia.com-device_base.yaml
```

hash 자체를 전역 고정값으로 사용하지 않는다. driver와 library 구성이 바뀌면 CDI 내용도 달라질 수 있다.

### 8. UVM 누락 시 DRA Plugin만 재생성

먼저 대상 노드에 active DRA consumer가 없는지 다시 확인한다.

```bash
kubectl get pods -A \
  --field-selector "spec.nodeName=$NODE" \
  -o jsonpath='{range .items[*]}{.metadata.namespace}{"/"}{.metadata.name}{" claims="}{range .spec.resourceClaims[*]}{.resourceClaimName}{end}{"\n"}{end}' |
  grep 'claims=.' || true
```

출력이 없고 NVIDIA driver와 UVM device가 준비된 상태에서만 대상 노드의 DRA Plugin 한 개를 재생성한다.

```bash
OLD_DRA_POD="$DRA_POD"

kubectl -n "$GPU_NS" delete pod "$OLD_DRA_POD" --wait=true

kubectl -n "$GPU_NS" get pods \
  --field-selector "spec.nodeName=$NODE" \
  -l nvidia-dra-driver-gpu-component=kubelet-plugin \
  -o wide
```

DaemonSet이 새 Pod를 만든다. 새 Pod가 `2/2 Running`이 된 뒤 CDI UVM 항목과 ResourceSlice를 다시 확인한다.

```bash
kubectl get resourceslices \
  -o custom-columns='NAME:.metadata.name,NODE:.spec.nodeName,DRIVER:.spec.driver,DEVICES:.spec.devices[*].name' |
  grep -E "NAME|$NODE"
```

GPU driver Pod, GPU Operator 전체 DaemonSet, 다른 GPU 노드의 DRA Plugin은 재시작하지 않는다.

## Smoke test

노드가 아직 cordon 상태라면 테스트 Pod에 `node.kubernetes.io/unschedulable` toleration을 넣는다. 단일 GPU 노드에서 legacy와 DRA 테스트를 동시에 실행하지 않는다.

현재 환경에서 사용한 테스트 image:

```text
nvcr.io/nvidia/gpu-operator:v25.10.1
```

GPU Operator 버전이 바뀌면 현재 CUDA Validator image와 맞춘다.

테스트 Namespace를 별도로 만들고 완료 후 전체 삭제한다.

```bash
TEST_NS="twinx-gpu-smoke-$(date +%Y%m%d-%H%M%S)"
kubectl create namespace "$TEST_NS"
```

### 일반 `nvidia.com/gpu` 테스트

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: gpu-legacy-smoke
  namespace: <test-namespace>
spec:
  restartPolicy: Never
  runtimeClassName: nvidia
  tolerations:
    - key: node.kubernetes.io/unschedulable
      operator: Exists
      effect: NoSchedule
  nodeSelector:
    kubernetes.io/hostname: <node-uuid>
  containers:
    - name: cuda-test
      image: nvcr.io/nvidia/gpu-operator:v25.10.1
      command: ["sh", "-c"]
      args:
        - nvidia-smi --query-gpu=name,driver_version --format=csv,noheader && vectorAdd
      resources:
        limits:
          nvidia.com/gpu: 1
```

성공 로그:

```text
NVIDIA GeForce RTX 3070, 580.82.07
Test PASSED
Done
```

### DRA `ResourceClaim` 테스트

```yaml
apiVersion: resource.k8s.io/v1
kind: ResourceClaim
metadata:
  name: gpu-claim
  namespace: <test-namespace>
spec:
  devices:
    requests:
      - name: gpu
        exactly:
          deviceClassName: gpu.nvidia.com
---
apiVersion: v1
kind: Pod
metadata:
  name: gpu-dra-smoke
  namespace: <test-namespace>
spec:
  restartPolicy: Never
  runtimeClassName: nvidia
  tolerations:
    - key: node.kubernetes.io/unschedulable
      operator: Exists
      effect: NoSchedule
  nodeSelector:
    kubernetes.io/hostname: <node-uuid>
  resourceClaims:
    - name: gpu
      resourceClaimName: gpu-claim
  containers:
    - name: cuda-test
      image: nvcr.io/nvidia/gpu-operator:v25.10.1
      command: ["sh", "-c"]
      args:
        - nvidia-smi --query-gpu=name,driver_version --format=csv,noheader && vectorAdd
      resources:
        claims:
          - name: gpu
```

DRA 테스트 Pod에는 `nvidia.com/gpu` limit이 없어야 한다.

```bash
kubectl -n <test-namespace> get pod gpu-dra-smoke \
  -o jsonpath='claim={.spec.resourceClaims[0].resourceClaimName}{" legacyGpuLimit="}{.spec.containers[0].resources.limits.nvidia\.com/gpu}{"\n"}'

kubectl -n <test-namespace> logs gpu-dra-smoke
```

DRA Plugin 로그에서 Claim UID와 `gpu-0` 준비를 확인할 수 있다.

```bash
CLAIM_UID=$(kubectl -n <test-namespace> get resourceclaim gpu-claim \
  -o jsonpath='{.metadata.uid}')

kubectl -n gpu-nvidia logs \
  -l nvidia-dra-driver-gpu-component=kubelet-plugin \
  -c gpus --since=10m --prefix |
  grep "$CLAIM_UID"
```

### 정리와 uncordon

```bash
kubectl delete namespace <test-namespace> --wait=true
kubectl uncordon "$NODE"
```

GPU Operator driver manager가 먼저 uncordon했다면 `already uncordoned`가 출력될 수 있다. 최종 상태가 schedulable인지 확인한다.

## 최종 검증 명령

```bash
kubectl get nodes \
  -o custom-columns='NAME:.metadata.name,READY:.status.conditions[?(@.type=="Ready")].status,UNSCHEDULABLE:.spec.unschedulable,GPU:.status.allocatable.nvidia\.com/gpu'

kubectl -n gpu-nvidia get daemonset \
  nvidia-device-plugin-daemonset \
  nvidia-dra-driver-gpu-kubelet-plugin

kubectl get clusterpolicy
kubectl get deviceclasses
kubectl get resourceslices -o wide
```

Argo 관리 클러스터:

```bash
kubectl --context mobilex-argocd -n argo get applications \
  twinx \
  twinx-nvidia-gpu-operator \
  twinx-nvidia-gpu-dra \
  -o custom-columns='NAME:.metadata.name,SYNC:.status.sync.status,HEALTH:.status.health.status,OPERATION:.status.operationState.phase'
```

성공 기준:

```text
twinx                       Synced   Healthy   Succeeded
twinx-nvidia-gpu-operator   Synced   Healthy   Succeeded
twinx-nvidia-gpu-dra        Synced   Healthy   Succeeded
```

## Rollback과 중단 기준

### Greeter 비활성화 중단 기준

VINE desktop/VM session을 이 노드에서 의도적으로 사용해야 한다면 Greeter를 임의로 비활성화하지 않는다. workload owner와 VINE 구성 의도를 먼저 확인한다.

이번 TwinX는 full VINE Application stack을 배포하지 않았고 Kubernetes container GPU를 목표로 하므로 Greeter를 비활성화했다.

### reboot 중단 기준

다음 상태에서는 reboot하지 않는다.

- 미확인 workload 존재
- VolumeAttachment 존재
- active ResourceClaim workload 존재
- drain 실패 원인 불명
- Cilium 또는 control plane 비정상

### DRA Plugin 재시작 중단 기준

대상 노드에서 DRA Claim을 사용하는 Pod가 실행 중이면 Plugin을 삭제하지 않는다. 먼저 workload owner와 maintenance window를 확인한다.

## 재발 방지

### 현재 운영 체크리스트

- [ ] 새 GPU 노드 Join 뒤 GPU Operator Validator가 통과했는가
- [ ] GPU PCI driver가 `nvidia`이고 override가 `(null)`인가
- [ ] `openark-vine-greeter`가 disabled/inactive인가
- [ ] reboot 뒤 새 Boot ID와 Node Ready를 확인했는가
- [ ] GPU allocatable이 `1`로 복구됐는가
- [ ] DRA CDI에 `nvidia-uvm`과 `nvidia-uvm-tools`가 모두 있는가
- [ ] 노드별 `gpu.nvidia.com` ResourceSlice가 있는가
- [ ] 일반 GPU CUDA test가 통과했는가
- [ ] DRA ResourceClaim CUDA test가 통과했는가
- [ ] test namespace를 삭제했는가
- [ ] 노드가 최종 schedulable인가

### 장기 개선 후보

현재 DRA chart는 GPU Operator driver 준비와 DRA Plugin 초기화 순서를 완전히 보장하지 못한다. 다음 중 하나를 별도 변경으로 검토한다.

1. DRA Kubelet Plugin이 `/run/nvidia/driver/dev/nvidia-uvm`과 `nvidia-uvm-tools`를 기다리는 init gate
2. UVM 누락을 탐지하는 운영 점검 또는 alert
3. 새 GPU 노드 Join/reboot 뒤 DRA Plugin을 선택적으로 재생성하는 자동화
4. container GPU 노드에서 VINE Greeter가 다시 enable되지 않도록 provisioning profile 분리

이 개선은 이번 작업에 포함하지 않았다. upstream chart와 SmartX 공용 Application 영향 범위를 확인한 뒤 별도 PR로 다룬다.

## 남은 위험

- 새 GPU 노드 Join이나 reboot 때 DRA CDI 초기화 race가 반복될 수 있다.
- KISS reprovision 또는 OS 재설치가 Greeter를 다시 enable할 수 있다.
- `/etc/modules-load.d/vfio-pci.conf`로 vfio module이 로드되는 것과 GPU가 실제 vfio에 bind되는 것은 다르다. module 존재만 보고 파일을 삭제하지 않는다.
- Management Box에 과거 VINE bind metadata가 남아 있을 수 있다. owner 확인 없이 label을 삭제하지 않는다.
- GPU Operator 또는 NVIDIA DRA version이 바뀌면 test image, CDI 파일 구조와 device class가 달라질 수 있다.

## 관련 문서

- [DataX Gen12 Compute의 default 경유 TwinX 재조인](../cluster-lifecycle/datax-default-twinx-worker-migration-2026-08-24.md)
- [KISS Node Lifecycle 한눈에 보기](../../hardware/provisioning/kiss-node-lifecycle-at-a-glance.md)
- [과거 TwinX NVIDIA DRA Driver Rollout](twinx-nvidia-dra-driver-rollout-2026-06-28.md)
- [Kubernetes Dynamic Resource Allocation](https://kubernetes.io/docs/concepts/scheduling-eviction/dynamic-resource-allocation/)
- [NVIDIA DRA Driver for GPUs](https://github.com/NVIDIA/k8s-dra-driver-gpu)
