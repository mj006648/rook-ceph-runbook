# ScaleX Isaac TwinX C 클러스터 실제 배포 절차

> 실행일: 2026-07-24
> 대상: `mj006648/scalex-isaac-twinx` → `SJoon99/scalex-federation` → Tower Argo CD/Karmada → member `c`
> 결과: Portal `10.33.143.11`, GB10 ARM64 Isaac Sim 생성·WebRTC startup·Nucleus 인증·삭제/GPU 반환 검증 완료

이 문서는 계획이 아니라 실제 실행한 순서와 확인 결과를 기록한다. Git에는 Secret 값이나
Harbor/Nucleus 비밀번호를 넣지 않는다.

## 1. 최종 구조

~~~text
mj006648/scalex-isaac-twinx
  chart/                  Portal, RBAC, Service, Karmada policy
  images/portal/          Tekton이 빌드하는 Portal image
  src/                    FastAPI Portal와 정적 UI
  manifests/              사용하지 않음

SJoon99/scalex-federation
  argocd/appproject.yaml
  releases/scalex-isaac-twinx/
    release.yaml
    runtime-values.yaml   {}
    values.yaml           Tekton promotion이 생성

Tower
  Tekton child-build
    -> Portal image build/push
    -> immutable digest 결정
    -> Federation promotion PR 생성
  Argo CD
    -> child chart + Federation values 렌더
  Karmada
    -> member c로 전파

member c
  namespace/scalex-isaac-twinx
  secret/harbor-regcred
  secret/nucleus-cred
  deployment/service/isaac-portal
  Portal이 요청 시 Isaac ResourceClaim/Deployment 생성
~~~

Isaac Sim runtime image는 child의 `images/` 아래에서 다시 빌드하지 않는다. 이미 Harbor에 있는
amd64/arm64 image를 immutable digest로 참조한다. `images/portal/Dockerfile`만 Tower가 자동으로
발견하고 빌드한다.

## 2. 실제 사용한 identity

### Child

~~~text
repository: https://github.com/mj006648/scalex-isaac-twinx.git
release: v0.2.4
source SHA: 118de5390cc610d065061d6be66f7611c737496f
commit: release v0.2.4
~~~

### Federation

~~~text
repository: https://github.com/SJoon99/scalex-federation.git
promotion PR: #14
promotion commit: 5648cf90b2363a6010256c7e20a169b69215aa8e
runtime-values.yaml: {}
state: active
~~~

### Images

~~~text
Portal:
10.34.25.18/tower-ci/scalex-isaac-twinx/portal:v0.2.4
@sha256:41a9af63c75b6c36deffb1f638b33df0e33959686172f60cd8798fa8e52042ac

Isaac Sim amd64:
10.34.25.18/omniverse/isaac-sim
@sha256:eaaa811e907f2bf5fd2878c21d61ac3daa23420e935431e7f3bbad6842a2bd46

Isaac Sim arm64:
10.34.25.18/omniverse/isaac-sim
@sha256:fcf7946c583e9cb2f4bb33306ca21d3a3d304025d7ebecc286b801e4769c55b0
~~~

`tag`는 사람이 읽는 release identity이고, 실제 실행 identity는 `@sha256:...`다.

## 3. 먼저 알아야 할 값

~~~bash
CHILD_REPO=https://github.com/mj006648/scalex-isaac-twinx.git
FEDERATION_REPO=https://github.com/SJoon99/scalex-federation.git
CHILD_NAME=scalex-isaac-twinx
NAMESPACE=scalex-isaac-twinx
MEMBER_CLUSTER=c
PORTAL_IP=10.33.143.11
NUCLEUS_IP=10.33.143.10
~~~

Tower host에서는 현재 환경의 kubeconfig helper인 `tkubectl`을 사용했다. member C host에서는
`kubectl`을 사용했다. 두 명령을 섞으면 다른 클러스터에 리소스를 만들 수 있으므로 매 단계에서
context/namespace를 확인한다.

## 4. Child chart 설정

