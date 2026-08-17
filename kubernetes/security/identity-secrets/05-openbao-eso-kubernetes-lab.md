# OpenBao + ESO Kubernetes lab

## 요약

이 lab은 Kubernetes 안에 OpenBao와 External Secrets Operator(ESO)를 설치하고, OpenBao KV v2 값을 ESO가 Kubernetes `Secret`으로 동기화하는 전체 흐름을 검증합니다.

실습 범위:

- OpenBao 설치와 안전한 init/unseal 경고
- KV v2 mount와 sample secret
- 최소 권한 OpenBao policy
- Kubernetes auth: projected short-lived ServiceAccount token + TokenReview
- namespaced `SecretStore` 우선 구성
- 선택적 `ClusterSecretStore`
- `ExternalSecret` data/dataFrom/template
- rotation, deletion, failure injection
- cleanup
- production promotion checklist

> 공개 runbook 규칙: root token, unseal key, recovery key, OpenBao client token, AppRole secret_id를 문서, Git, shell history, issue, PR comment에 쓰지 않습니다.

## 버전 기준과 검증 caveat

2026-08-17 기준 문서 확인:

- OpenBao latest caveat: v2.6.1, 2026-07-22
- ESO latest caveat: v2.9.0, 2026-08-07, chart 2.9.0
- ESO current docs의 core manifest apiVersion: `external-secrets.io/v1`
- ESO `PushSecret`/`ClusterPushSecret`: `external-secrets.io/v1alpha1`
- ESO generators: `generators.external-secrets.io/v1alpha1`
- OpenBao dedicated provider: alpha, tested only ESO v0.16.1 + OpenBao v2.2.0
- OpenBao dedicated provider 명시 범위: KV only, auth AppRole/Kubernetes/token/UserPass
- VaultDynamicSecret-on-OpenBao: compatibility inference입니다. 이 lab은 KV v2 sync를 검증하고, dynamic secret은 production promotion 전에 별도 lab에서 검증합니다.

공식 URL:

- OpenBao release notes: <https://openbao.org/community/release-notes/2-6-0/>
- OpenBao Helm Kubernetes auth example: <https://openbao.org/docs/2.5.x/platform/k8s/helm/examples/kubernetes-auth/>
- OpenBao Kubernetes auth: <https://openbao.org/docs/2.4.x/auth/kubernetes/>
- OpenBao KV v2: <https://openbao.org/docs/secrets/kv/kv-v2/>
- OpenBao seal/unseal: <https://openbao.org/docs/next/concepts/seal/>
- ESO getting started/API docs: <https://external-secrets.io/latest/api/externalsecret/>
- ESO SecretStore: <https://external-secrets.io/main/api/secretstore/>
- ESO Vault provider: <https://external-secrets.io/latest/provider/hashicorp-vault/>
- ESO OpenBao provider: <https://external-secrets.io/main/provider/openbao/>

배포 전 실제 CRD 확인:

```bash
kubectl api-resources | rg 'external-secrets|generators'
kubectl explain externalsecret.spec
kubectl explain secretstore.spec.provider.vault
```

## 전제조건

필요 도구:

- Kubernetes cluster
- `kubectl`
- `helm`
- `jq`
- `rg`
- OpenBao CLI가 container image 안에 있거나 `kubectl exec`로 사용할 수 있는 환경

namespace:

```bash
kubectl create namespace openbao --dry-run=client -o yaml | kubectl apply -f -
kubectl create namespace external-secrets --dry-run=client -o yaml | kubectl apply -f -
kubectl create namespace demo --dry-run=client -o yaml | kubectl apply -f -
```

label:

```bash
kubectl label namespace demo secrets.example.com/tenant=demo --overwrite
```

## Helm repository

네트워크 접근이 가능한 운영 터미널에서 실행합니다.

```bash
helm repo add openbao https://openbao.github.io/openbao-helm
helm repo add external-secrets https://charts.external-secrets.io
helm repo update
```

chart version 확인:

```bash
helm search repo openbao/openbao --versions | head
helm search repo external-secrets/external-secrets --versions | head
```

