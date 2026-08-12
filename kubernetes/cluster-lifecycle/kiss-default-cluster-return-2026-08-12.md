# KISS EdgeX/TwinX/DataX 노드 default 클러스터 복귀 2026-08-12

> 작성일: 2026-08-12
>
> 상태: Compute 10대와 기존 E300 Control Plane 2대의 `default / Compute` 복귀 완료. Box 12대 `Running`, default Kubernetes Node 12대 `Ready`
>
> 선행 기록: [EdgeX/TwinX KISS 클러스터 구성 및 Compute 재조인 복구](edgex-twinx-kiss-cluster-join-recovery-2026-07-24.md), [DataX Worker 재조인 복구 및 네트워크 안정화](datax-worker-rejoin-recovery-2026-07-24.md)

## 결과 요약

7월에 `default`에서 EdgeX/TwinX/DataX로 이동했던 Gen12 Compute와 EdgeX/TwinX E300 Control Plane을 다시 `default / Compute`로 복귀시켰다.

| 구분 | 대상 수 | 최종 Box | 최종 default Node |
| --- | ---: | --- | --- |
| EdgeX Gen12 Compute | 4 | `default / Compute / Running` | 4대 `Ready` |
| TwinX Gen12 Compute | 4 | `default / Compute / Running` | 4대 `Ready` |
| DataX Gen12 Compute | 2 | `default / Compute / Running` | 2대 `Ready` |
| EdgeX/TwinX E300 | 2 | `default / Compute / Running` | 2대 `Ready` |
| 합계 | 12 | 12대 `Running` | 12대 `Ready` |

성공 기준은 다음을 모두 만족하는 것이다.

1. KISS Box가 `default / Compute / Running`이다.
2. default API의 같은 UUID Node가 `Ready`이다.
3. kubelet과 containerd가 `active`이고 kubelet 로그에 신규 `x509` 오류가 없다.
4. kubelet 인증서의 Node UUID와 Box UUID가 일치하고 CA가 default CA와 일치한다.
5. CNI 설정과 nginx API proxy가 default 백업본으로 복구됐다.
6. E300은 이전 EdgeX/TwinX control-plane의 etcd와 static Pod가 재기동되지 않는다.

이번 작업에서는 다음을 하지 않았다.

- 기존 default Control Plane의 설정·서비스 변경
- `kubeadm reset -f`
- 이전 EdgeX/TwinX/DataX API의 stale Node 오브젝트 삭제
- E300의 `/opt/etcd` 데이터 또는 백업 삭제
- 백업 디렉터리 삭제
- `vine-session` Pod 강제 삭제 또는 재시작

접속 정보, 비밀번호, private key, kubeconfig·인증서 본문, bootstrap token은 Git에 기록하지 않는다.

## 작업 대상과 실제 백업

### EdgeX Compute

| UUID | Alias | 7월 default 원본 | 8월 live 격리본 |
| --- | --- | --- | --- |
| `d739a00e-2328-a790-8928-1c697ad987c0` | `edgex-desktop-gen12-01` | `/root/edgex-compute-prejoin-20260724-061758` | `/root/default-return-hold-20260812-062448` |
| `64348c94-d302-c07a-bce5-1c697ad99e5b` | `edgex-desktop-gen12-02` | `/root/edgex-compute-prejoin-20260724-061845` | `/root/default-return-hold-20260812-065006` |
| `da454806-2e36-f088-f10e-1c697ad8c03f` | `edgex-desktop-gen12-03` | `/root/edgex-compute-prejoin-20260724-061949` | `/root/default-return-hold-20260812-063312` |
| `3adacb41-faad-9484-7477-1c697ad8c17d` | `edgex-desktop-gen12-04` | `/root/edgex-compute-prejoin-20260724-061630` | `/root/default-return-hold-20260812-065941` |

### TwinX Compute

