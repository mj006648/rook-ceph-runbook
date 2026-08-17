# External Secrets Operator: OpenBao 연동 운영 기본기

## 요약

External Secrets Operator(ESO)는 외부 secret provider의 값을 Kubernetes `Secret`으로 동기화하는 controller입니다.

OpenBao와 함께 쓸 때 ESO는 보통 아래 일을 합니다.

- OpenBao에 Kubernetes auth 또는 token/AppRole로 로그인합니다.
- KV path에서 값을 읽습니다.
- `ExternalSecret` spec에 따라 Kubernetes `Secret`을 만들거나 갱신합니다.
- status, condition, event, metric으로 reconcile 결과를 남깁니다.

> 공개 runbook 규칙: provider token, OpenBao token, AppRole secret_id, unseal key, root token 값은 절대 문서나 Git에 쓰지 않습니다.

## 버전 기준과 공식 문서

2026-08-17 기준 확인 사항:

- ESO latest caveat: v2.9.0, release date 2026-08-07, Helm chart 2.9.0
- ESO core API 예시는 current docs 기준 `external-secrets.io/v1`
- `PushSecret`, `ClusterPushSecret`, generator 일부는 `external-secrets.io/v1alpha1` 또는 `generators.external-secrets.io/v1alpha1`
- OpenBao dedicated provider 문서는 alpha이며, tested 범위를 `External Secrets Operator v0.16.1` + `OpenBao v2.2.0`으로 명시합니다.
- OpenBao dedicated provider 명시 범위: KV only, auth는 AppRole/Kubernetes/token/UserPass
- OpenBao에서 VaultDynamicSecret generator 사용은 Vault provider compatibility에 근거한 추론입니다. 공식 OpenBao dedicated provider support로 쓰면 안 됩니다.

공식 URL:

- ESO ExternalSecret API: <https://external-secrets.io/latest/api/externalsecret/>
- ESO SecretStore API: <https://external-secrets.io/main/api/secretstore/>
- ESO API spec: <https://external-secrets.io/main/api/spec/>
- ESO ownership/deletion policy: <https://external-secrets.io/latest/guides/ownership-deletion-policy/>
- ESO HashiCorp Vault provider: <https://external-secrets.io/latest/provider/hashicorp-vault/>
- ESO OpenBao provider: <https://external-secrets.io/main/provider/openbao/>
- ESO Cluster Generator: <https://external-secrets.io/latest/api/generator/cluster/>
- ESO PushSecret: <https://external-secrets.io/main/api/pushsecret/>
- ESO ClusterPushSecret: <https://external-secrets.io/main/api/clusterpushsecret/>

실제 배포 전 확인:

```bash
kubectl api-resources | rg 'external-secrets|generators'
kubectl explain externalsecret.spec
kubectl explain secretstore.spec.provider
kubectl explain pushsecret.spec
```

## ESO가 해결하는 문제

Kubernetes workload는 보통 `Secret`을 env나 volume으로 읽습니다. 하지만 secret 원본은 OpenBao, cloud secret manager, password generator, PKI 등 외부 시스템에 있을 수 있습니다.

ESO는 이 간극을 controller reconcile로 메웁니다.

```text
OpenBao KV
  -> SecretStore provider config
  -> ExternalSecret
  -> Kubernetes Secret
  -> Pod env/volume
```

ESO가 해주는 일:

- provider 인증
- remote secret fetch
- key mapping
- template rendering
- Kubernetes Secret create/update/delete
- reconcile retry
- status/condition/event 기록

ESO가 해주지 않는 일:

- application hot reload 보장
- OpenBao unseal 자동화
- 잘못된 OpenBao policy 설계 보정
- dynamic secret lease와 app lifecycle 자동 설계
- root token 안전 보관

## controller 구조

ESO 배포에는 보통 아래 구성요소가 있습니다.

- controller manager: `ExternalSecret`, `SecretStore` 등을 reconcile합니다.
- webhook: CRD admission validation/conversion을 담당합니다.
- cert-controller: webhook certificate를 관리합니다.
- CRDs: `ExternalSecret`, `SecretStore`, `ClusterSecretStore`, `ClusterExternalSecret`, `PushSecret`, generator 등

상태 확인:

```bash
kubectl -n external-secrets get deploy,pod,svc
kubectl get crd | rg 'external-secrets|generators'
```

로그:

