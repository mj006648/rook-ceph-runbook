# DataX Worker 재조인 복구 및 네트워크 안정화 2026-07-24

> 작성일: 2026-07-24
> 상태: 두 Worker의 KISS 재조인 완료. DataX CNI는 의도적으로 배포하지 않았으며 Control Plane과 두 Worker는 `NetworkPluginNotReady` 상태이다.
> 대상: `datax-desktop-gen12-01` / `fc1258ba-b25c-2724-d6d7-1c697ad99de0`, `datax-desktop-gen12-02` / `fe47eb1b-cf4c-f3fb-5057-1c697ad99f56`
>
> 후속 복귀: [KISS EdgeX/TwinX/DataX 노드 default 클러스터 복귀 2026-08-12](kiss-default-cluster-return-2026-08-12.md)

## Current status

`datax-desktop-gen12-01`을 `default` 클러스터에서 `datax / Compute`로 이동하는 과정에서 이전 클러스터의 kubelet 인증정보와 불안정한 Wi-Fi 경로가 겹쳐 KISS Join이 반복 실패했다. 같은 절차를 `datax-desktop-gen12-02`에 반복 적용해 인증정보 백업·분리, NetworkManager bond primary 영구 보정, KISS Join, stale CNI 분리까지 검증했다.

두 복구 모두 `kubeadm reset -f`를 실행하지 않았다. 이전 인증정보만 root 전용 디렉터리에 백업한 뒤 live 경로에서 이동하고, KISS가 DataX 인증정보를 새로 만들게 했다. 첫 번째 Worker는 bond active slave를 유선 NIC로 임시 전환했고, 두 번째 Worker는 잘못 저장된 `primary=enp108s0`를 실제 유선 NIC인 `primary=enp109s0`로 보정한 뒤 runtime primary도 같은 값으로 맞췄다.

최종 확인 상태:

| 항목 | 결과 |
| --- | --- |
| Box | `datax / Compute / Running` |
| KISS Join | 완료 후 TTL에 의해 Job 삭제 |
| DataX Node 등록 | 성공 |
| kubelet | `active` |
| kubelet CA | DataX cluster CA와 SHA-256 fingerprint 일치 |
| API proxy upstream | `10.32.119.93:6443` |
| bond | `gen12-01`: active만 `enp109s0`; `gen12-02`: primary/active 모두 `enp109s0` |
| NetworkManager | `gen12-02`의 saved bond primary를 `enp109s0`로 보정 |
| CNI | 미배포 |
| Control Plane / 두 Worker | 모두 `NotReady`, `NetworkPluginNotReady` |

`NotReady`는 이번 작업의 실패가 아니다. DataX에 CNI를 배포하지 않기로 했기 때문에 예상되는 상태이다. 성공 기준은 Box `Running`, DataX API의 Node 리소스 존재, 올바른 kubelet CA, 안정적인 API 통신이다.

## 작업 대상과 식별자

| 구분 | 값 |
| --- | --- |
| Worker aliases | `datax-desktop-gen12-01`, `datax-desktop-gen12-02` |
| Worker UUID / Node name | `fc1258ba-b25c-2724-d6d7-1c697ad99de0`, `fe47eb1b-cf4c-f3fb-5057-1c697ad99f56` |
| Worker IP | `10.38.35.101`, `10.38.35.102` |
| DataX Control Plane alias | `datax-control-plane-supermicro-01` |
| DataX Control Plane UUID | `00000000-0000-0000-0000-ac1f6bc5f502` |
| DataX API upstream | `10.32.119.93:6443` |
| 관리 tmux session | `chang2` |

접속 비밀번호, kubeconfig 내용, private key, certificate 본문은 이 저장소에 기록하지 않는다.

### 두 번째 Worker 반복 검증 기록

`datax-desktop-gen12-02`에서도 같은 이전 `default` kubelet CA가 확인됐다. 사용자가 Box를 edit하고 reboot하기 전까지 다음 사전 작업만 수행했다.

| 구분 | 실제 경로 또는 결과 |
| --- | --- |
| kubelet 인증정보 backup | `/root/kubelet-before-datax-20260724-041905` |
| NetworkManager backup | `/root/networkmanager-before-datax-20260724-042126` |
| 기존 kubelet live 경로 | backup 아래로 이동 후 부재 확인 |
| saved bond primary | 잘못된 `enp108s0`에서 실제 유선 `enp109s0`로 보정 |
| KISS Join | 완료 후 TTL에 의해 Job 삭제 |
| DataX Node | 등록 성공, 신규 DataX CA 사용 |
| stale CNI backup | `/root/cni-before-datax-cleanup-20260724-043516` |
| 최종 CNI 상태 | `NetworkPluginNotReady`, 의도한 결과 |

이 기록은 backup 경로와 실행 순서를 재현하기 위한 것이다. 각 디렉터리에는 인증서·키 또는 네트워크 설정이 포함될 수 있으므로 노드 밖으로 무단 복사하거나 Git에 추가하지 않는다.

## Symptom

### 1. DataX API에는 도달하지만 kubelet 등록 실패

Worker의 nginx API proxy는 DataX Control Plane을 가리키고 있었지만 kubelet은 이전 `default` 클러스터 CA를 신뢰하고 있었다.

```text
x509: certificate signed by unknown authority
Unable to register node with API server
```

KISS Join 작업 내부에서도 DataX API에 접근한 뒤 대상 Node를 조회했으나 `NotFound`가 발생했다. 즉 API 경로 자체보다 Node 등록 단계가 핵심 실패 지점이었다.

### 2. 재조인 중 SSH와 API 통신이 간헐적으로 끊김

인증정보를 정리한 뒤에는 x509 오류가 사라졌지만 다음 현상이 반복됐다.

- ping 성공과 실패가 교대로 발생
- `box-ssh`에서 `No route to host`
- Join 작업에서 DataX Control Plane `10.32.119.93` ping 100% loss
- Ansible SSH가 긴 connect timeout 상태로 대기