| UUID | Alias | 7월 default 원본 | 8월 live 격리본 |
| --- | --- | --- | --- |
| `02ca34fa-a216-9a2b-2a0a-1c697ad8c177` | `twinx-desktop-gen12-01` | `/root/twinx-compute-prejoin-20260724-062739` | `/root/default-return-hold-20260812-070624` |
| `9bc3697d-072e-dc48-29fb-1c697ad99e51` | `twinx-desktop-gen12-02` | `/root/twinx-compute-prejoin-20260724-062839` | `/root/default-return-hold-20260812-070710` |
| `9bcc45d1-c926-ab80-b3c0-1c697ad99d94` | `twinx-desktop-gen12-03` | `/root/twinx-compute-prejoin-20260724-062935` | `/root/default-return-hold-20260812-070755` |
| `a924e3df-003a-b402-91c6-1c697ad8c15e` | `twinx-desktop-gen12-04` | `/root/twinx-compute-prejoin-20260724-063046` | `/root/default-return-hold-20260812-070841` |

### DataX Compute

| UUID | Alias | 7월 원본 | 8월 live 격리본 |
| --- | --- | --- | --- |
| `fc1258ba-b25c-2724-d6d7-1c697ad99de0` | `datax-desktop-gen12-01` | kubelet `/root/kubelet-before-datax-20260724-023912`, CNI `/root/cni-before-datax-cleanup-20260724-035256` | `/root/default-return-hold-20260812-071801` |
| `fe47eb1b-cf4c-f3fb-5057-1c697ad99f56` | `datax-desktop-gen12-02` | kubelet `/root/kubelet-before-datax-20260724-041905`, CNI `/root/cni-before-datax-cleanup-20260724-043516` | `/root/default-return-hold-20260812-071944` |

`datax-desktop-gen12-01`에는 7월의 단일 full manifest가 없었다. 따라서 교체 전에 다음 증거를 각각 확인했다.

- default kubelet/CA 파일의 SHA-256
- kubelet client certificate의 subject, 만료일, Node UUID
- default API 응답
- CNI 파일의 정확한 SHA-256
- 현재 live/source 파일의 별도 manifest

하나라도 식별이 불가능하거나 UUID가 다르면 복원을 중단해야 한다. 다른 노드의 kubelet PKI를 복사해서는 안 된다.

### EdgeX/TwinX E300

| UUID | Alias | 7월 default 원본 | 8월 최종 control-plane backup |
| --- | --- | --- | --- |
| `00000000-0000-0000-0000-0cc47a9f8416` | `edgex-control-plane-e300-01` | `/root/control-plane-before-repurpose-20260724-135152` | `/root/edgex-final-control-plane-backup-20260812-170040` |
| `00000000-0000-0000-0000-ac1f6be5de08` | `twinx-control-plane-e300-01` | `/root/control-plane-before-repurpose-20260724-134956` | `/root/twinx-final-control-plane-backup-20260812-171120` |

최종 etcd snapshot 확인값:

| 대상 | revision | key count | snapshot 크기 |
| --- | ---: | ---: | ---: |
| EdgeX E300 | `3632659` | `1447` | 약 `8.3 MB` |
| TwinX E300 | `3635447` | `1667` | 약 `9.1 MB` |

백업은 인증서와 key를 포함할 수 있으므로 root-only 권한을 유지하고 외부나 Git으로 복사하지 않는다.

## 확인된 실패 원인

### 1. KISS `Running`만으로 Kubernetes 복귀 성공을 판단할 수 없음

관측된 상태 전이는 다음과 같았다.

```text
Disconnected -> Commissioning -> Joining -> Running
```

완료된 Join Job은 TTL로 삭제될 수 있다. 그러나 E300은 Box가 `Running`이 된 뒤에도 default Node가 `NotReady`였고, live 경로에는 이전 source 클러스터의 CA, static manifest, etcd 설정이 남아 있었다.

따라서 Box 상태와 별개로 Node Ready, kubelet CA/UUID, systemd, static Pod, listening port를 검증해야 한다.

### 2. E300의 실제 etcd data directory가 `/var/lib/etcd`가 아니었음

`/var/lib/etcd`가 없다는 이유로 etcd 데이터가 없다고 판단하면 안 된다. 실제 환경은 다음과 같았다.

