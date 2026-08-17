# SPIRE Kubernetes 실습: Helm hardened chart로 SVID 발급부터 장애 분석까지

이 문서는 Kubernetes에서 SPIRE를 안전하게 설치하고, workload가 X.509-SVID를 받는 흐름을 끝까지 확인하는 실습이다. 목표는 “설치 성공”이 아니라 다음을 직접 추적하는 것이다.

- 어떤 chart version과 SPIRE app version을 썼는가
- Server, Agent, Controller Manager, CSI driver가 어떤 namespace에서 동작하는가
- 어떤 `ClusterSPIFFEID`가 어떤 Pod에 어떤 SPIFFE ID를 부여하는가
- Workload API socket이 Pod에 어떻게 전달되는가
- SVID가 언제 만료되고 어떻게 회전되는가
- 장애가 났을 때 설치, attestation, registration, socket, client 중 어디가 문제인지 구분할 수 있는가

조사 기준 시점은 **2026-08-17**이다. 공식 SPIRE release 기준은 **v1.15.2, 2026-07-09 공개**다. SPIFFE Kubernetes quickstart는 Kubernetes **1.29-1.34**에서 테스트되었다고 명시한다. 이 문서는 공식 **helm-charts-hardened** 경로를 우선 사용한다.

## 안전 범위

이 실습은 학습용 클러스터에서만 실행한다.

- 운영 trust domain을 사용하지 않는다.
- 운영 kubeconfig, 운영 signing key, 운영 OpenBao token을 사용하지 않는다.
- SVID private key를 확인 목적으로 파일에 쓰더라도 실습 Pod 안 임시 경로에만 둔다.
- `REPLACE_ME`, `<PLACEHOLDER>`는 실제 secret이 아니다. 그대로 Git에 남겨도 되는 자리표시자다.
- lab-only value는 운영 권장값이 아니다. 운영 전에는 HA, datastore, backup, telemetry, PSS, RBAC, network policy를 별도로 설계한다.

## 실습에서 사용할 변수

터미널마다 먼저 실행한다.

```bash
export LAB_MGMT_NAMESPACE=spire-mgmt
export LAB_SPIRE_SERVER_NAMESPACE=spire-server
export LAB_SPIRE_SYSTEM_NAMESPACE=spire-system
export LAB_APP_NAMESPACE=identity-demo
export LAB_CLUSTER_NAME=identity-lab
export LAB_TRUST_DOMAIN=lab.example.org
export LAB_RELEASE_NAME=spire
export LAB_HELM_REPO_NAME=spiffe
export LAB_HELM_REPO_URL=https://spiffe.github.io/helm-charts-hardened/
```

값 확인:

```bash
printf 'cluster=%s\ntrustDomain=%s\nmgmtNs=%s\nserverNs=%s\nsystemNs=%s\nappNs=%s\n' \
  "$LAB_CLUSTER_NAME" \
  "$LAB_TRUST_DOMAIN" \
  "$LAB_MGMT_NAMESPACE" \
  "$LAB_SPIRE_SERVER_NAMESPACE" \
  "$LAB_SPIRE_SYSTEM_NAMESPACE" \
  "$LAB_APP_NAMESPACE"
```

## Prerequisites

필수 도구:

- `kubectl`
- `helm`
- Kubernetes 1.29-1.34 범위의 학습용 클러스터 권장
- cluster-admin 수준 권한이 있는 lab kubeconfig
- 기본 StorageClass 또는 SPIRE Server persistence에 사용할 StorageClass

버전 확인:

```bash
kubectl version
kubectl cluster-info
kubectl get nodes -o wide
kubectl get storageclass
helm version
```

StorageClass가 없으면 Server StatefulSet의 PVC가 `Pending`이 될 수 있다. kind/minikube/kubeadm 환경은 기본 storage provisioner를 먼저 확인한다.

```bash
kubectl get pvc -A
kubectl describe storageclass <STORAGE_CLASS_NAME>
```

## 공식 version pin 확인

절대 `latest`를 암묵적으로 믿지 않는다. 설치 직전 chart와 app version을 기록한다.

```bash
helm repo add "$LAB_HELM_REPO_NAME" "$LAB_HELM_REPO_URL"
helm repo update

helm search repo "$LAB_HELM_REPO_NAME/spire" --versions | head -20
helm search repo "$LAB_HELM_REPO_NAME/spire-crds" --versions | head -20
```

2026-08-17 조사 시점에는 SPIRE upstream 최신 release가 `v1.15.2`이고, 공식 다운로드 문서는 container image tag가 leading `v` 없이 `1.15.2`라고 안내한다.

