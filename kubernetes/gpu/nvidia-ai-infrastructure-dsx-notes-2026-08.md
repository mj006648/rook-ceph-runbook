# NVIDIA AI Infrastructure와 DSX 학습 메모 (2026-08)

## 문서 성격과 공개 범위

교수님이 검토를 요청한 NVIDIA AI Infrastructure/DSX 발표 자료를 바탕으로,
AI Factory의 핵심 개념을 NetAI 운영 관점에서 다시 정리했다.

원본에는 비공개 표시가 있으므로 슬라이드 원문, 도면, 세부 용량 산정값,
제품 로드맵과 파트너 정보는 옮기지 않았다. 아래 내용은 2026-09-02 기준
NVIDIA 공개 문서로 다시 확인할 수 있는 개념과 운영 시사점만 담는다.
원본 PDF도 저장소에 포함하지 않는다.

이 문서는 TwinX/MiniX가 NVIDIA DSX를 구축했거나 인증받았다는 의미가 아니다.

## 세 줄 요약

1. **AI Factory는 GPU 서버 묶음이 아니라 데이터부터 추론까지 이어지는 생산 시스템이다.**
2. **성능은 GPU 단품보다 compute, network, storage, power/cooling, software의 공동 설계에 좌우된다.**
3. **운영 목표는 설치 완료가 아니라 token 처리량, 전력 효율, 복구 가능성, 격리와 감사 가능성이다.**

## 1. AI Factory로 보는 이유

전통적인 데이터센터가 여러 범용 애플리케이션을 수용하는 공간이라면, AI Factory는
데이터를 받아 모델을 학습·조정하고 대규모 추론 결과를 지속적으로 생산하는 데
최적화된 시스템이다.

```text
data acquisition/curation
        -> training or model customization
        -> inference serving
        -> application feedback and new data
        -> repeat
```

따라서 운영 질문도 `GPU가 Ready인가?`에서 끝나지 않는다.

- 데이터와 checkpoint가 GPU에 충분히 빠르게 도달하는가?
- 분산 작업의 east-west 통신이 병목 없이 동작하는가?
- 모델 배포 후 latency, throughput, token cost를 함께 측정하는가?
- GPU 고장 시 작업 격리, 재시작, 교체 절차가 있는가?
- 전력과 냉각 한계 안에서 실제 유효 처리량을 최대화하는가?
- tenant별 compute, network, storage, identity가 분리되고 변경 이력이 남는가?

## 2. 기존 데이터센터와 다른 설계 축

| 설계 축 | AI workload 특성 | 운영에서 볼 것 |
| --- | --- | --- |
| Compute | 고밀도 accelerator와 큰 GPU memory | GPU health, topology, utilization, 오류·throttling |
| Network | 분산 학습·추론의 대량 east-west traffic | RDMA/fabric 상태, latency, loss, topology 정합성 |
| Storage | dataset, checkpoint, embedding의 높은 throughput/IOPS | hot/cold tier, ingest 속도, checkpoint 복구 시간 |
| Power/Cooling | 지속 부하와 높은 rack 전력 밀도 | power budget, 온도, cooling 여유, throttling |
| Platform | 여러 사용자와 workload의 동적 자원 경쟁 | scheduler, quota, topology-aware placement, rollback |
| Security | 고가 자원과 민감한 데이터의 multi-tenancy | OOB 분리, workload identity, 암호화, 삭제·감사 증적 |

각 축을 따로 최대화하는 대신 end-to-end 병목을 줄여야 한다. GPU를 추가해도
storage나 fabric이 따라오지 못하면 token 생산량은 비례해 늘지 않는다.

## 3. 공개 DSX 구성요소 읽기

NVIDIA는 DSX를 AI Factory의 설계, 시뮬레이션, 구축, 운영을 위한 reference design,
API, software library와 기술의 묶음으로 설명한다.