```text
ETCD_DATA_DIR=/opt/etcd
ETCD_NAME=etcd1
```

반드시 service unit과 environment를 읽어 실제 data directory와 인증서 경로를 찾고 그 값으로 snapshot을 만든다.

### 3. `/etc/hosts`의 localhost 누락

E300에서 `localhost`가 DNS `10.64.0.3`으로 전달돼 다음 오류가 발생했다.

```text
lookup localhost on 10.64.0.3:53
```

nginx와 kubelet이 사용하는 `localhost:6443` 경로를 복구하려면 `/etc/hosts`에 loopback 매핑이 있어야 한다.

### 4. 이전 kube-apiserver 프로세스가 6443을 점유

source control-plane container가 남아 있으면 nginx가 다음 오류로 기동하지 못한다.

```text
bind() to 0.0.0.0:6443 failed (98: Address already in use)
```

E300을 worker로 돌릴 때는 source kube-apiserver/etcd를 확실히 중지하고, 6443의 owner가 nginx인지 확인해야 한다.

### 5. Control Plane은 단순 worker 인증서 교체 대상이 아님

E300에는 kubelet 인증정보 외에 source etcd, control-plane static manifests, 인증서, container가 함께 있었다. Compute처럼 `kubelet.conf`와 CNI만 바꾸면 split-brain 또는 이전 API 재기동 위험이 있다.

## 공통 사전 게이트

### KISS 작업 중복 방지

관리 클러스터에서 대상별로 확인한다.

```bash
UUID='<box-uuid>'

kubectl get box "$UUID" \
  -o custom-columns='ALIAS:.metadata.labels.dash\.ulagbulag\.io/alias,SPEC_CLUSTER:.spec.group.clusterName,SPEC_ROLE:.spec.group.role,BIND_CLUSTER:.status.bindGroup.clusterName,BIND_ROLE:.status.bindGroup.role,STATE:.status.state,UPDATED:.status.lastUpdated'

kubectl -n kiss get job,pod |
  grep -E "NAME|box-(commission|join)-$UUID" || true
```

Commission/Join Job이 파일을 수정하는 동안 같은 live 파일을 이동하지 않는다. Job 완료 또는 TTL 삭제, Box 상태 전환을 확인한 뒤 진행한다.

### default 기준값 확보

민감한 내용은 출력하지 않고 fingerprint, subject, 날짜, HTTP status만 확인한다.

```bash
sudo openssl x509 -in /etc/kubernetes/ssl/ca.crt \
  -noout -subject -fingerprint -sha256

sudo awk '/client-certificate-data:/{print $2; exit}' \
  /etc/kubernetes/kubelet.conf |
  base64 -d | openssl x509 -noout -subject -dates

curl -sk -o /dev/null -w '%{http_code}\n' https://localhost:6443/
```

이번 작업에서 확인한 default CA SHA-256 fingerprint는 다음과 같다.

```text
AB:93:F6:25:CF:4A:72:97:BC:1A:1A:F6:AA:14:14:3D:6E:10:5B:1D:69:9C:66:BB:72:2D:DA:07:2C:82:0A:9F
```

## Compute 10대 복귀 절차

### 1. 원본과 live 상태 대조

노드마다 다음을 비교한다.

- 원본 backup의 `kubelet.conf`, kubelet PKI, CA, CNI 존재 여부
- manifest가 있으면 `sha256sum -c`
- client certificate subject에 대상 UUID가 포함되는지
- 인증서 유효기간이 남았는지
- CA fingerprint가 default CA인지
- default API가 TLS 경로를 통해 응답하는지

### 2. 현재 source live 파일을 root-only backup으로 격리

