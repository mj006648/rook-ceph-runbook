# OpenBao 기본기: seal, token, policy, Kubernetes auth

## 요약

OpenBao는 secret 값을 단순히 저장하는 서버가 아니라 **인증, 권한, 수명, 감사, 폐기**를 한곳에서 강제하는 secret management system입니다.

이 문서는 운영자가 External Secrets Operator(ESO), Argo CD, Kubernetes workload와 함께 OpenBao를 쓸 때 먼저 알아야 하는 개념을 정리합니다.

> 공개 runbook 규칙: unseal key, recovery key, root token, client token 값은 절대 문서, Git, shell history, issue, PR comment에 쓰지 않습니다.

## 버전 기준과 공식 문서

이 문서는 2026-08-17 기준으로 아래 문서를 확인하고 작성했습니다.

- OpenBao release notes: <https://openbao.org/community/release-notes/2-6-0/>
- OpenBao latest로 확인한 버전 caveat: v2.6.1, release date 2026-07-22
- Seal/Unseal concepts: <https://openbao.org/docs/next/concepts/seal/>
- `operator init`: <https://openbao.org/docs/commands/operator/init/>
- `operator unseal`: <https://openbao.org/docs/next/commands/operator/unseal/>
- Storage stanza: <https://openbao.org/docs/configuration/storage/>
- Raft storage: <https://openbao.org/docs/next/configuration/storage/raft/>
- Authentication concepts: <https://openbao.org/docs/concepts/auth/>
- Tokens: <https://openbao.org/docs/2.5.x/concepts/tokens/>
- Lease, renew, revoke: <https://openbao.org/docs/2.5.x/concepts/lease/>
- Policy concepts: <https://openbao.org/docs/2.5.x/concepts/policies/>
- KV v2: <https://openbao.org/docs/secrets/kv/kv-v2/>
- Audit devices: <https://openbao.org/docs/next/audit/>
- Kubernetes auth: <https://openbao.org/docs/2.4.x/auth/kubernetes/>
- Kubernetes auth API: <https://openbao.org/api-docs/next/auth/kubernetes/>

문서 URL에 `next`, `2.4.x`, `2.5.x`가 섞여 있습니다. 개념은 OpenBao 2.6.x에서도 유효하지만, 실제 운영 manifest와 CLI flag는 배포한 OpenBao 버전의 문서에서 다시 확인합니다.

## OpenBao가 해결하는 문제

Kubernetes `Secret`만 쓰면 값은 etcd에 저장되고 Pod에 주입됩니다. 하지만 아래 운영 질문은 Kubernetes Secret만으로 풀기 어렵습니다.

- 누가 어떤 secret을 읽었는지 남는가?
- 앱별로 path 단위 최소 권한을 줄 수 있는가?
- DB 계정을 요청할 때마다 동적으로 만들고 TTL 이후 자동 폐기할 수 있는가?
- 인증 토큰을 즉시 revoke할 수 있는가?
- secret engine별로 감사 로그, TTL, rotation을 통제할 수 있는가?
- 클러스터 장애 후 암호화된 저장소와 seal material을 분리해서 복구할 수 있는가?

OpenBao는 secret 값을 API 뒤에 숨기고, 모든 접근을 token, policy, lease, audit log로 통제합니다.

## 큰 구조

OpenBao 서버는 대략 아래 흐름으로 요청을 처리합니다.

```text
client
  -> listener/TLS
  -> auth method 또는 token 검증
  -> ACL policy 평가
  -> secret engine 또는 sys endpoint
  -> barrier 암호화 계층
  -> storage backend
  -> audit devices
```

주요 구성요소:

- Listener: HTTP API를 받는 입구입니다. 운영에서는 TLS를 기본으로 둡니다.
- Auth method: Kubernetes, AppRole, token, userpass, LDAP 같은 로그인 방식입니다.
- Token store: 로그인 결과로 발급된 OpenBao token을 관리합니다.
- Policy engine: token에 붙은 policy로 path별 capability를 평가합니다.
- Secrets engine: KV, PKI, database처럼 실제 secret을 저장하거나 생성합니다.
- Barrier: storage에 쓰는 대부분의 데이터를 암호화하는 내부 경계입니다.
- Seal: barrier key를 풀기 위한 root key 보호 장치입니다.
- Storage: 암호화된 OpenBao 데이터를 저장하는 durable backend입니다.
- Audit device: 요청과 응답을 감사 로그로 남기는 장치입니다.

## barrier, seal, storage를 구분하기

