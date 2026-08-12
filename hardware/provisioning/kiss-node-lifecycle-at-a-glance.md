# KISS 노드 프로비저닝·클러스터 Join 한눈에 보기

> 범위: Box 등록·수정 → PXE/OS 설치 → Commissioning → Kubernetes Join → 검증
>
> 대상: 신규 장비 · OS 재설치 장비 · 기존 클러스터 이동 장비
>
> 기록: 2026-07-13 PXE 복구 · 2026-07-24 EdgeX/TwinX/DataX Join · 2026-08-12 default 복귀

## 전체 흐름

```text
장비 정보·UUID 확인
        ↓
KISS Box 등록·수정
        ↓
┌──────────────────────┬──────────────────────┐
│ 신규 / OS 재설치    │ 기존 OS / 재조인     │
│ PXE boot             │ 기존 설정 backup    │
│ Ubuntu autoinstall   │ cluster·role 변경   │
│ UEFI disk boot       │ reboot               │
└──────────┬───────────┴──────────┬───────────┘
           └────────────┬─────────┘
                        ↓
Disconnected → Commissioning → Joining → Running
                        ↓
        Box + Node + 서비스 + 인증서 검증
```

## 두 작업 경로

| 구분 | 신규 장비·OS 재설치 | 기존 장비·클러스터 이동 |
| --- | --- | --- |
| 시작점 | 빈 장비·재설치 대상 | 기존 Ubuntu·Kubernetes 설정 |
| 부팅 | PXE | disk reboot |
| 핵심 작업 | autoinstall·UEFI·첫 disk boot | backup·Cluster/Role 변경·재조인 |
| 공통 종착점 | Commissioning → Joining → Running | Commissioning → Joining → Running |
| 핵심 위험 | HWE package·GRUB·stale UEFI | stale CA·kubelet PKI·CNI·network |

## 1. 작업 전 확인

### Box 필수값

| 항목 | 의미 |
| --- | --- |
| Name | 장비 UUID · Kubernetes Node 이름 |
| Alias | 운영자 식별 이름 |
| Cluster | `default`, `edgex`, `twinx`, `datax` |
| Role | `ControlPlane`, `Compute`, `Desktop` |
| PowerType | `Ipmi`, `IntelAMT`, 미지원 |
| PowerAddress | IPMI/AMT 주소 |
| PrimaryAddress | OS/KISS 통신 주소 |

### 안전 게이트

- UUID·Alias 일치
- 현재/목적 Cluster·Role 확인
- workload·PV·VolumeAttachment 확인
- kubelet·CNI·NetworkManager backup
- KISS active Job 부재
- Control Plane: etcd·static Pod·local data 확인
- 원격 전원 또는 현장 reboot 준비
- credential·key·인증서 본문의 Git 기록 금지

## 2-A. PXE 설치 경로

```text
PXE boot 지정
  → KISS kernel/initrd/cloud-config
  → Ubuntu autoinstall
  → storage·network·kernel
  → GRUB·UEFI entry
  → installer reboot
  → PXE override 해제
  → Ubuntu disk boot
  → SSH·KISS agent 응답
```

### 설치 완료 기준

- Subiquity 완료 reboot
- EFI partition·GRUB 파일 존재
- 새 `Ubuntu` UEFI entry
- UEFI entry GUID와 EFI PARTUUID 일치
- persistent PXE override 해제
- 설치된 OS의 SSH 응답
- PXE live 임시 계정의 target OS 잔존 없음

### 실제 E300 PXE 장애

| 증상 | 원인 | 처리 |
| --- | --- | --- |
| HWE package 없음 | HWE marker + 순간 offline 판정 | one-shot GA kernel · KISS template 개선 대상 |
| GRUB EFI 등록 실패 | 과거 partition의 stale NVRAM entry | PARTUUID 비교 · stale disk entry만 제거 |
| 설치 후 다시 PXE | persistent PXE override | override 해제 · 새 `Ubuntu` 선택 |
| 수동 target 조립 필요처럼 보임 | Subiquity 미완료 | 수동 완료 금지 · installer 전체 restart |

## 2-B. 기존 장비 재조인 경로

```text
source 설정 확인
  → root-only backup + checksum
  → kubelet 중지
  → stale kubelet/CA/CNI 분리
  → Box Cluster·Role 변경
  → reboot
  → Commissioning
  → Joining
  → 목적 클러스터 Node 등록
```

### backup 최소 세트

- `/etc/kubernetes`
- `/var/lib/kubelet/pki`
- `/etc/cni/net.d`
- `/etc/nginx`
- NetworkManager 설정
- `/etc/hosts`
- `MANIFEST.sha256`
- Control Plane: 실제 etcd data + snapshot + static manifests

### 인증정보 기준

- kubelet certificate의 대상 UUID
- 목적 클러스터 CA fingerprint
- 인증서 유효기간
- 다른 노드 PKI 복사 금지
- worker의 `admin.conf` 사용 금지
- `kubeadm reset -f` 기본 금지

## 3. KISS 상태 전이

```text
Disconnected → Commissioning → Joining → Running
```