```bash
TS=$(date +%Y%m%d-%H%M%S)
HOLD="/root/default-return-hold-$TS"
sudo install -d -m 700 "$HOLD"

sudo systemctl stop kubelet
sudo cp -a /etc/kubernetes "$HOLD/" 2>/dev/null || true
sudo cp -a /var/lib/kubelet/pki "$HOLD/kubelet-pki" 2>/dev/null || true
sudo cp -a /etc/cni/net.d "$HOLD/cni-net.d" 2>/dev/null || true
sudo cp -a /etc/nginx "$HOLD/nginx" 2>/dev/null || true

sudo find "$HOLD" -xdev -type f ! -name MANIFEST.sha256 \
  -exec sha256sum {} \; |
  sudo tee "$HOLD/MANIFEST.sha256" >/dev/null
sudo chmod -R go-rwx "$HOLD"
```

### 3. 해당 노드의 default 원본을 atomic restore

아래 `DEFAULT_BACKUP`은 반드시 같은 UUID의 표에 있는 경로를 사용한다. 실제 backup 구조를 `find`로 먼저 확인하고 경로를 맞춘다.

```bash
DEFAULT_BACKUP='<same-node-default-backup>'

# 예시: live 파일을 hold 아래로 이동한 뒤 같은 노드의 원본을 복원한다.
sudo install -d -m 755 /etc/kubernetes /etc/kubernetes/ssl
sudo install -d -m 700 /var/lib/kubelet/pki
sudo install -d -m 755 /etc/cni/net.d

# cp 명령은 각 backup의 실제 디렉터리 구조에 맞춰 실행한다.
# 다른 노드의 kubelet.conf 또는 PKI를 재사용하지 않는다.
```

복원 후 즉시 재확인한다.

```bash
sudo awk '/client-certificate-data:/{print $2; exit}' \
  /etc/kubernetes/kubelet.conf |
  base64 -d | openssl x509 -noout -subject -dates

sudo awk '/certificate-authority-data:/{print $2; exit}' \
  /etc/kubernetes/kubelet.conf |
  base64 -d | openssl x509 -noout -fingerprint -sha256
```

### 4. 서비스와 Node 검증

```bash
sudo systemctl restart containerd
sudo systemctl start kubelet
sudo systemctl is-active containerd kubelet

sudo journalctl -u kubelet --since '10 minutes ago' --no-pager |
  grep -Ei 'x509|certificate|unauthorized|forbidden|failed|error' |
  tail -n 100
```

관리 클러스터의 default kubeconfig로 다음을 확인한다.

```bash
kubectl get node '<node-uuid>' -o wide
kubectl get volumeattachment -o wide |
  grep '<node-uuid>' || true
```

각 desktop에서 Ceph `VolumeAttachment` 2개가 `ATTACHED=true`인 것도 확인했다.

노드마다 임의로 2분씩 기다리지는 않았다. 단, backup·UUID·CA·KISS active Job 확인과 파일 교체는 노드별로 순차 수행했고, KISS/Node 수렴은 여러 대를 묶어 관찰했다.

## E300 Control Plane 2대의 worker 복귀 절차

> 이 절은 기존 default Control Plane을 변경하는 절차가 아니다. 대상은 이전에 EdgeX/TwinX Control Plane으로 사용했다가 사용자가 `default / Compute`로 변경하고 재부팅한 E300 두 대뿐이다.

### 1. live control-plane 감사

```bash
sudo systemctl is-active kubelet containerd etcd nginx || true
sudo systemctl is-enabled etcd || true
sudo ss -lntp | grep -E ':(2379|2380|6443|10250)\b' || true
sudo find /etc/kubernetes/manifests -maxdepth 1 -type f -printf '%f\n' | sort
sudo ctr -n k8s.io containers list |
  grep -E 'kube-(apiserver|controller-manager|scheduler)|etcd' || true
sudo systemctl cat etcd
sudo systemctl show etcd -p Environment -p EnvironmentFiles
```

### 2. 실제 etcd 설정으로 health와 snapshot 확보

실제 `ETCD_DATA_DIR`, endpoint, CA, client certificate/key를 service environment에서 읽는다. 인증서나 key 본문은 출력하지 않는다.