OpenBao 장애 복구에서 가장 흔한 혼동은 storage와 seal을 같은 것으로 보는 것입니다.

### barrier

Barrier는 OpenBao가 storage에 쓰는 대부분의 데이터를 암호화하는 계층입니다. OpenBao glossary는 root key, keyring, seal의 관계를 설명합니다.

운영적으로는 이렇게 기억합니다.

- storage에는 OpenBao 데이터가 저장됩니다.
- 그 데이터는 barrier에 의해 암호화됩니다.
- barrier를 열려면 root key가 필요합니다.
- root key는 seal mechanism으로 보호됩니다.

### seal

Seal은 root key를 보호하는 메커니즘입니다. OpenBao 프로세스가 시작되면 기본적으로 sealed 상태입니다. Sealed 상태에서는 storage 위치는 알지만, storage 내용을 복호화할 수 없습니다.

Sealed 상태에서 가능한 일은 제한됩니다.

- 가능: seal status 확인, unseal 시도, health 확인
- 불가능: auth login, KV read/write, policy 변경, secrets engine 사용

ESO 로그에서 `Vault is sealed`, `Code: 503`이 보이면 secret path나 policy보다 먼저 OpenBao `Sealed` 상태를 봅니다.

### storage

Storage는 암호화된 데이터를 저장하는 위치입니다.

예:

- file storage: 단일 노드 lab 또는 간단한 standalone
- integrated storage raft: OpenBao 자체 Raft replication
- PostgreSQL 등 외부 storage backend

OpenBao 공식 storage 문서는 대부분의 use case에서 integrated storage를 권장합니다. Raft는 HA를 지원하고 OpenBao 운영 범위 안에서 모니터링할 수 있기 때문입니다.

### 실전 구분

```bash
kubectl -n openbao exec openbao-0 -- sh -lc '
  export BAO_ADDR=http://127.0.0.1:8200
  bao status
'
```

해석:

```text
Initialized     true
Sealed          true
Storage Type    raft
HA Enabled      true
```

- `Initialized=true`: storage backend는 이미 OpenBao cluster로 초기화됐습니다.
- `Sealed=true`: barrier를 아직 열지 못했습니다.
- `Storage Type=raft`: 데이터 저장 방식입니다. unseal 방식이 아닙니다.
- `HA Enabled=true`: leader/standby cluster 모드입니다.

## init은 한 번만 한다

`bao operator init`은 storage backend를 OpenBao cluster로 처음 준비하는 작업입니다. 이미 initialized인 cluster에 다시 실행하면 안 됩니다.

상태만 확인하는 안전한 명령:

```bash
kubectl -n openbao exec openbao-0 -- sh -lc '
  export BAO_ADDR=http://127.0.0.1:8200
  bao operator init -status
'
```

출력 해석:

```text
OpenBao is initialized
```

또는 exit code로 확인합니다.

```bash
kubectl -n openbao exec openbao-0 -- sh -lc '
  export BAO_ADDR=http://127.0.0.1:8200
  bao operator init -status >/tmp/openbao-init-status.txt
  rc=$?
  cat /tmp/openbao-init-status.txt
  exit "$rc"
'
```

주의:

- `Initialized=true`이면 `bao operator init`을 실행하지 않습니다.
- root token은 초기 bootstrap 뒤 폐기하거나 break-glass 보관소로 이동합니다.
- unseal key 또는 recovery key는 GitOps Secret, Slack, password manager export 파일에 평문으로 두지 않습니다.

## Shamir unseal

기본 Shamir seal에서는 초기화 시 unseal key share가 생성되고, threshold 개수만큼 입력해야 barrier가 열립니다.

안전한 unseal 방식:

```bash
kubectl -n openbao exec -it openbao-0 -- sh -lc '
  export BAO_ADDR=http://127.0.0.1:8200
  bao operator unseal
'
```

프롬프트가 뜨면 key share를 붙여넣습니다. key를 명령 인자로 넣지 않습니다.

하지 말아야 할 방식:

```bash
# 금지: unseal key가 shell history와 process args에 남습니다.
bao operator unseal '<UNSEAL_KEY_VALUE>'
```

다중 노드 Shamir cluster에서는 각 노드를 threshold만큼 unseal해야 합니다. 한 노드에 key 1개, 다른 노드에 key 1개씩 분산 입력해도 cluster 전체 threshold가 채워지는 방식이 아닙니다.

## auto unseal과 recovery key

Auto unseal은 KMS, HSM, transit seal 같은 외부 신뢰 장치가 root key 복호화를 도와주는 방식입니다.