| DSX 영역 | 역할 | NetAI 관점의 질문 |
| --- | --- | --- |
| DSX Sim | 논리 인프라와 digital twin을 이용한 사전·사후 검증 | 변경 전에 topology, power, cooling 영향을 검증할 수 있는가? |
| DSX OS | Kubernetes 기반 scheduling, runtime, provisioning, monitoring, remediation | GPU 수명주기를 GitOps와 자동 복구 절차로 관리하는가? |
| DSX MaxLPS | 고정된 power envelope 안에서 performance-per-watt 최적화 | GPU 사용률만 아니라 유효 작업량/전력도 측정하는가? |
| DSX Exchange | compute/network와 power/cooling 계층 사이의 IT/OT event 교환 | BMS·전력 이벤트와 cluster 이벤트를 연결할 수 있는가? |
| DSX Flex | grid·현장 발전·저장장치 신호에 따른 전력 orchestration | 전력 제약 시 workload 감속·이동·중지 정책이 있는가? |
| Reference Designs | 세대별 compute, network, storage, facility 설계 기준 | 현재 규모에 맞는 반복 가능한 단위를 정의했는가? |

DSX OS 공개 구성에는 GPU/Network Operator, topology-aware scheduling,
GPU monitoring/remediation, bare-metal provisioning, attestation, rack management 등이
포함된다. Kubernetes는 AI Factory의 전부가 아니라 물리 인프라와 AI workload를
연결하는 운영 제어면으로 보는 편이 맞다.

## 4. 안정적으로 가져갈 운영 원칙

### 4.1 반복 가능한 단위로 확장한다

node나 GPU를 임의로 하나씩 붙이기보다 compute, fabric port, storage bandwidth,
power/cooling budget을 함께 만족하는 작은 단위를 정의하고 반복한다.
확장 전에는 그 단위의 성능과 장애 복구를 먼저 검증한다.

### 4.2 네트워크 역할을 분리한다

공개 NVIDIA reference architecture는 GPU compute east-west, CPU/storage/user traffic,
out-of-band management의 역할을 구분한다. 작은 환경에서 물리 fabric을 합치더라도
VLAN/VRF, ACL, routing과 관측 지표에서는 경계를 유지한다. 특히 BMC/OOB는
일반 사용자망과 직접 섞지 않고 제한된 관리 경로에서만 접근한다.

### 4.3 평균 사용률보다 유효 산출량을 본다

- training: samples/sec, step time, scaling efficiency, checkpoint time, retry 수
- inference: time to first token, inter-token latency, tokens/sec, error, queue time
- infrastructure: GPU/fabric error, storage latency, power, temperature, throttling
- operation: job completion time, SLO 달성률, token당 비용과 전력

### 4.4 장애 조치를 API와 runbook으로 만든다

```text
detect -> isolate/cordon -> drain or checkpoint -> diagnose
       -> reset/repair/replace -> revalidate -> return to service
```

조치가 다른 workload에 미치는 영향, 중단 기준, rollback과 검증 명령까지 남긴다.

### 4.5 multi-tenancy를 처음부터 설계한다

NVIDIA의 공개 AI cloud 요구사항을 그대로 의무화할 필요는 없지만, 다음은 좋은
control objective다.

- tenant별 network, compute, storage, control plane 경계
- OIDC 기반 사용자·workload identity와 최소 권한 RBAC
- etcd/secret과 persistent data의 at-rest encryption
- 보안 정책 변경과 관리 API 호출의 audit log
- 장비 재할당 전 disk, GPU memory, TPM/BIOS 상태의 sanitization
- TPM 2.0, UEFI Secure Boot와 attestation 검토

## 5. NetAI 저장소와 연결

