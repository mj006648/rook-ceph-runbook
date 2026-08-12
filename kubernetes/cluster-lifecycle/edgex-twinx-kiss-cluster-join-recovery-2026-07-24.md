# EdgeX/TwinX KISS 클러스터 구성 및 Compute 재조인 복구 2026-07-24

> 작성일: 2026-07-24
>
> 상태: EdgeX/TwinX Control Plane 구성 및 각 Compute 4대의 KISS Join 완료
>
> CNI 정책: 의도적으로 미배포. 모든 Node의 `Ready=False`/`NotReady`는 예상 상태
>
> 선행 기록: [DataX Worker 재조인 복구 및 네트워크 안정화](datax-worker-rejoin-recovery-2026-07-24.md)
>
> 후속 복귀: [KISS EdgeX/TwinX/DataX 노드 default 클러스터 복귀 2026-08-12](kiss-default-cluster-return-2026-08-12.md)

## 결과 요약

기존 `default` 클러스터의 E300 두 대를 각각 EdgeX/TwinX Control Plane으로 구성하고 Gen12 desktop 네 대씩을 각 클러스터의 Compute Node로 이동했다.

| 클러스터 | Control Plane | Compute | Box | Node 등록 | CNI |
| --- | --- | ---: | --- | --- | --- |
| EdgeX | `edgex-control-plane-e300-01` | 4대 | 모두 `Running` | 4대 성공 | 미배포 |
| TwinX | `twinx-control-plane-e300-01` | 4대 | 모두 `Running` | 4대 성공 | 미배포 |

CNI를 배포하지 않았으므로 Control Plane과 Compute의 `Ready=False`는 실패가 아니다. 성공 기준은 Box `Running`, 대상 API의 Node 리소스 존재, Join Job `Complete` 또는 성공 후 TTL 삭제였다.

## 대상 장비

### Control Plane

| 클러스터 | UUID | Alias |
| --- | --- | --- |
| EdgeX | `00000000-0000-0000-0000-0cc47a9f8416` | `edgex-control-plane-e300-01` |
| TwinX | `00000000-0000-0000-0000-ac1f6be5de08` | `twinx-control-plane-e300-01` |

### EdgeX Compute

| UUID | Alias |
| --- | --- |
| `d739a00e-2328-a790-8928-1c697ad987c0` | `edgex-desktop-gen12-01` |
| `64348c94-d302-c07a-bce5-1c697ad99e5b` | `edgex-desktop-gen12-02` |
| `da454806-2e36-f088-f10e-1c697ad8c03f` | `edgex-desktop-gen12-03` |
| `3adacb41-faad-9484-7477-1c697ad8c17d` | `edgex-desktop-gen12-04` |

### TwinX Compute

| UUID | Alias |
| --- | --- |
| `02ca34fa-a216-9a2b-2a0a-1c697ad8c177` | `twinx-desktop-gen12-01` |
| `9bc3697d-072e-dc48-29fb-1c697ad99e51` | `twinx-desktop-gen12-02` |
| `9bcc45d1-c926-ab80-b3c0-1c697ad99d94` | `twinx-desktop-gen12-03` |
| `a924e3df-003a-b402-91c6-1c697ad8c15e` | `twinx-desktop-gen12-04` |

| 클러스터 | worker nginx upstream |
| --- | --- |
| EdgeX | `10.32.14.142:6443` |
| TwinX | `10.32.47.196:6443` |

접속 비밀번호, private key, kubeconfig·인증서 본문, bootstrap token은 Git에 기록하지 않는다.

## 확인된 실패 사슬

1. Compute에 이전 `default` 클러스터의 kubelet 인증정보와 CNI 설정이 남아 있었다.
2. stale 인증정보를 분리하자 kubelet이 뜨지 않아 static nginx API proxy도 사라졌다.
3. proxy seed용 CA와 kubelet을 계속 두면 kubeadm preflight가 파일과 10250 포트를 거부했다.
4. Control Plane에는 kubeadm bootstrap ConfigMap과 RBAC도 빠져 있었다.
5. 실패 횟수를 소진한 Job은 Box를 `Failed`로 만들고 자동 재생성되지 않았다.

단순 `kubelet.conf` 이동, 단순 재부팅, `kubeadm reset -f` 중 하나만으로는 해결되지 않았다.

## 오류별 진단표