```bash
kubectl -n external-secrets logs deploy/external-secrets --tail=200
```

webhook 로그:

```bash
kubectl -n external-secrets logs deploy/external-secrets-webhook --tail=200
```

## reconcile 흐름

`ExternalSecret` 하나의 reconcile은 대략 아래 순서입니다.

```text
1. ExternalSecret spec 읽기
2. secretStoreRef로 SecretStore/ClusterSecretStore 찾기
3. provider config validation
4. provider auth 수행
5. spec.data와 spec.dataFrom remoteRef 읽기
6. conversion/decoding/rewrite/template 적용
7. target Secret create/update/delete policy 적용
8. status.conditions, refreshTime, syncedResourceVersion 갱신
9. event와 metric 기록
```

수동 reconcile 유도:

```bash
stamp=$(date +%s)
kubectl -n demo annotate externalsecret app-config force-sync="$stamp" --overwrite
```

GitOps drift를 피하려면 임시 annotation 제거:

```bash
kubectl -n demo annotate externalsecret app-config force-sync- --overwrite
```

## CRD 지도

### SecretStore

`SecretStore`는 namespaced provider config입니다. 같은 namespace의 `ExternalSecret`이 참조합니다.

OpenBao/Vault provider 예:

```yaml
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
```

특징:

- namespace 경계 안에서 동작합니다.
- tenant별 권한 분리에 유리합니다.
- `serviceAccountRef`는 보통 같은 namespace의 service account를 가리킵니다.

### ClusterSecretStore

`ClusterSecretStore`는 cluster-scoped provider config입니다.

```yaml
apiVersion: external-secrets.io/v1
kind: ClusterSecretStore
metadata:
  name: openbao-shared
spec:
  provider:
    vault:
      server: http://openbao.openbao.svc.cluster.local:8200
      path: secret
      version: v2
      auth:
        kubernetes:
          mountPath: kubernetes
          role: eso-shared
          serviceAccountRef:
            name: external-secrets
            namespace: external-secrets
```

주의:

- `ClusterSecretStore`에서 `serviceAccountRef`, `secretRef`를 쓰면 namespace를 명시해야 합니다.
- 편하지만 blast radius가 큽니다.
- multi-tenant cluster에서는 controller class, namespace policy, admission policy와 함께 제한합니다.

### ExternalSecret

`ExternalSecret`은 어떤 remote secret을 어떤 Kubernetes Secret으로 만들지 정의합니다.

```yaml
apiVersion: external-secrets.io/v1
kind: ExternalSecret
metadata:
  name: app-config
  namespace: demo
spec:
  refreshPolicy: Periodic
  refreshInterval: 15m
  secretStoreRef:
    name: openbao
    kind: SecretStore
  target:
    name: app-config
    creationPolicy: Owner
    deletionPolicy: Retain
    template:
      engineVersion: v2
      type: Opaque
      data:
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
```

### ClusterExternalSecret

`ClusterExternalSecret`은 matching namespace에 `ExternalSecret`을 배포하는 cluster-scoped 리소스입니다.

```yaml
apiVersion: external-secrets.io/v1
kind: ClusterExternalSecret
metadata:
  name: pull-secret-sync
spec:
  externalSecretName: pull-secret
  namespaceSelectors:
    - matchLabels:
        secrets.example.com/pull-secret: "enabled"
  refreshTime: 10m
  externalSecretSpec:
    refreshPolicy: Periodic
    refreshInterval: 1h
    secretStoreRef:
      name: openbao-shared
      kind: ClusterSecretStore
    target:
      name: registry-pull-secret
      creationPolicy: Owner
      deletionPolicy: Retain
    dataFrom:
      - extract:
          key: platform/registry/pull-secret
```

주의:

- namespace selector 실수는 여러 namespace에 secret을 배포합니다.
- tenant namespace label 권한을 제한해야 합니다.
- `externalSecretSpec` 안의 권한은 store policy에 의해 제한되어야 합니다.

### PushSecret

`PushSecret`은 Kubernetes Secret 값을 provider 쪽으로 push합니다. API는 current docs 기준 `external-secrets.io/v1alpha1`입니다.