| AI Factory 운영 축 | 현재 문서 | 다음 보강점 |
| --- | --- | --- |
| GPU resource lifecycle | [NVIDIA DRA rollout](twinx-nvidia-dra-driver-rollout-2026-06-28.md) | workload별 ResourceClaim, topology, quota 검증 |
| Physical node lifecycle | [KISS node lifecycle](../../hardware/provisioning/kiss-node-lifecycle-at-a-glance.md) | GPU firmware/driver baseline과 입고·퇴역 검사 |
| Workload placement | [Karmada lab](../multicluster/karmada/) | Kueue quota와 accelerator topology 연계 |
| Network operations | [Networking runbooks](../networking/) | compute/storage/OOB traffic 역할과 SLO 명시 |
| Identity and secrets | [Identity/Secrets path](../security/identity-secrets/) | GPU workload identity, tenant boundary, audit 연결 |
| Observability | [Observability notes](../observability/) | DCGM/fabric/storage/power 지표의 공통 dashboard |
| Power and facilities | [Hardware power notes](../../hardware/power/) | node/rack power budget, 온도, throttling 대응 |

## 6. NetAI에서 먼저 만들 운영 기준선

### P0. 자산과 topology

- node별 GPU/NIC/DPU/storage/firmware/driver/container runtime inventory
- GPU-NIC-NUMA 연결과 label/taint/DeviceClass의 정합성
- workload, storage, cluster management, BMC/OOB network 경계
- 장애 추적에 쓸 안정적인 node/GPU/NIC 식별자

```bash
kubectl get nodes -o wide
kubectl get nodes --show-labels
kubectl get deviceclass
kubectl get resourceslice -o wide
nvidia-smi --query-gpu=index,uuid,name,pci.bus_id,temperature.gpu,power.draw,power.limit --format=csv
```

### P0. workload SLO와 telemetry

- training과 inference를 분리해 golden workload 선정
- job duration, queue time, GPU health, network/storage latency 수집
- inference는 TTFT, inter-token latency, tokens/sec를 함께 기록
- 하드웨어 이벤트와 Kubernetes event, workload failure를 같은 시간축에서 조회

### P0. break-fix와 rollback

- GPU Xid/ECC, NIC/fabric, node unreachable별 진단 분기
- cordon/drain/checkpoint 가능 조건과 즉시 중단 조건
- driver/operator/firmware 변경의 canary, rollback, 재검증 절차
- 수리 후 burn-in과 production 복귀 승인 기준

### P1. power/cooling과 보안

- node/rack/site power budget, 온도, throttling 기준 기록
- 전력 제한 시 우선순위별 queue, pause, migrate, stop 정책
- BMC/OOB 접근 경로와 계정·감사 정책 점검
- namespace뿐 아니라 network/storage/device까지 tenant 격리 검증
- workload identity, short-lived credential, secret rotation 연결
- 장비 재할당·폐기 시 sanitization 체크리스트

## 7. 교수님과 확인할 질문

1. 기대 결과는 **개념 이해**, **NetAI 현황 진단**, **도입 설계** 중 어디까지인가?
2. 우선 workload는 training, fine-tuning, batch inference, online inference 중 무엇인가?
3. 현재 병목은 GPU, network, storage, power/cooling, 운영 자동화 중 무엇인가?
4. TwinX/MiniX를 하나의 pool로 볼지 역할별 factory cell로 분리할지?
5. GPU utilization 대신 어떤 workload SLO와 token efficiency를 기준으로 둘지?
6. 시설·전력·BMS telemetry를 Kubernetes/Prometheus와 어디까지 연결할지?

## 참고

모든 링크는 2026-09-02에 확인했다. NVIDIA 문서는 계속 갱신되므로 실제 설계 전에는
대상 제품 세대와 문서 버전을 다시 고정해야 한다.

- [NVIDIA DSX Documentation](https://docs.nvidia.com/dsx)
- [NVIDIA Enterprise AI Factory Design Guide](https://docs.nvidia.com/ai-enterprise/planning-resource/ai-factory-white-paper/latest/ai-factory-overview.html)
- [NVIDIA DSX Software Components](https://docs.nvidia.com/dsx/ncp/part-2-software-components/nvidia-software-components)
- [NVIDIA Requirements for AI Clouds](https://docs.nvidia.com/dsx/ncp/nvidia-requirements-for-ai-clouds/home)
- [NVIDIA HGX AI Factory Network Architecture](https://docs.nvidia.com/enterprise-reference-architectures/hgx-ai-factory-h100-h200-b200/latest/network-logical-architecture.html)