`scalex-isaac-twinx/chart/Chart.yaml`:

~~~yaml
apiVersion: v2
name: scalex-isaac-twinx
type: application
version: 0.2.4
appVersion: "0.2.4"
~~~

`scalex-isaac-twinx/chart/values.yaml`의 핵심값:

~~~yaml
fullnameOverride: "isaac-portal"

images:
  portal:
    repository: 10.34.25.18/tower-ci/scalex-isaac-twinx/portal
    tag: v0.2.4
    pullPolicy: IfNotPresent

portal:
  writeEnabled: true
  service:
    type: ClusterIP

instanceImages:
  amd64: 10.34.25.18/omniverse/isaac-sim@sha256:eaaa811e907f2bf5fd2878c21d61ac3daa23420e935431e7f3bbad6842a2bd46
  arm64: 10.34.25.18/omniverse/isaac-sim@sha256:fcf7946c583e9cb2f4bb33306ca21d3a3d304025d7ebecc286b801e4769c55b0

imagePullSecrets:
  - harbor-regcred

nucleus:
  server: omniverse://10.33.143.10/
  projectPath: Projects/demonstration
  secretName: nucleus-cred
  userKey: OMNI_USER
  passwordKey: OMNI_PASS

stream:
  arm64:
    startXvfb: false
    hostNetwork: true

karmada:
  enabled: true
  placement:
    cluster: c
  portalService:
    exposure: member-lb
    loadBalancerIP: 10.33.143.11
    annotationKey: lbipam.cilium.io/ips
~~~

중요한 경계:

- child `values.yaml`에는 Portal의 `digest`와 `sourceRevision`을 쓰지 않는다.
- 두 값은 Tekton promotion이 Federation `values.yaml`에 생성한다.
- Isaac runtime digest는 Portal build 대상이 아니므로 child가 직접 고정한다.
- ARM64 GB10은 검증된 `hostNetwork=true`와 node InternalIP를 사용한다.
- Secret 이름만 values에 있고 Secret 값/manifest는 없다.
- chart는 Namespace 또는 Secret을 만들지 않는다.

## 5. Child 로컬 검증과 push

~~~bash
cd ~/git/scalex-isaac-twinx

python3 -m venv .venv
. .venv/bin/activate
pip install -r src/tests/requirements.txt

helm lint --strict chart

helm template scalex-isaac-twinx chart \
  --namespace scalex-isaac-twinx \
  >/tmp/scalex-isaac-twinx.yaml

PYTHONPATH=src python -m pytest src/tests

docker build \
  -f images/portal/Dockerfile \
  -t scalex-isaac-twinx:v0.2.4-local .

git add README.md chart src
git commit -m 'release v0.2.4'
git push origin main
~~~

이번 실행에서는 Portal 단위 테스트 `38 passed`, Helm-dependent 테스트 `9 skipped`인 로컬
환경에서 node4 Helm 3로 strict lint와 template을 별도 확인했다. Portal Docker build도 통과했다.

새 SHA 확인:

~~~bash
CHILD_SHA=$(git ls-remote \
  https://github.com/mj006648/scalex-isaac-twinx.git \
  refs/heads/main | awk '{print $1}')

test "${#CHILD_SHA}" -eq 40
echo "$CHILD_SHA"
~~~

커밋을 새로 만들면 SHA가 바뀌는 것이 정상이다. SHA가 바뀌어도 manifest 내용이 같은 한
Kubernetes 동작이 달라지는 것은 아니지만, promotion은 정확한 source를 재현하기 위해 새 SHA를
기록한다.

## 6. Federation 최초 등록

이미 등록돼 있다면 이 절은 다시 하지 않는다.

### 6.1 AppProject source 허용

`scalex-federation/argocd/appproject.yaml`의 `spec.sourceRepos`에 추가한다.

~~~yaml
- https://github.com/mj006648/scalex-isaac-twinx.git
~~~

이 변경만으로 배포는 시작되지 않는다.

### 6.2 disabled release 생성

