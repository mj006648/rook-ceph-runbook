# TwinX Ceph 내부 풀 배치 복구 — 단일 SSD 실험 환경

> 작성일: 2026-09-10
>
> 상태: **선택 Sync 2건 성공, 대상 65 PG 복구 완료. 추가 내부 풀의 32 PG가 inactive로 남아 전체 정상화는 미완료.**
>
> 범위: 관리·RGW 로그 풀 3개의 가용성 복구. 원본 데이터 삭제, PV/PVC 변경, 복제 수 감소는 하지 않는다.

## Current status

| 항목 | 확인 결과 |
| --- | --- |
| 환경 | Rook 1.17.6 / Ceph 19.2.2, TwinX 실험 스토리지 |
| OSD 배치 | l40s 한 호스트, **같은 NVMe 한 개의 논리 장치 3개** |
| 문제 | 최초 대상 65 PG는 복구됨. 추가 `default.rgw.control`의 32 PG가 inactive |
| 원본 풀 | `trident-kci-rgw-data` 128 PG는 active+clean |
| 메타데이터 풀 | `trident-kci-rgw-meta` 8 PG는 active+clean |
| 적용 GitOps | 별도 수동 앱 `rook-ceph-pool-recovery`, 자동 Sync는 꺼진 상태 유지 |
| 로컬 검증 | 테스트 20개, Helm, API 서버 dry-run 통과 |
| 실제 적용 | 새 Application 1개와 대상 풀 CR 3개만 선택 Sync, 두 작업 Succeeded |
| 엄격한 사후 검증 | 추가 풀이 발견되어 실패. 예외를 무시하거나 baseline을 바꾸지 않음 |
| 물리 장애 보호 | 이번 변경으로 추가되지 않음 |

이 문서는 공개 운영 노트다. 인증값, kubeconfig, keyring, 인증서 본문, 장치 일련번호와 전체 클러스터 원시 덤프는 게시하지 않는다. 상세 내부 근거 링크는 아래에 별도로 구분했다.

## Symptom

Trident v3 입력 준비 파일럿의 사전 점검에서 다음 경고를 확인했다.

~~~text
PG_AVAILABILITY: 65 pgs inactive
PG_DEGRADED: 65 pgs undersized
MON_DISK_BIG
MON_DISK_LOW
~~~

Argo CD의 Application이 Healthy이거나 Rook CR이 Ready여도 Ceph의 모든 PG가 데이터를 제공할 수 있다는 뜻은 아니다. 데이터 가용성은 Ceph 상태를 별도로 확인해야 한다.

## Diagnosis

### 1. 풀의 복제 설정과 실제 배치 확인

기존 Rook toolbox에서 읽기 전용으로 확인한다.

~~~bash
kubectl -n rook-ceph exec deploy/rook-ceph-tools -- ceph health detail
kubectl -n rook-ceph exec deploy/rook-ceph-tools -- ceph osd tree
kubectl -n rook-ceph exec deploy/rook-ceph-tools -- ceph osd pool ls detail --format json \
  | jq '[.[] | select(.pool_name == ".mgr" or .pool_name == "default.rgw.log" or .pool_name == "trident-kci-store.rgw.log") | {pool_name, pool_id, size, min_size, crush_rule, pg_num}]'
kubectl -n rook-ceph exec deploy/rook-ceph-tools -- ceph osd crush rule dump --format json
~~~

적용 전 관측한 대상:

| 풀 | pool ID | PG 수 | size | min_size | 복구 전 failure domain |
| --- | ---: | ---: | ---: | ---: | --- |
| `.mgr` | 1 | 1 | 3 | 2 | host |
| `default.rgw.log` | 20 | 32 | 3 | 2 | host |
| `trident-kci-store.rgw.log` | 31 | 32 | 3 | 2 | host |

ID는 당시 TwinX의 값이다. 다른 클러스터에서 그대로 가정하지 말고 조회 결과를 사용한다. 풀이 비어 있다고 보고되어도 inactive PG의 통계를 근거로 삭제하지 않는다.

### 2. OSD 수와 물리 디스크 수를 구분