ESO chart는 요청 기준 latest caveat가 2.9.0입니다. 실제 운영에서는 chart values와 CRD migration note를 함께 확인합니다.

## OpenBao 설치

Lab용 standalone file storage 예입니다. 운영 promotion 전에는 HA Raft, TLS, audit, backup을 별도 설계합니다.

```bash
helm upgrade --install openbao openbao/openbao \
  --namespace openbao \
  --set server.dev.enabled=false \
  --set server.standalone.enabled=true \
  --set server.ha.enabled=false \
  --set='server.standalone.config=ui = true

listener "tcp" {
  address = "[::]:8200"
  cluster_address = "[::]:8201"
  tls_disable = true
}

storage "file" {
  path = "/openbao/data"
}

api_addr = "http://openbao.openbao.svc.cluster.local:8200"'
```

주의:

- `tls_disable=true`는 lab 전용입니다.
- 운영에서는 TLS listener와 CA 배포를 구성합니다.
- file storage standalone은 HA가 아닙니다.
- sealed 상태를 readiness success로 만들면 Pod는 Ready인데 ESO는 실패할 수 있습니다.

Pod 확인:

```bash
kubectl -n openbao rollout status statefulset/openbao
kubectl -n openbao get pod,svc,pvc
```

## init 전 안전 경고

먼저 initialized 여부를 확인합니다.

```bash
kubectl -n openbao exec openbao-0 -- sh -lc '
  export BAO_ADDR=http://127.0.0.1:8200
  bao operator init -status
'
```

`OpenBao is initialized`라면 init을 다시 실행하지 않습니다.

새 lab storage에서만 init합니다. 실제 unseal key/root token 출력은 터미널에 표시됩니다. 문서에 복사하지 않습니다. 가능하면 PGP 암호화를 사용합니다.

```bash
kubectl -n openbao exec -it openbao-0 -- sh -lc '
  export BAO_ADDR=http://127.0.0.1:8200
  bao operator init -key-shares=5 -key-threshold=3
'
```

운영 권장:

- init 출력은 승인된 secret ceremony에서만 다룹니다.
- 화면 공유와 터미널 녹화를 끕니다.
- root token은 bootstrap 후 revoke 또는 break-glass 금고에 보관합니다.
- unseal key share는 서로 다른 보관자에게 나눕니다.
- GitOps repo에 절대 넣지 않습니다.

## unseal

unseal key를 command argument에 넣지 않습니다.

```bash
kubectl -n openbao exec -it openbao-0 -- sh -lc '
  export BAO_ADDR=http://127.0.0.1:8200
  bao operator unseal
'
```

threshold를 만족할 때까지 반복합니다.

상태 확인:

```bash
kubectl -n openbao exec openbao-0 -- sh -lc '
  export BAO_ADDR=http://127.0.0.1:8200
  bao status
'
```

필수:

```text
Initialized     true
Sealed          false
```

## root token 로그인

root token을 인자로 넣지 않습니다.

```bash
kubectl -n openbao exec -it openbao-0 -- sh -lc '
  export BAO_ADDR=http://127.0.0.1:8200
  bao login
'
```

이 세션은 bootstrap에만 사용합니다.

## audit enable

Lab에서는 stdout 또는 file audit를 선택합니다. 운영에서는 declarative audit config와 log pipeline을 설계합니다.

```bash
kubectl -n openbao exec openbao-0 -- sh -lc '
  export BAO_ADDR=http://127.0.0.1:8200
  bao audit list
'
```

file audit:

```bash
kubectl -n openbao exec openbao-0 -- sh -lc '
  export BAO_ADDR=http://127.0.0.1:8200
  mkdir -p /openbao/audit
  bao audit enable file file_path=/openbao/audit/audit.log
'
```

OpenBao v2.3.2+에서는 API/CLI audit creation에 `unsafe_allow_api_audit_creation=true`가 필요할 수 있습니다. 실패하면 server config 방식으로 전환합니다.

## KV v2 준비

KV v2 mount 확인:

```bash
kubectl -n openbao exec openbao-0 -- sh -lc '
  export BAO_ADDR=http://127.0.0.1:8200
  bao secrets list
'
```

`secret/`이 없으면 enable:

```bash
kubectl -n openbao exec openbao-0 -- sh -lc '
  export BAO_ADDR=http://127.0.0.1:8200
  bao secrets enable -path=secret kv-v2
'
```

sample secret 작성:

```bash
kubectl -n openbao exec -i openbao-0 -- sh -lc '
  export BAO_ADDR=http://127.0.0.1:8200
  bao kv put -mount=secret apps/demo/database \
    username=demo_app \
    password=-
' <<'EOF'
CHANGE-ME-LAB-PASSWORD
EOF
```

이 값은 lab placeholder입니다. 운영 비밀번호는 터미널 argument나 Git에 넣지 않습니다.

읽기:

```bash
kubectl -n openbao exec openbao-0 -- sh -lc '
  export BAO_ADDR=http://127.0.0.1:8200
  bao kv get -mount=secret apps/demo/database
'
```

version 확인:

```bash
kubectl -n openbao exec openbao-0 -- sh -lc '
  export BAO_ADDR=http://127.0.0.1:8200
  bao kv metadata get -mount=secret apps/demo/database
'
```

## ESO 설치

CRD 포함 설치:

```bash
helm upgrade --install external-secrets external-secrets/external-secrets \
  --namespace external-secrets \
  --version 2.9.0 \
  --set installCRDs=true
```

rollout:

```bash
kubectl -n external-secrets rollout status deploy/external-secrets
kubectl -n external-secrets rollout status deploy/external-secrets-webhook
kubectl -n external-secrets rollout status deploy/external-secrets-cert-controller
```

CRD 확인:

```bash
kubectl get crd | rg 'external-secrets|generators'
kubectl api-resources | rg 'ExternalSecret|SecretStore|PushSecret|Generator|external-secrets'
```

## Kubernetes auth service account

ESO가 OpenBao에 로그인할 때 사용할 demo namespace service account를 만듭니다.

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: eso-openbao
  namespace: demo
```

적용:

```bash
kubectl apply -f - <<'EOF'
apiVersion: v1
kind: ServiceAccount
metadata:
  name: eso-openbao
  namespace: demo
EOF
```

OpenBao server service account가 TokenReview API를 호출할 수 있어야 합니다. chart가 만든 service account 이름을 확인합니다.

```bash
kubectl -n openbao get serviceaccount
```

예시는 `openbao` service account를 가정합니다.

```bash
kubectl apply -f - <<'EOF'
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: openbao-tokenreview
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:auth-delegator
subjects:
  - kind: ServiceAccount
    name: openbao
    namespace: openbao
EOF
```

## Kubernetes auth enable/config

OpenBao 안에서 Kubernetes auth method를 활성화합니다.

```bash
kubectl -n openbao exec openbao-0 -- sh -lc '
  export BAO_ADDR=http://127.0.0.1:8200
  bao auth list | grep -q "^kubernetes/" || bao auth enable kubernetes
'
```

OpenBao가 Kubernetes 안에서 돌고 있으므로 local service account token을 reviewer JWT로 사용하게 `token_reviewer_jwt`를 생략합니다.

```bash
kubectl -n openbao exec openbao-0 -- sh -lc '
  export BAO_ADDR=http://127.0.0.1:8200
  bao write auth/kubernetes/config \
    kubernetes_host="https://${KUBERNETES_SERVICE_HOST}:${KUBERNETES_SERVICE_PORT}"
'
```

config 확인:

```bash
kubectl -n openbao exec openbao-0 -- sh -lc '
  export BAO_ADDR=http://127.0.0.1:8200
  bao read auth/kubernetes/config
'
```

Kubernetes 1.21+에서는 projected short-lived service account token이 기본입니다. OpenBao Kubernetes auth는 TokenReview API를 사용해 JWT를 검증합니다.

## 최소 권한 policy

ESO가 demo path를 읽는 데 필요한 KV v2 read policy만 만듭니다.

```bash
kubectl -n openbao exec -i openbao-0 -- sh -lc '
  export BAO_ADDR=http://127.0.0.1:8200
  bao policy write eso-demo-read -