### 3. Worker만 일시적으로 `Ready`

DataX Control Plane은 CNI 미초기화로 `NotReady`인데 Worker는 잠시 `Ready`로 표시됐다. 그러나 Worker에 DataX CNI가 배포된 것은 아니었다.

Worker에 이전 환경의 CNI 설정이 남아 있었다.

```text
/etc/cni/net.d/05-cilium.conflist
/etc/cni/net.d/00-multus.conf
/etc/cni/net.d/multus.d/multus.kubeconfig
```

`00-multus.conf`는 `cilium-cni`에 위임하고 있었고 DataX에는 실제 Cilium daemon이 없었다. 이 상태에서 runtime이 `NetworkReady=true`를 보고했으므로 Worker의 `Ready`는 정상적인 DataX Pod network를 증명하지 않는 오해 소지가 있는 상태였다.

## Diagnosis

### 1. Box와 Join 상태 확인

관리 클러스터에서 실행한다.

```bash
UUID=fc1258ba-b25c-2724-d6d7-1c697ad99de0

kubectl get box "$UUID" \
  -o custom-columns='ALIAS:.metadata.labels.dash\.ulagbulag\.io/alias,SPEC_CLUSTER:.spec.group.clusterName,SPEC_ROLE:.spec.group.role,BIND_CLUSTER:.status.bindGroup.clusterName,BIND_ROLE:.status.bindGroup.role,STATE:.status.state,UPDATED:.status.lastUpdated'

kubectl -n kiss get job,pod | grep -E "NAME|box-(commission|join)-$UUID" || true
```

Join Job은 완료 후 TTL 정책으로 삭제될 수 있다. Job이 없다는 사실만으로 실패라고 판단하지 말고 Box 상태, Node 등록, kubelet 로그를 함께 확인한다.

### 2. kubelet의 API endpoint와 인증 오류 확인

Worker에서 실행한다.

```bash
sudo grep -H '^[[:space:]]*server:' \
  /etc/kubernetes/kubelet.conf \
  /etc/kubernetes/bootstrap-kubelet.conf 2>/dev/null || true

sudo systemctl --no-pager --full status kubelet

sudo journalctl -u kubelet --since '30 minutes ago' --no-pager |
  grep -Ei 'register|certificate|x509|unauthorized|forbidden|refused|timeout|network|cni|error|fail' |
  tail -n 150
```

로컬 `https://localhost:6443`는 worker의 nginx API proxy이다. 이 주소가 보인다고 잘못된 것은 아니다. 실제 upstream을 따로 확인한다.

```bash
sudo grep -nE 'upstream|server [0-9]|listen|proxy_pass' \
  /etc/nginx/nginx.conf
```

### 3. kubelet CA와 DataX CA의 fingerprint 비교

인증서 본문이나 private key를 출력하지 않고 fingerprint만 비교한다.

Worker:

```bash
sudo awk '/certificate-authority-data:/{print $2; exit}' \
  /etc/kubernetes/kubelet.conf |
  base64 -d |
  openssl x509 -noout -subject -fingerprint -sha256
```

DataX Control Plane:

```bash
sudo openssl x509 \
  -in /etc/kubernetes/ssl/ca.crt \
  -noout -subject -fingerprint -sha256
```

복구 전에는 두 CA가 달랐고, 복구 후에는 다음 fingerprint로 일치했다.

```text
FC:A5:C2:39:4A:81:5D:50:61:5C:5F:23:C7:E9:DE:1F:94:8A:B4:08:25:80:F5:0F:B1:4D:16:FE:05:A7:71:E6
```

### 4. bond와 물리 링크 확인

Worker의 bond interface 이름은 `master`였다.

```bash
ip -brief address
ip route
cat /proc/net/bonding/master
```

장애 당시 관측값:

```text
Bonding Mode: fault-tolerance (active-backup)
Primary Slave: None
Currently Active Slave: wlp0s20f3

wlp0s20f3: MII Status up
enp109s0:   MII Status up, 1 Gbps, full duplex
```

두 링크가 모두 up인데 primary가 지정되지 않아 Wi-Fi가 active slave였다. 정확한 AP·무선 드라이버·물리 환경의 근본 원인까지는 확정하지 못했지만, active slave를 유선으로 바꾸자 2분간 24/24 ping이 성공했고 Join이 바로 진행됐다. 따라서 Wi-Fi 경로 불안정이 Join retry의 직접 유발 요인이었다는 근거는 강하다.

runtime 상태만 보고 영구 설정도 맞다고 판단하면 안 된다. 두 번째 Worker에서는 runtime active slave가 이미 `enp109s0`였지만 NetworkManager profile에는 존재하지 않는 `primary=enp108s0`가 저장돼 있었다.

```bash
nmcli -f NAME,UUID,TYPE,DEVICE connection show
nmcli -g bond.options connection show 11-kiss-bond-master
sudo grep -RHE '^\[bond\]|^interface-name=|^options=' \
  /etc/NetworkManager/system-connections
```

확인해야 하는 값은 다음 세 가지다.

- `/proc/net/bonding/master`의 `Primary Slave`
- `/proc/net/bonding/master`의 `Currently Active Slave`
- NetworkManager bond profile의 saved `primary=`

### 5. CNI 상태가 실제 배포와 일치하는지 확인

Worker:

```bash
sudo find /etc/cni/net.d -maxdepth 3 -type f \
  -printf '%TY-%Tm-%Td %TH:%TM:%TS  %p\n' | sort

sudo crictl info 2>/dev/null |
  grep -A8 -B2 -E 'NetworkReady|NetworkPluginNotReady|lastCNILoadStatus'
```

DataX Control Plane:

```bash
kubectl -n kube-system get pods -o wide
kubectl get nodes -o wide
kubectl get node fc1258ba-b25c-2724-d6d7-1c697ad99de0 \
  -o jsonpath='{range .status.conditions[*]}{.type}{"\t"}{.status}{"\t"}{.reason}{"\t"}{.message}{"\n"}{end}'
```