| 오류/상태 | 의미 | 대응 |
| --- | --- | --- |
| `x509: certificate signed by unknown authority` | 이전 kubelet CA | stale kubelet 인증정보를 root-only backup으로 이동 |
| `Node ... not found` | API 도달 성공, Node 등록 전 | kubelet Join 로그와 CA 확인 |
| `127.0.0.1:6443 ... connection refused` | local static nginx proxy 없음 | 임시 seed로 proxy 기동 |
| `FileAvailable--etc-kubernetes-ssl-ca.crt` | seed CA가 live 경로에 남음 | proxy 기동 후 CA 이동 |
| `Port-10250 ... in use` | seed kubelet이 계속 실행 중 | proxy 기동 후 kubelet 중지 |
| `cluster-info ... NotFound` | bootstrap signer/cluster-info 누락 | `kubeadm init phase bootstrap-token` |
| `kubeadm-config ... forbidden` | bootstrap RBAC 누락 | 표준 Role/RoleBinding 복구 |
| `kubeadm-config ... not found` | ConfigMap 없음 | `upload-config kubeadm` |
| `kubelet-config ... not found` | ConfigMap 없음 | `upload-config kubelet` |
| Box `Failed`, Job 없음 | retry 소진 | 준비 후 reboot로 재커미셔닝 |
| Node `NotReady` | CNI 미배포 | 이번 작업에서는 정상 |

## 변경 전 안전 게이트

클러스터 이동과 재부팅은 workload에 영향을 준다.

```bash
NODE='<node-uuid>'
kubectl get pods -A --field-selector "spec.nodeName=$NODE" -o wide
kubectl get pv,pvc -A
box-ssh '<node-alias>' \
  'sudo ctr -n k8s.io containers list; sudo findmnt; systemctl is-active etcd || true'
```

Control Plane 후보는 기존 etcd, non-DaemonSet workload, local PV/hostPath를 감사하고 snapshot을 확보한다. 세부 항목은 DataX 런북의 `향후 Control Plane 전환 전 별도 게이트` 절을 참고한다.

이번 Compute 복구에서는 `/var/lib/containerd`, local disk/PV, iptables 전체, CNI interface 전체, nginx upstream, Box 리소스를 초기화하지 않았다.

## Control Plane bootstrap 복구

Compute를 재부팅하기 전에 이 절을 완료하는 것이 가장 안전하다.

### cluster-info와 bootstrap signer

```bash
sudo kubeadm init phase bootstrap-token \
  --config /etc/kubernetes/kubeadm-config.yaml

sudo kubectl --kubeconfig /etc/kubernetes/admin.conf \
  -n kube-public get configmap cluster-info

sudo kubectl --kubeconfig /etc/kubernetes/admin.conf \
  auth can-i get configmap/cluster-info \
  -n kube-public --as=system:anonymous
```

anonymous 조회 결과가 `yes`여야 한다.

### kubeadm/kubelet ConfigMap

```bash
sudo kubeadm init phase upload-config kubeadm \
  --config /etc/kubernetes/kubeadm-config.yaml
sudo kubeadm init phase upload-config kubelet \
  --config /etc/kubernetes/kubeadm-config.yaml

sudo kubectl --kubeconfig /etc/kubernetes/admin.conf \
  -n kube-system get configmap kubeadm-config kubelet-config
```

실행 중 `clusterDNS`가 설정값 `10.112.0.3`, 권장값 `10.112.0.10`이라는 경고가 있었지만 ConfigMap 생성은 성공했다. CNI/DNS를 배포하지 않았으므로 DNS 동작은 이번 검증 범위가 아니다.

### bootstrap 최소 RBAC

정상 DataX와 비교해 EdgeX/TwinX에는 다음 두 Role/RoleBinding이 없었다.

- `kubeadm:nodes-kubeadm-config`
- `kubeadm:kubelet-config`

없는 경우에만 생성한다.

```bash
K='sudo kubectl --kubeconfig /etc/kubernetes/admin.conf -n kube-system'

$K get role kubeadm:nodes-kubeadm-config >/dev/null 2>&1 ||
  $K create role kubeadm:nodes-kubeadm-config \
    --verb=get --resource=configmaps --resource-name=kubeadm-config

$K get rolebinding kubeadm:nodes-kubeadm-config >/dev/null 2>&1 ||
  $K create rolebinding kubeadm:nodes-kubeadm-config \
    --role=kubeadm:nodes-kubeadm-config \
    --group=system:bootstrappers:kubeadm:default-node-token \
    --group=system:nodes

$K get role kubeadm:kubelet-config >/dev/null 2>&1 ||
  $K create role kubeadm:kubelet-config \
    --verb=get --resource=configmaps --resource-name=kubelet-config

$K get rolebinding kubeadm:kubelet-config >/dev/null 2>&1 ||
  $K create rolebinding kubeadm:kubelet-config \
    --role=kubeadm:kubelet-config \
    --group=system:nodes \
    --group=system:bootstrappers:kubeadm:default-node-token
```