' <<'EOF'
path "secret/data/apps/demo/*" {
  capabilities = ["read"]
}

path "secret/metadata/apps/demo/*" {
  capabilities = ["list", "read"]
}
EOF
```

policy 확인:

```bash
kubectl -n openbao exec openbao-0 -- sh -lc '
  export BAO_ADDR=http://127.0.0.1:8200
  bao policy read eso-demo-read
'
```

## Kubernetes auth role

demo namespace의 `eso-openbao` service account만 이 role로 로그인할 수 있게 묶습니다.

```bash
kubectl -n openbao exec openbao-0 -- sh -lc '
  export BAO_ADDR=http://127.0.0.1:8200
  bao write auth/kubernetes/role/eso-demo \
    bound_service_account_names=eso-openbao \
    bound_service_account_namespaces=demo \
    policies=eso-demo-read \
    ttl=15m
'
```

확인:

```bash
kubectl -n openbao exec openbao-0 -- sh -lc '
  export BAO_ADDR=http://127.0.0.1:8200
  bao read auth/kubernetes/role/eso-demo
'
```

## projected short-lived token 확인

Kubernetes TokenRequest API로 service account token이 발급되는지 확인합니다. token 값은 출력하지 않습니다.

```bash
kubectl -n demo create token eso-openbao --duration=10m >/tmp/eso-openbao.jwt
wc -c /tmp/eso-openbao.jwt
rm -f /tmp/eso-openbao.jwt
```

OpenBao auth login 자체는 ESO가 수행합니다. 사람이 token을 복사해 login 테스트하는 절차는 민감하므로 이 lab에서는 생략합니다. 필요하면 임시 파일과 stdin을 쓰고, 출력 token을 남기지 않는 별도 보안 절차에서 수행합니다.

## SecretStore 우선 구성

namespaced `SecretStore`를 먼저 씁니다.

```bash
kubectl apply -f - <<'EOF'
apiVersion: external-secrets.io/v1
kind: SecretStore
metadata:
  name: openbao
  namespace: demo
spec:
  provider:
    vault:
      server: http://openbao.openbao.svc.cluster.local:8200
      path: secret
      version: v2
      auth:
        kubernetes:
          mountPath: kubernetes
          role: eso-demo
          serviceAccountRef:
            name: eso-openbao
EOF
```

검증:

```bash
kubectl -n demo get secretstore openbao
kubectl -n demo describe secretstore openbao
```

Ready 확인:

```bash
kubectl -n demo get secretstore openbao -o jsonpath='{range .status.conditions[*]}{.type}{" "}{.status}{" "}{.reason}{" "}{.message}{"\n"}{end}'
```

## ExternalSecret: data mapping

remote property를 명시적으로 매핑합니다.

```bash
kubectl apply -f - <<'EOF'
apiVersion: external-secrets.io/v1
kind: ExternalSecret
metadata:
  name: app-database
  namespace: demo
spec:
  refreshPolicy: Periodic
  refreshInterval: 1m
  secretStoreRef:
    name: openbao
    kind: SecretStore
  target:
    name: app-database
    creationPolicy: Owner
    deletionPolicy: Retain
  data:
    - secretKey: username
      remoteRef:
        key: apps/demo/database
        property: username
    - secretKey: password
      remoteRef:
        key: apps/demo/database
        property: password
EOF
```

상태:

```bash
kubectl -n demo get externalsecret app-database
kubectl -n demo describe externalsecret app-database
```

Secret key 이름만 확인:

```bash
kubectl -n demo get secret app-database -o jsonpath='{.metadata.name}{" keys="}{.data}' | sed 's/[A-Za-z0-9+/_=-]\\{8,\\}/<redacted>/g'
echo
```

값을 출력하지 않습니다.

## ExternalSecret: template

template으로 앱이 원하는 env 형태를 만듭니다.

```bash
kubectl apply -f - <<'EOF'
apiVersion: external-secrets.io/v1
kind: ExternalSecret
metadata:
  name: app-env
  namespace: demo