~~~bash
kubectl -n rook-ceph exec deploy/rook-ceph-tools -- ceph osd metadata --format json \
  | jq '[.[] | {id, hostname, devices, bluestore_bdev_dev_node}]'
~~~

이 환경에서는 세 OSD가 모두 같은 `devices` 값을 보고했고, 서로 다른 device-mapper 장치를 사용했다. OSD가 3개라고 해서 독립 디스크가 3개인 것은 아니다. 장치 일련번호는 운영 환경에서 교차 확인하되 공개 노트에 복사하지 않는다.

### 3. Monitor 경고와 원본 저장 공간을 구분

~~~bash
kubectl -n rook-ceph exec deploy/rook-ceph-tools -- ceph df detail
kubectl -n rook-ceph exec deploy/rook-ceph-tools -- ceph config get mon mon_data_avail_warn
kubectl -n rook-ceph get pods -l app=rook-ceph-mon -o wide
~~~

Monitor Pod의 실제 mon-data 마운트 경로를 확인한 뒤 해당 경로에 `df -h`를 실행한다. 원본 데이터 풀의 사용량과 Monitor 파일시스템 사용량을 혼동하지 않는다.

이전 점검에서 Ceph 전체 여유는 약 1.07 TB였고, 경고가 발생한 두 Monitor 파일시스템에도 각각 약 1.28 TB, 130 GB가 남아 있었다. `MON_DISK_LOW`는 기본적으로 여유 **비율 30%**를 기준으로 발생한다. 따라서 곧바로 원본 데이터를 삭제해야 한다는 의미가 아니다.

Monitor DB는 약 29 GiB로 크기 경고도 있었다. 오래 inactive인 PG와 DB 증가의 관련성을 검토할 수 있지만, 이 점검만으로 인과관계를 확정하지 않는다. DB 파일 삭제, 경고 임계값 완화, 무조건적인 compaction은 이번 복구에 포함하지 않는다.

## Root cause

문제의 풀들은 복제본을 서로 다른 **host**에 배치하려고 하지만 OSD가 있는 host는 하나뿐이다. `size=3`, `min_size=2` 조건을 충족하지 못해 PG가 활성화되지 못했다.

현재 CRUSH 맵의 복사본으로 4,096개 입력값을 시험한 결과:

- host 기준 규칙: 모든 입력이 OSD 1개에만 배치됨.
- 기존 osd 기준 규칙: 모든 입력이 논리 OSD 3개에 배치됨.

이는 배치 가능성에 대한 모의 검증이다. 실제 PG 복구나 디스크 장애 보호를 검증한 결과가 아니다.

## Fix

### 1. 단일 SSD 실험 환경이라는 한계를 먼저 확정

이번에는 장비 구성을 바꾸지 않고 **서비스 가용성만 복구**한다. 복제 수 3과 최소 복제 수 2는 유지하고, 대상 풀의 failure domain만 `osd`로 바꾼다.

물리 디스크·서버 장애 보호가 필요하면 독립된 장치/호스트를 확보하는 별도 작업이 필요하다. 이 절차를 고가용성 운영 환경의 일반 권장 구성으로 사용하지 않는다.

### 2. 기존 자동 Sync 앱과 분리

TwinX GitOps에 다음을 추가했다.

~~~text
argocd/twinx-storage/values.yaml
  applications.rook-ceph-pool-recovery

argocd/twinx-storage/apps/rook-ceph-pool-recovery/
  pools.yaml
  verify.py
  README.md
~~~

- 새 앱은 수동 Sync 전용이며 `pools.yaml`만 읽는다.
- 기존 `rook-ceph-resources` 앱과 원본/메타데이터 풀을 다루는 PreSync 훅은 변경하지 않는다.
- `FailOnSharedResource=true`를 사용한다.
- 기존 풀을 새 CephBlockPool CR로 관리하므로 삭제 생명주기에 특히 주의한다.

`.mgr`의 선언 예시는 다음과 같다. 다른 두 풀도 같은 복제·삭제 보호 설정을 사용하되 Kubernetes 이름을 기존 풀 이름으로 지정하고 `application: rgw`를 사용한다.