Worker에 CNI 설정 파일만 있고 DataX Cilium daemon이 없다면 `Ready`를 정상 CNI의 증거로 사용하지 않는다.

## Root cause

### 확인된 원인 1: 이전 클러스터 kubelet 인증정보 잔존

| 항목 | 상태 |
| --- | --- |
| `/etc/kubernetes/kubelet.conf` | 2025-03-14 생성, 이전 `default` 클러스터 CA 포함 |
| nginx API upstream | DataX `10.32.119.93:6443`로 변경됨 |
| 결과 | DataX API 인증서를 이전 CA로 검증해 x509 실패 |

KISS의 클러스터 전환 과정이 기존 kubelet 인증정보를 자동으로 교체하지 못했다. API proxy와 kubelet 자격증명이 서로 다른 클러스터를 가리키는 혼합 상태였다.

### 확인된 원인 2: Wi-Fi가 bond의 active path

`active-backup` bond의 primary가 없고 Wi-Fi `wlp0s20f3`가 active slave였다. 이 경로에서 worker와 DataX Control Plane 사이의 ping 및 SSH가 간헐적으로 실패했다. 유선 `enp109s0`으로 전환한 뒤 연결이 안정되고 Join이 완료됐다.

Wi-Fi 자체의 물리적 결함인지, AP·라우팅·드라이버·절전 설정 문제인지는 별도 조사 대상이다.

### 별도 상태: 이전 CNI 설정 잔존

이전 Cilium/Multus 설정은 Join 실패의 최초 x509 원인은 아니지만, DataX CNI가 없는 상태에서 Worker를 `Ready`처럼 보이게 만들었다. 운영자가 Join과 Pod network가 모두 정상이라고 오판할 수 있으므로 백업 후 live 경로에서 분리했다.

## 작업 전 안전 기준

### 중단 기준

다음 중 하나라도 해당하면 인증정보를 이동하지 않는다.

- 해당 노드에 이동하면 안 되는 Kubernetes workload가 남아 있음
- 사용자 Docker/containerd workload 또는 VM이 확인됨
- local PV, hostPath, mount 사용 여부를 확인하지 못함
- KISS Commission/Join이 live 파일을 동시에 수정 중임
- worker의 콘솔 또는 대체 접속 경로가 없음

### workload 영향 확인

향후 같은 작업에서는 Box edit 전에 기존 클러스터와 worker 양쪽을 확인한다.

기존 클러스터:

```bash
kubectl get node fc1258ba-b25c-2724-d6d7-1c697ad99de0 -o wide
kubectl get pods -A \
  --field-selector spec.nodeName=fc1258ba-b25c-2724-d6d7-1c697ad99de0 \
  -o wide
```

Worker:

```bash
sudo crictl ps -a
docker ps -a 2>/dev/null || true
findmnt
sudo find /var/lib/kubelet/pods -maxdepth 3 -type d -name volumes \
  -print 2>/dev/null
```

클러스터 이동은 기존 클러스터가 이 노드의 pod를 계속 관리할 수 없게 만드는 작업이다. `kubeadm reset`을 쓰지 않더라도 kubelet 중지와 재부팅은 해당 노드의 Kubernetes workload를 중단시킬 수 있다. 기존 workload가 없다는 확인 또는 drain·서비스 이관이 먼저다.

이번 작업에서 다음 항목은 삭제하거나 초기화하지 않았다.

- `/var/lib/containerd`
- local disk와 PV 데이터
- `/mnt/openark-vine-session`
- iptables
- CNI interface
- nginx proxy 설정
- DataX Control Plane 설정

위 경로를 건드리지 않았다는 사실이 그 안의 workload 무중단을 보장하지는 않는다. 특히 kubelet 중지와 재부팅의 영향은 별도로 평가해야 한다.

## Fix

아래는 다음 재조인 때 사용할 안전한 순서다. 실제 2026-07-24 작업에서는 인증정보 정리 후 사용자가 Box를 `datax / Compute`로 변경하고 재부팅했다.

### 0단계 — 동시에 실행 중인 KISS 작업 확인

관리 클러스터:

```bash
UUID=fc1258ba-b25c-2724-d6d7-1c697ad99de0
kubectl -n kiss get job,pod | grep -E "NAME|box-(commission|join)-$UUID" || true
```

Join이 파일을 수정하는 중이면 먼저 KISS가 제공하는 정상 중단 경로를 사용한다. Job을 임의 삭제하거나 suspend하기 전에 controller가 즉시 재생성하는지 확인한다.

### 1단계 — kubelet 중지와 기존 인증정보 백업

Worker:

```bash
sudo systemctl stop kubelet

TS=$(date +%Y%m%d-%H%M%S)
BACKUP=/root/kubelet-before-datax-$TS
sudo install -d -m 700 "$BACKUP"

sudo cp -a /etc/kubernetes/kubelet.conf \
  "$BACKUP/kubelet.conf.original"
sudo cp -a /var/lib/kubelet/pki \
  "$BACKUP/kubelet-pki.original"
sudo cp -a /etc/kubernetes/pki/ca.crt \
  "$BACKUP/ca.crt.original"

sudo sha256sum \
  /etc/kubernetes/kubelet.conf \
  "$BACKUP/kubelet.conf.original"
```

인증서와 키가 포함되므로 backup은 `/root`, mode `700`에 두고 Git, 메신저, 일반 사용자 홈에 복사하지 않는다.

### 2단계 — 삭제 대신 live 경로 밖으로 이동