spec:
  refreshPolicy: Periodic
  refreshInterval: 1m
  secretStoreRef:
    name: openbao
    kind: SecretStore
  target:
    name: app-env
    creationPolicy: Owner
    deletionPolicy: Retain
    template:
      engineVersion: v2
      type: Opaque
      data:
        APP_USERNAME: "{{ .username }}"
        APP_PASSWORD: "{{ .password }}"
        DATABASE_URL: "postgres://{{ .username }}:{{ .password }}@postgres.demo.svc:5432/app"
  data:
    - secretKey: username
      remoteRef:
        key: apps/demo/database
        property: username
    - secretKey: password
      remoteRef:
        key: apps/demo/database
        property: password
EOF
```

key 확인:

```bash
kubectl -n demo get secret app-env -o jsonpath='{range $k,$v := .data}{$k}{"\n"}{end}'
```

## ExternalSecret: dataFrom

remote object 전체를 가져옵니다.

```bash
kubectl apply -f - <<'EOF'
apiVersion: external-secrets.io/v1
kind: ExternalSecret
metadata:
  name: app-database-all
  namespace: demo
spec:
  refreshPolicy: Periodic
  refreshInterval: 1m
  secretStoreRef:
    name: openbao
    kind: SecretStore
  target:
    name: app-database-all
    creationPolicy: Owner
    deletionPolicy: Retain
  dataFrom:
    - extract:
        key: apps/demo/database
EOF
```

key 확인:

```bash
kubectl -n demo get secret app-database-all -o jsonpath='{range $k,$v := .data}{$k}{"\n"}{end}'
```

## rotation 실험

OpenBao 값을 갱신합니다.

```bash
kubectl -n openbao exec -i openbao-0 -- sh -lc '
  export BAO_ADDR=http://127.0.0.1:8200
  bao kv put -mount=secret apps/demo/database \
    username=demo_app \
    password=-
' <<'EOF'
CHANGE-ME-LAB-PASSWORD-ROTATED
EOF
```

manual refresh:

```bash
stamp=$(date +%s)
kubectl -n demo annotate externalsecret app-database force-sync="$stamp" --overwrite
kubectl -n demo annotate externalsecret app-env force-sync="$stamp" --overwrite
kubectl -n demo annotate externalsecret app-database-all force-sync="$stamp" --overwrite
```

status:

```bash
kubectl -n demo get externalsecret
kubectl -n demo get externalsecret app-database -o jsonpath='{.status.refreshTime}{" "}{.status.syncedResourceVersion}{"\n"}'
```

cleanup annotation:

```bash
kubectl -n demo annotate externalsecret app-database force-sync- --overwrite
kubectl -n demo annotate externalsecret app-env force-sync- --overwrite
kubectl -n demo annotate externalsecret app-database-all force-sync- --overwrite
```

앱 반영 주의:

- env var로 주입한 Pod는 재시작 전 값이 바뀌지 않습니다.
- volume mount Secret은 kubelet sync 지연이 있습니다.
- application hot reload는 별도 구현입니다.

## deletion policy 실험

provider secret 삭제 전 현재 metadata를 봅니다.

```bash
kubectl -n openbao exec openbao-0 -- sh -lc '
  export BAO_ADDR=http://127.0.0.1:8200
  bao kv metadata get -mount=secret apps/demo/database
'
```

soft delete:

```bash
kubectl -n openbao exec openbao-0 -- sh -lc '
  export BAO_ADDR=http://127.0.0.1:8200
  bao kv delete -mount=secret apps/demo/database
'
```

ESO reconcile:

```bash
stamp=$(date +%s)
kubectl -n demo annotate externalsecret app-database force-sync="$stamp" --overwrite
kubectl -n demo describe externalsecret app-database
kubectl -n demo get secret app-database
```

`deletionPolicy: Retain`이면 target Secret은 남고 ExternalSecret은 오류 condition을 보일 수 있습니다.

복구:

```bash
kubectl -n openbao exec -i openbao-0 -- sh -lc '
  export BAO_ADDR=http://127.0.0.1:8200
  bao kv put -mount=secret apps/demo/database \
    username=demo_app \
    password=-