핵심 차이:

- Shamir: unseal key share가 barrier를 여는 데 직접 필요합니다.
- Auto unseal: seal mechanism이 root key 복호화를 담당합니다.
- Recovery key: auto unseal 환경에서 generate-root 같은 quorum 작업을 승인하는 용도입니다.

중요한 오해:

- Recovery key는 auto unseal backend가 망가졌을 때 root key를 직접 복호화하는 만능 키가 아닙니다.
- KMS key를 삭제하고 backup만 있으면 복구된다는 보장은 없습니다.
- Auto unseal은 운영 편의성을 올리지만 KMS/HSM lifecycle 의존성을 만듭니다.

## token

OpenBao client token은 web session cookie와 비슷합니다. 사용자는 auth method로 로그인하고 token을 받습니다. 이후 API 요청은 token으로 인증되고 policy로 권한이 제한됩니다.

안전한 token lookup:

```bash
kubectl -n openbao exec -it openbao-0 -- sh -lc '
  export BAO_ADDR=http://127.0.0.1:8200
  bao token lookup
'
```

현재 token renew:

```bash
kubectl -n openbao exec -it openbao-0 -- sh -lc '
  export BAO_ADDR=http://127.0.0.1:8200
  bao token renew
'
```

token을 인자로 넣는 명령은 피합니다.

```bash
# 금지: token 값이 history/process args에 남습니다.
bao token lookup '<CLIENT_TOKEN>'
```

운영 원칙:

- root token은 일상 운영에 사용하지 않습니다.
- 앱 token은 짧은 TTL과 최소 policy를 부여합니다.
- token revoke는 그 token으로 생성된 lease까지 같이 영향을 줄 수 있습니다.
- token accessor 조회 권한도 강력한 권한입니다. accessor로 revoke가 가능하기 때문입니다.

## lease, renewal, revocation

Dynamic secret과 service type token에는 lease가 붙습니다. Lease는 “이 값이 이 시간 동안 유효하다”는 계약입니다.

동적 secret 예:

```bash
kubectl -n openbao exec openbao-0 -- sh -lc '
  export BAO_ADDR=http://127.0.0.1:8200
  bao read database/creds/app-readonly -format=json | jq "{lease_id, lease_duration, renewable}"
'
```

lease renew:

```bash
kubectl -n openbao exec -it openbao-0 -- sh -lc '
  export BAO_ADDR=http://127.0.0.1:8200
  printf "Lease ID: "
  read -r lease_id
  bao lease renew "$lease_id"
'
```

lease revoke:

```bash
kubectl -n openbao exec -it openbao-0 -- sh -lc '
  export BAO_ADDR=http://127.0.0.1:8200
  printf "Lease ID to revoke: "
  read -r lease_id
  bao lease revoke "$lease_id"
'
```

KV는 lease 기반 dynamic secret이 아닙니다. KV read에서 lease duration처럼 보이는 값이 나와도 DB 계정처럼 자동 폐기되는 secret으로 해석하면 안 됩니다.

## auth method

Auth method는 “누가 로그인할 수 있는가”를 정합니다. 대표적인 방식:

- token: 이미 발급된 token으로 접근합니다.
- userpass: 실습에는 쉽지만 운영 machine auth로는 보통 권장하지 않습니다.
- AppRole: CI/CD나 non-Kubernetes workload에서 자주 씁니다.
- Kubernetes: ServiceAccount token을 TokenReview API로 검증합니다.
- JWT/OIDC: Kubernetes issuer discovery 또는 외부 IdP 기반 인증에 씁니다.
- LDAP/GitHub 등: 사용자 인증에 씁니다.

auth method 목록:

```bash
kubectl -n openbao exec openbao-0 -- sh -lc '
  export BAO_ADDR=http://127.0.0.1:8200
  bao auth list
'
```

Kubernetes auth 활성화:

```bash
kubectl -n openbao exec openbao-0 -- sh -lc '
  export BAO_ADDR=http://127.0.0.1:8200
  bao auth enable kubernetes
'
```

이미 있으면 실패할 수 있으므로 GitOps나 bootstrap job에서는 idempotent하게 확인 후 처리합니다.

## policy와 capability

OpenBao policy는 path 기반 ACL입니다. 기본은 deny입니다. 빈 policy는 아무 권한도 주지 않습니다.

주요 capability:

- `create`: 새 값 생성
- `read`: 값 읽기
- `update`: 기존 값 변경 또는 대부분의 create/update API
- `patch`: 일부 field patch
- `delete`: 삭제
- `list`: prefix listing
- `sudo`: 일부 privileged sys 작업
- `deny`: 명시적 거부

KV v2 최소 read policy 예:

```hcl
path "secret/data/apps/demo/*" {
  capabilities = ["read"]
}

path "secret/metadata/apps/demo/*" {
  capabilities = ["list", "read"]
}
```

쓰기까지 허용하는 policy 예:

```hcl
path "secret/data/apps/demo/*" {
  capabilities = ["create", "update", "read", "delete"]
}

path "secret/metadata/apps/demo/*" {
  capabilities = ["list", "read", "delete"]
}
```

policy 적용:

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

필요 권한 확인:

```bash
kubectl -n openbao exec openbao-0 -- sh -lc '
  export BAO_ADDR=http://127.0.0.1:8200
  bao kv get -mount=secret -output-policy apps/demo/config
'
```

## KV v1과 KV v2

KV v1:

- path에 바로 값을 저장합니다.
- version history가 없습니다.
- API path와 사람이 보는 path가 비교적 단순합니다.

KV v2:

- versioned secret입니다.
- data path와 metadata path가 분리됩니다.
- soft delete, undelete, destroy 같은 version lifecycle이 있습니다.

OpenBao KV v2 공식 문서는 `-mount=secret` 형태를 권장합니다. 사람이 보는 logical path와 실제 API path `secret/data/...`를 혼동하지 않기 위해서입니다.

KV v2 활성화:

```bash
kubectl -n openbao exec openbao-0 -- sh -lc '
  export BAO_ADDR=http://127.0.0.1:8200
  bao secrets enable -path=secret kv-v2
'
```

KV v2 쓰기:

```bash
kubectl -n openbao exec -i openbao-0 -- sh -lc '
  export BAO_ADDR=http://127.0.0.1:8200
  bao kv put -mount=secret apps/demo/config \
    username=demo \
    password=-
' <<'EOF'
CHANGE-ME-LAB-PASSWORD
EOF
```

위 예시는 lab placeholder입니다. 운영 credential은 문서에 넣지 말고, 사람 입력 프롬프트나 승인된 secret injection 절차를 사용합니다.

KV v2 읽기:

```bash
kubectl -n openbao exec openbao-0 -- sh -lc '
  export BAO_ADDR=http://127.0.0.1:8200
  bao kv get -mount=secret apps/demo/config
'
```

KV v2 metadata:

```bash
kubectl -n openbao exec openbao-0 -- sh -lc '
  export BAO_ADDR=http://127.0.0.1:8200
  bao kv metadata get -mount=secret apps/demo/config
'
```

ESO에서 KV v2를 읽을 때는 provider path를 mount인 `secret`으로 두고, remote key는 mount 아래 logical path인 `apps/demo/config`로 둡니다.

## dynamic secrets engine

Dynamic secrets engine은 요청 시 credential을 만들고 lease가 끝나면 revoke합니다.

예:

- database engine: DB 사용자/비밀번호를 동적으로 생성
- PKI engine: certificate 발급
- cloud secret engine: cloud IAM credential 발급

Database dynamic credential 흐름:

```text
app or ESO
  -> OpenBao auth
  -> read database/creds/app-readonly
  -> OpenBao creates DB user
  -> returns username/password + lease_id
  -> renew or revoke lease
```

PKI 흐름:

```text
client
  -> OpenBao auth
  -> write pki/issue/app
  -> receives cert/key/ca_chain + lease
```

주의:

- dynamic secret은 consumer가 renewal/replacement를 이해해야 합니다.
- Kubernetes Secret에 dynamic credential을 복사하면 lease 만료와 app reload를 함께 설계해야 합니다.
- ESO의 OpenBao dedicated provider는 KV 중심 alpha 범위입니다. dynamic secret은 ESO Vault provider 또는 VaultDynamicSecret generator compatibility로 접근할 수 있지만, OpenBao에서 공식적으로 검증된 범위와 inference를 구분해야 합니다.

## audit

Audit device는 OpenBao API 요청/응답을 기록합니다. 감사 로그는 보안상 민감합니다. 값은 기본적으로 HMAC/hash 처리되지만 path, accessor, metadata, error는 운영상 민감할 수 있습니다.

상태 확인:

```bash
kubectl -n openbao exec openbao-0 -- sh -lc '
  export BAO_ADDR=http://127.0.0.1:8200
  bao audit list
'
```

파일 audit enable 예:

```bash
kubectl -n openbao exec openbao-0 -- sh -lc '
  export BAO_ADDR=http://127.0.0.1:8200
  bao audit enable file file_path=/openbao/audit/audit.log
'
```

OpenBao v2.3.2 이후 API/CLI로 audit device를 만들려면 서버 설정에서 `unsafe_allow_api_audit_creation=true`가 필요할 수 있습니다. 운영에서는 declarative audit configuration을 선호합니다.

중요한 운영 사실:

- 처음 initialize한 OpenBao에는 audit이 자동으로 켜져 있지 않습니다.
- audit device가 모두 실패하면 OpenBao 요청 처리가 막힐 수 있습니다.
- 여러 audit device를 두면 tamper 확인과 가용성에 도움이 됩니다.
- audit device를 disable 후 같은 path로 다시 enable해도 HMAC salt가 달라질 수 있습니다.

## HA, DR, backup

OpenBao HA는 API availability와 leader election 문제이고, backup은 암호화된 storage 데이터와 seal material lifecycle 문제입니다.

운영 확인:

```bash
kubectl -n openbao exec openbao-0 -- sh -lc '
  export BAO_ADDR=http://127.0.0.1:8200
  bao status
'
```

Raft peer 확인:

```bash
kubectl -n openbao exec openbao-0 -- sh -lc '
  export BAO_ADDR=http://127.0.0.1:8200
  bao operator raft list-peers
'
```

Raft snapshot은 운영 정책에 맞게 별도 보관합니다. snapshot 파일도 암호화된 OpenBao 데이터이므로 민감 자산입니다.

```bash
kubectl -n openbao exec openbao-0 -- sh -lc '
  export BAO_ADDR=http://127.0.0.1:8200
  bao operator raft snapshot save /tmp/openbao-raft.snap
'
```

위 명령은 pod 내부 `/tmp`에 snapshot을 만듭니다. 실제 운영에서는 snapshot 반출, 암호화, retention, 복구 리허설 절차가 필요합니다.

복구 체크리스트:

- storage snapshot/PVC backup이 있는가?
- seal backend 또는 Shamir unseal key threshold를 만족할 수 있는가?
- auto unseal KMS/HSM key가 삭제되지 않았는가?
- OpenBao version과 storage backend compatibility를 확인했는가?
- audit log와 snapshot을 같은 권한 경계에 두지 않았는가?
- restore rehearsal를 실제로 해봤는가?

## Kubernetes auth

Kubernetes auth는 ServiceAccount JWT를 받아 Kubernetes TokenReview API로 검증합니다.

OpenBao가 Kubernetes 안에서 실행될 때 권장되는 단순 설정은 local service account token을 reviewer JWT로 쓰는 방식입니다. OpenBao 공식 문서는 이 경우 `token_reviewer_jwt`와 `kubernetes_ca_cert`를 생략하고 pod의 service account token/CA를 읽도록 설명합니다.

TokenReview 권한:

```yaml
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
```

auth config:

```bash
kubectl -n openbao exec openbao-0 -- sh -lc '
  export BAO_ADDR=http://127.0.0.1:8200
  bao write auth/kubernetes/config \
    kubernetes_host="https://${KUBERNETES_SERVICE_HOST}:${KUBERNETES_SERVICE_PORT}"
'
```

role:

```bash
kubectl -n openbao exec openbao-0 -- sh -lc '
  export BAO_ADDR=http://127.0.0.1:8200
  bao write auth/kubernetes/role/eso-demo \
    bound_service_account_names=external-secrets \
    bound_service_account_namespaces=external-secrets \
    policies=eso-demo-read \
    ttl=15m
'
```

Kubernetes 1.21+ 주의:

- BoundServiceAccountTokenVolume이 기본입니다.
- Pod token은 짧은 수명과 audience를 가질 수 있습니다.
- `disable_iss_validation=true`가 새 mount의 권장 기본입니다.
- Kubernetes auth는 TokenReview API를 사용하므로 revoked service account token 검증에 유리합니다.
- JWT/OIDC auth는 TokenReview를 쓰지 않으므로 token expiry 전 강제 revoke 성격이 다릅니다.

## 안전한 운영 습관

### secret 값을 argument에 넣지 않기

금지:

```bash
bao login '<TOKEN>'
bao operator unseal '<UNSEAL_KEY>'
bao write secret/data/app password='<PASSWORD>'
```

권장:

```bash
kubectl -n openbao exec -it openbao-0 -- sh -lc '
  export BAO_ADDR=http://127.0.0.1:8200
  bao login
'
```

