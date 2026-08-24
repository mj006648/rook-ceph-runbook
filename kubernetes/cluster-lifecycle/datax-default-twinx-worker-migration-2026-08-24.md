# DataX Gen12 Compute의 default 경유 TwinX 재조인 2026-08-24

> 작성일: 2026-08-24
>
> 상태: 단일 Gen12 Compute를 `datax`에서 안전하게 분리하고 `default` 복귀를 검증한 뒤 `twinx / Compute`로 재조인 완료. 최종 Box `Running`, TwinX Node `Ready`
>
> 선행 기록: [KISS EdgeX/TwinX/DataX 노드 default 클러스터 복귀](kiss-default-cluster-return-2026-08-12.md), [EdgeX/TwinX KISS 클러스터 구성 및 Compute 재조인 복구](edgex-twinx-kiss-cluster-join-recovery-2026-07-24.md), [KISS Node Lifecycle 한눈에 보기](../../hardware/provisioning/kiss-node-lifecycle-at-a-glance.md)

## 결과 요약

작업 범위는 다음 UUID 한 대뿐이었다.

| 항목 | 작업 전 | 중간 검증 | 최종 |
| --- | --- | --- | --- |
| UUID | `3adacb41-faad-9484-7477-1c697ad8c17d` | 동일 | 동일 |
| Alias | `datax-desktop-gen12-04` | `twinx-desktop-gen12-05` | `twinx-desktop-gen12-05` |
| Box cluster | `datax` | `default` | `twinx` |
| Role | `Compute` | `Compute` | `Compute` |
| Box state | `Running` | `Running` | `Running` |
| Kubernetes Node | DataX `Ready` | default `Ready` | TwinX `Ready` |

최종 완료 근거:

- Box alias `twinx-desktop-gen12-05`
- Box `spec=twinx`, `bind=twinx`, `state=Running`
- `containerd`, `kubelet` 모두 `active`
- TwinX API의 동일 UUID Node `Ready=True`
- kubelet client certificate CN이 동일 UUID
- CA 검증을 켠 `https://localhost:6443/` 응답 `403`
- nginx upstream `10.32.47.196:6443`
- 최근 5분 `x509`, `unknown authority`, `unauthorized`, `forbidden`, `connection refused` 0건
- kubelet health endpoint 응답 `ok`

이번 작업에서는 다음을 하지 않았다.

- 대상 외 Box 또는 Node 변경
- `kubeadm reset -f`
- 다른 노드의 kubelet PKI 복사
- KISS CronJob schedule 변경 또는 수동 Job 생성
- 기존 cluster의 stale Node 오브젝트 삭제
- backup 삭제
- 인증서 본문, private key, 비밀번호, bootstrap token의 Git 기록

## 작업 경로와 안전 원칙

작업은 로컬 `scalex` tmux에서 관리 호스트의 `chang` tmux에 접속해 수행했다. 접속 endpoint와 credential은 이 문서에 기록하지 않는다.

모든 worker 명령 앞에 다음 UUID guard를 사용했다.

```bash
test "$(hostname)" = "3adacb41-faad-9484-7477-1c697ad8c17d"
```

tmux에는 implicit buffer를 사용하지 않았다. 새 named buffer에 명령을 넣고 내용을 확인한 뒤 composer를 비우고 paste/submit했다.

## 실제 backup과 rollback anchor

| 단계 | 경로 | 내용 |
| --- | --- | --- |
| DataX 이탈 전 | `/root/datax-pre-default-20260824-025601` | DataX live Kubernetes, kubelet PKI, CNI, nginx, NetworkManager, hosts |
| default 원본 | `/root/edgex-compute-prejoin-20260724-061630/snapshot` | 같은 UUID의 기존 default worker 원본 |
| TwinX 이탈 전 default | `/root/default-pre-twinx-20260824-043353` | 정상 default live 상태와 이후 `stale-live` 격리본 |