' <<'EOF'
CHANGE-ME-LAB-PASSWORD-RESTORED
EOF
```

다시 reconcile:

```bash
stamp=$(date +%s)
kubectl -n demo annotate externalsecret app-database force-sync="$stamp" --overwrite
kubectl -n demo get externalsecret app-database
```

## failure injection: sealed

주의: 이 실험은 secret sync를 일부러 깨뜨립니다. lab cluster에서만 합니다.

seal:

```bash
kubectl -n openbao exec openbao-0 -- sh -lc '
  export BAO_ADDR=http://127.0.0.1:8200
  bao operator seal
'
```

ESO 증상:

```bash
stamp=$(date +%s)
kubectl -n demo annotate externalsecret app-database force-sync="$stamp" --overwrite
kubectl -n demo describe externalsecret app-database
kubectl -n external-secrets logs deploy/external-secrets --since=5m | rg 'sealed|503|app-database|openbao|vault'
```

복구:

```bash
kubectl -n openbao exec -it openbao-0 -- sh -lc '
  export BAO_ADDR=http://127.0.0.1:8200
  bao operator unseal
'
```

threshold만큼 반복 후:

```bash
kubectl -n openbao exec openbao-0 -- sh -lc '
  export BAO_ADDR=http://127.0.0.1:8200
  bao status | sed -n "/Initialized/p;/Sealed/p"
'
```

## failure injection: auth role 오류

role 이름을 틀리게 바꿉니다.

```bash
kubectl -n demo patch secretstore openbao --type=merge -p '{"spec":{"provider":{"vault":{"auth":{"kubernetes":{"role":"eso-demo-wrong"}}}}}}'
```

reconcile:

```bash
stamp=$(date +%s)
kubectl -n demo annotate externalsecret app-database force-sync="$stamp" --overwrite
kubectl -n demo describe externalsecret app-database
```

복구:

```bash
kubectl -n demo patch secretstore openbao --type=merge -p '{"spec":{"provider":{"vault":{"auth":{"kubernetes":{"role":"eso-demo"}}}}}}'
```

## failure injection: path 오류

remoteRef path를 없는 path로 바꿉니다.

```bash
kubectl -n demo patch externalsecret app-database --type=json -p='[
  {"op":"replace","path":"/spec/data/0/remoteRef/key","value":"apps/demo/missing"},
  {"op":"replace","path":"/spec/data/1/remoteRef/key","value":"apps/demo/missing"}
]'
```

확인:

```bash
stamp=$(date +%s)
kubectl -n demo annotate externalsecret app-database force-sync="$stamp" --overwrite
kubectl -n demo describe externalsecret app-database
```

복구:

```bash
kubectl -n demo patch externalsecret app-database --type=json -p='[
  {"op":"replace","path":"/spec/data/0/remoteRef/key","value":"apps/demo/database"},
  {"op":"replace","path":"/spec/data/1/remoteRef/key","value":"apps/demo/database"}
]'
```

## failure injection: policy 오류

OpenBao role에 빈 policy를 붙여 permission denied를 만듭니다.

```bash
kubectl -n openbao exec -i openbao-0 -- sh -lc '
  export BAO_ADDR=http://127.0.0.1:8200
  bao policy write eso-demo-empty -
' <<'EOF'
# intentionally empty for lab failure injection
EOF
```

role 변경:

```bash
kubectl -n openbao exec openbao-0 -- sh -lc '
  export BAO_ADDR=http://127.0.0.1:8200
  bao write auth/kubernetes/role/eso-demo \
    bound_service_account_names=eso-openbao \
    bound_service_account_namespaces=demo \
    policies=eso-demo-empty \
    ttl=15m
'
```

확인:

```bash
stamp=$(date +%s)
kubectl -n demo annotate externalsecret app-database force-sync="$stamp" --overwrite
kubectl -n demo describe externalsecret app-database
```

복구:

```bash
kubectl -n openbao exec openbao-0 -- sh -lc '
  export BAO_ADDR=http://127.0.0.1:8200
  bao write auth/kubernetes/role/eso-demo \
    bound_service_account_names=eso-openbao \
    bound_service_account_namespaces=demo \
    policies=eso-demo-read \
    ttl=15m