~~~text
releases/scalex-isaac-twinx/
  release.yaml
  runtime-values.yaml
~~~

최초 `release.yaml`:

~~~yaml
name: scalex-isaac-twinx
namespace: scalex-isaac-twinx
state: disabled
disabledReason: Initial image promotion and C activation are pending.
renderer: helm/v1

source:
  repoURL: https://github.com/mj006648/scalex-isaac-twinx.git
  path: chart
  branch: main

values:
  path: releases/scalex-isaac-twinx/values.yaml

promotion:
  mode: tracking

requiredKinds:
  - ClusterRole
  - ClusterRoleBinding
  - ClusterPropagationPolicy
~~~

최초 `runtime-values.yaml`:

~~~yaml
{}
~~~

최초 등록 PR에서는 `values.yaml`을 수동으로 만들지 않는다. Tekton promotion이 생성한다.

~~~bash
test ! -e releases/scalex-isaac-twinx/values.yaml
python3 tests/test_promotion_contract.py
~~~

## 7. C namespace와 Secret 준비

이번 작업에서는 요청에 따라 `c-k8s` patch를 추가하지 않고 C에서 namespace를 직접 만들었다.

~~~bash
kubectl create namespace scalex-isaac-twinx \
  --dry-run=client -o yaml | kubectl apply -f -

kubectl label namespace scalex-isaac-twinx \
  app.kubernetes.io/part-of=scalex-federation-poc \
  scalex.io/release=isaac-twinx \
  --overwrite
~~~

기존 `omniverse` namespace에 검증된 Secret이 있다면 값을 복호화하거나 출력하지 않고 복사한다.

~~~bash
for secret in harbor-regcred nucleus-cred; do
  kubectl -n omniverse get secret "$secret" -o json |
    jq '
      del(
        .metadata.uid,
        .metadata.resourceVersion,
        .metadata.creationTimestamp,
        .metadata.managedFields,
        .metadata.ownerReferences
      )
      | .metadata.namespace="scalex-isaac-twinx"
    ' |
    kubectl apply -f -
done
~~~

키 이름만 확인한다.