default 원본 경로에 `edgex`가 들어간 이유는 같은 UUID가 7월 작업 당시 `edgex-desktop-gen12-04` alias로 사용됐기 때문이다. backup 선택 기준은 경로명의 alias가 아니라 client certificate의 Node UUID와 default CA fingerprint다.

새 backup은 root-only 디렉터리로 만들고 `MANIFEST.sha256`을 생성한 뒤 `sha256sum -c`로 검증했다. TwinX 전환 전에 live에서 이동한 stale 항목에는 별도 `STALE-LIVE-MANIFEST.sha256`을 생성했다.

backup에는 인증서와 key가 포함될 수 있다. Git이나 일반 사용자 경로로 복사하지 않고 삭제도 별도 승인 작업으로 남긴다.

## 작업 전 DataX 상태

Box와 DataX Node에서 확인한 기준값:

- Box `spec=datax`, `bind=datax`, `state=Running`
- DataX Node `Ready`
- source control plane alias `datax-control-plane-supermicro-01`
- 일반 system ReplicaSet Pod 10개
- VolumeAttachment 0개
- kubelet과 containerd `active`
- nginx upstream `10.32.119.93:6443`
- CNI `05-cilium.conflist`
- bond primary/active path `enp109s0`

source drain 전 workload와 VolumeAttachment를 확인했고, 정확한 UUID만 cordon/drain했다. drain 뒤에는 DaemonSet과 Node-owned nginx proxy만 남았다.

## 전체 시간선

시간은 관리 호스트와 Kubernetes event의 UTC 기준이다.

| 시각 | 상태와 작업 |
| --- | --- |
| `02:56` 전후 | DataX Node cordon/drain, DataX live backup 완료 |
| `02:59` | 사용자가 Box를 `default / Compute`로 변경 |
| `03:00` | hourly `box-reset` 시작 |
| `03:04:47` | default 전환용 `box-reset` 성공 완료 |
| `03:09:45` | 자동 reboot가 없음을 확인하고 같은 UUID를 수동 reboot |
| `03:12:12` | default `Joining`, 동일 UUID Node 생성 및 `Ready` 확인 |
| `03:19:21` | Box `default/default/Running` |
| `04:30` | default Node의 일반 Pod 0개, VolumeAttachment 0개 확인 후 cordon/drain |
| `04:33:53` | 정상 default backup 생성, kubelet 중지 |
| `04:35:36` | 사용자가 Box를 `twinx / Compute`로 변경 |
| `04:38` | stale default kubelet/CA/CNI 분리 완료 |
| `05:00` | hourly TwinX 전환용 `box-reset` 시작 |
| `05:05` 전후 | `box-reset` 성공 완료 |
| `05:24:09` | 자동 reboot가 없음을 확인하고 같은 UUID를 수동 reboot |
| `05:24:38` | 새 boot 시작 |
| `05:26:01` | Box `twinx/twinx/Joining`, `box-join` 시작 |
| `05:32:01` | Box `twinx/twinx/Running` |
| `05:32` 전후 | `box-join` 성공 완료 후 TTL 정리 |
| `05:53` | TwinX Node `Ready=True`, 최근 인증/proxy 오류 0건 최종 확인 |

## 1. DataX에서 안전 분리

### 사전 게이트

```bash
UUID='3adacb41-faad-9484-7477-1c697ad8c17d'

kubectl get node "$UUID" -o wide
kubectl get pods -A --field-selector "spec.nodeName=$UUID" -o wide
kubectl get volumeattachments.storage.k8s.io \
  -o custom-columns='NAME:.metadata.name,NODE:.spec.nodeName,ATTACHED:.status.attached,PV:.spec.source.persistentVolumeName'
kubectl -n kiss get job,pod --no-headers | grep "$UUID" || true
```

active KISS Job, 미확인 workload, VolumeAttachment가 있으면 중단한다.

### cordon과 drain

```bash
kubectl cordon "$UUID"

kubectl drain "$UUID" \
  --ignore-daemonsets \
  --delete-emptydir-data \
  --timeout=5m \
  --dry-run=server

kubectl drain "$UUID" \
  --ignore-daemonsets \
  --delete-emptydir-data \
  --timeout=5m
```