```bash
# 문서 조사 기준 예시. 실제 설치 전에는 위 helm search 결과를 사용한다.
export LAB_SPIRE_APP_VERSION=1.15.2

# chart version은 설치 직전 helm search 결과에서 고른다.
export LAB_SPIRE_CHART_VERSION=<REPLACE_WITH_HELM_SEARCH_RESULT>
export LAB_SPIRE_CRDS_CHART_VERSION=<REPLACE_WITH_HELM_SEARCH_RESULT>
```

chart version을 자동으로 고르고 싶다면, lab에서만 다음처럼 현재 repo의 최상단 version을 임시 pin으로 기록할 수 있다.

```bash
export LAB_SPIRE_CHART_VERSION="$(
  helm search repo "$LAB_HELM_REPO_NAME/spire" --versions \
    | awk 'NR==2 {print $2}'
)"
export LAB_SPIRE_CRDS_CHART_VERSION="$(
  helm search repo "$LAB_HELM_REPO_NAME/spire-crds" --versions \
    | awk 'NR==2 {print $2}'
)"
printf 'spire chart=%s\nspire-crds chart=%s\n' "$LAB_SPIRE_CHART_VERSION" "$LAB_SPIRE_CRDS_CHART_VERSION"
```

자동 선택은 재현성이 낮다. 실습 기록에는 실제 선택한 chart version을 남긴다.

## Namespace 전략

helm-charts-hardened 문서는 production deployment에서 management namespace와 SPIRE service namespace를 분리하는 구성을 설명한다.

| Namespace | 용도 |
| --- | --- |
| `spire-mgmt` | Helm release와 CRD chart 관리 |
| `spire-server` | SPIRE Server, Controller Manager, OIDC Discovery Provider |
| `spire-system` | SPIRE Agent, CSI driver처럼 node/system 권한이 필요한 구성 |
| `identity-demo` | 실습 workload |

Namespace 생성은 chart의 recommendation으로 맡길 수 있지만, 실습에서는 먼저 상태를 보이게 만들기 위해 app namespace만 직접 만든다.

```bash
kubectl create namespace "$LAB_APP_NAMESPACE" --dry-run=client -o yaml | kubectl apply -f -
kubectl label namespace "$LAB_APP_NAMESPACE" \
  pod-security.kubernetes.io/enforce=restricted \
  pod-security.kubernetes.io/audit=restricted \
  pod-security.kubernetes.io/warn=restricted \
  --overwrite
```

## values 파일 작성

다음 파일은 lab용이다. trust domain, cluster name, namespace creation, recommendations를 명시한다.

```bash
mkdir -p /tmp/spire-lab
cat > /tmp/spire-lab/values.yaml <<EOF
global:
  openshift: false
  spire:
    recommendations:
      enabled: true
    namespaces:
      create: true
      server:
        name: ${LAB_SPIRE_SERVER_NAMESPACE}
      system:
        name: ${LAB_SPIRE_SYSTEM_NAMESPACE}
    clusterName: ${LAB_CLUSTER_NAME}
    trustDomain: ${LAB_TRUST_DOMAIN}
    caSubject:
      country: ZZ
      organization: SPIRE Lab
      commonName: ${LAB_TRUST_DOMAIN}

# Lab note:
# 기본 ClusterSPIFFEID는 모든 Pod에
# spiffe://<trust-domain>/ns/<namespace>/sa/<service-account>
# 형태의 ID를 줄 수 있다. 운영에서는 namespaceSelector로 범위를 좁히거나
# workload별 ClusterSPIFFEID를 선언한다.
spire-server:
  controllerManager:
    identities:
      clusterSPIFFEIDs:
        default:
          enabled: true
EOF
```

파일 내용 확인:

```bash
sed -n '1,220p' /tmp/spire-lab/values.yaml
```

주의:

- 위 설정은 lab-only다.
- 운영에서는 external datastore, backup, HA, ingress/federation, telemetry, resource requests/limits, PSS, NetworkPolicy를 별도로 다룬다.
- chart value 이름은 chart version에 따라 바뀔 수 있다. `helm show values`로 확인한다.

```bash
helm show values "$LAB_HELM_REPO_NAME/spire" \
  --version "$LAB_SPIRE_CHART_VERSION" \
  > /tmp/spire-lab/spire-default-values.yaml

grep -nE 'trustDomain|clusterName|controllerManager|clusterSPIFFEIDs|recommendations|namespaces' \
  /tmp/spire-lab/spire-default-values.yaml | head -80
```

## 설치

CRD chart를 먼저 설치한다.

```bash
helm upgrade --install --create-namespace \
  -n "$LAB_MGMT_NAMESPACE" \
  spire-crds "$LAB_HELM_REPO_NAME/spire-crds" \
  --version "$LAB_SPIRE_CRDS_CHART_VERSION"
```

SPIRE stack을 설치한다.

```bash
helm upgrade --install \
  -n "$LAB_MGMT_NAMESPACE" \
  "$LAB_RELEASE_NAME" "$LAB_HELM_REPO_NAME/spire" \
  --version "$LAB_SPIRE_CHART_VERSION" \
  -f /tmp/spire-lab/values.yaml
```