```yaml
apiVersion: external-secrets.io/v1alpha1
kind: PushSecret
metadata:
  name: push-app-bootstrap
  namespace: demo
spec:
  refreshInterval: 1h
  secretStoreRefs:
    - name: openbao
      kind: SecretStore
  selector:
    secret:
      name: app-bootstrap
  updatePolicy: IfNotExists
  deletionPolicy: None
  data:
    - match:
        secretKey: bootstrap-password
      remoteRef:
        remoteKey: apps/demo/bootstrap
        property: password
```

주의:

- push는 source of truth 방향을 뒤집습니다.
- GitOps bootstrap에는 편해 보이지만 권한과 감사가 더 복잡합니다.
- OpenBao dedicated provider의 push 지원 범위는 반드시 배포 버전에서 확인합니다.

### ClusterPushSecret

`ClusterPushSecret`은 여러 namespace에 `PushSecret`을 만듭니다. API는 `external-secrets.io/v1alpha1`입니다.

```yaml
apiVersion: external-secrets.io/v1alpha1
kind: ClusterPushSecret
metadata:
  name: push-tenant-bootstrap
spec:
  pushSecretName: tenant-bootstrap
  namespaceSelectors:
    - matchLabels:
        secrets.example.com/push-bootstrap: "enabled"
  refreshTime: 10m
  pushSecretSpec:
    refreshInterval: 1h
    secretStoreRefs:
      - name: openbao-shared
        kind: ClusterSecretStore
    selector:
      secret:
        name: tenant-bootstrap
    updatePolicy: IfNotExists
    deletionPolicy: None
    dataTo:
      - storeRef:
          name: openbao-shared
          kind: ClusterSecretStore
        match:
          regexp: "^bootstrap-.*"
        rewrite:
          - regexp:
              source: "^bootstrap-"
              target: "tenants/"
```

### generators

ESO generators는 provider에서 읽는 대신 값을 생성하거나 외부 API에서 ephemeral credential을 받아옵니다.

Password generator 예:

```yaml
apiVersion: generators.external-secrets.io/v1alpha1
kind: Password
metadata:
  name: app-password
  namespace: demo
spec:
  length: 32
  digits: 5
  symbols: 5
  symbolCharacters: "-_$@"
  noUpper: false
  allowRepeat: true
---
apiVersion: external-secrets.io/v1
kind: ExternalSecret
metadata:
  name: generated-app-password
  namespace: demo
spec:
  refreshPolicy: CreatedOnce
  target:
    name: generated-app-password
    creationPolicy: Orphan
    immutable: true
  dataFrom:
    - sourceRef:
        generatorRef:
          apiVersion: generators.external-secrets.io/v1alpha1
          kind: Password
          name: app-password
```

VaultDynamicSecret generator는 ESO docs에 있습니다. OpenBao에 대해 사용할 때는 Vault compatibility에 근거한 추론으로 표시하고, lab에서 명시 검증 후 운영에 올립니다.

## `spec.data`

`data`는 remote secret의 특정 property를 Kubernetes Secret key로 매핑합니다.

```yaml
spec:
  data:
    - secretKey: username
      remoteRef:
        key: apps/demo/database
        property: username
    - secretKey: password
      remoteRef:
        key: apps/demo/database
        property: password
```

장점:

- 키 이름을 명확히 통제합니다.
- 필요한 property만 가져옵니다.
- template에서 어떤 값이 들어오는지 예측하기 쉽습니다.

## `spec.dataFrom`

`dataFrom`은 remote secret의 여러 property를 한 번에 가져옵니다.

```yaml
spec:
  dataFrom:
    - extract:
        key: apps/demo/config
```

주의:

- remote secret에 새 property가 추가되면 Kubernetes Secret에도 들어올 수 있습니다.
- rewrite, conversion, decoding 정책을 명확히 둡니다.
- tenant boundary에서는 `data`를 우선 고려합니다.

## template

`target.template`은 만들어질 Kubernetes Secret의 type, labels, annotations, data 모양을 정합니다.

```yaml
spec:
  target:
    name: app-env
    template:
      engineVersion: v2
      type: Opaque
      metadata:
        labels:
          app.kubernetes.io/name: demo
      data:
        APP_USERNAME: "{{ .username }}"
        APP_PASSWORD: "{{ .password }}"
        DATABASE_URL: "postgres://{{ .username }}:{{ .password }}@postgres.demo.svc:5432/app"
```

주의:

- template 결과도 Kubernetes Secret에 저장됩니다.
- template에 secret 값을 합쳐 넣으면 downstream rotation 단위가 커집니다.
- 앱이 env var로 읽으면 Secret update만으로 프로세스 값이 바뀌지 않습니다.