~~~bash
kubectl -n scalex-isaac-twinx \
  get secret harbor-regcred nucleus-cred -o json |
  jq -r '
    .items[]
    | [
        .metadata.name,
        .type,
        ((.data // {}) | keys | join(","))
      ]
    | @tsv
  '
~~~

예상:

~~~text
harbor-regcred  kubernetes.io/dockerconfigjson  .dockerconfigjson
nucleus-cred    Opaque                          OMNI_PASS,OMNI_USER
~~~

기존 Secret이 없다면 팀 표준인 External Secrets 또는 Sealed Secrets를 사용한다. 평문 Secret
값을 Git, shell history, 문서, chat에 넣지 않는다.

## 8. Tower 사전 확인

node4에서:

~~~bash
tkubectl get namespace tower-ci

tkubectl -n tower-ci get pipeline child-build \
  -o jsonpath='{.spec.params[*].name}{"\n"}'

tkubectl -n tower-ci get secret \
  harbor-builder federation-promotion-github-app

tkubectl get storageclass rook-ceph-block-hot

tkubectl auth can-i \
  create pipelineruns.tekton.dev \
  --namespace tower-ci
~~~

Pipeline에는 다음 parameter가 있어야 한다.

~~~text
child-name
repo-url
source-revision
chart-path
build-targets
allowed-kinds
~~~

`build-targets`는 입력하지 않았다. Pipeline이 `images/portal/Dockerfile`을 자동 발견한다.

## 9. 한 개의 PipelineRun YAML 생성

node4 홈에는 `~/scalex-isaac-twinx.yaml` 한 개만 유지했다.

~~~bash
CHILD_SHA=$(git ls-remote \
  https://github.com/mj006648/scalex-isaac-twinx.git \
  refs/heads/main | awk '{print $1}')

test "${#CHILD_SHA}" -eq 40

cat > ~/scalex-isaac-twinx.yaml <<EOF
apiVersion: tekton.dev/v1
kind: PipelineRun
metadata:
  generateName: scalex-isaac-twinx-release-build-
  namespace: tower-ci
  labels:
    app.kubernetes.io/part-of: tekton-ci
    scalex.io/child-name: scalex-isaac-twinx
spec:
  pipelineRef:
    name: child-build

  taskRunTemplate:
    serviceAccountName: tekton-ci-runner
    podTemplate:
      securityContext:
        fsGroup: 65532
        fsGroupChangePolicy: OnRootMismatch

  params:
    - name: child-name
      value: scalex-isaac-twinx

    - name: repo-url
      value: https://github.com/mj006648/scalex-isaac-twinx.git

    - name: source-revision
      value: $CHILD_SHA

    - name: chart-path
      value: chart

    - name: allowed-kinds
      value: ClusterRole,ClusterRoleBinding,ClusterPropagationPolicy

  workspaces:
    - name: source
      volumeClaimTemplate:
        spec:
          accessModes: [ReadWriteOnce]
          storageClassName: rook-ceph-block-hot
          resources:
            requests:
              storage: 5Gi
EOF
~~~

SHA 확인:

~~~bash
grep -A1 'name: source-revision' ~/scalex-isaac-twinx.yaml
~~~

## 10. Tekton 실행과 확인

~~~bash
tkubectl -n tower-ci create \
  -f ~/scalex-isaac-twinx.yaml
~~~

`generateName`을 사용하므로 `apply`가 아니라 `create`다.

이번 run:

~~~text
scalex-isaac-twinx-release-build-mfl69
~~~

완료 대기:

~~~bash
RUN=scalex-isaac-twinx-release-build-mfl69

tkubectl -n tower-ci wait \
  --for=condition=Succeeded \
  pipelinerun/$RUN \
  --timeout=30m

tkubectl -n tower-ci get taskrun \
  -l tekton.dev/pipelineRun=$RUN \
  -o json |
  jq -r '
    .items[]
    | [
        .metadata.labels["tekton.dev/pipelineTask"],
        .status.conditions[0].status,
        .status.conditions[0].reason
      ]
    | @tsv
  ' |
  sort
~~~

실제 결과:

~~~text
Tasks Completed: 7
Failed: 0
clone                     Succeeded
validate-input            Succeeded
derive-targets            Succeeded
helm-validate             Succeeded
build-push                Succeeded
create-promotion-payload  Succeeded
promote                   Succeeded
~~~

## 11. Promotion PR 검토와 merge

Tekton이 생성한 PR #14에서 정확히 두 파일만 바뀌었다.

~~~text
releases/scalex-isaac-twinx/release.yaml
releases/scalex-isaac-twinx/values.yaml
~~~

`release.yaml`:

~~~yaml
promotion:
  mode: tracking
  resolvedRevision: 118de5390cc610d065061d6be66f7611c737496f
~~~

generated `values.yaml`:

~~~yaml
images:
  portal:
    repository: 10.34.25.18/tower-ci/scalex-isaac-twinx/portal
    tag: v0.2.4
    pullPolicy: IfNotPresent
    digest: sha256:41a9af63c75b6c36deffb1f638b33df0e33959686172f60cd8798fa8e52042ac
    sourceRevision: 118de5390cc610d065061d6be66f7611c737496f
~~~

검토 기준:

- `sourceRevision`이 실행한 `CHILD_SHA`와 같은가
- `digest`가 `sha256:` + 64자리인가
- tag가 child `chart/values.yaml`과 같은가
- 다른 release를 변경하지 않았는가
- `runtime-values.yaml`이 계속 `{}`인가
- 최초 단계라면 `state: disabled`가 유지되는가

~~~bash
python3 tests/test_promotion_contract.py
~~~

검토 후 PR을 merge한다.

## 12. 최초 activation

최초 promotion과 C namespace/Secret 준비가 끝난 뒤 별도 변경으로 활성화한다.

~~~yaml
state: active
~~~

`disabledReason`은 제거한다. 이번 v0.2.4 update 시점에는 이미 active였기 때문에 promotion PR
merge 직후 automated sync가 시작됐다.

## 13. Argo refresh와 상태 확인

Application:

~~~text
federation-scalex-isaac-twinx
~~~

automated sync가 켜져 있으므로 보통 수동 sync는 필요 없다. 즉시 다시 읽게 할 때는 hard refresh만
요청한다.

~~~bash
tkubectl -n argo annotate application \
  federation-scalex-isaac-twinx \
  argocd.argoproj.io/refresh=hard \
  --overwrite
~~~

확인:

~~~bash
tkubectl -n argo get application \
  federation-scalex-isaac-twinx -o json |
  jq '{
    sync: .status.sync.status,
    health: .status.health.status,
    revisions: .status.sync.revisions,
    operation: .status.operationState.phase
  }'
~~~

이번 성공값:

~~~text
sync: Synced
health: Healthy
operation: Succeeded
child revision: 118de5390cc610d065061d6be66f7611c737496f
federation revision: 5648cf90b2363a6010256c7e20a169b69215aa8e
~~~

## 14. C Portal 검증

~~~bash
kubectl -n scalex-isaac-twinx rollout status \
  deployment/isaac-portal \
  --timeout=5m

kubectl -n scalex-isaac-twinx get pod,service -o wide

curl -fsS http://10.33.143.11/healthz | jq .
curl -fsS http://10.33.143.11/api/config | jq .
curl -fsS http://10.33.143.11/api/gpus | jq .
~~~

실제 결과:

~~~text
Portal Pod: Running / Ready
Service: LoadBalancer
External IP: 10.33.143.11
writeEnabled: true
imageConfigured: true
nucleusConfigured: true
GPU: NVIDIA GB10 / arm64 / Available
~~~

live Portal image:

~~~text
10.34.25.18/tower-ci/scalex-isaac-twinx/portal:v0.2.4
@sha256:41a9af63c75b6c36deffb1f638b33df0e33959686172f60cd8798fa8e52042ac
~~~

## 15. 실제 Isaac 인스턴스 생성

Available이며 WebRTC-compatible인 GPU UUID를 API에서 현재값으로 고른다. UUID를 Git values에
고정하지 않는다.

~~~bash
GPU_UUID=$(curl -fsS http://10.33.143.11/api/gpus |
  jq -r '
    .items[]
    | select(
        .status == "Available"
        and .compatibleWithWebRTC == true
      )
    | .uuid
  ' |
  head -1)

test -n "$GPU_UUID"

curl -fsS \
  -X POST \
  http://10.33.143.11/api/instances \
  -H 'Content-Type: application/json' \
  --data "{
    \"name\": \"scalex-e2e-gb10\",
    \"gpuUUID\": \"$GPU_UUID\"
  }" |
  jq .