```bash
sudo kubectl --kubeconfig /etc/kubernetes/admin.conf \
  auth can-i get configmap/kubeadm-config -n kube-system \
  --as=system:bootstrap:test01 \
  --as-group=system:bootstrappers:kubeadm:default-node-token

sudo kubectl --kubeconfig /etc/kubernetes/admin.conf \
  auth can-i get configmap/kubelet-config -n kube-system \
  --as=system:bootstrap:test01 \
  --as-group=system:bootstrappers:kubeadm:default-node-token
```

두 결과 모두 `yes`여야 한다.

## Compute 사전 백업과 stale 설정 분리

### KISS 작업 확인

```bash
UUID='<node-uuid>'
kubectl get box "$UUID" \
  -o custom-columns='ALIAS:.metadata.labels.dash\.ulagbulag\.io/alias,SPEC_CLUSTER:.spec.group.clusterName,SPEC_ROLE:.spec.group.role,BIND_CLUSTER:.status.bindGroup.clusterName,BIND_ROLE:.status.bindGroup.role,STATE:.status.state,UPDATED:.status.lastUpdated'
kubectl -n kiss get job,pod |
  grep -E "NAME|box-(commission|join)-$UUID" || true
```

KISS가 파일을 수정 중일 때 같은 파일을 이동하지 않는다.

### root-only backup

EdgeX:

```bash
TS=$(date +%Y%m%d-%H%M%S)
BACKUP="/root/edgex-compute-prejoin-$TS"
sudo install -d -m 700 "$BACKUP"
printf '%s\n' "$BACKUP" |
  sudo tee /root/.edgex-prejoin-last-backup >/dev/null
```

TwinX:

```bash
TS=$(date +%Y%m%d-%H%M%S)
BACKUP="/root/twinx-compute-prejoin-$TS"
sudo install -d -m 700 "$BACKUP"
printf '%s\n' "$BACKUP" |
  sudo tee /root/.twinx-prejoin-last-backup >/dev/null
```

```bash
sudo cp -a /etc/kubernetes "$BACKUP/" 2>/dev/null || true
sudo cp -a /var/lib/kubelet/pki "$BACKUP/kubelet-pki" 2>/dev/null || true
sudo cp -a /etc/cni/net.d "$BACKUP/cni-net.d" 2>/dev/null || true
sudo cp -a /etc/NetworkManager/system-connections \
  "$BACKUP/networkmanager" 2>/dev/null || true

sudo find "$BACKUP" -xdev -type f ! -name MANIFEST.sha256 \
  -exec sha256sum {} \; |
  sudo tee "$BACKUP/MANIFEST.sha256" >/dev/null
sudo chmod -R go-rwx "$BACKUP"
```

### stale kubelet/CNI 분리

```bash
sudo systemctl stop kubelet

sudo test -e /etc/kubernetes/kubelet.conf &&
  sudo mv /etc/kubernetes/kubelet.conf "$BACKUP/kubelet.conf.stale"
sudo test -d /var/lib/kubelet/pki &&
  sudo mv /var/lib/kubelet/pki "$BACKUP/kubelet-pki.stale"
sudo test -e /etc/kubernetes/ssl/ca.crt &&
  sudo mv /etc/kubernetes/ssl/ca.crt "$BACKUP/ca.crt.stale"
sudo test -d /etc/cni/net.d &&
  sudo mv /etc/cni/net.d "$BACKUP/cni-net.d.stale"
sudo install -d -m 755 /etc/cni/net.d

sudo test ! -e /etc/kubernetes/kubelet.conf
sudo test ! -e /etc/kubernetes/ssl/ca.crt
```

`/etc/nginx/nginx.conf`는 대상 upstream으로 정상 변경돼 있었으므로 삭제하지 않았다.

### Wi-Fi와 유선 경로

```bash
nmcli -f NAME,UUID,TYPE,DEVICE connection show
cat /proc/net/bonding/master 2>/dev/null || true
ip route
```

Gen12 desktop은 유선 NIC를 bond primary/active path로 사용하고 Wi-Fi autoconnect를 비활성화한 뒤 연결을 내렸다. profile은 삭제하지 않았고 변경 전 NetworkManager 구성을 백업했다. NIC/profile 이름은 장비마다 다시 확인한다.

## Box edit와 reboot