## refreshPolicy

ESO current docs 기준 `spec.refreshPolicy`는 세 가지입니다.

### Periodic

기본값입니다. `refreshInterval`마다 provider를 다시 읽습니다.

```yaml
spec:
  refreshPolicy: Periodic
  refreshInterval: 15m
```

`refreshInterval: 0`은 backwards compatibility 때문에 한 번만 생성하는 방식처럼 동작합니다. 명확성을 위해 `CreatedOnce`를 선호합니다.

### OnChange

`ExternalSecret` metadata/spec 변경 시 sync합니다.

```yaml
spec:
  refreshPolicy: OnChange
```

수동 rotation window에 맞춰 annotation으로 trigger할 때 유용합니다.

### CreatedOnce

`ExternalSecret` object의 첫 reconcile에서 sync하고 멈춥니다. 단, object 삭제 후 재생성하면 status가 reset되어 다시 sync할 수 있습니다.

불변 bootstrap secret 패턴:

```yaml
spec:
  refreshPolicy: CreatedOnce
  target:
    name: app-bootstrap
    creationPolicy: Orphan
    immutable: true
```

## creationPolicy와 deletionPolicy

`target.creationPolicy`는 Kubernetes Secret을 어떻게 만들고 소유할지 정합니다.

- `Owner`: 기본값입니다. Secret에 ownerReference를 붙이고 ExternalSecret 삭제 시 Secret도 삭제될 수 있습니다.
- `Orphan`: Secret을 만들지만 ownerReference를 붙이지 않습니다.
- `Merge`: 기존 Secret에 key를 merge합니다. Secret이 없으면 실패합니다.
- `None`: 현재 일반 create/update 용도로 쓰지 않습니다.

`target.deletionPolicy`는 provider 쪽 secret이 없어졌을 때 Kubernetes Secret을 어떻게 처리할지 정합니다.

- `Retain`: target Secret을 유지하고 상태 오류를 남깁니다.
- `Delete`: provider secret이 사라지면 target Secret을 삭제합니다.
- `Merge`: provider에 없는 key를 target Secret에서 제거합니다.

ESO lifecycle 문서는 일부 조합을 금지합니다. 특히 기존 Secret을 예기치 않게 삭제할 수 있는 조합을 피합니다.

안전한 기본값:

```yaml
target:
  creationPolicy: Owner
  deletionPolicy: Retain
```

공유 Secret에 일부 key만 주입할 때:

```yaml
target:
  creationPolicy: Merge
  deletionPolicy: Retain
```

이 경우 target Secret은 사전에 만들어야 합니다.

## status와 conditions

빠른 상태 확인:

```bash
kubectl -n demo get externalsecret
kubectl -n demo get secretstore
```

상세 condition:

```bash
kubectl -n demo describe externalsecret app-config
kubectl -n demo get externalsecret app-config -o jsonpath='{range .status.conditions[*]}{.type}{" "}{.status}{" "}{.reason}{" "}{.message}{"\n"}{end}'
```

SecretStore condition:

```bash
kubectl -n demo get secretstore openbao -o jsonpath='{range .status.conditions[*]}{.type}{" "}{.status}{" "}{.reason}{" "}{.message}{"\n"}{end}'
```

자주 보는 reason:

- `SecretSynced`: sync 성공
- `SecretSyncedError`: provider read, auth, template, target Secret 작업 실패
- `ValidationFailed`: store/provider config validation 실패
- `InvalidProviderConfig`: provider 설정 오류

실제 reason 문자열은 ESO 버전에 따라 달라질 수 있으므로 `describe`와 controller log를 같이 봅니다.

## RBAC와 multitenancy

기본 원칙:

- tenant namespace에는 namespaced `SecretStore`를 우선 사용합니다.
- `ClusterSecretStore`는 platform team이 관리합니다.
- tenant가 `ClusterExternalSecret`이나 `ClusterPushSecret`을 만들 수 없게 합니다.
- ESO controller service account 권한은 필요한 CRD와 Secret 작업으로 제한합니다.
- OpenBao policy는 Kubernetes namespace/service account에 묶습니다.

OpenBao Kubernetes role 예:

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

tenant namespace service account:

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: eso-openbao
  namespace: demo