배포 상태 확인:

```bash
helm -n "$LAB_MGMT_NAMESPACE" list
helm -n "$LAB_MGMT_NAMESPACE" status "$LAB_RELEASE_NAME"
kubectl get ns | grep -E 'spire|identity-demo'
kubectl get pods -n "$LAB_SPIRE_SERVER_NAMESPACE" -o wide
kubectl get pods -n "$LAB_SPIRE_SYSTEM_NAMESPACE" -o wide
```

rollout 대기:

```bash
kubectl -n "$LAB_SPIRE_SERVER_NAMESPACE" rollout status statefulset/spire-server --timeout=180s
kubectl -n "$LAB_SPIRE_SYSTEM_NAMESPACE" rollout status daemonset/spire-agent --timeout=180s
kubectl -n "$LAB_SPIRE_SYSTEM_NAMESPACE" rollout status daemonset/spiffe-csi-driver --timeout=180s
```

리소스 이름은 chart version에 따라 다를 수 있다. 실패하면 먼저 실제 이름을 확인한다.

```bash
kubectl -n "$LAB_SPIRE_SERVER_NAMESPACE" get deploy,sts,ds,svc,cm,secret
kubectl -n "$LAB_SPIRE_SYSTEM_NAMESPACE" get deploy,sts,ds,svc,cm,secret
```

## 설치 검증

Server log:

```bash
kubectl -n "$LAB_SPIRE_SERVER_NAMESPACE" logs statefulset/spire-server -c spire-server --tail=200
```

Agent log:

```bash
kubectl -n "$LAB_SPIRE_SYSTEM_NAMESPACE" logs daemonset/spire-agent -c spire-agent --tail=200
```

Controller Manager log:

```bash
kubectl -n "$LAB_SPIRE_SERVER_NAMESPACE" logs statefulset/spire-server -c spire-controller-manager --tail=200
```

컨테이너 이름이 다르면 확인 후 바꾼다.

```bash
kubectl -n "$LAB_SPIRE_SERVER_NAMESPACE" get pod spire-server-0 \
  -o jsonpath='{range .spec.containers[*]}{.name}{"\n"}{end}'
```

CRD 확인:

```bash
kubectl api-resources | grep -Ei 'spiffe|spire'
kubectl get clusterspiffeids
kubectl get clusterstaticentries 2>/dev/null || true
kubectl get clusterfederatedtrustdomains 2>/dev/null || true
```

기본 ClusterSPIFFEID가 있다면 상세 확인:

```bash
kubectl get clusterspiffeids -o yaml
```

기대할 수 있는 기본 ID 형태:

```text
spiffe://lab.example.org/ns/<pod-namespace>/sa/<service-account-name>
```

이는 chart 문서의 기본 ClusterSPIFFEID 설명과 맞다. 운영에서는 모든 Pod에 기본 ID를 주는 정책이 맞는지 반드시 검토한다.

## 실습 workload 배포

service account를 만든다.

```bash
kubectl -n "$LAB_APP_NAMESPACE" create serviceaccount spiffe-demo \
  --dry-run=client -o yaml | kubectl apply -f -
```

socket을 mount할 Pod를 만든다. 아래 manifest는 lab-only다. image는 `alpine`을 사용해 socket mount와 파일 확인을 위한 대기 Pod를 만든다.

```bash
cat > /tmp/spire-lab/demo-pod.yaml <<'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: spiffe-demo
  namespace: identity-demo
  labels:
    app: spiffe-demo
spec:
  serviceAccountName: spiffe-demo
  restartPolicy: Always
  containers:
  - name: app
    image: alpine:3.20
    command: ["/bin/sh", "-c"]
    args:
    - |
      while true; do
        date
        ls -l /spiffe-workload-api || true
        sleep 300
      done
    securityContext:
      allowPrivilegeEscalation: false
      capabilities:
        drop: ["ALL"]
      runAsNonRoot: true
      runAsUser: 65532
      seccompProfile:
        type: RuntimeDefault
    volumeMounts:
    - name: spiffe-workload-api
      mountPath: /spiffe-workload-api
      readOnly: true
  volumes:
  - name: spiffe-workload-api
    csi:
      driver: "csi.spiffe.io"
      readOnly: true
EOF

kubectl apply -f /tmp/spire-lab/demo-pod.yaml
kubectl -n "$LAB_APP_NAMESPACE" wait --for=condition=Ready pod/spiffe-demo --timeout=180s
```

Pod 확인:

```bash
kubectl -n "$LAB_APP_NAMESPACE" get pod spiffe-demo -o wide
kubectl -n "$LAB_APP_NAMESPACE" describe pod spiffe-demo
kubectl -n "$LAB_APP_NAMESPACE" exec spiffe-demo -- ls -l /spiffe-workload-api
```