```bash
sudo mv /etc/kubernetes/kubelet.conf \
  "$BACKUP/kubelet.conf.default"
sudo mv /var/lib/kubelet/pki \
  "$BACKUP/kubelet-pki.default"
sudo mv /etc/kubernetes/pki/ca.crt \
  "$BACKUP/ca.crt.default"

sudo find "$BACKUP" -maxdepth 2 -printf '%M %u:%g %p\n'
sudo test ! -e /etc/kubernetes/kubelet.conf
sudo test ! -e /var/lib/kubelet/pki
sudo test ! -e /etc/kubernetes/pki/ca.crt

sudo find "$BACKUP" -xdev -type f ! -name MANIFEST.sha256 \
  -exec sha256sum {} \; |
  sudo tee "$BACKUP/MANIFEST.sha256" >/dev/null
sudo chmod 600 "$BACKUP/MANIFEST.sha256"
```

이번 작업에서 사용한 실제 backup:

```text
/root/kubelet-before-datax-20260724-023912
├── ca.crt.default
├── ca.crt.original
├── kubelet.conf.default
├── kubelet.conf.original
├── kubelet-pki.default/
└── kubelet-pki.original/
```

원본과 복사본 `kubelet.conf`의 SHA-256은 일치했다.

```text
0428018eebc5c7c03d45d7a356af92afd8c524f7b3357f7ddb73b3ecbfe5e0f5
```

### 3단계 — Box를 `datax / Compute`로 변경

관리 클러스터:

```bash
kubectl edit box fc1258ba-b25c-2724-d6d7-1c697ad99de0
```

의도한 값:

```yaml
spec:
  group:
    clusterName: datax
    role: Compute
```

변경 후 spec과 status를 구분해서 본다. spec은 요청값이고 status는 실제 적용 결과이다.

```bash
kubectl get box fc1258ba-b25c-2724-d6d7-1c697ad99de0 \
  -o custom-columns='SPEC_CLUSTER:.spec.group.clusterName,SPEC_ROLE:.spec.group.role,BIND_CLUSTER:.status.bindGroup.clusterName,BIND_ROLE:.status.bindGroup.role,STATE:.status.state'
```

### 4단계 — 재부팅 판단

Box edit 자체와 kubelet 인증정보 재생성에는 원칙적으로 재부팅이 필수는 아니다. 다음 경우에만 재부팅한다.

- KISS Commission 절차가 reboot를 요구함
- kubelet·proxy·network 상태가 이전 lifecycle에서 남아 정상 전환되지 않음
- 콘솔 또는 대체 접속 경로가 확보됨

이번 작업에서는 사용자가 재부팅했다. 단, sysfs로 변경한 bond active slave는 재부팅하면 원래 선택 정책으로 돌아갈 수 있으므로 네트워크 검증 없이 반복 재부팅하지 않는다.

### 5단계 — Join 진행 확인

관리 클러스터:

```bash
UUID=fc1258ba-b25c-2724-d6d7-1c697ad99de0

watch -n 2 \
  "kubectl get box $UUID -o custom-columns='STATE:.status.state,UPDATED:.status.lastUpdated'; \
   kubectl -n kiss get job,pod | grep -E 'NAME|box-(commission|join)-$UUID' || true"
```

Job이 존재하는 동안:

```bash
kubectl -n kiss logs job/box-join-$UUID -f
```

다음 로그가 다시 나오면 인증정보 문제다.

```text
x509: certificate signed by unknown authority
Unable to register node with API server
```

ping, `No route to host`, SSH timeout이 반복되면 인증서를 더 지우지 말고 네트워크 경로를 먼저 확인한다.

### 6단계 — 유선을 bond primary로 지정

먼저 runtime과 saved profile을 모두 확인한다.

```bash
cat /proc/net/bonding/master
nmcli -f NAME,UUID,TYPE,DEVICE connection show
nmcli -g bond.options connection show 11-kiss-bond-master
```

NetworkManager 설정 전체와 변경 전 option을 먼저 백업한다.

```bash
TS=$(date +%Y%m%d-%H%M%S)
NM_BACKUP=/root/networkmanager-before-datax-$TS
BOND_PROFILE=11-kiss-bond-master

sudo install -d -m 700 "$NM_BACKUP"
sudo cp -a /etc/NetworkManager/system-connections \
  "$NM_BACKUP/"
nmcli -g bond.options connection show "$BOND_PROFILE" |
  sudo tee "$NM_BACKUP/bond-options.before.txt" >/dev/null
```

기존 option 전체를 보존하면서 `primary`만 실제 유선 NIC로 바꾼다. profile 이름과 NIC 이름은 장비마다 다시 확인한다.

```bash
OLD=$(nmcli -g bond.options connection show "$BOND_PROFILE")
case ",$OLD," in
  *,primary=*)
    NEW=$(printf '%s\n' "$OLD" |
      sed -E 's/(^|,)primary=[^,]*/\1primary=enp109s0/')
    ;;
  *)
    NEW="${OLD:+$OLD,}primary=enp109s0"
    ;;
esac

sudo nmcli connection modify "$BOND_PROFILE" bond.options "$NEW"
nmcli -g bond.options connection show "$BOND_PROFILE" |
  sudo tee "$NM_BACKUP/bond-options.after.txt" >/dev/null
```

`nmcli` 수행 중 D-Bus 응답 단절 오류가 발생해도 곧바로 재실행하지 않는다. 두 번째 Worker에서는 다음 오류가 출력됐지만 saved profile 변경은 이미 적용돼 있었다.

```text
Message recipient disconnected from message bus without replying
```

먼저 `nmcli`, 실제 keyfile, NetworkManager 상태를 다시 읽어 적용 여부를 확인한다. NetworkManager restart는 원격 접속을 끊을 수 있으므로 콘솔 또는 대체 접속 수단이 있는 유지보수 시간에만 수행한다.

runtime primary도 유선으로 맞춘다.

```bash
echo enp109s0 | sudo tee /sys/class/net/master/bonding/primary

grep -E 'Bonding Mode|Primary Slave|Currently Active Slave' \
  /proc/net/bonding/master
nmcli -g bond.options connection show "$BOND_PROFILE"
```