'
```

## failure injection: KV version 오류

SecretStore의 `version`을 `v1`로 틀리게 바꿉니다.

```bash
kubectl -n demo patch secretstore openbao --type=merge -p '{"spec":{"provider":{"vault":{"version":"v1"}}}}'
```

증상 확인:

```bash
stamp=$(date +%s)
kubectl -n demo annotate externalsecret app-database force-sync="$stamp" --overwrite
kubectl -n demo describe externalsecret app-database
```

복구:

```bash
kubectl -n demo patch secretstore openbao --type=merge -p '{"spec":{"provider":{"vault":{"version":"v2"}}}}'
```

## optional ClusterSecretStore

platform team이 공용 store를 제공해야 할 때만 사용합니다.

OpenBao role은 ESO controller namespace service account에 묶습니다. service account 이름은 설치 값을 확인합니다.

```bash
kubectl -n external-secrets get serviceaccount
```

예시는 `external-secrets` service account를 가정합니다.

```bash
kubectl -n openbao exec openbao-0 -- sh -lc '
  export BAO_ADDR=http://127.0.0.1:8200
  bao write auth/kubernetes/role/eso-cluster-demo \
    bound_service_account_names=external-secrets \
    bound_service_account_namespaces=external-secrets \
    policies=eso-demo-read \
    ttl=15m
'
```

ClusterSecretStore:

```bash
kubectl apply -f - <<'EOF'
apiVersion: external-secrets.io/v1
kind: ClusterSecretStore
metadata:
  name: openbao-demo
spec:
  provider:
    vault:
      server: http://openbao.openbao.svc.cluster.local:8200
      path: secret
      version: v2
      auth:
        kubernetes:
          mountPath: kubernetes
          role: eso-cluster-demo
          serviceAccountRef:
            name: external-secrets
            namespace: external-secrets
EOF
```

ExternalSecret에서 참조:

```bash
kubectl apply -f - <<'EOF'
apiVersion: external-secrets.io/v1
kind: ExternalSecret
metadata:
  name: app-database-cluster-store
  namespace: demo
spec:
  refreshPolicy: Periodic
  refreshInterval: 1m
  secretStoreRef:
    name: openbao-demo
    kind: ClusterSecretStore
  target:
    name: app-database-cluster-store
    creationPolicy: Owner
    deletionPolicy: Retain
  data:
    - secretKey: username
      remoteRef:
        key: apps/demo/database
        property: username
EOF
```

검증:

```bash
kubectl get clustersecretstore openbao-demo
kubectl -n demo get externalsecret app-database-cluster-store
```

## validation checklist

OpenBao:

```bash
kubectl -n openbao exec openbao-0 -- sh -lc '
  export BAO_ADDR=http://127.0.0.1:8200
  bao status
  bao auth list
  bao secrets list
  bao policy read eso-demo-read
'
```

SecretStore:

```bash
kubectl -n demo get secretstore openbao -o yaml
kubectl -n demo get secretstore openbao -o jsonpath='{range .status.conditions[*]}{.type}{" "}{.status}{" "}{.reason}{"\n"}{end}'
```

ExternalSecret:

```bash
kubectl -n demo get externalsecret
kubectl -n demo get externalsecret app-database -o jsonpath='{.status.refreshTime}{" "}{.status.syncedResourceVersion}{"\n"}'
```

Kubernetes Secret metadata only:

```bash
kubectl -n demo get secret app-database -o jsonpath='{.metadata.name}{" "}{.type}{" "}{.metadata.resourceVersion}{"\n"}'
kubectl -n demo get secret app-database -o jsonpath='{range $k,$v := .data}{$k}{"\n"}{end}'
```

ESO logs:

```bash
kubectl -n external-secrets logs deploy/external-secrets --since=15m | rg 'app-database|openbao|vault|error|denied|sealed'
```

## cleanup

lab 리소스 삭제:

```bash
kubectl -n demo delete externalsecret app-database app-env app-database-all app-database-cluster-store --ignore-not-found
kubectl -n demo delete secret app-database app-env app-database-all app-database-cluster-store --ignore-not-found
kubectl -n demo delete secretstore openbao --ignore-not-found
kubectl delete clustersecretstore openbao-demo --ignore-not-found
kubectl -n demo delete serviceaccount eso-openbao --ignore-not-found
```

OpenBao lab data 삭제:

```bash
kubectl -n openbao exec openbao-0 -- sh -lc '
  export BAO_ADDR=http://127.0.0.1:8200
  bao kv metadata delete -mount=secret apps/demo/database
  bao policy delete eso-demo-read || true
  bao policy delete eso-demo-empty || true