~~~

이번 응답:

~~~text
name: scalex-e2e-gb10
node architecture: arm64
product: NVIDIA GB10
streamIP: 10.33.201.193
status: Pending
~~~

생성된 리소스:

~~~bash
kubectl -n scalex-isaac-twinx get \
  resourceclaim,deployment,pod,service \
  -l app.kubernetes.io/instance=scalex-e2e-gb10 \
  -o wide
~~~

GB10 hostNetwork 경로에서는 인스턴스별 Service를 만들지 않는다. ResourceClaim, Deployment,
Pod가 생성되고 node InternalIP를 stream IP로 사용한다.

실제 계약:

~~~text
ResourceClaim: allocated,reserved
runtime image: ARM64 immutable digest
hostNetwork: true
dnsPolicy: ClusterFirstWithHostNet
imagePullSecrets: harbor-regcred
Nucleus Secret refs: nucleus-cred / OMNI_USER / OMNI_PASS
public stream IP: 10.33.201.193
~~~

## 16. Isaac/WebRTC/Nucleus 성공 확인

Pod Ready:

~~~bash
kubectl -n scalex-isaac-twinx wait \
  --for=condition=Ready \
  pod \
  -l app.kubernetes.io/instance=scalex-e2e-gb10 \
  --timeout=15m
