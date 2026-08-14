# K8s 部署規劃設計

## 文件定位

本文件對應 `requirements.md`，定義 Opscenter 部署到 Kubernetes 的建議架構。此階段只做規劃設計，不建立實際部署檔。

## 目標拓樸

```text
Client
  |
  v
Ingress / TLS
  |-- /api/*             -> opscenter-api Service -> opscenter-api Deployment
  |-- /api/v1/healthz/*  -> opscenter-api Service -> opscenter-api Deployment
  |-- /metrics           -> internal only / ServiceMonitor
  `-- /*                 -> opscenter-web Service -> opscenter-web Deployment

opscenter-api
  |-- PostgreSQL writer
  |-- PostgreSQL reader, optional
  `-- Redis
```

## 設計原則

- 前後端分離 image，但使用同一個外部網域。
- 前端 image 只服務靜態檔，不包含 Vite dev server。
- 後端 image 只負責 API、health、metrics 與 background worker。
- 後端獨立 image 使用 Go build tag `api_only`，使 `web.Dist()` 回傳 `nil`，不依賴或內嵌前端 `web/dist`；整合式根目錄 Dockerfile 維持內嵌前端資產。
- Secret 不進 image。
- 可選擇使用 `vault-agent` mutating webhook 注入 Secret。
- PostgreSQL image 只提供 PostgreSQL 18 與 extension，不內建正式資料，也不執行 Opscenter 專案初始化。
- Schema / seed / migration 由 DBA 或手動維運流程執行，不放在 API container 啟動流程。
- metrics 不公開給一般使用者。
- Helm chart 固定放在 `deployments/helm`。
- Kustomize base / overlays 固定放在 `deployments/kustomize`。
- Kustomize app base 與 PostgreSQL base 分離，允許 API / Web 與 DB 使用不同 namespace。

## Image 規劃

### opscenter-web

建議路徑：

```text
opscenter-frontend/Dockerfile
opscenter-frontend/deploy/nginx.conf
```

設計：

```text
build stage:
  image: node LTS
  action:
    npm ci
    npm run build

runtime stage:
  image: nginx stable alpine 或 nginx unprivileged
  copy:
    dist -> /usr/share/nginx/html
    nginx.conf -> /etc/nginx/conf.d/default.conf
  port:
    8080
```

前端 Nginx 設定檔固定放置：

```text
opscenter-frontend/deploy/nginx.conf
```

Nginx 必要設定：

```nginx
server {
    listen 8080;
    server_name _;

    root /usr/share/nginx/html;
    index index.html;

    location / {
        try_files $uri $uri/ /index.html;
    }

    location = /healthz {
        access_log off;
        return 200 "ok\n";
    }
}
```

注意：

- `/api/*` 應由 Ingress 先轉到 API，不應進入 Web Nginx。
- 前端呼叫 API 使用 `/api/v1` 相對路徑。
- 靜態資源可設定長 cache，`index.html` 不設定長 cache。

前端 Makefile 需提供：

```bash
make -C opscenter-frontend docker-build TAG=20260721
make -C opscenter-frontend docker-login
make -C opscenter-frontend docker-push REMOTE_TAG=20260721
make -C opscenter-frontend docker-build-push TAG=20260721 REMOTE_TAG=20260721
make -C opscenter-frontend docker-print-config
```

前端 Docker 變數：

```text
IMAGE=opscenter-web
TAG=dev
LOCAL_IMAGE=$(IMAGE):$(TAG)
REMOTE_IMAGE=vincent119/opscenter-web
REMOTE_TAG=$(TAG)
```

Makefile 標準流程不暴露 platform 與 base image version，避免日常 build / push 過度複雜。若需覆寫 Dockerfile `ARG`，由維運人員直接使用原生 `docker build --build-arg ...`。

Makefile shell 固定使用 `/bin/sh`，避免 GitLab runner 或 Linux builder 沒安裝 zsh 時無法執行 Docker 指令。

操作文件：

```text
opscenter-frontend/README.md
deployments/README.md
```

### opscenter-api

建議路徑：

```text
opscenter-server/Dockerfile
```

設計：

```text
build stage:
  image: golang
  action:
    go mod download
    go test ./...
    go build ./cmd/server

runtime stage:
  image: distroless 或 alpine
  copy:
    server binary
    config template, optional
  env:
    APP_ENV
    APP_PORT
    DB_HOST
    DB_PORT
    DB_NAME
    DB_USER
    DB_PASSWORD
    REDIS_ADDR
    REDIS_PASSWORD
    JWT_SECRET
    METRICS_PASSWORD
```

runtime image 套件需包含：

```text
vips
libwebp
libheif
libheif-aom
```

原因：

- API 啟動時會初始化 govips / libvips，並驗證 WebP 與 AVIF 輸出能力。
- `libheif` 只提供 HEIF/AVIF 容器支援時，不代表已具備 AVIF encoder。
- Alpine runtime 需安裝 `libheif-aom`，讓 `heifsave` 可用 AV1 encoder 輸出 AVIF。

API listen port 沿用目前設定：

```text
APP_PORT=9998
```

後端 Makefile 需提供：

```bash
make -C opscenter-server docker-build TAG=20260721
make -C opscenter-server docker-login
make -C opscenter-server docker-push REMOTE_TAG=20260721
make -C opscenter-server docker-build-push TAG=20260721 REMOTE_TAG=20260721
make -C opscenter-server docker-print-config
```

後端 Docker 變數：

```text
IMAGE=opscenter-api
TAG=dev
LOCAL_IMAGE=$(IMAGE):$(TAG)
REMOTE_IMAGE=vincent119/opscenter-api
REMOTE_TAG=$(TAG)
```

Makefile 標準流程不暴露 platform 與 base image version，避免日常 build / push 過度複雜。若需覆寫 Dockerfile `ARG`，由維運人員直接使用原生 `docker build --build-arg ...`。

Makefile shell 固定使用 `/bin/sh`，避免 GitLab runner 或 Linux builder 沒安裝 zsh 時無法執行 Docker 指令。

操作文件：

```text
opscenter-server/README.md
deployments/README.md
```

設定載入方式：

```text
1. 程式先讀取工作目錄下的 config/config.yaml。
2. 若檔案不存在，使用程式內 defaults。
3. 接著套用環境變數覆寫。
```

Kubernetes 建議：

```text
ConfigMap:
  提供非敏感環境變數

Secret:
  透過 env 注入 DB_PASSWORD、JWT_SECRET、Redis password 等敏感值

Secret sync:
  若由 vault-agent 提供完整 config.yaml，使用 opscenter-api-config-secret
  mount 到 /app/config，Secret key 為 config.yaml
```

限制：

- 目前程式入口呼叫 `config.Load("config/config.yaml")`，尚未支援任意 `CONFIG_PATH`。
- 若要改成可設定 config path，需另開 task 修改 `cmd/server/main.go` 與 config 測試。
- `opscenter-api-secret` 保留給 `envFrom` 敏感環境變數，不得同時承載完整 config file。
- `opscenter-api-config-secret` 需在 Pod 啟動前已包含 `config.yaml`，否則 API 會因讀不到設定檔而啟動失敗。

Kubernetes probe：

```text
livenessProbe:
  path: /api/v1/healthz/live

readinessProbe:
  path: /api/v1/healthz/ready
```

關機設定：

```text
terminationGracePeriodSeconds >= app shutdown timeout + 5 到 10 秒 buffer
preStop 可保留短暫 sleep，讓 Ingress endpoint 更新
```

## Ingress 設計

建議 path：

```text
/api(/|$)(.*)       -> opscenter-api:9998
/metrics            -> 不透過公開 Ingress，或加 allowlist / auth
/(.*)               -> opscenter-web:8080
```

使用同網域的優點：

- 前端不需要 CORS。
- OIDC / SAML redirect URL 較穩定。
- Cookie 與 token 行為集中。
- 使用者只需要記住一個 URL。

注意：

- `/api` route 必須優先於 `/` route。
- Web Nginx 的 SPA fallback 不可攔截 API request。
- metrics 建議由 Prometheus 透過 ServiceMonitor 抓取。
- AWS Load Balancer Controller 使用 `ingressClassName: alb`。
- HTTP 轉 HTTPS 使用 `alb.ingress.kubernetes.io/ssl-redirect: "443"`，不使用未掛到 backend path 的 `actions.ssl-redirect`。
- ACM certificate ARN 與 ALB security groups 屬於環境專屬值，需放在 overlay patch，不放在 base Ingress。
- ALB 會對不同 backend Service 建立不同 target group，health check path 應放在 Service annotations：
  - `opscenter-api` 使用 `/api/v1/healthz/live`。
  - `opscenter-web` 使用 `/healthz`。

## PostgreSQL 設計

### 使用策略

正式環境優先順序：

1. 外部 managed PostgreSQL。
2. Kubernetes PostgreSQL Operator。
3. 自行維護 StatefulSet。

本 spec 仍規劃 PostgreSQL Dockerfile，原因是目前 schema 依賴 `ulid` extension；若採 in-cluster PostgreSQL，需要可重建且可版本控管的 DB image。

### PostgreSQL Dockerfile 規劃

建議路徑：

```text
deployments/postgres/Dockerfile
deployments/postgres/Makefile
```

Dockerfile 初稿：

```dockerfile
ARG PG_MAJOR=18
ARG PGRX_VERSION=0.17.0

FROM postgres:${PG_MAJOR}-bookworm AS pgx-ulid-builder

USER root

RUN apt-get update \
    && apt-get install -y --no-install-recommends \
        build-essential \
        ca-certificates \
        clang \
        curl \
        git \
        libclang-dev \
        pkg-config \
    && rm -rf /var/lib/apt/lists/*

ENV CARGO_HOME=/usr/local/cargo
ENV RUSTUP_HOME=/usr/local/rustup
ENV PATH=/usr/local/cargo/bin:${PATH}

RUN curl -fsSL https://sh.rustup.rs \
    | sh -s -- -y --profile minimal

RUN cargo install cargo-pgrx --version ${PGRX_VERSION} --locked

RUN cargo pgrx init --pg${PG_MAJOR} /usr/lib/postgresql/${PG_MAJOR}/bin/pg_config

RUN git clone --depth=1 https://github.com/pksunkara/pgx_ulid.git /tmp/pgx_ulid

WORKDIR /tmp/pgx_ulid

RUN cargo pgrx install --release --pg-config /usr/lib/postgresql/${PG_MAJOR}/bin/pg_config

FROM postgres:${PG_MAJOR}-bookworm

ARG PG_MAJOR=18

COPY --from=pgx-ulid-builder /usr/share/postgresql/${PG_MAJOR}/extension/*ulid* /usr/share/postgresql/${PG_MAJOR}/extension/
COPY --from=pgx-ulid-builder /usr/lib/postgresql/${PG_MAJOR}/lib/*ulid* /usr/lib/postgresql/${PG_MAJOR}/lib/
```

說明：

- 這是規劃稿，實作時需驗證 `pgx_ulid` build 後的 control / sql / so 檔名。
- PostgreSQL 版本固定以 18.x 規劃。
- `PGRX_VERSION` 需與 `pgx_ulid` 使用的 `pgrx` 版本相容，並需確認支援 PostgreSQL 18。
- image 不包含正式 DB 密碼。
- image 不包含正式資料。
- image 不包含 `docker-entrypoint-initdb.d` 專案初始化腳本。
- image 不提供 Opscenter 專案專屬 tablespace 初始化腳本。
- Opscenter tablespace 目錄由 DBA 或維運人員手動建立。
- 若正式環境使用 managed PostgreSQL，仍需由 DBA 在 DB 端安裝對應 extension。

Image build 入口：

```bash
make -C deployments/postgres build
```

建置前檢查 Docker daemon：

```bash
make -C deployments/postgres doctor
```

Docker Hub 登入與推送：

```bash
make -C deployments/postgres login
make -C deployments/postgres push REMOTE_TAG=tagname
make -C deployments/postgres build-push TAG=18-20260716 REMOTE_TAG=tagname
make -C deployments/postgres buildx-push REMOTE_TAG=18-20260716
make -C deployments/postgres buildx-push REMOTE_TAG=18-20260716 PLATFORMS=linux/amd64
make -C deployments/postgres buildx-push REMOTE_TAG=18-20260716 CARGO_BUILD_JOBS=2
make -C deployments/postgres buildx-push REMOTE_TAG=18-20260716 PLATFORMS=linux/amd64 BUILDX_BUILDER=<builder-name>
```

指定本機來源 image：

```bash
make -C deployments/postgres push LOCAL_IMAGE=opscenter-postgres:18 REMOTE_TAG=tagname
```

推送規則：

```text
local image:  opscenter-postgres:<TAG>
remote image: vincent119/postgres:<REMOTE_TAG>
```

限制：

- Makefile 不保存 Docker Hub 密碼。
- `push` 只負責 tag 與 push，不隱含登入；登入需由 `login` 明確執行。

Makefile 需支援覆寫：

```text
IMAGE
TAG
LOCAL_IMAGE
REMOTE_IMAGE
REMOTE_TAG
PG_MAJOR
PGRX_VERSION
PGX_ULID_REPO
PGX_ULID_REF
PLATFORM
PLATFORMS
BUILDX_BUILDER
CARGO_BUILD_JOBS
```

`pksunkara/pgx_ulid` 的預設分支使用 `master`，因此 `PGX_ULID_REF` 預設值不得設為 `main`。

Image platform 規則：

- `make build` 預設使用 `PLATFORM=linux/amd64`，避免在 Apple Silicon 本機 build 後只產出 `linux/arm64` image。
- 若 Kubernetes node 是 ARM64，需使用 `PLATFORM=linux/arm64`。
- `buildx-push` 預設使用 `PLATFORMS=linux/amd64,linux/arm64`，讓同一個 tag 同時支援 AMD64 與 ARM64。
- 若只要單一平台，可明確覆寫 `PLATFORMS=linux/amd64` 或 `PLATFORMS=linux/arm64`。
- `pgrx` 編譯在 multi-arch buildx 下記憶體用量較高，預設使用 `CARGO_BUILD_JOBS=1` 降低記憶體峰值。
- Apple Silicon 本機透過 QEMU 模擬 `linux/amd64` 編譯 Rust / C toolchain 可能出現 linker `SIGSEGV`，正式推送建議使用原生 amd64 builder、遠端 BuildKit builder 或 CI runner。
- `BUILDX_BUILDER` 可指定 buildx builder，避免誤用本機 QEMU builder。
- 上線前可用 `make -C deployments/postgres inspect-remote REMOTE_TAG=<tag>` 檢查 Docker Hub manifest。

PostgreSQL StatefulSet probe 規則：

- readiness / liveness 需透過 `sh -ec` 執行 `pg_isready`，讓 `POSTGRES_USER` 與 `POSTGRES_DB` 由 shell 展開。
- probe 不得依賴容器執行使用者作為預設 DB role，避免 PostgreSQL log 持續出現 `role "root" does not exist`。
- probe 指令需明確使用 `-U "$POSTGRES_USER"` 與 `-d "$POSTGRES_DB"`。

### tablespace 維運邊界

PostgreSQL image 只提供 PostgreSQL 18.x 與 `ulid` extension，不包含 Opscenter 專案專屬 tablespace 初始化。需要 in-cluster PostgreSQL tablespace 時，由 DBA 或維運人員在確認目標資料目錄、PVC 掛載狀態與 owner 後手動建立。

規劃目錄：

```text
/var/lib/postgresql/data/tablespaces/opscenter
/var/lib/postgresql/data/tablespaces/opscenter_temp
```

注意：

- repository 不提供 `docker-entrypoint-initdb.d` 或 `manual` tablespace 腳本。
- 建立 database、role、schema、extension 與 `generate_ulid()` 交由 DBA 或手動維運流程執行。

### DB schema 初始化規劃

本階段不規劃獨立 Kubernetes Job 執行 migration。DB schema 初始化與變更由 DBA 或手動維運流程執行，原因如下：

```text
DBA / 維運流程
  1. 審核 opscenter-server/sql/*.sql
  2. 確認目標環境、角色、權限與 extension
  3. 依序執行尚未套用的 SQL
  4. 紀錄執行版本與結果
```

原則：

- 不在 API container 啟動時自動跑 migration。
- 不在本階段新增獨立 Kubernetes Job。
- migration 需有人工審核、回復計畫與可重跑策略。
- `0008_database_roles_and_grants.sql` 涉及角色與權限，正式環境應由 DBA 執行或核准。
- 每次 schema 變更追加新序號 SQL，不修改已執行 SQL。

### StatefulSet 規劃

若使用 in-cluster PostgreSQL：

```text
kind: StatefulSet
name: opscenter-postgres
volumeClaimTemplates:
  - data
service:
  ClusterIP
secret:
  POSTGRES_PASSWORD
  postgres superuser password
  app_user password 由 schema / role SQL 或 DBA 維運流程另外建立
```

PostgreSQL official image 的 `POSTGRES_USER` 是初始化 superuser。本部署規劃固定使用 `postgres` 作為 bootstrap superuser，避免把應用帳號誤當成 cluster superuser。`app_user` 需由後續 schema / role SQL 或 DBA 維運流程建立與授權。

Kustomize 需將 PostgreSQL 拆成獨立 base：

```text
deployments/kustomize/base
  API / Web / Redis / Ingress / ServiceMonitor

deployments/kustomize/postgres/base
  PostgreSQL Service
  PostgreSQL StatefulSet
  PostgreSQL Secret 範本
  PostgreSQL backup ServiceAccount
  PostgreSQL backup CronJob
```

PostgreSQL data volume 命名：

```text
volumeMount name: postgres-data
PVC template name: postgres-data
PVC disk label: opscenter.openai.io/disk-name=opscenter-postgres-data
PVC disk annotation: opscenter.openai.io/aws-ebs-name-tag=opscenter-postgres-data
StorageClass: opscenter-postgres-gp3
AWS EBS CSI tagSpecification_1: Name=opscenter-postgres-data
```

說明：

- PVC metadata label / annotation 讓 Kubernetes 內可追蹤掛載磁碟用途。
- AWS EBS 實體 volume 的 `Name` tag 由 EBS CSI StorageClass `tagSpecification_*` 產生。
- dev / prod overlay 需 patch 成環境專屬 StorageClass 與 `Name` tag，避免同叢集多環境磁碟標籤撞名。
- 若正式環境使用既有 StorageClass，需保留等效的 EBS tag policy。

Namespace 建議：

```text
dev app: opscenter-dev
dev db:  opscenter-db-dev
prod app: opscenter
prod db:  opscenter-db
```

當 API 與 PostgreSQL 位於不同 namespace 時，API `DB_HOST` 需使用跨 namespace service DNS：

```text
opscenter-postgres.<db-namespace>.svc.cluster.local
```

### 備份 CronJob 與 IRSA

PostgreSQL 備份採獨立 CronJob，不放在 PostgreSQL 主 container。備份 Job 使用獨立 ServiceAccount，避免把 S3 權限授予資料庫主 Pod。

ServiceAccount：

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: opscenter-postgres-backup
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::<account-id>:role/<role-name>
```

CronJob：

```text
schedule: 0 18 * * *
timezone: Asia/Taipei
image: opscenter-postgres-backup:18
command:
  1. pg_dump 產生 compressed dump
  2. 使用 IRSA 取得 AWS 權限
  3. 上傳至 S3 bucket / prefix
```

限制：

- `opscenter-postgres-backup:18` 需包含 `pg_dump`、`gzip`、`aws` 指令。
- CronJob manifest 只能放 bucket / prefix / region 等非敏感設定或 placeholder。
- 不得在 manifest 放入 AWS access key 或 secret key。
- IRSA role arn 由 overlay patch 注入；base 只能保留 placeholder。
- 開發環境可保留 placeholder，不要求實際可上傳。

限制：

- 單 pod PostgreSQL 適合開發 / 測試 / 小型內部環境。
- 正式高可用建議使用 PostgreSQL Operator 或 managed PostgreSQL。
- backup / restore 不可依賴 PVC snapshot 單一手段。

## Redis 設計

開發 / 測試：

```text
opscenter-redis Deployment 或 StatefulSet
Service: opscenter-redis
```

正式環境：

```text
外部 Redis 或具備 HA 的 Redis 方案
```

設定：

```text
REDIS_ADDR=opscenter-redis:6379
REDIS_PASSWORD 從 Secret 注入
```

## ConfigMap / Secret 設計

### ConfigMap

放非敏感設定：

```text
APP_ENV
APP_PORT
DB_HOST
DB_PORT
DB_NAME
REDIS_ADDR
METRICS_ENABLED
METRICS_PATH
HEALTH_LIVENESS_PATH
HEALTH_READINESS_PATH
```

### Secret

放敏感設定：

```text
DB_USER
DB_PASSWORD
REDIS_PASSWORD
JWT_SECRET
METRICS_PASSWORD
OIDC_CLIENT_SECRET
SAML_PRIVATE_KEY
WEBHOOK secrets
```

原則：

- Secret 不提交到 git。
- 正式環境建議接 External Secrets 或 sealed secret 類方案。
- 若環境已有 Vault，建議可用 `vault-agent` Pod injection 或 Secret sync 取代部分 Kubernetes Secret。
- 本地範例只能使用 placeholder。

## Vault Agent 設計

可選方案：

```text
https://github.com/vincent119/vault-agent
```

用途：

- 使用 mutating webhook 依 Pod label / annotations 注入 Secret。
- 從 Vault KV v2 或 AWS Secrets Manager 讀取指定 key。
- 可直接做 Pod env injection。
- 可透過 Secret sync 覆寫 Kubernetes Secret.Data。

### Webhook 前置條件

`vault-agent` 有兩層 webhook filter，兩者都必須符合。

Namespace 必須有 label：

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: opscenter
  labels:
    vault-agent.io/admission-webhooks: "enabled"
```

Pod template labels 必須有：

```yaml
spec:
  template:
    metadata:
      labels:
        inject-vault-agent: "true"
```

注意：

- `inject-vault-agent: "true"` 必須放在 `spec.template.metadata.labels`。
- 放在 Deployment 最外層 `metadata.labels` 不會被 webhook 的 objectSelector 命中。
- Namespace 沒有 `vault-agent.io/admission-webhooks: "enabled"` 時，API Server 不會把 Pod admission request 送給 `vault-agent`。

### 模式 A：Pod env injection

建議第一版用於 `opscenter-api` 的敏感環境變數，因目前 Go config 會在讀取 `config/config.yaml` 後套用 env 覆寫。

範例：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: opscenter-api
  namespace: opscenter
spec:
  selector:
    matchLabels:
      app: opscenter-api
  template:
    metadata:
      labels:
        app: opscenter-api
        inject-vault-agent: "true"
      annotations:
        inject-vault-agent: "true"
        inject-vault-agent.backend: "vault"
        inject-vault-agent.path: "opscenter/prod/api"
        inject-vault-agent.keys: '["DB_USER","DB_PASSWORD","JWT_SECRET","REDIS_PASSWORD","METRICS_PASSWORD"]'
    spec:
      containers:
        - name: api
          image: opscenter-api
```

建議注入 key：

```text
DB_USER
DB_PASSWORD
REDIS_PASSWORD
JWT_SECRET
METRICS_PASSWORD
OIDC_CLIENT_SECRET
SSO_COMPANY_OIDC_CLIENT_SECRET
SSO_COMPANY_SAML_PRIVATE_KEY
WEBHOOK_SECRET_ENCRYPTION_KEY
S3_ACCESS_KEY_ID
S3_SECRET_ACCESS_KEY
```

優點：

- 不需要改 Go 程式。
- 符合目前 env 覆寫 config file 的載入順序。
- Secret 不需要提交到 Kubernetes Secret manifest。

限制：

- Secret 變更不會自動套用到既有 process。
- 需要 rollout restart 才能讓新 Pod 重新注入 Secret。
- 需要先部署並啟用 `vault-agent` mutating webhook。

### 模式 B：Secret sync

適合需要讓既有 K8s Secret 維持同步的場景。

`vault-agent` sync worker 會掃描帶有 `inject-vault-agent=true` label 的 Secret，並使用相同 annotations 取得資料後覆寫 `Secret.Data`。

範例：

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: opscenter-api-secret
  namespace: opscenter
  labels:
    inject-vault-agent: "true"
  annotations:
    inject-vault-agent.backend: "vault"
    inject-vault-agent.path: "opscenter/prod/api"
    inject-vault-agent.keys: '["DB_USER","DB_PASSWORD","JWT_SECRET","REDIS_PASSWORD"]'
type: Opaque
data: {}
```

使用方式：

```yaml
envFrom:
  - secretRef:
      name: opscenter-api-secret
```

若 Secret sync 用於完整設定檔，使用獨立 Secret，避免覆蓋 env secret：

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: opscenter-api-config-secret
  labels:
    inject-vault-agent: "true"
  annotations:
    inject-vault-agent: "true"
    inject-vault-agent.backend: "vault"
    inject-vault-agent.path: "im/devops/opscenter"
    inject-vault-agent.keys: '["config.yaml"]'
type: Opaque
data: {}
```

Deployment mount：

```yaml
volumeMounts:
  - name: api-config
    mountPath: /app/config
    readOnly: true
volumes:
  - name: api-config
    secret:
      secretName: opscenter-api-config-secret
      items:
        - key: config.yaml
          path: config.yaml
```

限制：

- Secret sync 更新 Secret 後，已啟動 Pod 的 env 不會自動更新。
- 若使用 envFrom，仍需 rollout restart 才能套用新值。
- 若掛載為 volume，檔案可能更新，但目前 API 不會熱載入 config。

### Backend / path / keys

annotations：

```text
inject-vault-agent: "true"
inject-vault-agent.backend: "vault" 或 "aws"
inject-vault-agent.path: secret path
inject-vault-agent.keys: JSON array string，可省略表示抓取所有 key-value
```

Vault path 說明：

```text
Vault backend 使用 KV v2 path，不包含 mount。
AWS backend 使用 Secrets Manager secret name 或 ARN。
```

安全要求：

- Vault token 不得寫入 image。
- AppRole `role_id` / `secret_id` 不得寫入 image。
- Vault policy 需最小權限，只能讀取 Opscenter 所需 path。
- Vault / AWS Secret path 需依環境分層，不同 namespace 不共用同一組正式 secret。

### 建議 Secret Path

```text
secret/data/opscenter/{env}/config
secret/data/opscenter/{env}/db
secret/data/opscenter/{env}/redis
secret/data/opscenter/{env}/jwt
secret/data/opscenter/{env}/sso
secret/data/opscenter/{env}/metrics
secret/data/opscenter/{env}/storage
```

### 與現有 Go Config 的關係

目前後端設定載入順序：

```text
1. 讀取 config/config.yaml。
2. 檔案不存在時使用 defaults。
3. 套用環境變數覆寫。
```

因此 Vault Agent 第一版最穩定的接法是使用 Pod env injection 或 Secret sync 提供敏感環境變數。若仍保留 mounted `config/config.yaml`，需定義優先權：

```text
mounted config/config.yaml < vault-agent injected env 或 Kubernetes Secret env
```

也就是 env 仍可覆寫 config file。

## Kubernetes Resource 規劃

### Web

```text
Deployment:
  replicas: 2
  image: opscenter-web
  port: 8080

Service:
  port: 8080
```

### API

```text
Deployment:
  replicas: 2
  image: opscenter-api
  port: 9998
  envFrom:
    ConfigMap
    Secret
  optional:
    vault-agent Pod template labels / annotations
    或 Secret sync 後透過 envFrom 注入

Service:
  port: 9998
```

注意：

- API 多 pod 時，scheduler 已有分散式鎖規劃；所有 scheduler 任務仍需確認 lock 保護。
- readiness 需依 DB / Redis 檢查結果決定。
- metrics endpoint 不應出現在公開 Ingress。

### PostgreSQL

```text
StatefulSet:
  replicas: 1
  image: opscenter-postgres
  pvc:
    data
```

### Redis

```text
Deployment 或 StatefulSet:
  replicas: 1
  image: redis
```

## 目錄規劃

本次部署實作需依下列目錄落地：

```text
deployments/
  helm/
    Chart.yaml
    values.yaml
    templates/
      api-deployment.yaml
      api-service.yaml
      web-deployment.yaml
      web-service.yaml
      ingress.yaml
      configmap.yaml
      secret.example.yaml
      servicemonitor.yaml
      postgres-statefulset.yaml
      redis.yaml
  kustomize/
    base/
      api-deployment.yaml
      api-service.yaml
      web-deployment.yaml
      web-service.yaml
      ingress.yaml
      configmap.yaml
      secret.example.yaml
      service-monitor.yaml
      redis.yaml
      kustomization.yaml
    overlays/
      dev/
        kustomization.yaml
      prod/
        kustomization.yaml
    postgres/
      base/
        kustomization.yaml
        storageclass.yaml
        service.yaml
        statefulset.yaml
        secret.example.yaml
        backup-serviceaccount.yaml
        backup-cronjob.yaml
      overlays/
        dev/
          kustomization.yaml
        prod/
          kustomization.yaml
  postgres/
    Dockerfile

opscenter-frontend/
  deploy/
    nginx.conf
```

Port 固定：

```text
opscenter-web: 8080
opscenter-api: 9998
PostgreSQL: 5432
Redis: 6379
```

## 驗收規劃

### Image

- `opscenter-web` image 內需存在 `index.html` 與 hashed assets。
- `opscenter-api` image 啟動後 health live 通過。
- `opscenter-postgres` image 可執行 `CREATE EXTENSION IF NOT EXISTS ulid;`。

### Kubernetes

- manifests 可 dry-run。
- `kubectl get pods` 全部 ready。
- `/` 可載入前端。
- `/api/v1/healthz/ready` 回傳 DB / Redis ready。
- `/api/v1` 同網域呼叫不需要 CORS。
- Prometheus 可抓取 API metrics。

### PostgreSQL

需驗證：

```sql
CREATE EXTENSION IF NOT EXISTS ulid;
SELECT generate_ulid();
SELECT extname FROM pg_extension WHERE extname IN ('ulid', 'pgx_ulid', 'btree_gist');
```

若 `generate_ulid()` 尚未建立，需先執行初始化 SQL 中的 wrapper 建立步驟。