또는 stdin/file descriptor를 사용합니다. 단, 파일로 만들 때도 권한과 삭제를 관리합니다.

### root token 최소화

초기 root token은 bootstrap 후 아래 작업만 수행하고 보관/폐기합니다.

- audit enable
- 최소 admin auth method 구성
- 운영 policy 작성
- break-glass 절차 검증

일상 작업은 named auth method와 제한된 policy로 합니다.

### sealed를 app 장애로 오해하지 않기

ESO, Argo CD, application이 동시에 secret sync 실패를 보이면 먼저 OpenBao seal status를 확인합니다.

```bash
kubectl -n openbao exec openbao-0 -- sh -lc '
  export BAO_ADDR=http://127.0.0.1:8200
  bao status | sed -n "/Initialized/p;/Sealed/p"
'
```

### readiness probe 오해

OpenBao health endpoint는 query parameter에 따라 sealed 상태도 ready로 보이게 만들 수 있습니다.

예:

```text
/v1/sys/health?standbyok=true&sealedcode=204&uninitcode=204
```

이 설정은 Kubernetes pod readiness와 secret service readiness를 다르게 만듭니다. ESO가 실패하면 `/sys/health` probe보다 실제 auth login과 KV read를 봅니다.

## 자주 틀리는 생각

- “PVC가 있으니 unseal key는 필요 없다.”  
  틀렸습니다. PVC에는 암호화된 데이터가 있고 seal을 열 material은 별도입니다.

- “Initialized=true니까 정상이다.”  
  부족합니다. `Sealed=false`까지 확인해야 auth와 secrets engine이 동작합니다.

- “Recovery key가 있으면 KMS 삭제 후에도 복구된다.”  
  일반적으로 틀린 가정입니다. auto unseal backend lifecycle을 보호해야 합니다.

- “KV v2 path는 secret/foo이다.”  
  CLI logical path는 그렇게 보일 수 있지만 API policy는 `secret/data/foo`, metadata는 `secret/metadata/foo`입니다.

- “list 권한은 read보다 약하다.”  
  list도 민감합니다. secret 이름 자체가 정보입니다.

- “audit log는 안전하게 아무 데나 보내도 된다.”  
  틀렸습니다. audit log는 민감 운영 데이터입니다.

- “ESO가 OpenBao를 지원하니 모든 OpenBao engine이 공식 지원이다.”  
  틀렸습니다. ESO OpenBao dedicated provider는 alpha이고 명시 범위가 제한됩니다. Vault provider compatibility와 공식 OpenBao support를 구분합니다.

## 당직자 빠른 진단

OpenBao 상태:

```bash
kubectl -n openbao exec openbao-0 -- sh -lc '
  export BAO_ADDR=http://127.0.0.1:8200
  bao status
'
```

auth method:

```bash
kubectl -n openbao exec openbao-0 -- sh -lc '
  export BAO_ADDR=http://127.0.0.1:8200
  bao auth list
'
```

secret engine:

```bash
kubectl -n openbao exec openbao-0 -- sh -lc '
  export BAO_ADDR=http://127.0.0.1:8200
  bao secrets list
'
```

policy:

```bash
kubectl -n openbao exec openbao-0 -- sh -lc '
  export BAO_ADDR=http://127.0.0.1:8200
  bao policy list
'
```

audit:

```bash
kubectl -n openbao exec openbao-0 -- sh -lc '
  export BAO_ADDR=http://127.0.0.1:8200
  bao audit list
'
```

ESO 관련 Kubernetes auth role:

```bash
kubectl -n openbao exec openbao-0 -- sh -lc '
  export BAO_ADDR=http://127.0.0.1:8200
  bao read auth/kubernetes/role/eso-demo
'
```

## 참고

- OpenBao release notes: <https://openbao.org/community/release-notes/2-6-0/>
- OpenBao seal/unseal: <https://openbao.org/docs/next/concepts/seal/>
- OpenBao init: <https://openbao.org/docs/commands/operator/init/>
- OpenBao storage: <https://openbao.org/docs/configuration/storage/>
- OpenBao policy concepts: <https://openbao.org/docs/2.5.x/concepts/policies/>
- OpenBao KV v2: <https://openbao.org/docs/secrets/kv/kv-v2/>
- OpenBao audit: <https://openbao.org/docs/next/audit/>
- OpenBao Kubernetes auth: <https://openbao.org/docs/2.4.x/auth/kubernetes/>