```

tenant가 자기 namespace에서만 SecretStore를 쓰게 하려면 Kubernetes RBAC도 같이 둡니다.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: external-secret-author
  namespace: demo
rules:
  - apiGroups: ["external-secrets.io"]
    resources: ["externalsecrets", "secretstores"]
    verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
  - apiGroups: [""]
    resources: ["secrets"]
    verbs: ["get", "list", "watch"]
```

주의:

- Secret write 권한을 tenant에게 주면 ESO target Secret을 우회해서 직접 쓸 수 있습니다.
- Secret read 권한은 곧 secret value read 권한입니다.
- `ClusterSecretStore` 참조를 허용하면 OpenBao policy가 마지막 방어선이 됩니다.

## security checklist

SecretStore:

- OpenBao server URL은 TLS를 씁니다.
- `caBundle` 또는 `caProvider`로 CA를 검증합니다.
- static token보다 Kubernetes auth 또는 AppRole을 선호합니다.
- token/AppRole secret은 namespace와 RBAC로 보호합니다.
- `ClusterSecretStore`는 꼭 필요한 경우만 씁니다.

ExternalSecret:

- remoteRef는 필요한 path/property만 가져옵니다.
- `dataFrom.extract`는 remote object 전체 노출을 검토합니다.
- target Secret owner/deletion policy를 의도적으로 선택합니다.
- secret value를 annotation/label/template metadata에 넣지 않습니다.
- app reload 전략을 별도로 둡니다.

OpenBao:

- ESO 전용 policy를 별도로 만듭니다.
- KV v2 data path와 metadata path 권한을 구분합니다.
- root token을 ESO에 주지 않습니다.
- audit device를 켭니다.
- sealed 상태 알람을 둡니다.

## observability

리소스 상태:

```bash
kubectl get secretstore,clustersecretstore
kubectl -n demo get externalsecret
kubectl -n demo describe externalsecret app-config
```

event:

```bash
kubectl -n demo get event --sort-by=.lastTimestamp | tail -40
```

controller log:

```bash
kubectl -n external-secrets logs deploy/external-secrets --since=30m | rg 'app-config|openbao|vault|error|denied|sealed'
```

metrics endpoint 확인:

```bash
kubectl -n external-secrets get svc
kubectl -n external-secrets port-forward svc/external-secrets-metrics 8080:8080
```

다른 터미널:

```bash
curl -fsS http://127.0.0.1:8080/metrics | rg 'externalsecret|secretstore|reconcile|error'
```

서비스 이름과 port는 chart values에 따라 다를 수 있으므로 `kubectl -n external-secrets get svc -o yaml`로 확인합니다.

## troubleshooting

### Store가 Ready가 아니다

```bash
kubectl -n demo describe secretstore openbao
kubectl -n external-secrets logs deploy/external-secrets --since=15m | rg 'openbao|vault|SecretStore|error'
```

확인할 것:

- OpenBao service DNS가 맞는가?
- TLS CA가 맞는가?
- OpenBao가 sealed 상태인가?
- Kubernetes auth role 이름이 맞는가?
- serviceAccountRef namespace/name이 맞는가?
- ClusterSecretStore에서 namespace를 빠뜨리지 않았는가?

### ExternalSecret이 SecretSyncedError

```bash
kubectl -n demo describe externalsecret app-config
kubectl -n demo get externalsecret app-config -o yaml
```

확인할 것:

- `secretStoreRef.kind`가 맞는가?
- remote key path가 KV mount 아래 logical path인가?
- KV v2 property 이름이 맞는가?
- OpenBao policy에 `secret/data/...` read가 있는가?
- template에서 없는 key를 참조하지 않는가?

### target Secret이 갱신되지 않는다

```bash
kubectl -n demo get externalsecret app-config -o jsonpath='{.spec.refreshPolicy}{" "}{.spec.refreshInterval}{"\n"}'
kubectl -n demo get externalsecret app-config -o jsonpath='{.status.refreshTime}{" "}{.status.syncedResourceVersion}{"\n"}'
```

확인할 것:

- `refreshPolicy: CreatedOnce`인가?
- `refreshPolicy: OnChange`인데 spec/metadata 변경이 없었는가?
- target Secret이 `immutable: true`인가?
- app이 env var로 읽어 재시작 전에는 반영되지 않는가?

### path는 맞는데 permission denied

KV v2 policy path를 확인합니다.