일반적으로 CSI mount 안에 Workload API socket이 보인다. socket 파일 이름은 chart/driver 구성에 따라 다를 수 있으므로 직접 확인한다.

```bash
kubectl -n "$LAB_APP_NAMESPACE" exec spiffe-demo -- \
  sh -c 'find /spiffe-workload-api -maxdepth 2 \( -type s -o -type l -o -type f \) -print'
```

## 명시적인 ClusterSPIFFEID 추가

기본 ClusterSPIFFEID만으로도 SVID를 받을 수 있지만, 학습을 위해 workload 전용 ID를 추가한다. 운영에서는 이 패턴이 더 안전하다.

```bash
cat > /tmp/spire-lab/demo-clusterspiffeid.yaml <<EOF
apiVersion: spire.spiffe.io/v1alpha1
kind: ClusterSPIFFEID
metadata:
  name: identity-demo-spiffe-demo
spec:
  className: "spiffe"
  spiffeIDTemplate: "spiffe://${LAB_TRUST_DOMAIN}/ns/{{ .PodMeta.Namespace }}/sa/{{ .PodSpec.ServiceAccountName }}/app/spiffe-demo"
  podSelector:
    matchLabels:
      app: spiffe-demo
  namespaceSelector:
    matchLabels:
      kubernetes.io/metadata.name: ${LAB_APP_NAMESPACE}
  dnsNameTemplates:
  - "spiffe-demo.${LAB_APP_NAMESPACE}.svc"
EOF

kubectl apply -f /tmp/spire-lab/demo-clusterspiffeid.yaml
kubectl get clusterspiffeid identity-demo-spiffe-demo -o yaml
```

주의:

- `className` 기본값이나 필요 여부는 controller-manager chart/version에 따라 다를 수 있다.
- CRD schema가 위 manifest를 거부하면 `kubectl explain clusterspiffeid.spec`로 현재 필드를 확인한다.

```bash
kubectl explain clusterspiffeid.spec
kubectl describe clusterspiffeid identity-demo-spiffe-demo
kubectl -n "$LAB_SPIRE_SERVER_NAMESPACE" logs statefulset/spire-server -c spire-controller-manager --tail=200
```

## SVID를 안전하게 가져오기

SVID 확인은 lab 전용 debug Pod에서 한다. 운영 애플리케이션에서는 language SDK나 proxy가 Workload API stream을 직접 소비해야 한다.

### 방법 A: 공식 spire-agent image를 one-shot client로 사용

공식 SPIRE image tag는 release version에서 leading `v`를 뺀 `1.15.2` 형태다.
`v1.15.2`의 공식 `spire-agent` image는 `scratch` 기반이라 `/bin/sh`, `ls`,
`openssl` 같은 일반 도구가 없다. shell script 대신 `spire-agent api fetch x509`
binary를 직접 실행한다.

```bash
cat > /tmp/spire-lab/svid-fetcher.yaml <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: svid-fetcher
  namespace: ${LAB_APP_NAMESPACE}
  labels:
    app: spiffe-demo
spec:
  serviceAccountName: spiffe-demo
  restartPolicy: Never
  containers:
  - name: fetcher
    image: ghcr.io/spiffe/spire-agent:${LAB_SPIRE_APP_VERSION}
    command: ["/opt/spire/bin/spire-agent"]
    args:
    - api
    - fetch
    - x509
    - -socketPath
    - /spiffe-workload-api/spire-agent.sock
    securityContext:
      allowPrivilegeEscalation: false
      capabilities:
        drop: ["ALL"]
      readOnlyRootFilesystem: true
      runAsNonRoot: true
      runAsUser: 65532
      seccompProfile:
        type: RuntimeDefault
    volumeMounts:
    - name: spiffe-workload-api
      mountPath: /spiffe-workload-api
      readOnly: true
  volumes:
  - name: spiffe-workload-api
    csi:
      driver: "csi.spiffe.io"
      readOnly: true
EOF

kubectl apply -f /tmp/spire-lab/svid-fetcher.yaml
kubectl -n "$LAB_APP_NAMESPACE" logs -f pod/svid-fetcher
```

정상이라면 CLI가 SPIFFE ID와 SVID 유효 기간을 출력한다. private key PEM은
출력하거나 공유하지 않는다. 공식 image에는 shell이 없으므로 `kubectl exec ... -- sh`
또는 `openssl` 진단을 이 Pod에 시도하지 않는다. 더 자세한 검사는 공식 SPIFFE SDK나
검토한 version-pinned debug image로 수행한다.

socket 파일 이름이 다르면 manifest의 `-socketPath`를 실제 경로로 바꾼다.
non-root UID가 socket을 열 수 없다면 root로 완화하기 전에 CSI driver의 공식 socket
permission 설정과 chart values를 확인한다.