두 번째 Worker의 최종 값:

```text
Primary Slave: enp109s0 (primary_reselect better)
Currently Active Slave: enp109s0
mode=active-backup,downdelay=0,miimon=100,primary=enp109s0,primary_reselect=better,updelay=0
```

첫 번째 Worker에서는 `active_slave`만 임시로 바꿨으므로 reboot 또는 NetworkManager reload 뒤 재확인이 필요하다. 두 번째 Worker에서는 saved profile과 runtime primary를 모두 보정했지만 reboot 뒤에도 반드시 다시 확인한다.

API 경로는 ICMP만으로 판정하지 않는다. 다음 검사는 TLS 인증을 검증하기 위한 것이 아니라 API listener의 연속 도달성만 보는 진단이다.

```bash
OK=0
FAIL=0
for i in $(seq 1 10); do
  if curl -ksS --connect-timeout 2 --max-time 4 \
    https://10.32.119.93:6443/livez | grep -qx ok; then
    OK=$((OK + 1))
  else
    FAIL=$((FAIL + 1))
  fi
  sleep 1
done
printf 'ok=%s fail=%s\n' "$OK" "$FAIL"
```

두 번째 Worker에서는 gateway가 10/10 응답하고 유선 NIC error/drop이 0이었지만 Control Plane ICMP 손실과 `/livez` 3/10 실패가 관측됐다. Join은 완료됐으나 이 결과는 완전한 네트워크 정상화를 뜻하지 않으며 별도 경로 품질 점검 대상이다.

긴급 rollback이 필요하고 Wi-Fi 경로가 정상임을 확인한 경우에만 기존 backup을 기준으로 saved profile과 runtime 값을 함께 되돌린다. Wi-Fi connection 자체를 삭제하지 않는다.

### 7단계 — stale CNI 설정 백업 및 live 경로에서 분리

이 단계는 **DataX CNI를 의도적으로 배포하지 않는 경우**에만 적용한다. 곧 올바른 Cilium/Multus를 배포할 예정이라면 GitOps·KISS 소유권을 먼저 확인한다.

Worker:

```bash
TS=$(date +%Y%m%d-%H%M%S)
CNI_BACKUP=/root/cni-before-datax-cleanup-$TS
sudo install -d -m 700 "$CNI_BACKUP"

sudo cp -a /etc/cni/net.d \
  "$CNI_BACKUP/net.d.snapshot"

sudo mv /etc/cni/net.d/00-multus.conf \
  "$CNI_BACKUP/" 2>/dev/null || true
sudo mv /etc/cni/net.d/05-cilium.conflist \
  "$CNI_BACKUP/" 2>/dev/null || true
sudo mv /etc/cni/net.d/multus.d \
  "$CNI_BACKUP/" 2>/dev/null || true

sudo find /etc/cni/net.d -maxdepth 3 -printf '%M %u:%g %p\n'
sudo find "$CNI_BACKUP" -maxdepth 4 -printf '%M %u:%g %p\n'

sudo find "$CNI_BACKUP" -xdev -type f ! -name MANIFEST.sha256 \
  -exec sha256sum {} \; |
  sudo tee "$CNI_BACKUP/MANIFEST.sha256" >/dev/null
sudo chmod 600 "$CNI_BACKUP/MANIFEST.sha256"
```

이번 작업에서 사용한 실제 backup:

```text
# datax-desktop-gen12-01
/root/cni-before-datax-cleanup-20260724-035256

# datax-desktop-gen12-02
/root/cni-before-datax-cleanup-20260724-043516
```

공통으로 이동한 CNI 파일과 전체 `net.d.snapshot`을 보존했다. `gen12-02` backup에는 다음처럼 `MANIFEST.sha256`도 함께 생성했다.

```text
├── 00-multus.conf
├── 05-cilium.conflist
├── multus.d/
├── net.d.snapshot/
└── MANIFEST.sha256
```

live `/etc/cni/net.d`에는 `.cni-concurrency.lock`만 남았다. CNI binary, Cilium interface, iptables, containerd data는 삭제하지 않았다.

이번에는 kubelet 또는 containerd를 재시작하지 않아도 runtime이 변경을 감지했다.

```text
NetworkReady=false
reason: NetworkPluginNotReady
message: cni plugin not initialized
```

### 8단계 — backup 보존과 정리

노드마다 다음 세 종류를 구분해 보존한다.

| backup | 필수 내용 | 생성 시점 |
| --- | --- | --- |
| kubelet 인증정보 | `kubelet.conf`, `/var/lib/kubelet/pki`, cluster `ca.crt`, checksum | Box edit 전 |
| NetworkManager | `system-connections` 전체, bond options 변경 전·후 | bond 변경 전 |
| stale CNI | `/etc/cni/net.d` snapshot과 live 경로에서 이동한 파일, checksum | Join 성공 후 CNI 오판 정리 전 |

2026-07-24 실제 backup:

| Worker | kubelet | NetworkManager | CNI |
| --- | --- | --- | --- |
| `gen12-01` | `/root/kubelet-before-datax-20260724-023912` | 별도 영구 변경 없음 | `/root/cni-before-datax-cleanup-20260724-035256` |
| `gen12-02` | `/root/kubelet-before-datax-20260724-041905` | `/root/networkmanager-before-datax-20260724-042126` | `/root/cni-before-datax-cleanup-20260724-043516` |

backup은 root 인증정보와 이전 network 설정을 포함하므로 DataX 운영 검증이 끝날 때까지 해당 노드의 `/root`에만 보관한다. 다른 노드의 backup을 복원하지 않는다.