```bash
sudo ETCDCTL_API=3 etcdctl \
  --endpoints='<actual-client-endpoint>' \
  --cacert='<actual-ca>' \
  --cert='<actual-client-cert>' \
  --key='<actual-client-key>' \
  endpoint health

sudo ETCDCTL_API=3 etcdctl \
  --endpoints='<actual-client-endpoint>' \
  --cacert='<actual-ca>' \
  --cert='<actual-client-cert>' \
  --key='<actual-client-key>' \
  snapshot save '<root-only-backup>/etcd-snapshot.db'

sudo ETCDCTL_API=3 etcdctl snapshot status \
  '<root-only-backup>/etcd-snapshot.db' -w table
```

### 3. 최종 root-only backup

최소 포함 항목:

- etcd systemd unit와 environment
- 실제 etcd data directory
- `/etc/kubernetes`
- kubelet PKI
- CNI 설정
- nginx 설정
- NetworkManager 설정
- `/etc/hosts`
- source Node/Pod 목록
- etcd snapshot과 status
- 전체 `MANIFEST.sha256`

```bash
BACKUP='/root/<cluster>-final-control-plane-backup-<timestamp>'
sudo install -d -m 700 "$BACKUP"
# 위 항목을 cp -a로 수집한 뒤:
sudo find "$BACKUP" -xdev -type f ! -name MANIFEST.sha256 \
  -exec sha256sum {} \; |
  sudo tee "$BACKUP/MANIFEST.sha256" >/dev/null
sudo chmod -R go-rwx "$BACKUP"
```

### 4. source control-plane 정지 후 default worker 원본 복원

순서가 중요하다.

1. kubelet 중지
2. source etcd 중지 및 disable
3. live `/etc/kubernetes`, kubelet PKI, CNI, nginx 등을 final backup 아래 `retired-live`로 이동
4. 같은 E300의 7월 default backup에서 worker kubelet/CA/CNI/nginx 설정 복원
5. `/etc/hosts`에 localhost가 없으면 loopback 매핑 추가
6. 남은 source kube-apiserver/controller/scheduler/etcd container 중지
7. containerd와 kubelet 시작

```bash
sudo systemctl stop kubelet
sudo systemctl disable --now etcd

# source control-plane container가 남았는지 확인하고 중지한다.
sudo ctr -n k8s.io containers list |
  grep -E 'kube-(apiserver|controller-manager|scheduler)|etcd' || true

# localhost가 없을 때만 추가한다.
grep -Eq '^[[:space:]]*127\.0\.0\.1[[:space:]]+.*\blocalhost\b' /etc/hosts ||
  printf '127.0.0.1 localhost\n' | sudo tee -a /etc/hosts >/dev/null

sudo systemctl restart containerd
sudo systemctl start kubelet
```

`etcd` 재시작은 이 복귀 절차의 일부가 아니다. source 클러스터 rollback이라는 별도 변경이며 split-brain 위험을 검토한 뒤에만 수행한다.

### 5. E300 전용 완료 검증

```bash
sudo systemctl is-active kubelet containerd nginx
sudo systemctl is-active etcd || true       # inactive 예상
sudo systemctl is-enabled etcd || true      # disabled 예상

sudo find /etc/kubernetes/manifests -maxdepth 1 -type f -printf '%f\n' | sort
# nginx static manifest만 있어야 한다.

sudo ctr -n k8s.io containers list |
  grep -E 'kube-(apiserver|controller-manager|scheduler)|etcd' || true
# 출력 0건 예상

sudo ss -lntp | grep ':6443'
# nginx가 소유해야 한다.

curl -sk -o /dev/null -w '%{http_code}\n' https://localhost:6443/
# 인증정보 없는 요청의 403은 TLS와 default API proxy 도달 성공으로 해석한다.
```

최종 E300 두 대에서 다음을 확인했다.

| 항목 | EdgeX E300 | TwinX E300 |
| --- | --- | --- |
| Box | `default / Compute / Running` | `default / Compute / Running` |
| default Node | `Ready` | `Ready` |
| kubelet/containerd | `active` | `active` |
| etcd | `inactive / disabled` | `inactive / disabled` |
| static manifest | nginx only | nginx only |
| source control-plane container | 0 | 0 |
| kubelet CA | default CA | default CA |
| system DaemonSet Pod | 정상 | 정상 |