```bash
kubectl -n "$LAB_APP_NAMESPACE" get pod svid-fetcher -o yaml | grep -A20 spiffe-workload-api
kubectl -n "$LAB_APP_NAMESPACE" logs pod/svid-fetcher
```

### 방법 B: SDK 샘플 workflow 사용

공식 quickstart는 workload container가 Workload API로 X.509-SVID를 가져오는 흐름을 보여준다. 더 실제적인 client/server mTLS는 SPIFFE/SPIRE 예제 repository나 사용하는 언어 SDK의 mTLS 예제를 사용한다.

개념적으로 client와 server는 다음을 해야 한다.

1. 각자 Workload API에서 X.509-SVID와 bundle을 구독한다.
2. TLS server는 server certificate로 자신의 SVID를 제시한다.
3. TLS client는 server certificate의 SPIFFE ID가 기대한 ID인지 검증한다.
4. TLS client도 client certificate를 제시한다.
5. server는 client SPIFFE ID를 검증한 뒤 authorization policy를 평가한다.

실습에서는 직접 private key를 추출해서 오래 저장하지 않는다. SDK가 memory에서 certificate source를 갱신하도록 한다.

## mTLS 개념 검증 checklist

실제 샘플을 붙일 때는 다음을 확인한다.

```bash
# client Pod와 server Pod가 서로 다른 service account를 쓰는지 확인
kubectl -n "$LAB_APP_NAMESPACE" get pod -o custom-columns=NAME:.metadata.name,SA:.spec.serviceAccountName

# 각 Pod가 받는 SPIFFE ID 확인
kubectl -n "$LAB_APP_NAMESPACE" logs pod/<CLIENT_POD>
kubectl -n "$LAB_APP_NAMESPACE" logs pod/<SERVER_POD>

# server가 client SPIFFE ID allowlist를 가지고 있는지 확인
kubectl -n "$LAB_APP_NAMESPACE" get cm,secret
```

허용 policy 예:

```text
allow client:
  spiffe://lab.example.org/ns/identity-demo/sa/spiffe-demo/app/spiffe-demo

deny:
  trust domain mismatch
  namespace mismatch
  service account mismatch
  expired SVID
```

## Rotation 관찰

SVID TTL은 chart/server 설정과 entry 설정에 따라 다르다. 먼저 현재 SVID의 만료 시각을 기록한다.

```bash
kubectl -n "$LAB_APP_NAMESPACE" delete pod svid-fetcher --ignore-not-found
kubectl apply -f /tmp/spire-lab/svid-fetcher.yaml
kubectl -n "$LAB_APP_NAMESPACE" logs -f pod/svid-fetcher | tee /tmp/spire-lab/svid-fetcher.log
grep -E 'notBefore|notAfter|URI:spiffe' /tmp/spire-lab/svid-fetcher.log
```

streaming client로 rotation을 보려면 `spire-agent api fetch x509`를 종료하지 않는 방식이나 SDK 샘플을 사용한다. 단발 fetch는 “현재 SVID 확인”에 가깝다.

관찰 포인트:

- `notAfter`가 지나기 전에 새 SVID가 내려오는가
- 애플리케이션이 TLS config를 갱신하는가
- peer connection이 재시작 없이 새 certificate를 사용하는가
- Server/Agent log에 signing error나 reconnect가 없는가

## Telemetry와 관측

helm-charts-hardened recommendations에는 Prometheus exporter 노출 recommendation이 포함된다. SPIRE v1.15.2 release에는 Prometheus metrics endpoint TLS 지원 관련 변경도 포함되어 있다.

주의: SPIRE telemetry 문서 일부는 최신 release보다 version 표시가 뒤처질 수 있다. 문서의 “Latest” 표시와 실제 release/changelog를 함께 확인한다.

기본 확인:

```bash
kubectl -n "$LAB_SPIRE_SERVER_NAMESPACE" get svc -o wide
kubectl -n "$LAB_SPIRE_SYSTEM_NAMESPACE" get svc -o wide
kubectl -n "$LAB_SPIRE_SERVER_NAMESPACE" get servicemonitor,podmonitor 2>/dev/null || true
kubectl -n "$LAB_SPIRE_SYSTEM_NAMESPACE" get servicemonitor,podmonitor 2>/dev/null || true
```

로그에서 볼 것:

```bash
kubectl -n "$LAB_SPIRE_SERVER_NAMESPACE" logs statefulset/spire-server -c spire-server --tail=300 | \
  grep -Ei 'error|warn|attest|svid|bundle|entry|jwt|x509' || true

kubectl -n "$LAB_SPIRE_SYSTEM_NAMESPACE" logs daemonset/spire-agent -c spire-agent --tail=300 | \
  grep -Ei 'error|warn|attest|workload|selector|svid|socket' || true
```

운영 alert 후보:

- Server unavailable
- Agent unavailable per node
- SVID signing error rate
- Workload API error rate
- node attestation failure
- Controller Manager reconciliation error
- datastore latency/error
- bundle endpoint failure
- certificate expiration window

## HA와 production hardening

이 lab values는 production ready가 아니다. 운영 전에는 최소한 다음을 결정한다.

### Server HA와 datastore

- StatefulSet replica와 leader/lock 동작 확인
- external SQL datastore 또는 지원되는 production datastore 선택
- datastore backup/restore drill
- signing key 보호 방식
- trust bundle rotation 절차
- disaster recovery RTO/RPO

### Namespace와 Pod Security

- `spire-server` namespace는 restricted에 가깝게 유지
- `spire-system` namespace는 CSI/DaemonSet 때문에 privileged가 필요할 수 있음
- namespace 생성과 label ownership 문서화
- 누가 `ClusterSPIFFEID`를 만들 수 있는지 RBAC 제한

### Registration policy

- 기본 ClusterSPIFFEID를 모든 namespace에 열지 않는다.
- namespaceSelector로 적용 범위를 줄인다.
- service account별 ID를 만든다.
- label selector를 권한 근거로 쓸 때 label 변경 권한을 admission/RBAC으로 제한한다.
- one workload, one meaningful identity 원칙을 우선한다.

### Network와 exposure

- Server API는 필요한 Agent/Controller Manager만 접근
- federation bundle endpoint만 필요한 경우에 노출
- OIDC Discovery Provider는 JWT-SVID 외부 검증이 필요할 때만 노출
- ingress TLS와 DNS ownership 확인

### Secret과 key 관리

- SPIRE Server signing key 백업과 접근통제
- chart values에 secret material 평문 저장 금지
- SVID private key 파일 덤프 금지
- debug command 결과 수거와 삭제

### Upgrade

- chart major/minor skip 금지. 공식 chart upgrade 문서는 한 major/minor씩 올릴 것을 설명한다.
- CRD upgrade notes 먼저 확인
- staging에서 SVID 발급, rotation, federation, JWT validation 재검증
- rollback 시 datastore schema와 CRD 호환성 확인

## Failure injection

운영이 아닌 lab에서만 수행한다.

### 1. socket mount 제거

목적: delivery 문제와 registration 문제를 구분한다.

```bash
kubectl -n "$LAB_APP_NAMESPACE" delete pod spiffe-demo --ignore-not-found

kubectl -n "$LAB_APP_NAMESPACE" run no-socket \
  --image=alpine:3.20 \
  --serviceaccount=spiffe-demo \
  --restart=Never \
  --command -- sh -c 'ls -l /spiffe-workload-api; sleep 30'

kubectl -n "$LAB_APP_NAMESPACE" logs pod/no-socket
```

기대:

- `/spiffe-workload-api`가 없다.
- 이 경우 SPIRE registration이 정상이어도 workload는 Workload API에 연결할 수 없다.

정리:

```bash
kubectl -n "$LAB_APP_NAMESPACE" delete pod no-socket --ignore-not-found
kubectl apply -f /tmp/spire-lab/demo-pod.yaml
```

### 2. selector 불일치

목적: ClusterSPIFFEID가 Pod를 매칭하지 못할 때 증상을 본다.

```bash
kubectl -n "$LAB_APP_NAMESPACE" label pod spiffe-demo app=wrong --overwrite
kubectl describe clusterspiffeid identity-demo-spiffe-demo
kubectl -n "$LAB_SPIRE_SERVER_NAMESPACE" logs statefulset/spire-server -c spire-controller-manager --tail=200
```

다시 복구:

```bash
kubectl -n "$LAB_APP_NAMESPACE" label pod spiffe-demo app=spiffe-demo --overwrite
kubectl describe clusterspiffeid identity-demo-spiffe-demo
```

주의: Pod label을 직접 바꾸는 것은 lab 전용이다. Deployment가 관리하는 Pod라면 재생성될 수 있다.

### 3. service account mismatch

목적: ID가 service account에 묶인다는 점을 확인한다.

```bash
kubectl -n "$LAB_APP_NAMESPACE" create serviceaccount other-sa --dry-run=client -o yaml | kubectl apply -f -
kubectl -n "$LAB_APP_NAMESPACE" run wrong-sa \
  --image=alpine:3.20 \
  --serviceaccount=other-sa \
  --labels=app=spiffe-demo \
  --restart=Never \
  --command -- sh -c 'sleep 300'

kubectl -n "$LAB_APP_NAMESPACE" get pod wrong-sa -o jsonpath='{.spec.serviceAccountName}{"\n"}'
```

기대:

- selector가 label만 보면 match될 수 있다.
- SPIFFE ID template에 service account가 들어가면 `other-sa`용 ID가 렌더링될 수 있다.
- 따라서 label만으로 권한을 판단하지 말고 namespace/service account selector를 함께 제한해야 한다.

정리:

```bash
kubectl -n "$LAB_APP_NAMESPACE" delete pod wrong-sa --ignore-not-found
kubectl -n "$LAB_APP_NAMESPACE" delete serviceaccount other-sa --ignore-not-found
```

### 4. Agent 장애 관찰

목적: 특정 node의 Workload API 장애 증상을 본다.

DaemonSet을 직접 삭제하거나 scale할 수는 없으므로 운영과 공유되는 cluster에서는 하지 않는다. lab에서만 특정 Agent Pod를 삭제해 재생성을 관찰한다.

```bash
kubectl -n "$LAB_SPIRE_SYSTEM_NAMESPACE" get pod -l app.kubernetes.io/name=agent -o wide
kubectl -n "$LAB_SPIRE_SYSTEM_NAMESPACE" delete pod <SPIRE_AGENT_POD_NAME>
kubectl -n "$LAB_SPIRE_SYSTEM_NAMESPACE" get pod -w
```

복구 확인:

```bash
kubectl -n "$LAB_SPIRE_SYSTEM_NAMESPACE" rollout status daemonset/spire-agent --timeout=180s
kubectl -n "$LAB_SPIRE_SYSTEM_NAMESPACE" logs daemonset/spire-agent -c spire-agent --tail=200
```

## Troubleshooting map

### Server Pod가 Pending

확인:

```bash
kubectl -n "$LAB_SPIRE_SERVER_NAMESPACE" get pod,pvc
kubectl -n "$LAB_SPIRE_SERVER_NAMESPACE" describe pod spire-server-0
kubectl -n "$LAB_SPIRE_SERVER_NAMESPACE" describe pvc
kubectl get storageclass
```

가능한 원인:

- 기본 StorageClass 없음
- persistence storageClass 오기입
- quota 부족

### Agent가 Ready가 아님

확인:

```bash
kubectl -n "$LAB_SPIRE_SYSTEM_NAMESPACE" get pod -o wide
kubectl -n "$LAB_SPIRE_SYSTEM_NAMESPACE" describe pod <AGENT_POD_NAME>
kubectl -n "$LAB_SPIRE_SYSTEM_NAMESPACE" logs <AGENT_POD_NAME> -c spire-agent --tail=300
```

가능한 원인:

- node attestation 실패
- Server address/service 접근 불가
- ServiceAccount token/audience 문제
- hostPath/socket 권한 문제

### ClusterSPIFFEID가 적용되지 않음

확인:

```bash
kubectl get clusterspiffeid identity-demo-spiffe-demo -o yaml
kubectl describe clusterspiffeid identity-demo-spiffe-demo
kubectl -n "$LAB_APP_NAMESPACE" get pod --show-labels
kubectl -n "$LAB_APP_NAMESPACE" get ns --show-labels
kubectl -n "$LAB_SPIRE_SERVER_NAMESPACE" logs statefulset/spire-server -c spire-controller-manager --tail=300
```

가능한 원인:

- namespaceSelector 불일치
- podSelector 불일치
- template rendering error
- Controller Manager가 Server API에 접근 실패
- CRD schema와 manifest version mismatch

### Workload API socket이 없음

확인:

```bash
kubectl -n "$LAB_APP_NAMESPACE" describe pod spiffe-demo
kubectl -n "$LAB_APP_NAMESPACE" get pod spiffe-demo -o yaml | grep -A20 -B5 'csi.spiffe.io'
kubectl -n "$LAB_SPIRE_SYSTEM_NAMESPACE" logs daemonset/spiffe-csi-driver --tail=200
```

가능한 원인:

- CSI driver 미설치/NotReady
- Pod volume manifest 누락
- driver 이름 mismatch
- Pod Security/admission에서 CSI volume 거부

### SVID fetch가 PermissionDenied

확인:

```bash
kubectl -n "$LAB_APP_NAMESPACE" logs pod/svid-fetcher
kubectl -n "$LAB_SPIRE_SYSTEM_NAMESPACE" logs daemonset/spire-agent -c spire-agent --tail=300 | grep -Ei 'denied|selector|workload|svid' || true
kubectl get clusterspiffeids -o yaml
```

가능한 원인:

- workload selector와 registration entry가 매칭되지 않음
- parentID/Agent identity mismatch
- Pod가 예상 service account가 아님
- 기본 ClusterSPIFFEID를 꺼놓고 workload 전용 ID가 없음

### mTLS handshake 실패

확인:

```bash
kubectl -n "$LAB_APP_NAMESPACE" logs pod/<CLIENT_POD> --tail=200
kubectl -n "$LAB_APP_NAMESPACE" logs pod/<SERVER_POD> --tail=200
```

분류:

- certificate chain 검증 실패: bundle 문제
- SPIFFE ID mismatch: authorization allowlist 문제
- expired certificate: rotation/client reload 문제
- unknown authority: federation/bundle 전달 문제
- connection refused: identity 이전의 network/service 문제