~~~

Portal state:

~~~bash
curl -fsS http://10.33.143.11/api/instances |
  jq '.items[] | select(.name=="scalex-e2e-gb10")'
~~~

`Initializing`은 오류가 아니다. 현재 설정은 시작 후 120초까지 Initializing으로 표시한 뒤
`Running`으로 전환한다.

Isaac startup marker:

~~~bash
POD=$(kubectl -n scalex-isaac-twinx get pod \
  -l app.kubernetes.io/instance=scalex-e2e-gb10 \
  -o jsonpath='{.items[0].metadata.name}')

kubectl -n scalex-isaac-twinx logs "$POD" --tail=6000 |
  grep -Ei 'livestream|WebRTC' |
  tail -100
~~~

실제 확인:

~~~text
omni.kit.livestream.core startup
omni.kit.livestream.webrtc startup
omni.kit.livestream.app startup
10.33.201.193:49100 TCP reachable
~~~

Nucleus 인증 결과는 Secret 값을 출력하지 않고 auth status만 확인한다.

~~~bash
kubectl -n omniverse logs \
  omniverse-nucleus-0 \
  -c nucleus-auth \
  --since=10m |
  grep -E "InternalCredentials.auth|status.*OK" |
  tail -20
~~~

실제 확인:

~~~text
username: omniverse
status: OK
~~~

## 17. 테스트 인스턴스 삭제와 GPU 반환

~~~bash
curl -sS \
  -o /dev/null \
  -w '%{http_code}\n' \
  -X DELETE \
  http://10.33.143.11/api/instances/scalex-e2e-gb10
~~~

예상 HTTP status:

~~~text
204
~~~

리소스 제거 확인:

~~~bash
kubectl -n scalex-isaac-twinx get \
  resourceclaim,deployment,pod,service \
  -l app.kubernetes.io/instance=scalex-e2e-gb10
~~~

GPU 반환 확인:

~~~bash
curl -fsS http://10.33.143.11/api/gpus |
  jq '{summary, items: [.items[] | {product, nodeArchitecture, status, allocatedBy}]}'
~~~

이번 실행에서는 삭제 후 약 36초에 instance 리소스가 0개가 되고 GB10이 다시 `Available`로
돌아왔다.

## 18. 새 버전을 다시 배포할 때

Portal source 또는 chart를 바꾸면 다음 순서를 반복한다.

~~~text
1. chart/Chart.yaml version/appVersion 증가
2. chart/values.yaml images.portal.tag 증가
3. 테스트와 Helm lint/template
4. child main commit/push
5. 새 40자 CHILD_SHA 조회
6. ~/scalex-isaac-twinx.yaml의 source-revision 갱신
7. 새 PipelineRun create
8. 7 Task 성공 확인
9. promotion PR의 digest/sourceRevision 검토
10. promotion PR merge
11. Argo Synced/Healthy 확인
12. C live image/env/API 확인
~~~

같은 tag를 재사용하지 않는다. source commit이 달라지면 SHA도 달라지는 것이 정상이다.
Federation generated values의 digest/sourceRevision을 수동 편집하지 않는다.

## 19. 주의점

### writer는 하나만 운영

기존 direct `omniverse-isaac-saas` Portal과 Federation Portal을 동시에
`writeEnabled=true`로 운영하지 않는다. 같은 GPU와 instance 리소스를 두 controller가 관리하면
충돌할 수 있다.

### Portal image와 Isaac image의 lifecycle은 다름

~~~text
images/portal/Dockerfile
  -> 모든 child commit에서 Tower가 자동 build
  -> Federation values.yaml에 digest promotion

instanceImages.amd64 / instanceImages.arm64
  -> 별도 isaac-twinx/runtime build pipeline 산출물
  -> 이미 검증된 Harbor digest를 child values에서 참조
~~~

Isaac Sim Dockerfile을 `scalex-isaac-twinx/images/` 아래에 넣으면 Tower가 Portal 변경마다 대형
runtime image까지 자동 build 대상으로 인식할 수 있으므로 넣지 않았다.