```bash
mapfile -t BACKUPS < <(
  sudo find /root -maxdepth 1 -type d \
    \( -name 'kubelet-before-datax-*' \
    -o -name 'networkmanager-before-datax-*' \
    -o -name 'cni-before-datax-cleanup-*' \) \
    -print
)
((${#BACKUPS[@]} > 0)) || {
  echo 'no DataX rejoin backup found' >&2
  exit 1
}

sudo du -sh "${BACKUPS[@]}"
sudo find "${BACKUPS[@]}" -xdev -type f \
  ! -name MANIFEST.sha256 -exec sha256sum {} \; |
  sudo tee /root/datax-rejoin-backup-sha256-20260724.txt >/dev/null
sudo chmod 600 /root/datax-rejoin-backup-sha256-20260724.txt
```

backup 삭제는 다음 조건을 모두 만족한 뒤 별도 승인으로 수행한다.

- DataX Node 등록과 CA 검증 완료
- 이전 `default` 클러스터로 rollback하지 않기로 결정
- 올바른 CNI 배포 또는 CNI 미배포 정책 확정
- 필요한 설정을 별도 보안 backup에 보존

## Verification

### 1. Box 상태

관리 클러스터:

```bash
kubectl get box fc1258ba-b25c-2724-d6d7-1c697ad99de0 \
  -o custom-columns='ALIAS:.metadata.labels.dash\.ulagbulag\.io/alias,CLUSTER:.status.bindGroup.clusterName,ROLE:.status.bindGroup.role,STATE:.status.state,UPDATED:.status.lastUpdated'
```

실제 결과:

```text
datax-desktop-gen12-01  datax  Compute  Running
```

### 2. DataX Node 등록

DataX Control Plane:

```bash
kubectl get node fc1258ba-b25c-2724-d6d7-1c697ad99de0 -o wide
```

`NotFound`가 아니고 실제 Node 리소스가 보여야 한다.

### 3. kubelet과 신규 파일 시각

Worker:

```bash
systemctl is-active kubelet
sudo stat -c '%y  %n' /etc/kubernetes/kubelet.conf
sudo journalctl -u kubelet --since '20 minutes ago' --no-pager |
  grep -c 'x509: certificate signed by unknown authority'
```

실제 확인:

```text
kubelet: active
kubelet.conf: 2026-07-24 03:43:50.603401422 +0000
recent x509 count: 0
```

### 4. CA fingerprint

Worker와 DataX Control Plane에서 `Diagnosis`의 fingerprint 명령을 각각 실행한다. 두 값이 같아야 한다.

### 5. 네트워크 안정성

```bash
grep -E 'Primary Slave|Currently Active Slave' /proc/net/bonding/master
nmcli -g bond.options connection show 11-kiss-bond-master
ping -c 10 -W 1 10.47.255.254

OK=0
FAIL=0
for i in $(seq 1 10); do
  if curl -ksS --connect-timeout 2 --max-time 4 \
    https://10.32.119.93:6443/livez | grep -qx ok; then
    OK=$((OK + 1))
  else
    FAIL=$((FAIL + 1))
  fi
done
printf 'ok=%s fail=%s\n' "$OK" "$FAIL"
```

기대값은 primary와 active slave가 모두 실제 유선 NIC이고, gateway와 API `/livez`가 지속적으로 응답하는 것이다. ICMP 하나만 성공하거나 Join이 한 번 완료된 것만으로 네트워크가 완전히 정상이라고 결론내리지 않는다.

### 6. CNI 미배포 상태

DataX Control Plane:

```bash
kubectl get nodes -o wide
kubectl get node fc1258ba-b25c-2724-d6d7-1c697ad99de0 \
  -o jsonpath='{range .status.conditions[?(@.type=="Ready")]}{.status}{"\t"}{.reason}{"\t"}{.message}{"\n"}{end}'
```

현재 의도한 결과:

```text
False  KubeletNotReady  Network plugin returns error: cni plugin not initialized
```

Control Plane과 Worker 모두 이 상태여야 stale CNI 설정으로 인한 거짓 `Ready`가 제거된 것이다.

## Rollback

### 이전 kubelet 인증정보 복원

아래 경로는 `gen12-01`의 실제 예시다. 다른 Worker에서는 그 노드에서 생성·검증한 정확한 backup 경로로 바꾸며, 다른 노드의 backup을 사용하지 않는다.

이 rollback은 노드를 다시 이전 `default` 클러스터에 붙이는 결정이 내려졌을 때만 사용한다. DataX 재조인 중에는 실행하지 않는다.

```bash
sudo systemctl stop kubelet

BACKUP=/root/kubelet-before-datax-20260724-023912
sudo mv /etc/kubernetes/kubelet.conf \
  /root/kubelet.conf.datax.rollback-hold 2>/dev/null || true
sudo mv /var/lib/kubelet/pki \
  /root/kubelet-pki.datax.rollback-hold 2>/dev/null || true
sudo mv /etc/kubernetes/pki/ca.crt \
  /root/ca.crt.datax.rollback-hold 2>/dev/null || true

sudo cp -a "$BACKUP/kubelet.conf.original" \
  /etc/kubernetes/kubelet.conf
sudo cp -a "$BACKUP/kubelet-pki.original" \
  /var/lib/kubelet/pki
sudo cp -a "$BACKUP/ca.crt.original" \
  /etc/kubernetes/pki/ca.crt
```

복원 후 proxy upstream도 이전 클러스터와 일치해야 한다. nginx가 계속 DataX를 가리키는 상태에서 이전 kubelet 인증정보만 복원하면 최초의 혼합 상태가 재발한다.

### 이전 CNI 설정 복원

정확히 같은 이전 Cilium/Multus 환경으로 돌아갈 때만 복원한다. DataX에 CNI daemon이 없는 상태에서 파일만 복원하면 다시 오해 소지가 있는 `Ready` 상태가 발생할 수 있다.

```bash
CNI_BACKUP=/root/cni-before-datax-cleanup-20260724-035256

sudo cp -a "$CNI_BACKUP/00-multus.conf" \
  /etc/cni/net.d/
sudo cp -a "$CNI_BACKUP/05-cilium.conflist" \
  /etc/cni/net.d/
sudo cp -a "$CNI_BACKUP/multus.d" \
  /etc/cni/net.d/
```