~~~yaml
apiVersion: ceph.rook.io/v1
kind: CephBlockPool
metadata:
  name: builtin-mgr
  namespace: rook-ceph
  annotations:
    argocd.argoproj.io/sync-options: Prune=false,Delete=false
spec:
  name: .mgr
  application: mgr
  failureDomain: osd
  enableCrushUpdates: true
  replicated:
    size: 3
    requireSafeReplicaSize: true
  parameters:
    min_size: "2"
~~~

`enableCrushUpdates`는 기존 풀의 CRUSH 규칙 변경을 허용한다. 기존 공용 `replicated_rule` 자체를 수정하지 않는다. 풀을 삭제·재생성하거나 `size=1`, `min_size=1`로 낮추는 방식은 사용하지 않는다.

### 3. 적용 전 상태 저장 후 선택적으로 Sync

GitOps 저장소 루트에서:

~~~bash
python3 argocd/twinx-storage/apps/rook-ceph-pool-recovery/verify.py capture \
  --output /path/to/ceph-pool-recovery-before.json
~~~

대상 ID·복제 설정·PG 수와 소유권이 준비 당시와 같은지 확인한다. 다르면 강제로 맞추지 말고 범위를 다시 검토한다.

Argo CD에서:

1. `twinx-storage-root-app`을 Refresh한다.
2. **새 `rook-ceph-pool-recovery` Application 리소스 하나만 선택 Sync**한다. 다른 자식 앱이나 bootstrap 전체를 같이 Sync하지 않는다.
3. 생성된 `rook-ceph-pool-recovery`에서 자동 Sync가 꺼져 있는지 확인한다.
4. diff가 대상 CephBlockPool 3개뿐인지 확인하고 일반 수동 Sync한다.
5. Force/Replace/삭제로 재시도하지 않는다. 기존 `rook-ceph-resources` 앱을 대신 Sync하지 않는다.

**2026-09-10에 위임받은 선택 Sync를 실행했다.** Git 커밋 `48f1fc3`에 고정하고 Prune/Force 없이 요청했으며, 실제 처리된 리소스가 새 Application 1개와 대상 CephBlockPool 3개뿐인지 확인했다.

## Verification

~~~bash
python3 argocd/twinx-storage/apps/rook-ceph-pool-recovery/verify.py verify \
  --before /path/to/ceph-pool-recovery-before.json \
  --output /path/to/ceph-pool-recovery-after.json
~~~

적용 후 다음을 확인한다.

- 클러스터 FSID/context와 기존 pool ID 유지.
- 대상 풀의 `size=3`, `min_size=2`, PG 수 유지 및 osd 기준 규칙 적용.
- 대상 65 PG가 `active+clean`, CephBlockPool 상태 Ready.
- 비대상 풀의 ID·size·min_size·crush_rule·pg_num에 예상 밖 변경이 없음.
- 남은 Ceph 경고를 별도로 기록.

검증 도구는 조회만 하며 관측 실패를 성공으로 처리하지 않는다. 원시 관측과 판정을 같은 산출물에 저장하고 기존 파일을 덮어쓰지 않는다. 적용 전 실제 검사에서는 아직 바뀌지 않은 PG·규칙·CR 상태에 대해 예상대로 실패 판정이 나왔다. 복구를 시도했다가 실패한 결과가 아니다.

풀 복구가 통과해도 Monitor 공간 경고 등으로 **파일럿 사전 점검은 여전히 보류될 수 있다**. 실제 성능 측정과 다른 결과를 구분한다.

### 2026-09-10 실제 실행 결과

| 풀 | ID | size / min_size | 적용 후 상태 |
| --- | ---: | --- | --- |
| `.mgr` | 1 | 3 / 2 | 1 PG active+clean, osd 규칙 |
| `default.rgw.log` | 20 | 3 / 2 | 32 PG active+clean, osd 규칙 |
| `trident-kci-store.rgw.log` | 31 | 3 / 2 | 32 PG active+clean, osd 규칙 |
| `default.rgw.control` — 추가 발견 | 32 | 3 / 2 | 32 PG undersized+peered/inactive, host 규칙 |