운영자가 Box의 `spec.group`을 다음 구조로 변경했다.

```yaml
spec:
  group:
    clusterName: edgex  # 또는 twinx
    role: Compute
```

`Failed` Box에 임의 retry annotation을 추가해도 Job이 다시 생성되지 않았고 operator는 `Waiting for being changed`를 기록했다. annotation은 제거했다.

이번 환경에서는 reboot 뒤 다음 전이가 새 Job을 생성했다.

```text
Failed → Commissioning → Joining → Running
```

단, reboot하면 worker의 임시 nginx proxy도 사라지므로 다음 seed 절차를 즉시 수행할 준비가 필요하다.

## localhost:6443 proxy bootstrap

worker의 kubeadm endpoint `https://localhost:6443`은 kubelet static pod로 실행되는 nginx proxy다. stale kubelet 설정을 모두 분리하면 proxy도 사라져 `connection refused`가 발생한다.

### target CA 확인

```bash
# 대상 Control Plane
sudo openssl x509 -in /etc/kubernetes/ssl/ca.crt \
  -noout -subject -fingerprint -sha256

# worker의 root-only backup
sudo openssl x509 -in "$BACKUP/target-cluster-ca.crt" \
  -noout -subject -fingerprint -sha256
```

두 CA fingerprint가 같아야 한다. `$BACKUP/target-cluster-ca.crt`는 승인된 보안 전송 경로로 미리 배치한 대상 클러스터 CA 사본이며, 이전 클러스터의 stale CA를 사용하면 안 된다. 인증서 본문은 로그/Git에 남기지 않는다.

### proxy seed 임시 복원

이 seed는 이전 자격증명으로 Join하기 위한 것이 아니라 kubelet이 static nginx proxy를 containerd에 생성하게 하기 위한 것이다.

```bash
sudo systemctl stop kubelet
sudo install -d -m 755 /etc/kubernetes/ssl /var/lib/kubelet
sudo cp "$BACKUP/target-cluster-ca.crt" /etc/kubernetes/ssl/ca.crt
sudo cp "$BACKUP/kubelet.conf.stale" /etc/kubernetes/kubelet.conf
sudo cp -a "$BACKUP/kubelet-pki.stale" /var/lib/kubelet/pki
sudo systemctl start kubelet

sudo ss -lnt 'sport = :6443'
curl --cacert /etc/kubernetes/ssl/ca.crt \
  https://localhost:6443/version
```

실제 backup에는 `runtime-seed-used-*`, `runtime-seed-ca-used-*`, `post-reboot-pki-*`, `runtime-seed-reboot-used-*` 형태로 보존했다.

### proxy 기동 후 seed 재분리

proxy container 생성 뒤 kubelet을 중지해도 containerd의 nginx proxy는 유지됐다.

```bash
TS=$(date +%H%M%S)
RUNTIME_BACKUP="$BACKUP/runtime-seed-reboot-used-$TS"
sudo install -d -m 700 "$RUNTIME_BACKUP"
sudo systemctl stop kubelet

sudo test -e /etc/kubernetes/kubelet.conf &&
  sudo mv /etc/kubernetes/kubelet.conf "$RUNTIME_BACKUP/kubelet.conf"
sudo test -e /etc/kubernetes/ssl/ca.crt &&
  sudo mv /etc/kubernetes/ssl/ca.crt "$RUNTIME_BACKUP/ca.crt"
sudo test -d /var/lib/kubelet/pki &&
  sudo mv /var/lib/kubelet/pki "$RUNTIME_BACKUP/pki"
```

재부팅 직후 생성된 최소 `kubelet.key`/`kubelet.crt` pki를 별도 보관했다면 seed pki를 치운 뒤 원래 위치로 복원한다.

최종 preflight 조건:

```bash
sudo test ! -e /etc/kubernetes/kubelet.conf
sudo test ! -e /etc/kubernetes/ssl/ca.crt
test "$(systemctl is-active kubelet || true)" = inactive
sudo ss -lnt 'sport = :6443' | grep LISTEN
```

```text
kubelet.conf   = absent
ca.crt         = absent
kubelet        = inactive
localhost:6443 = listening
```

이 상태가 `ca.crt already exists`와 `Port-10250`을 동시에 방지한다.

## Join 진행 확인

```bash
UUID='<node-uuid>'
kubectl get box "$UUID" \
  -o custom-columns='ALIAS:.metadata.labels.dash\.ulagbulag\.io/alias,CLUSTER:.spec.group.clusterName,ROLE:.spec.group.role,STATE:.status.state,UPDATED:.status.lastUpdated'
kubectl -n kiss get job,pod |
  grep -E "NAME|box-(commission|join)-$UUID" || true
kubectl -n kiss logs "job/box-join-$UUID" --tail=120
```

