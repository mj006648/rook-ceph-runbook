# Omniverse Isaac SaaS SmartX 이관 문서

이 디렉터리는 원본 `SmartX-Team/Omniverse/isaac-saas`를 필요한 기능만 남긴 `isaac-twinx` MVP로 축소하고, MiniX에서 실제 검증한 뒤 SmartX `eecs-k8s` 공통 app catalog와 GPU cluster preset으로 옮기는 전 과정을 분리해 기록한다.

## 문서 역할

| 파일 | 역할 | 언제 읽는가 |
| --- | --- | --- |
| [`ISAAC_UI_MVP_SCOPE.md`](./ISAAC_UI_MVP_SCOPE.md) | 제품/설계 기준. 원본에서 무엇을 유지·변경·제거했는지, UI·GPU 선택·image·Nucleus·Extension 경계를 설명 | 무엇을 만들었고 왜 기능을 뺐는지 이해할 때 |
| [`MINIX_GPU_DRA_POC_2026-07-13.md`](./MINIX_GPU_DRA_POC_2026-07-13.md) | MiniX Kubernetes 1.34.3 업그레이드, GPU Operator DRA 전환, RTX 3090 exact selection 실행 기록 | 클러스터/DRA 전제를 확인할 때 |
| [`MINIX_ISAAC_SIM_E2E_2026-07-14.md`](./MINIX_ISAAC_SIM_E2E_2026-07-14.md) | Harbor, Isaac Sim 6.0 image, DRA instance, Nucleus, WebRTC endpoint의 실제 통합 실행 증거 | standalone 구현이 실제로 동작했는지 확인할 때 |
| [`SMARTX_PRE_MIGRATION_REHEARSAL_2026-07-14.md`](./SMARTX_PRE_MIGRATION_REHEARSAL_2026-07-14.md) | 개인 `smartx-k8s`/`twinx-k8s` chart·preset 렌더와 MiniX 적용 결과 | 실제 eecs/c 변경 전 SmartX 구조 검증 결과를 볼 때 |
| [`TWINX_CONTROL1_GPU_MIG_PREVIEW_2026-07-15.md`](./TWINX_CONTROL1_GPU_MIG_PREVIEW_2026-07-15.md) | 실제 TwinX GPU/MIG inventory와 `TwinX-Ops/argocd/omniverse` 읽기 전용 포털 준비·안전 검증 기록 | 이기종 GPU/MIG와 TwinX Argo 배포 경계를 확인할 때 |
| [`TWINX_ISAAC_SIM_E2E_2026-07-15.md`](./TWINX_ISAAC_SIM_E2E_2026-07-15.md) | TwinX L40S/A6000 동시 실행, 신규 Nucleus 실제 인증, Extension 포함, 과거 IP 제거, 삭제·GPU 반환 증거 | 현재 실제 성공 기준을 확인할 때 |
| [`C_DGX_SPARK_4K_KERNEL_WEBRTC_VALIDATION_2026-07-21.md`](./C_DGX_SPARK_4K_KERNEL_WEBRTC_VALIDATION_2026-07-21.md) | C DGX Spark 64K ELF 오류의 원인, 4K NVIDIA 커널 전환, ARM64 자동 선택, hostNetwork WebRTC와 Nucleus 인증의 실제 성공 기록·rollback 절차 | C의 GB10 ARM64 E2E 결과를 확인할 때 |
| [`SMARTX_MIGRATION_PLAN.md`](./SMARTX_MIGRATION_PLAN.md) | 실제 구현 가이드. eecs-k8s/c-k8s의 어느 파일에 어떤 코드를 넣고 왜 넣는지, 검증·rollback까지 설명 | 실제 이관 코드를 작성하거나 검토할 때 |
| [`SCALEX_FEDERATION_SINGLE_CLUSTER_PLAN.md`](./SCALEX_FEDERATION_SINGLE_CLUSTER_PLAN.md) | `isaac-twinx` child Helm chart와 `scalex-federation` release catalog를 이용해 Karmada 경로로 TwinX 한 곳에만 Portal을 배포하는 계획 | 실제 ScaleX Federation child 구조와 단일-target 배포를 준비할 때 |
| [`SCALEX_ISAAC_TWINX_C_DEPLOYMENT_2026-07-24.md`](./SCALEX_ISAAC_TWINX_C_DEPLOYMENT_2026-07-24.md) | `scalex-isaac-twinx` v0.2.4를 Tower Tekton/Federation/Karmada로 C에 배포하고 GB10 생성·WebRTC·Nucleus·삭제까지 검증한 실제 절차 | 같은 배포를 처음부터 재현하거나 새 버전을 promotion할 때 |
| [`../omniverse-nucleus/TWINX_EXECUTION_2026-07-15.md`](../omniverse-nucleus/TWINX_EXECUTION_2026-07-15.md) | TwinX-Ops raw app으로 새 `omniverse` namespace에 Nucleus를 실행한 기록. 기존 `oos-sim`과 외부 Nucleus `10.38.38.32` 비변경 경계 포함 | TwinX Isaac 실행 환경의 Nucleus 경계를 확인할 때 |

