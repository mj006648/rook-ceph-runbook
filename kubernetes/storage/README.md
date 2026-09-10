# Kubernetes Storage

Rook-Ceph, OSD, PVC, local disk, LV, and Kubernetes storage recovery runbooks.

## Quick map

| Last update | Topic | Document | Contents |
| --- | --- | --- | --- |
| 2026-09-10 | Ceph / PG placement | [TwinX Ceph 내부 풀 배치 복구](twinx-ceph-internal-pool-placement-recovery-2026-09-10.md) | 같은 NVMe의 논리 OSD, host 배치 불일치, 수동 Sync·검증·삭제 방지 |
| 2026-06-25 | Rook-Ceph | [Rook-Ceph Reinstall](rook-ceph-reinstall.md) | Rook-Ceph 전체 재설치와 단계별 bring-up 절차 |
| 2026-06-25 | Local disk / LV | [LV Preparation](lv-preparation.md) | stale LVM PV 때문에 OSD prepare가 실패하는 문제 |