| 상태 | 의미 | 확인 지점 |
| --- | --- | --- |
| `Disconnected` | OS/KISS 통신 대기 | 전원·boot mode·network |
| `Commissioning` | OS 기본 구성 | commission Job/Pod 로그 |
| `Joining` | Kubernetes 등록 | join Job/Pod·kubelet 로그 |
| `Running` | KISS workflow 완료 | Node·서비스·인증서 별도 확인 |
| `Failed` | retry 소진·단계 실패 | 최신 오류·Box 갱신 시각 |
| Job 없음 | 완료 후 TTL 삭제 가능 | Box + Node + kubelet 교차 확인 |

> 핵심: `Box Running ≠ Node Ready ≠ workload 정상`

## 4. Commissioning·Join 확인

```bash
UUID='<box-uuid>'

kubectl get box "$UUID" \
  -o custom-columns='ALIAS:.metadata.labels.dash\.ulagbulag\.io/alias,CLUSTER:.spec.group.clusterName,ROLE:.spec.group.role,STATE:.status.state,ADDRESS:.status.primaryAddress,UPDATED:.status.lastUpdated'

kubectl -n kiss get job,pod |
  grep -E "NAME|box-(commission|join)-$UUID" || true

kubectl -n kiss logs "job/box-commission-$UUID" --tail=120
kubectl -n kiss logs "job/box-join-$UUID" --tail=120

box-ssh '<node-alias>' \
  'systemctl is-active containerd kubelet'
```

### Join 추가 확인

| 대상 | 추가 확인 |
| --- | --- |
| EdgeX/TwinX | bootstrap ConfigMap·RBAC · `localhost:6443` nginx proxy |
| DataX | 유선 bond primary/active · stale Cilium/Multus |
| CNI 미배포 클러스터 | `NetworkPluginNotReady`·Node `NotReady` 허용 |
| default 복귀 | default CA·기존 Node UUID·CNI·VolumeAttachment |

## 5. 대표 장애

| 증상 | 원인 | 처리 방향 |
| --- | --- | --- |
| `x509: unknown authority` | 이전 클러스터 CA | stale 인증정보 분리 · 목적 CA |
| `Node ... not found` | API 도달·등록 전 | Join 로그 · 인증서 UUID |
| `localhost:6443 refused` | nginx proxy 없음 | manifest·upstream·port owner |
| `ca.crt already exists` | seed CA 잔존 | proxy 기동 후 seed 재분리 |
| `Port-10250 in use` | 기존 kubelet | kubelet 중지 후 preflight |
| `cluster-info NotFound` | bootstrap 정보 누락 | ConfigMap·signer 복구 |
| `kubeadm-config forbidden` | bootstrap RBAC 누락 | 최소 Role·RoleBinding |
| `No route to host` | Wi-Fi bond 경로 | 유선 primary·active 보정 |
| Box `Failed`, Job 없음 | retry 소진 | 원인 제거 → reboot → 새 Commissioning |

## 6. Control Plane → Compute 복귀

```text
실제 etcd 설정 확인
  → health
  → fresh snapshot
  → control-plane 전체 backup
  → source etcd stop + disable
  → source static manifest·PKI 격리
  → 동일 장비의 worker 설정 복원
  → source control-plane container 정지
  → nginx :6443 인계
  → kubelet 시작
```

### E300 완료 기준

- etcd `inactive / disabled`
- static manifest `nginx only`
- source control-plane container `0`
- `:6443` owner `nginx`
- 목적 kubelet CA
- 목적 Node `Ready`

## 7. 최종 완료 기준

### KISS

- 목적 Cluster·Role
- Box `Running`
- PrimaryAddress 정상
- Commission/Join 완료 또는 TTL 삭제

### Kubernetes

- 목적 API의 Node 존재
- 의도한 Ready 조건
- kubelet/containerd `active`
- 신규 x509 오류 없음
- CA·Node UUID 일치
- CNI 상태와 배포 정책 일치
- storage attachment 정상

### PXE 장비

- disk boot
- SSH 응답
- UEFI entry·PARTUUID 일치
- persistent PXE override 없음

## 중단 조건

- UUID 불일치
- backup·checksum 불명
- 목적 CA 불명
- certificate UUID 불일치
- active KISS Job과 수동 파일 작업 중복
- workload·local data 영향 불명
- PXE boot entry 삭제 대상 불명
- Control Plane etcd health·snapshot 실패
- 실제 etcd data directory 불명

## 실제 수행 결과

| 시점 | 작업 | 결과 |
| --- | --- | --- |
| 2026-07-13 | E300 PXE/UEFI 복구 | autoinstall · disk boot · Commissioning |
| 2026-07-24 | EdgeX/TwinX Control Plane 구성 | E300 2대 구성 |
| 2026-07-24 | EdgeX/TwinX/DataX Compute Join | Compute 10대 등록 |
| 2026-08-12 | 전체 노드 default 복귀 | Box 12대 Running · Node 12대 Ready |

## 상세 기록

- [E300 Ubuntu 24.04 PXE/UEFI 복구](supermicro-e300-ubuntu-24-04-pxe-uefi-recovery-2026-07-13.md)
- [EdgeX/TwinX Control Plane·Compute Join](../../kubernetes/cluster-lifecycle/edgex-twinx-kiss-cluster-join-recovery-2026-07-24.md)
- [DataX Worker Join·네트워크 복구](../../kubernetes/cluster-lifecycle/datax-worker-rejoin-recovery-2026-07-24.md)
- [EdgeX/TwinX/DataX → default 복귀](../../kubernetes/cluster-lifecycle/kiss-default-cluster-return-2026-08-12.md)