## 권장 읽는 순서

### 전체 흐름을 처음 이해할 때

1. `ISAAC_UI_MVP_SCOPE.md`
2. `MINIX_GPU_DRA_POC_2026-07-13.md`
3. `MINIX_ISAAC_SIM_E2E_2026-07-14.md`
4. `SMARTX_PRE_MIGRATION_REHEARSAL_2026-07-14.md`
5. `TWINX_CONTROL1_GPU_MIG_PREVIEW_2026-07-15.md`
6. `TWINX_ISAAC_SIM_E2E_2026-07-15.md`
7. C DGX Spark에서 시험할 때는 `C_DGX_SPARK_4K_KERNEL_WEBRTC_VALIDATION_2026-07-21.md`
8. `SMARTX_MIGRATION_PLAN.md`
9. ScaleX Federation/Karmada 구조를 이해할 때는 `SCALEX_FEDERATION_SINGLE_CLUSTER_PLAN.md`
10. 실제 C 배포를 재현할 때는 `SCALEX_ISAAC_TWINX_C_DEPLOYMENT_2026-07-24.md`
11. Nucleus 경계가 필요하면 `../omniverse-nucleus/TWINX_EXECUTION_2026-07-15.md`

### 바로 eecs-k8s/c-k8s 이관 작업을 할 때

1. `SMARTX_MIGRATION_PLAN.md`에서 저장소·파일·values 경계를 확인한다.
2. `TWINX_ISAAC_SIM_E2E_2026-07-15.md`에서 현재 image, Nucleus, DRA, 삭제 성공 기준을 확인한다.
3. `SMARTX_PRE_MIGRATION_REHEARSAL_2026-07-14.md`에서 기존 chart/preset 렌더 구조를 확인한다.
4. `TWINX_CONTROL1_GPU_MIG_PREVIEW_2026-07-15.md`는 읽기 전용 배포 당시의 안전 경계가 필요할 때만 본다.
5. 기능 범위에 의문이 있을 때만 `ISAAC_UI_MVP_SCOPE.md`로 돌아간다.

### ScaleX Federation child로 TwinX 한 곳에 배포할 때

1. `SCALEX_FEDERATION_SINGLE_CLUSTER_PLAN.md`에서 현재 `SJoon99/scalex-federation` release catalog 계약과 정확한 변경 파일을 확인한다.
2. `TWINX_ISAAC_SIM_E2E_2026-07-15.md`에서 Portal/Nucleus/WebRTC/Delete의 성공 기준을 확인한다.
3. `SMARTX_MIGRATION_PLAN.md`는 eecs-k8s/cluster preset 직접 배포와의 소유권 차이를 비교할 때만 참고한다.
4. 같은 TwinX Portal을 direct SmartX 경로와 Federation 경로가 동시에 관리하지 않도록 writer를 하나만 선택한다.
5. 실제 node4 PipelineRun, promotion, Argo/C 검증 명령은 `SCALEX_ISAAC_TWINX_C_DEPLOYMENT_2026-07-24.md`를 그대로 따른다.

## 문서 간 중복 방지 원칙