복원 후에는 실제 Cilium/Multus daemon, CRD, kubeconfig, cluster identity가 모두 일치하는지 별도로 검증한다.

## `kubeadm reset -f`를 사용하지 않은 이유

`kubeadm reset -f`는 인증정보만 지우는 명령이 아니다. kubeadm이 관리하는 node-local Kubernetes 상태를 넓게 정리하며, 환경에 따라 CNI·iptables·mount·runtime 잔존 상태에 대한 추가 수동 정리가 필요할 수 있다.

이번 장애의 확인된 원인은 오래된 kubelet 인증정보였으므로 다음 최소 범위만 처리했다.

```text
/etc/kubernetes/kubelet.conf
/var/lib/kubelet/pki
/etc/kubernetes/pki/ca.crt
```

이 최소 조치로 KISS가 올바른 DataX 인증정보를 재생성했고 Join이 성공했다. 따라서 `kubeadm reset -f`는 필요하지 않았다.

다음 경우에만 reset을 별도 변경 작업으로 검토한다.

- 최소 인증정보 정리 후에도 kubeadm state 충돌이 반복됨
- KISS/Kubespray의 공식 reset 절차가 확인됨
- workload, mount, local PV, runtime 영향을 사전 감사함
- 복구 backup과 콘솔 접속을 확보함
- 운영 승인과 maintenance window가 있음

## Control Plane의 `$HOME/.kube/config` 복사와의 관계

다음 명령은 DataX Control Plane에서 현재 사용자의 `kubectl` 설정을 만드는 작업이다.

```bash
mkdir -p "$HOME/.kube"
sudo cp /etc/kubernetes/admin.conf "$HOME/.kube/config"
sudo chown "$(id -u):$(id -g)" "$HOME/.kube/config"
```

이 명령은 다음을 변경하지 않는다.

- DataX cluster CA
- API server 인증서
- Worker의 `/etc/kubernetes/kubelet.conf`
- Worker의 kubelet certificate
- CNI

따라서 이번 Join 실패 원인이 아니다. `admin.conf`를 Worker의 kubelet 설정으로 복사해서도 안 된다.

## 향후 Control Plane 전환 전 별도 게이트

> 후속 결과: 아래 사전 조사 이후 EdgeX/TwinX Control Plane 구성과 Compute 8대 Join을 완료했다. 실제 수행 절차와 추가로 확인된 `cluster-info`, localhost API proxy, kubeadm ConfigMap/RBAC 문제는 [EdgeX/TwinX KISS 클러스터 구성 및 Compute 재조인 복구](edgex-twinx-kiss-cluster-join-recovery-2026-07-24.md)에 정리했다.

Worker의 인증정보 최소 분리 절차를 Control Plane 후보에 그대로 적용하면 안 된다. Control Plane 역할로 edit하기 전에 기존 workload와 로컬 etcd의 소유권·데이터를 먼저 확인한다.

2026-07-24 읽기 전용 사전 확인 결과:

| 후보 | 현재 상태 | 네트워크 | 추가 확인 사항 |
| --- | --- | --- | --- |
| `twinx-control-plane-e300-01` | `default / Compute / Running`, Node `Ready` | Wi-Fi는 bond 밖에서 down, 유선 `eno1` primary/active | `etcd` active, 실제 data dir과 member 구성을 확정해야 함 |
| `edgex-control-plane-e300-01` | `default / Compute / Running`, Node `Ready` | Wi-Fi 장치 없음, 유선 `eno2` primary/active | `virt-api`, `scalex-discovery` 등 non-DaemonSet workload와 단일-member etcd 확인 |

두 장비 모두 `/var/lib/etcd`가 없다는 사실만으로 etcd가 없다고 판단하면 안 된다. EdgeX 후보에서는 `/etc/etcd.env`의 `ETCD_DATA_DIR=/opt/etcd`와 2379/2380 listener가 확인됐다.

### 1. 변경 전 읽기 점검

```bash
systemctl is-active etcd
sudo systemctl cat etcd
sudo ss -lntp | grep -E ':(2379|2380)\b' || true
sudo grep -E \
  '^(ETCD_DATA_DIR|ETCD_NAME|ETCD_INITIAL_CLUSTER|ETCD_INITIAL_CLUSTER_STATE|ETCD_.*(CERT|KEY|CA))=' \
  /etc/etcd.env 2>/dev/null || true

sudo find /etc/kubernetes/manifests -maxdepth 1 -type f -printf '%f\n'
sudo test -e /etc/kubernetes/admin.conf && echo admin-conf-present || true
sudo test -d /var/lib/etcd && echo var-lib-etcd-present || true
```

기존 클러스터에서는 대상 Node에 배치된 workload를 확인한다.

```bash
NODE='candidate-node-uuid'
kubectl get pods -A --field-selector "spec.nodeName=$NODE" -o wide
```

DaemonSet이 아닌 pod, local PV, hostPath, VM, 사용자 container가 있으면 이관 또는 중단 계획이 확정되기 전에는 진행하지 않는다.

### 2. etcd membership와 health 확인

실제 인증서 경로는 `/etc/etcd.env`와 실행 중인 service argument에서 확인한다. 값이 확인되지 않으면 예시 경로를 추측하지 말고 중단한다. 다음 명령 전에 `ETCD_CA`, `ETCD_CERT`, `ETCD_KEY`를 확인된 실제 경로로 설정한다.

```bash
: "${ETCD_CA:?set verified etcd CA path}"
: "${ETCD_CERT:?set verified etcd client certificate path}"
: "${ETCD_KEY:?set verified etcd client key path}"

sudo ETCDCTL_API=3 etcdctl \
  --endpoints="https://127.0.0.1:2379" \
  --cacert="$ETCD_CA" \
  --cert="$ETCD_CERT" \
  --key="$ETCD_KEY" \
  endpoint health

sudo ETCDCTL_API=3 etcdctl \
  --endpoints="https://127.0.0.1:2379" \
  --cacert="$ETCD_CA" \
  --cert="$ETCD_CERT" \
  --key="$ETCD_KEY" \
  member list -w table
```