## Cleanup

실습 workload 삭제:

```bash
kubectl delete -f /tmp/spire-lab/svid-fetcher.yaml --ignore-not-found
kubectl delete -f /tmp/spire-lab/demo-pod.yaml --ignore-not-found
kubectl delete -f /tmp/spire-lab/demo-clusterspiffeid.yaml --ignore-not-found
kubectl -n "$LAB_APP_NAMESPACE" delete serviceaccount spiffe-demo --ignore-not-found
kubectl delete namespace "$LAB_APP_NAMESPACE" --ignore-not-found
```

SPIRE stack 삭제:

```bash
helm -n "$LAB_MGMT_NAMESPACE" uninstall "$LAB_RELEASE_NAME" || true
helm -n "$LAB_MGMT_NAMESPACE" uninstall spire-crds || true
```

CRD와 namespace 삭제는 신중히 한다. 같은 cluster에 다른 SPIRE 실습이 있으면 같이 깨질 수 있다.

```bash
# lab 전용 cluster에서만 실행한다.
kubectl delete crd -l app.kubernetes.io/part-of=spire --ignore-not-found
kubectl delete namespace "$LAB_SPIRE_SERVER_NAMESPACE" "$LAB_SPIRE_SYSTEM_NAMESPACE" "$LAB_MGMT_NAMESPACE" --ignore-not-found
```

임시 파일 삭제:

```bash
rm -rf /tmp/spire-lab
```

## 복습 문제

1. `spiffe://lab.example.org/ns/identity-demo/sa/spiffe-demo`에서 trust domain은 무엇인가?
2. SPIRE Server와 Agent 중 Workload API socket을 제공하는 쪽은 어느 쪽인가?
3. Controller Manager를 쓰는 환경에서 `spire-server entry create`로 수동 등록하면 어떤 문제가 생길 수 있는가?
4. X.509-SVID가 JWT-SVID보다 mTLS에 적합한 이유는 무엇인가?
5. 기본 ClusterSPIFFEID가 모든 Pod에 ID를 주는 구성의 위험은 무엇인가?
6. SVID fetch가 `PermissionDenied`일 때 registration 문제와 socket mount 문제를 어떻게 구분하는가?
7. federation이 authentication을 가능하게 해도 authorization을 자동 허용하지 않는 이유는 무엇인가?
8. 이미 발급된 SVID를 즉시 무효화하기 어렵다면 즉시 차단은 어디에서 해야 하는가?

## 정답 방향

1. `lab.example.org`
2. SPIRE Agent
3. Controller Manager reconcile이 수동 변경을 되돌리거나 drift를 만든다.
4. X.509-SVID는 TLS handshake에서 private key 소유를 증명하고 peer certificate를 bundle로 검증한다. JWT-SVID는 bearer token이라 replay 위험이 더 크다.
5. namespace/service account/RBAC가 약한 workload도 identity를 받을 수 있어 blast radius가 커진다.
6. socket mount 문제는 파일/socket 자체가 없고, registration 문제는 socket 연결 후 권한 거부나 selector mismatch log가 나온다.
7. federation은 상대 trust domain의 SVID를 검증할 수 있게 할 뿐, 업무 API 호출 허용 여부는 별도 정책이다.
8. application/proxy/API gateway/OPA/OpenBao 같은 authorization layer에서 deny한다.

## 참고

- SPIRE v1.15.2 release: https://github.com/spiffe/spire/releases
- SPIRE changelog: https://github.com/spiffe/spire/blob/main/CHANGELOG.md
- SPIRE downloads: https://spiffe.io/downloads/
- Kubernetes quickstart: https://spiffe.io/docs/latest/try/getting-started-k8s/
- About SPIRE Helm Charts Hardened: https://spiffe.io/docs/latest/spire-helm-charts-hardened-about/
- Helm chart installation: https://spiffe.io/docs/latest/spire-helm-charts-hardened-about/installation/
- Helm chart recommendations: https://spiffe.io/docs/latest/spire-helm-charts-hardened-about/recommendations/
- Helm chart namespaces: https://spiffe.io/docs/latest/spire-helm-charts-hardened-about/namespaces/
- Helm chart service selection: https://spiffe.io/docs/latest/spire-helm-charts-hardened-about/service-selection/
- Helm chart identifiers: https://spiffe.io/docs/latest/spire-helm-charts-hardened-about/identifiers/
- Helm chart upgrading: https://spiffe.io/docs/latest/spire-helm-charts-hardened-about/upgrading/
- Exposing SPIRE services: https://spiffe.io/docs/latest/spire-helm-charts-hardened-about/exposing/
- SPIRE Controller Manager: https://github.com/spiffe/spire-controller-manager
- SPIFFE Workload API: https://spiffe.io/docs/latest/spiffe-specs/spiffe_workload_api/