dry-run이 성공한 뒤에만 실제 drain을 수행한다.

### DataX live backup

최소 backup 대상:

- `/etc/kubernetes`
- `/var/lib/kubelet/pki`
- `/etc/cni/net.d`
- `/etc/nginx`
- `/etc/NetworkManager/system-connections`
- `/etc/hosts`

kubelet을 중지하고 backup checksum을 검증한 뒤에만 live 설정을 교체한다.

## 2. 같은 UUID의 default 원본 복원

이번 노드에는 같은 UUID의 default 원본이 남아 있었다.

```text
/root/edgex-compute-prejoin-20260724-061630/snapshot
```

복원 전에 다음을 확인했다.

- default CA fingerprint
- client certificate CN의 동일 UUID
- 인증서 유효기간
- kubelet endpoint `https://localhost:6443`
- default nginx upstream 3개
- default CNI 파일

확인한 default CA SHA-256 fingerprint:

```text
AB:93:F6:25:CF:4A:72:97:BC:1A:1A:F6:AA:14:14:3D:6E:10:5B:1D:69:9C:66:BB:72:2D:DA:07:2C:82:0A:9F
```

DataX live 파일은 삭제하지 않고 DataX backup의 `retired-live` 아래로 이동했다. 그다음 같은 노드의 default snapshot에서 Kubernetes, kubelet PKI, CNI, nginx를 복원했다.

Box를 `default / Compute`로 변경한 직후 kubelet 로그에 다음 오류가 잠시 나타났다.

```text
x509: certificate signed by unknown authority
```

이 시점에는 `box-reset`이 진행 중이고 default Node가 아직 생성되지 않았다. 단순 reboot나 CA 덮어쓰기를 반복하지 않고 다음을 교차 확인했다.

1. `box-reset` 성공 완료
2. 실제 boot time이 그대로여서 자동 reboot가 없었음
3. 수동 reboot 뒤 `Commissioning -> Joining -> Running`
4. default Node `Ready`
5. CA 검증을 켠 API 응답 `403`
6. 수렴 뒤 신규 인증 오류 0건

## 3. hourly KISS reset과 reboot 순서

이 환경의 per-Box reset CronJob은 다음 schedule이었다.

```text
@hourly
```

Box edit 직후 Job이 즉시 생성되지 않을 수 있다. 새 CronJob의 첫 실행은 다음 정시였다.

```bash
kubectl -n kiss get cronjob "box-reset-$UUID" \
  -o custom-columns='SCHEDULE:.spec.schedule,LAST:.status.lastScheduleTime,LAST_SUCCESS:.status.lastSuccessfulTime,ACTIVE:.status.active[*].name'
```

안전한 순서:

```text
Box edit
  -> 다음 hourly box-reset 시작
  -> Job Complete/event 확인
  -> 실제 boot time 확인
  -> 자동 reboot가 없을 때만 수동 reboot
  -> Commissioning
  -> Joining
  -> Running
```

reset 완료 전에 reboot하면 아직 default를 향하는 nginx proxy가 먼저 사라질 수 있다. Job이 TTL로 삭제됐으면 `NotFound`만 보고 실패로 판단하지 않고 CronJob `lastSuccessfulTime`, Kubernetes event, Box 상태를 함께 확인한다.

수동 reboot는 hostname guard 뒤에 수행했다.

```bash
test "$(hostname)" = "3adacb41-faad-9484-7477-1c697ad8c17d" &&
  sudo systemctl reboot
```

## 4. default에서 TwinX로 이동하기 전 게이트

default 복귀 성공을 확인한 뒤 곧바로 Box만 바꾸지 않는다. default Node가 다시 schedulable해졌을 수 있으므로 workload를 재감사한다.

이번 작업의 TwinX 변경 직전 상태:

```text
Box              = default/default/Running
Node Ready       = True
Unschedulable    = false
일반 Pod         = 0
VolumeAttachment = 0
```