'
```

auth role 삭제:

```bash
kubectl -n openbao exec openbao-0 -- sh -lc '
  export BAO_ADDR=http://127.0.0.1:8200
  bao delete auth/kubernetes/role/eso-demo || true
  bao delete auth/kubernetes/role/eso-cluster-demo || true
'
```

전체 Helm 삭제가 필요할 때:

```bash
helm -n external-secrets uninstall external-secrets
helm -n openbao uninstall openbao
```

PVC 삭제는 데이터 삭제입니다. lab cluster에서만 명시적으로 수행합니다.

```bash
kubectl -n openbao get pvc
```

삭제가 필요하면 대상 PVC 이름을 사람이 재확인한 뒤 실행합니다.

## production promotion checklist

OpenBao:

- HA Raft 또는 운영 storage backend를 설계했습니다.
- TLS listener와 CA 배포가 있습니다.
- auto unseal 또는 Shamir ceremony를 문서화했습니다.
- unseal/recovery/root material 보관자가 분리되어 있습니다.
- root token bootstrap 후 회수 절차가 있습니다.
- audit device가 declarative하게 구성되어 있습니다.
- audit failure가 OpenBao availability에 미치는 영향을 테스트했습니다.
- backup과 restore rehearsal를 완료했습니다.
- sealed 상태 alert가 있습니다.
- `Initialized=true`와 `Sealed=false`를 별도 모니터링합니다.

ESO:

- chart/CRD 버전을 고정했습니다.
- upgrade notes와 CRD conversion을 확인했습니다.
- `SecretStore`를 tenant 기본값으로 사용합니다.
- `ClusterSecretStore` 사용 namespace와 subject를 제한했습니다.
- controller service account RBAC를 검토했습니다.
- tenant가 cluster-scoped ESO 리소스를 만들 수 없습니다.
- status condition과 controller error metric alert가 있습니다.
- target Secret deletion policy가 의도와 맞습니다.
- app reload 전략이 있습니다.

OpenBao + ESO:

- OpenBao policy가 KV v2 `data`/`metadata` path를 정확히 씁니다.
- ESO role TTL이 짧고 renew/relogin 동작을 검증했습니다.
- TokenReview 권한이 OpenBao service account에만 부여됐습니다.
- OpenBao dedicated provider와 Vault provider compatibility 범위를 문서화했습니다.
- dynamic secret을 쓰면 lease 만료, ESO refresh, app reload, revoke 시나리오를 별도 검증했습니다.
- GitOps bootstrap paradox를 피하는 0단계 절차가 있습니다.

## 참고

- OpenBao release notes: <https://openbao.org/community/release-notes/2-6-0/>
- OpenBao Kubernetes auth: <https://openbao.org/docs/2.4.x/auth/kubernetes/>
- OpenBao KV v2: <https://openbao.org/docs/secrets/kv/kv-v2/>
- OpenBao seal/unseal: <https://openbao.org/docs/next/concepts/seal/>
- ESO ExternalSecret: <https://external-secrets.io/latest/api/externalsecret/>
- ESO SecretStore: <https://external-secrets.io/main/api/secretstore/>
- ESO Vault provider: <https://external-secrets.io/latest/provider/hashicorp-vault/>
- ESO OpenBao provider: <https://external-secrets.io/main/provider/openbao/>