member가 다른 운영 클러스터와 연결돼 있거나 용도를 설명할 수 없으면 Box role을 변경하지 않는다.

### 3. Control Plane backup 최소 세트

```bash
TS=$(date +%Y%m%d-%H%M%S)
CP_BACKUP=/root/control-plane-before-repurpose-$TS
sudo install -d -m 700 "$CP_BACKUP"

sudo cp -a /etc/etcd.env "$CP_BACKUP/" 2>/dev/null || true
sudo cp -a /etc/etcd "$CP_BACKUP/" 2>/dev/null || true
sudo cp -a /etc/systemd/system/etcd.service "$CP_BACKUP/" 2>/dev/null || true
sudo cp -a /etc/kubernetes "$CP_BACKUP/" 2>/dev/null || true
sudo cp -a /var/lib/kubelet/pki "$CP_BACKUP/" 2>/dev/null || true
sudo cp -a /etc/NetworkManager/system-connections \
  "$CP_BACKUP/" 2>/dev/null || true
```

실행 중인 etcd의 data directory를 단순 `cp`한 파일은 일관된 backup으로 간주하지 않는다. health와 member 구성을 확인한 뒤 snapshot을 별도로 만든다.

```bash
sudo ETCDCTL_API=3 etcdctl \
  --endpoints="https://127.0.0.1:2379" \
  --cacert="$ETCD_CA" \
  --cert="$ETCD_CERT" \
  --key="$ETCD_KEY" \
  snapshot save "$CP_BACKUP/etcd.snapshot.db"

sudo ETCDCTL_API=3 etcdctl \
  snapshot status "$CP_BACKUP/etcd.snapshot.db" -w table

sudo find "$CP_BACKUP" -xdev -type f ! -name MANIFEST.sha256 \
  -exec sha256sum {} \; |
  sudo tee "$CP_BACKUP/MANIFEST.sha256" >/dev/null
sudo chmod -R go-rwx "$CP_BACKUP"
```

certificate, private key, etcd snapshot은 Git·메신저·일반 사용자 홈에 복사하지 않는다. snapshot 검증 실패, etcd 소유권 불명, non-DaemonSet workload 미처리 중 하나라도 남으면 Control Plane edit을 중단한다.

## Prevention

### 클러스터 이동 전 체크리스트

- [ ] 기존 클러스터에서 대상 Node와 workload 확인
- [ ] direct Docker/containerd workload 확인
- [ ] local PV, hostPath, mount 확인
- [ ] KISS Commission/Join 작업이 동시에 실행 중인지 확인
- [ ] 기존 kubelet CA와 대상 cluster CA fingerprint 비교
- [ ] `/etc/kubernetes`, `/var/lib/kubelet/pki` root-only backup
- [ ] bond runtime primary·active slave와 NetworkManager saved `primary=`를 함께 확인
- [ ] NetworkManager profile 전체와 bond options 변경 전·후 backup
- [ ] gateway ping과 Control Plane API `/livez` 연속 도달성 확인
- [ ] Box spec과 status를 구분해 확인
- [ ] CNI를 배포할지, 의도적으로 미배포할지 결정
- [ ] rollback 시 proxy upstream까지 함께 되돌릴 계획 수립

### KISS 개선 후보

- clusterName 변경 시 기존 kubelet CA와 대상 CA가 다르면 Join 전에 명확히 중단
- stale `/etc/kubernetes/kubelet.conf`와 `/var/lib/kubelet/pki` 탐지
- Node `Ready`만 보지 말고 실제 대상 CNI daemon과 config 소유권 검증
- Join 전 worker에서 API endpoint까지 연속 reachability 검사
- active-backup bond에서 Wi-Fi가 active이거나 saved primary가 존재하지 않는 NIC이면 경고
- Control Plane 전환 시 local etcd와 non-DaemonSet workload를 자동 탐지하고 snapshot 없이는 중단

## Remaining risks

1. **두 Worker의 bond 영구성 수준이 다르다.**
   `gen12-01`은 runtime active slave만 바꿨고, `gen12-02`는 NetworkManager saved primary와 runtime primary를 모두 `enp109s0`로 보정했다. 어느 쪽도 reboot 뒤 확인을 생략하면 안 된다.

2. **Wi-Fi 불안정의 가장 낮은 계층 원인은 미확정이다.**
   유선 전환으로 증상은 해소됐지만 AP, signal, power-save, driver, route 중 무엇이 원인인지는 추가 진단이 필요하다.

3. **DataX CNI는 없다.**
   Control Plane과 Worker는 정상적으로 Node 등록됐지만 Pod network가 필요한 workload는 실행할 수 없다.

4. **이전 Rook-Ceph CSI 로그가 남아 있다.**
   kubelet 로그에 다음 stale unmount 오류가 관측됐다.

   ```text
   driver name csi-rook-ceph.cephfs.csi.ceph.com not found
   ```

   이번 작업에서는 mount, volume data, CSI state를 수정하지 않았다. 실제 남은 mount와 pod volume 경로는 별도 storage 점검이 필요하다.

5. **backup에는 인증자료가 포함된다.**
   `/root/kubelet-before-datax-*`, `/root/networkmanager-before-datax-*`, `/root/cni-before-datax-cleanup-*`는 장기 방치하지 말고 rollback 기간 종료 후 보안 정책에 따라 정리한다.

6. **Control Plane 후보의 기존 etcd 용도는 아직 확정하지 않았다.**
   특히 실제 data directory가 `/var/lib/etcd`가 아닐 수 있다. membership, workload 이관, 검증된 snapshot이 준비되기 전에는 Worker 절차를 적용하거나 Box role을 변경하지 않는다.