### ARM64 GB10

- C의 GB10 node는 `arm64`다.
- 검증 kernel은 `6.17.0-1026-nvidia`, 4K page size다.
- ARM image와 `hostNetwork=true`를 사용한다.
- stream IP는 node InternalIP `10.33.201.193`다.
- 같은 node에서 hostNetwork Isaac 인스턴스 두 개를 동시에 만들지 않는다.

### Secret

- `harbor-regcred`: private Harbor image pull
- `nucleus-cred`: `OMNI_USER`, `OMNI_PASS`
- 값은 Git/문서/log에 출력하지 않는다.
- Secret 복사는 base64 data를 decode하지 않은 상태로 수행한다.

### Karmada와 LoadBalancer

- placement `c`와 Portal LB `10.33.143.11`은 child chart 기본값이다.
- raw Portal Service는 ClusterIP다.
- Karmada OverridePolicy가 member C에서만 LoadBalancer로 바꾼다.
- Federation `runtime-values.yaml`은 현재 `{}`다.

### Argo

automated sync + selfHeal + prune가 켜져 있다. promotion merge 후 보통 자동 적용되며, 지연 시
hard refresh 후 상태를 본다. Application이 `Synced/Healthy/Succeeded`가 되기 전에 C 결과를
성공으로 판단하지 않는다.

## 20. Rollback

### instance만 제거

~~~bash
curl -X DELETE \
  http://10.33.143.11/api/instances/<instance-name>
~~~

ResourceClaim과 GPU 반환까지 확인한다.

### 긴급 write 차단

Federation `runtime-values.yaml`에 임시 override를 넣고 merge한다.

~~~yaml
portal:
  writeEnabled: false
~~~

Argo가 적용된 뒤 `/api/config`의 `writeEnabled=false`와 POST 403을 확인한다. 정상화할 때는
임시 override를 제거해 다시 `{}`로 돌린다.

### release 중지

Federation `release.yaml`:

~~~yaml
state: disabled
disabledReason: Emergency rollback.
~~~

merge 후 Argo/Karmada prune 범위를 확인한다. namespace와 Secret은 chart 소유가 아니므로
자동 삭제 대상으로 가정하지 않는다.

### 이전 Portal로 복귀

generated `values.yaml`을 임의로 과거 digest로 고치기보다, 되돌린 child commit으로 새 version과
새 PipelineRun/promotion을 만드는 것이 추적 가능하고 안전하다.

## 21. 최종 성공 체크리스트

~~~text
[x] child v0.2.4 commit/push
[x] 새 40자 source SHA 사용
[x] Tower PipelineRun 7/7 Succeeded
[x] promotion PR #14 generated values 검토/merge
[x] Federation promotion contract PASS
[x] Argo Synced / Healthy / Succeeded
[x] Portal v0.2.4 immutable digest 실행
[x] Portal LoadBalancer 10.33.143.11
[x] writeEnabled/imageConfigured/nucleusConfigured true
[x] harbor-regcred/nucleus-cred 준비
[x] GB10 Available 확인
[x] ResourceClaim allocated/reserved
[x] arm64 immutable image 자동 선택
[x] hostNetwork + node IP 10.33.201.193
[x] Isaac Pod Ready
[x] Portal state Running
[x] WebRTC extensions startup
[x] WebRTC TCP endpoint reachable
[x] Nucleus authentication status OK
[x] DELETE HTTP 204
[x] instance 리소스 0
[x] GB10 Available 반환
~~~

## 22. 한 문장 결론

`scalex-isaac-twinx` v0.2.4는 Portal만 Tower에서 빌드하고 기존 Harbor의 amd64/arm64 Isaac
Sim digest를 사용하며, Federation의 빈 runtime override와 child Karmada policy만으로 C에
배포되어 GB10 ARM64 Isaac 생성, WebRTC startup, Nucleus 인증, 삭제와 GPU 반환까지 실제
검증됐다.