정확한 UUID만 다시 cordon/drain한 뒤 정상 default 상태를 backup했다.

```text
/root/default-pre-twinx-20260824-043353
```

backup 전에 확인한 값:

- default CA fingerprint 일치
- client CN이 동일 UUID
- CA 검증 API 응답 `403`
- join 수렴 뒤 인증 오류 0건

## 5. stale default kubelet/CA/CNI 분리

Box를 `twinx / Compute`로 바꾼 뒤 active KISS Job이 없을 때 다음 live 상태를 분리했다.

- `/etc/kubernetes/kubelet.conf`
- `/var/lib/kubelet/pki`
- `/etc/kubernetes/ssl/ca.crt`
- `/etc/cni/net.d`

분리본은 다음 아래에 보존했다.

```text
/root/default-pre-twinx-20260824-043353/stale-live
```

live CNI 디렉터리는 빈 상태로 다시 만들었다.

```bash
sudo install -d -m 755 /etc/cni/net.d
```

### CA path alias 주의

이 노드에서는 다음 두 경로가 별도 CA 파일이 아니었다.

```text
/etc/kubernetes/pki -> /etc/kubernetes/ssl
/etc/kubernetes/pki/ca.crt == /etc/kubernetes/ssl/ca.crt
```

`ssl/ca.crt`를 먼저 이동한 뒤 `pki/ca.crt`를 다시 이동하려 하자 `No such file or directory`로 명령이 중단됐다. 이 실패는 추가 파일을 손상시키지 않았다. `readlink -f`로 두 경로가 같음을 확인한 뒤 남은 CNI 이동과 manifest 생성을 계속했다.

일반화된 안전 패턴:

```bash
SSL_CA=$(readlink -f /etc/kubernetes/ssl/ca.crt)
PKI_CA=$(readlink -f /etc/kubernetes/pki/ca.crt)

if [ "$SSL_CA" = "$PKI_CA" ]; then
  echo 'single CA target'
fi
```

경로별 존재 여부만 보고 두 번 이동하지 않는다. 실제 resolved target을 먼저 비교한다.

분리 완료 조건:

```text
kubelet                   = inactive
kubelet.conf              = absent
kubelet PKI               = absent
default CA                = absent
live CNI entry            = 0
localhost:6443 pre-reboot = listening
```

nginx 설정, containerd, NetworkManager는 이 단계에서 변경하지 않았다.

## 6. TwinX reset, reboot, join

`05:00` hourly reset Job은 성공 완료됐지만 실제 boot time은 이전 reboot 시각 그대로였다. 따라서 reset 완료를 확인한 뒤 같은 UUID만 수동 reboot했다.

reboot 뒤 관측한 상태 전이:

```text
Disconnected
  -> Joining
  -> Running
```

`box-join` 로그에서 대상 task는 `ok` 또는 `changed`였고 다음 오류는 없었다.

- `x509: unknown authority`
- `connection refused`
- `FileAvailable--...ca.crt`
- `Port-10250 in use`
- `forbidden`
- `unreachable`

Join Job은 성공 뒤 TTL로 삭제됐다. Kubernetes event에서 `Job completed`를 확인하고 Box 상태를 다시 조회했다.

## 7. 최종 TwinX 검증

### Box

```text
alias = twinx-desktop-gen12-05
spec  = twinx
bind  = twinx
state = Running
```

### 서비스와 Node

```text
containerd     = active
kubelet        = active
TwinX Node API = HTTP 200
Node Ready     = True
kubelet health = ok
```

worker에 `kubectl`이 없었으므로 kubelet client certificate로 자기 Node API를 직접 조회했다.

```bash
UUID='3adacb41-faad-9484-7477-1c697ad8c17d'

sudo curl --cacert /etc/kubernetes/ssl/ca.crt \
  --cert /var/lib/kubelet/pki/kubelet-client-current.pem \
  --key /var/lib/kubelet/pki/kubelet-client-current.pem \
  "https://localhost:6443/api/v1/nodes/$UUID"

curl -sS http://127.0.0.1:10248/healthz
```