```hcl
path "secret/data/apps/demo/*" {
  capabilities = ["read"]
}

path "secret/metadata/apps/demo/*" {
  capabilities = ["list", "read"]
}
```

`secret/apps/demo/*`가 아니라 `secret/data/apps/demo/*`입니다.

## GitOps bootstrap paradox

GitOps로 ESO와 OpenBao를 관리할 때 순환 의존이 생깁니다.

```text
Argo CD needs repo credentials
  -> repo credentials are ExternalSecret
  -> ExternalSecret needs ESO
  -> ESO needs SecretStore
  -> SecretStore needs OpenBao auth
  -> OpenBao auth needs bootstrap policy/token
```

해결 패턴:

- 0단계: OpenBao 설치, init, unseal, audit enable은 별도 break-glass 절차로 수행합니다.
- 1단계: 최소 admin auth와 ESO 전용 Kubernetes auth role/policy를 bootstrap합니다.
- 2단계: ESO CRD/controller를 설치합니다.
- 3단계: tenant SecretStore와 ExternalSecret을 GitOps로 배포합니다.
- 4단계: bootstrap token/root token을 회수하고 audit로 확인합니다.

피해야 할 패턴:

- root token을 Kubernetes Secret으로 저장해서 SecretStore가 읽게 하기
- unseal key를 SealedSecret/SOPS/ESO로 관리하기
- OpenBao가 필요해서 OpenBao unseal material을 OpenBao에 저장하기
- 모든 namespace가 같은 ClusterSecretStore와 같은 OpenBao role을 쓰기

## OpenBao dedicated provider vs Vault provider

ESO 문서에는 OpenBao provider page가 있지만, 문구상 HashiCorp Vault provider를 통한 integration이라고 설명하고 tested 범위를 `ESO v0.16.1` + `OpenBao v2.2.0`으로 제한합니다.

구분:

| 항목 | 공식 지원으로 말할 수 있는 것 | 추론 또는 별도 검증 필요 |
| --- | --- | --- |
| OpenBao dedicated provider | alpha, KV only, AppRole/Kubernetes/token/UserPass | 모든 OpenBao engine |
| Vault provider with OpenBao endpoint | Vault-compatible KV read/write 일부 | OpenBao version별 edge behavior |
| VaultDynamicSecret generator on OpenBao | Vault provider/generator compatibility 가능성 | OpenBao에서 공식 검증됐다는 주장 |
| PKI/database dynamic secret | OpenBao 자체 engine 기능 | ESO OpenBao provider의 공식 지원 범위 |

운영 문서에는 이렇게 씁니다.

```text
이 ExternalSecret은 ESO Vault provider를 OpenBao endpoint에 연결해 KV v2를 읽는다.
ESO OpenBao dedicated provider 문서의 공식 tested 범위는 v0.16.1/OpenBao v2.2.0이며,
dynamic secret 동작은 이 lab에서 별도 검증한 compatibility 결과다.
```

## 당직자 빠른 명령

전체 상태:

```bash
kubectl get secretstore,clustersecretstore
kubectl get externalsecret -A
```

특정 앱:

```bash
kubectl -n demo describe externalsecret app-config
kubectl -n demo get secret app-config -o jsonpath='{.metadata.creationTimestamp}{" "}{.metadata.resourceVersion}{"\n"}'
```

강제 reconcile:

```bash
stamp=$(date +%s)
kubectl -n demo annotate externalsecret app-config force-sync="$stamp" --overwrite
```

임시 annotation 제거:

```bash
kubectl -n demo annotate externalsecret app-config force-sync- --overwrite
```

OpenBao sealed 확인:

```bash
kubectl -n openbao exec openbao-0 -- sh -lc '
  export BAO_ADDR=http://127.0.0.1:8200
  bao status | sed -n "/Initialized/p;/Sealed/p"
'
```

## 참고

- ESO ExternalSecret: <https://external-secrets.io/latest/api/externalsecret/>
- ESO SecretStore: <https://external-secrets.io/main/api/secretstore/>
- ESO lifecycle: <https://external-secrets.io/latest/guides/ownership-deletion-policy/>
- ESO Vault provider: <https://external-secrets.io/latest/provider/hashicorp-vault/>
- ESO OpenBao provider: <https://external-secrets.io/main/provider/openbao/>
- ESO PushSecret: <https://external-secrets.io/main/api/pushsecret/>
- ESO API spec: <https://external-secrets.io/main/api/spec/>