## 최종 검증

### 12개 Node

```bash
kubectl get node \
  00000000-0000-0000-0000-0cc47a9f8416 \
  00000000-0000-0000-0000-ac1f6be5de08 \
  d739a00e-2328-a790-8928-1c697ad987c0 \
  64348c94-d302-c07a-bce5-1c697ad99e5b \
  da454806-2e36-f088-f10e-1c697ad8c03f \
  3adacb41-faad-9484-7477-1c697ad8c17d \
  02ca34fa-a216-9a2b-2a0a-1c697ad8c177 \
  9bc3697d-072e-dc48-29fb-1c697ad99e51 \
  9bcc45d1-c926-ab80-b3c0-1c697ad99d94 \
  a924e3df-003a-b402-91c6-1c697ad8c15e \
  fc1258ba-b25c-2724-d6d7-1c697ad99de0 \
  fe47eb1b-cf4c-f3fb-5057-1c697ad99f56 \
  -o wide
```

2026-08-12 최종 확인 시 12개 모두 `Ready`였다.

### Known issue: 일부 desktop의 vine-session

Node와 system 구성은 정상이나 다음 application Pod는 별도 장애가 남았다.

| 상태 | 노드 |
| --- | --- |
| `9/9` Healthy | TwinX 01, TwinX 03, EdgeX 01, EdgeX 02, EdgeX 03 |
| `8/9 Init:CrashLoopBackOff` | EdgeX 04, TwinX 02, TwinX 04, DataX 02 |
| `1/9 CrashLoopBackOff` | DataX 01 |

확인된 container 종료 이력에는 `picom` exit code `1`과 과거 `xorg`/`wireplumber` 종료가 포함됐다. 다음 근거 때문에 default 클러스터 재조인 실패와 분리해 application 후속 이슈로 남겼다.

- Node 12대 모두 `Ready`
- kubelet CA/UUID와 default API 통신 정상
- kube-system DaemonSet 정상
- desktop의 Ceph VolumeAttachment 정상

이번 작업에서는 사용자 세션 보존을 위해 Pod를 강제로 삭제하거나 재시작하지 않았다.

## Rollback 원칙

### Compute

복원 직후 default Node가 등록되지 않거나 인증정보가 잘못됐음이 확인되면:

1. kubelet 중지
2. 잘못 복원된 live 파일을 별도 실패 backup으로 이동
3. `default-return-hold-*`의 source live 파일을 원래 위치로 복구
4. KISS Box의 intended cluster/role과 rollback 방향 재확인
5. kubelet 시작 전 CA, UUID, nginx upstream 재검증

### E300

E300 rollback은 단순 worker rollback이 아니다. source control-plane 복구에는 etcd snapshot/data directory, static manifests, source CA, peer/member 상태가 함께 필요하다.

- final backup과 7월 default backup을 삭제하지 않는다.
- source etcd를 임의 재시작하지 않는다.
- source 클러스터가 실제로 폐기됐는지, 다른 member가 살아 있는지 먼저 확인한다.
- 승인된 rollback 계획 없이 `/opt/etcd`를 live로 되돌리지 않는다.

## 운영 주의사항

- Box `Running`은 필요조건이지 충분조건이 아니다.
- KISS Job이 active일 때 동일 파일을 수동 변경하지 않는다.
- 인증서 subject의 Node UUID가 대상과 다르면 즉시 중단한다.
- 다른 노드의 kubelet PKI를 복사하지 않는다.
- E300은 `/var/lib/etcd`뿐 아니라 service environment의 실제 data directory를 확인한다.
- E300 worker 전환 뒤에는 etcd `inactive/disabled`, static manifest nginx only, source control-plane container 0건을 확인한다.
- nginx 6443과 default API 응답을 함께 검증한다.
- 이전 source API의 stale Node 삭제는 별도 승인 작업으로 남긴다.
- backup 삭제는 retention과 rollback 가능성을 검토한 별도 승인 뒤에만 한다.