응답의 Node condition에서 `type=Ready`, `status=True`를 확인했다.

### CA와 kubelet client

최종 TwinX CA SHA-256 fingerprint:

```text
0D:7A:2A:8B:3D:29:C2:91:10:05:66:70:DB:12:BC:60:C3:17:0E:87:90:FE:55:72:B3:5D:B8:09:FC:28:C4:E6
```

최종 kubelet client:

```text
subject   = O = system:nodes, CN = system:node:3adacb41-faad-9484-7477-1c697ad8c17d
notBefore = 2026-08-24 05:26:35 GMT
notAfter  = 2027-08-24 05:26:35 GMT
```

CA 검증을 켠 API root 요청은 `403`이었다. 이는 anonymous root 요청의 권한은 거부됐지만 TLS와 target API 경로는 정상임을 뜻한다.

### nginx와 CNI

```text
nginx upstream = 10.32.47.196:6443
CNI            = 05-cilium.conflist
lock           = .cni-concurrency.lock
```

Join 직후 kubelet 로그 필터에는 초기 수렴 과정의 인증/proxy 관련 항목이 63건 남아 있었다. 과거 누적값만으로 실패를 판단하지 않고 최근 시간창을 다시 확인했다.

```text
최근 3분 오류 = 0
최근 5분 오류 = 0
```

## Rollback

TwinX join이 실패하고 원인이 제거되지 않으면 다음 순서를 따른다.

1. active KISS Job이 없는지 확인한다.
2. kubelet을 중지한다.
3. 실패한 TwinX live 파일을 새 root-only hold 아래로 이동한다.
4. `/root/default-pre-twinx-20260824-043353/MANIFEST.sha256`을 검증한다.
5. 같은 backup의 default Kubernetes, kubelet PKI, CNI, nginx를 원래 위치로 복원한다.
6. client CN, default CA fingerprint, nginx upstream을 재검증한다.
7. Box rollback 방향을 사용자와 확인한 뒤에만 kubelet을 시작한다.

DataX까지 되돌려야 하면 `/root/datax-pre-default-20260824-025601`을 별도 rollback anchor로 사용한다. default rollback과 DataX rollback을 한 번에 섞지 않는다.

## 재발 방지 체크리스트

- [ ] 대상 UUID를 hostname guard와 Box 조회 양쪽에서 확인했다.
- [ ] source Node를 cordon/drain했다.
- [ ] 일반 Pod와 VolumeAttachment가 0인지 확인했다.
- [ ] root-only backup과 checksum을 확보했다.
- [ ] active KISS Job과 수동 파일 작업을 겹치지 않았다.
- [ ] `readlink -f`로 CA alias를 확인했다.
- [ ] 이전 kubelet.conf, PKI, CA, CNI를 live 경로에서 분리했다.
- [ ] KISS reset schedule과 마지막 성공 시각을 확인했다.
- [ ] reset 완료 전 reboot하지 않았다.
- [ ] 자동 reboot 여부를 boot time으로 확인했다.
- [ ] Box `Running`뿐 아니라 Node `Ready=True`를 확인했다.
- [ ] target CA, client UUID, nginx upstream, CNI를 확인했다.
- [ ] 최근 시간창의 신규 인증/proxy 오류가 0인지 확인했다.
- [ ] rollback backup을 삭제하지 않았다.

## 남은 위험과 후속 작업

- source DataX와 default API의 stale Node 오브젝트 정리는 별도 승인 작업이다.
- backup retention과 삭제는 인증정보 보존 정책을 확인한 뒤 별도 수행한다.
- TwinX control-plane bootstrap ConfigMap/RBAC이 다시 사라지면 선행 2026-07-24 런북의 control-plane 복구 절차를 적용한다.
- hourly reset 정책은 운영 의도에 따른 설정이다. 일회성 작업을 위해 schedule을 임의로 바꾸거나 manual Job을 만들지 않는다.