긴 `kubectl wait` 대신 Pod 재시작 수와 최신 오류를 짧게 확인한다.

```bash
POD=$(kubectl -n kiss get pod -l "job-name=box-join-$UUID" \
  -o jsonpath='{.items[0].metadata.name}')
kubectl -n kiss get pod "$POD"
kubectl -n kiss logs "$POD" --tail=120 |
  grep -Ei 'fatal:|failed:|unreachable|already exists|Port-10250|connection refused|forbidden|not found|PLAY RECAP'
```

Job은 성공 후 TTL로 삭제될 수 있으므로 Job 부재만으로 실패라고 판단하지 않는다.

## 완료 검증

```bash
kubectl get boxes \
  -o custom-columns='NAME:.metadata.name,ALIAS:.metadata.labels.dash\.ulagbulag\.io/alias,CLUSTER:.spec.group.clusterName,ROLE:.spec.group.role,STATE:.status.state' |
  awk 'NR==1 || $2 ~ /^(edgex|twinx)-desktop-gen12-0[1-4]$/'
```

대상 Control Plane:

```bash
sudo kubectl --kubeconfig /etc/kubernetes/admin.conf get nodes \
  -o custom-columns='NAME:.metadata.name,READY:.status.conditions[?(@.type=="Ready")].status,CREATED:.metadata.creationTimestamp'
```

최종 결과는 EdgeX/TwinX Compute 4대씩 Box `Running`, Node 존재, `READY=False`였다.

## Job 실패 시 재시도 순서

1. Control Plane bootstrap/ConfigMap/RBAC 선복구
2. worker backup과 proxy seed 위치 확인
3. reboot
4. `Commissioning`과 SSH 복귀 확인
5. proxy seed 복원
6. `Joining` Job 생성 확인
7. seed 재분리와 kubelet 중지
8. Job과 Node 등록 검증

준비 없이 반복 reboot하면 proxy가 다시 사라져 같은 실패를 반복한다.

## 이번 작업에서 하지 않은 것

- `kubeadm reset -f`
- Box 삭제
- Control Plane 재초기화 또는 etcd 강제 reset
- containerd 데이터 삭제
- iptables 전체 flush
- CNI interface 강제 삭제
- EdgeX/TwinX CNI 배포
- worker에 `admin.conf` 복사
- 인증서 본문 직접 편집

Control Plane의 사용자용 kubectl 설정은 Join 실패 원인이 아니다.

```bash
mkdir -p "$HOME/.kube"
sudo cp /etc/kubernetes/admin.conf "$HOME/.kube/config"
sudo chown "$(id -u):$(id -g)" "$HOME/.kube/config"
```

`admin.conf`를 worker kubelet 설정으로 사용해서는 안 된다.

## Rollback

Rollback은 클러스터 이동 전체를 되돌리는 별도 변경 작업이다. 대상/이전 클러스터의 Node 처리, workload/storage, nginx upstream, NetworkManager saved/runtime 경로, backup checksum을 함께 검토한다. stale 파일만 live 경로에 복사하지 않는다.

## KISS 개선 항목

1. cluster 이동 전 기존/대상 kubelet CA fingerprint 자동 비교
2. stale kubelet.conf, pki, CNI config 탐지
3. localhost API proxy 의존성 preflight
4. proxy seed와 kubeadm preflight의 chicken-and-egg 제거
5. cluster-info, kubeadm-config, kubelet-config, RBAC 자동 검증
6. Box `Failed`의 명시적 retry API 제공
7. active-backup bond의 Wi-Fi 선택/잘못된 primary 경고
8. CNI 미배포 시 `NotReady`를 실패로 오판하지 않는 완료 조건

## Remaining risks

1. CNI가 없어 Pod network가 필요한 workload는 실행할 수 없다.
2. `clusterDNS`의 `10.112.0.3`과 권장값 `10.112.0.10` 차이를 별도로 검토해야 한다.
3. `/root/edgex-compute-prejoin-*`, `/root/twinx-compute-prejoin-*`와 seed backup에는 인증자료가 있어 접근 통제와 폐기 기한이 필요하다.
4. Wi-Fi의 최하위 원인은 미확정이다. 유선 우선화로 Join 경로만 안정화했다.
5. proxy seed는 운영 workaround이며 KISS lifecycle에서 제거해야 한다.