대상 3개의 pool ID·복제 설정·PG 수와 기존 비대상 풀의 비교 대상 설정은 유지됐다. 하지만 적용 전에는 없었던 `default.rgw.control` 풀이 나타나 엄격한 사후 검증은 실패했다. **Argo Sync 성공, 대상 PG 복구, 클러스터 전체 정상화는 서로 다른 결과**다.

추가 풀은 요청한 Argo 리소스에 없었다. Toolbox에는 17~19일째 남아 있는 RGW 관리 프로세스 8개가 있었고, 6개가 명시적 zone 없이 실행 중이었다. 기존 관리 요청이 log 풀 복구 뒤 default-zone 초기화를 이어갔을 가능성이 있지만, 생성 주체를 감사 로그로 확정한 것은 아니다.

추가 풀이나 기존 프로세스를 임의로 삭제·종료하지 않았다. 소유자·용도를 확인하고 후속 범위를 정하기 전에는 복구 대상 풀을 계속 추가하거나 global CRUSH 기본값을 바꾸지 않는다. 원본 데이터·PV/PVC·MinIO는 변경하지 않았으며 파일럿 성능 부하도 재개하지 않았다.

## Prevention

- OSD 개수뿐 아니라 실제 장치·호스트의 독립성을 확인한다.
- 자동 생성된 내부 풀도 원하는 failure domain·replica 조건과 일치하는지 점검한다.
- Argo Healthy, CR Ready, Ceph 데이터 가용성을 별개의 검증 항목으로 둔다.
- 공유 스토리지의 기존 풀을 CR로 관리하기 시작할 때는 삭제 정책과 rollback을 함께 검토한다.
- 실험 설정과 고가용성 운영 설정을 같은 것으로 설명하지 않는다.

## Remaining risks

- **같은 NVMe/호스트 장애에 대한 보호는 추가되지 않는다.**
- `Prune=false,Delete=false`는 Argo 경로의 보호다. 직접 `kubectl delete`로 풀 CR을 삭제하면 Rook이 기존 Ceph 풀까지 삭제할 수 있다. **CR 삭제를 rollback으로 사용하지 않는다.**
- 정상화되지 않는 경우 CR/PG 상태와 Operator 오류를 수집하고 같은 대상 범위의 전진 수정안을 검토한다. 데이터를 지우거나 복제 수를 낮춰 성공처럼 보이게 하지 않는다.
- 요청한 Sync와 대상 65 PG 복구는 검증했지만, 추가 내부 풀과 Monitor 경고가 남아 전체 정상화·파일럿 재개는 미완료다.

## 참고

공개 참고 문서:

- [Rook 1.17 CephBlockPool 설정](https://rook.io/docs/rook/v1.17/CRDs/Block-Storage/ceph-block-pool-crd/#pool-settings)
- [Ceph Monitor 및 PG health checks](https://docs.ceph.com/en/squid/rados/operations/health-checks/)
- [Argo CD Sync options](https://argo-cd.readthedocs.io/en/stable/user-guide/sync-options/)

내부 운영 근거 — 저장소 권한 필요:

- [TwinX GitOps 복구 변경 및 런북](https://github.com/SmartX-Team/TwinX-Ops/tree/48f1fc3cda31af802cefa0b37425b88f7424c65f/argocd/twinx-storage/apps/rook-ceph-pool-recovery)
- [Trident 실험 저장소의 준비·검증 기록](https://github.com/mj006648/Trident-Lakehouse-Experiments/blob/bf0ee141c87eaa79457563e3c7d95409ee1668e5/experiments/operations-v3/results/summary/ceph-pool-recovery-preparation-20260910.md)

- [실제 선택 Sync 결과와 남은 예외 — 내부 권한 필요](https://github.com/mj006648/Trident-Lakehouse-Experiments/blob/main/experiments/operations-v3/results/summary/ceph-pool-recovery-result-2026-09-10.md)

기존의 [Rook-Ceph 재설치 절차](rook-ceph-reinstall.md)나 [LV 준비 절차](lv-preparation.md)는 이번 복구 경로가 아니다. 재설치·LV 재구성으로 이 문제를 해결하려 하지 않는다.