```text
MVP_SCOPE      = 무엇을 만들고 왜 뺐는가
DRA_POC        = Kubernetes/GPU 기반이 준비됐는가
MINIX_E2E      = standalone 앱과 image가 실제로 실행됐는가
REHEARSAL      = SmartX chart/preset 모양으로도 동작했는가
TWINX_PREVIEW  = 실제 이기종 GPU/MIG cluster에 안전하게 올릴 모양인가
TWINX_E2E      = 현재 TwinX에서 launch/Nucleus/extensions/delete가 실제로 됐는가
C_GB10_4K      = C DGX Spark의 64K ELF 오류를 4K kernel과 hostNetwork로 어떻게 해결·검증했는가
MIGRATION_PLAN = 실제 eecs-k8s/cluster preset에 어떤 코드를 넣는가
FEDERATION_PLAN = isaac-twinx child chart/release를 Karmada로 TwinX 한 곳에 어떻게 전파하는가
```

같은 실행 결과를 여러 문서에 복사하지 않는다. 구현 가이드는 실행 증거 문서로 링크하고, 실행 증거는 제품 범위를 다시 설명하지 않는다.

## 현재 상태

```text
MiniX Kubernetes 1.34.3 + NVIDIA DRA: 검증 완료
MiniX WebRTC 2.0.0 영상/입력: 사용자 확인 완료
개인 SmartX/TwinX chart/preset 리허설: 완료
TwinX 일반 GPU/MIG 자동 inventory: 검증 완료
TwinX Isaac portal: 10.38.38.243, WRITE_ENABLED=true, Argo CD Synced/Healthy
TwinX Nucleus: 10.38.38.245, RBD 10Gi Retain, Pod 12/12 Running/Healthy
보호 대상: 기존 oos-sim 및 외부 Nucleus 10.38.38.32 비변경
isaac-twinx source/origin: c70d125, 48 tests passed
TwinX-Ops portal revision: c04b787, Argo CD Synced/Healthy
TwinX portal image: sha256:67fc3848cab38bb3503b71bdfc08b1d64e4702e9b95bbdd043287cbe1f0e9254
TwinX Isaac image: sha256:c3a5b1b3402f3f2d6185fccee158023da59e99748ce096c33d5a2404fdea9bb7
GPU UI: physical GPU index/PCI 표시, 메모리 GiB 통일
Isaac 상태: Pending -> Initializing(120초) -> Running, L40S WebRTC 사용자 확인 완료
TwinX L40S index 7 + sv4000-1 A6000 동시 launch: 완료
신규 Nucleus 실제 인증: 두 Isaac Pod의 omni.client stat Result.OK
Extension 9개: image 포함, 두 Pod의 Kit 검색 경로 등록 확인; 자동 활성화는 하지 않음
과거 MiniX 10.34.48.* 및 외부 Nucleus 10.38.38.32: 새 image/runtime extension scan에서 없음
두 E2E Delete: HTTP 204, Deployment/Service/ResourceClaim 0, 두 GPU Available 반환
C DGX Spark kernel/page size: 6.17.0-1026-nvidia / 4096, node Ready
C DGX Spark boot default: GRUB saved entry로 6.17.0-1026-nvidia 영구 고정, 재부팅하지 않음
C portal: isaac-twinx 0.2.2, amd64 control node에서 정상 실행
C DGX Spark ARM64 Isaac 6.0.1: architecture 자동 선택, DRA exact GPU, hostNetwork, node IP WebRTC 검증 완료
C DGX Spark WebRTC: 10.33.201.193, Client 2.0.0 영상/입력 사용자 확인 완료
C Nucleus: 10.33.143.10, nucleus-cred 참조, 신규 instance 인증 status OK
ScaleX C Portal: scalex-isaac-twinx v0.2.4, 10.33.143.11, Argo Synced/Healthy
ScaleX C E2E: GB10 ARM64 create/WebRTC/Nucleus status OK/delete/GPU 반환 완료
eecs-k8s/c-k8s Isaac 코드 반영: eecs 6acbd67, c main 1ac1b9d
```


Nucleus 자체의 eecs-k8s/c-k8s 이관과 C 클러스터 배포는 이미 완료됐으며, 관련 문서는 [`../omniverse-nucleus/`](../omniverse-nucleus/)에 있다. TwinX에서는 `eecs-k8s`/`c-k8s`를 변경하지 않고 `TwinX-Ops` raw app으로 새 `omniverse` namespace에 Nucleus를 실행했으며, 기존 `oos-sim`과 외부 Nucleus `10.38.38.32`는 변경하지 않았다. 세부 기록은 [`../omniverse-nucleus/TWINX_EXECUTION_2026-07-15.md`](../omniverse-nucleus/TWINX_EXECUTION_2026-07-15.md)를 기준으로 한다.
