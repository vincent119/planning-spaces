# K8s 部署規劃需求

## 文件定位

本 spec 定義 Opscenter 前端、後端與基礎服務部署到 Kubernetes 的規劃。部署拓樸採用：

- `opscenter-web`：React build 靜態檔，由 Nginx image 對外服務。
- `opscenter-api`：Gin API image，提供 API、health 與 metrics。
- Ingress：同網域 path-based routing。
- PostgreSQL：需支援目前 SQL 所需 extension，包含 `ulid`。

原始設計稿 `../2026-06-01_10-22_oncall-ticket-system` 不在本次修改範圍。

## 背景

目前系統包含前端 React、後端 Gin API、PostgreSQL、Redis、metrics、health check、scheduler 與通知 worker。若直接把前後端塞進同一個 image，會產生下列問題：

- 前端靜態檔更新會綁定 API image release。
- Nginx / CDN cache、SPA fallback 與靜態資源壓縮不易獨立調整。
- API 與 Web 資源需求不同，無法分開調整 replica、CPU 與 memory。
- Prometheus metrics 與 health check 路由邊界不清楚。

因此本次規劃採用前後端雙 image，Ingress 使用同網域路由，避免 CORS 與 cookie / SSO redirect 問題。

## 範圍

### 包含

- 前端 `opscenter-web` image 規劃。
- 後端 `opscenter-api` image 規劃。
- Nginx SPA static hosting 規劃。
- Ingress path routing 規劃。
- PostgreSQL image / Dockerfile 規劃。
- PostgreSQL StatefulSet / PVC /初始化流程規劃。
- PostgreSQL Kustomize manifests 需獨立於 app manifests，支援不同 namespace。
- PostgreSQL 定時備份 CronJob 與 IRSA 規劃。
- Helm chart 放置於 `deployments/helm`。
- Kustomize manifests 放置於 `deployments/kustomize`。
- Redis 部署或外部 Redis 連線規劃。
- ConfigMap / Secret / probes / metrics scrape 規劃。
- Vault Agent 提取設定檔或 Secret 的部署模式規劃。
- K8s manifests 與驗收 task 拆分。

### 不包含

- 本階段不執行 SQL。
- 本階段不部署到 Kubernetes。
- 本階段不調整應用程式邏輯。
- 本階段不修改初始設計稿。

## 需求 1：前端 Web Image

系統需要把前端打包為靜態檔 image，正式環境不得使用 Vite dev server。

### 驗收條件

- [ ] 1.1 前端 image 需使用 `npm run build` 產生 `dist`。
- [ ] 1.2 前端 image 需使用 Nginx serve 靜態檔。
- [ ] 1.3 Nginx 需支援 SPA fallback 到 `index.html`。
- [ ] 1.4 前端 API base URL 需使用相對路徑 `/api/v1`。
- [ ] 1.5 前端 image 不得包含 Node dev server runtime。
- [ ] 1.6 Nginx 設定檔位置需為 `opscenter-frontend/deploy/nginx.conf`。
- [ ] 1.7 Nginx listen port 固定為 `8080`。
- [ ] 1.8 前端 Makefile 需支援 Docker build、login、push 與 build-push 指令。
- [ ] 1.9 前端 Docker image repository 與 tag 需可由 Makefile 變數覆寫。
- [ ] 1.10 前端 Makefile 需相容 Linux CI runner，不得依賴 zsh。

## 需求 2：後端 API Image

系統需要把 Gin API 打包為獨立 image，並以 Kubernetes Deployment 執行。

### 驗收條件

- [ ] 2.1 後端 image 需使用 multi-stage build。
- [ ] 2.2 後端容器需由環境變數或 mounted config 取得 runtime 設定。
- [ ] 2.3 Secret 類設定不得寫入 image。
- [ ] 2.4 Deployment 需配置 livenessProbe 與 readinessProbe。
- [ ] 2.5 Deployment 需設定 graceful shutdown 所需的 termination grace period。
- [ ] 2.6 後端 Makefile 需支援 Docker build、login、push 與 build-push 指令。
- [ ] 2.7 後端 Docker image repository 與 tag 需可由 Makefile 變數覆寫。
- [ ] 2.8 後端 Makefile 需相容 Linux CI runner，不得依賴 zsh。
- [ ] 2.9 後端 runtime image 需提供 libvips、WebP 與 AVIF encoder 所需 runtime 套件，避免啟動時圖片轉檔能力檢查失敗。

## 需求 3：Ingress 與路由

系統需要使用同網域 path-based routing，讓前端與 API 使用相同 origin。

### 驗收條件

- [ ] 3.1 `/api/*` 需導向 `opscenter-api` Service。
- [ ] 3.2 `/api/v1/healthz/*` 需導向 `opscenter-api` Service。
- [ ] 3.3 `/` 需導向 `opscenter-web` Service。
- [ ] 3.4 metrics 不得直接公開給一般使用者。
- [ ] 3.5 前端靜態路由不得把 `/api/*` fallback 到 `index.html`。
- [ ] 3.6 使用 AWS Load Balancer Controller 時，Ingress 需指定 `alb` ingress class。
- [ ] 3.7 HTTP listener 需導向 HTTPS listener，不使用未接到 backend rule 的 redirect action annotation。
- [ ] 3.8 ALB health check 需依 backend Service 分開設定，API 使用 `/api/v1/healthz/live`，Web 使用 `/healthz`。

## 需求 4：PostgreSQL Image 與資料層

系統需要規劃可在 Kubernetes 使用的 PostgreSQL image，並保留正式環境可切換外部資料庫的能力。

### 驗收條件

- [ ] 4.1 PostgreSQL image 需基於 PostgreSQL 18.x。
- [ ] 4.2 PostgreSQL image 需安裝 `ulid` extension 所需檔案。
- [ ] 4.3 PostgreSQL image 不得包含正式密碼或正式資料。
- [ ] 4.4 PostgreSQL image 不得自動建立或提供 Opscenter tablespace 初始化腳本；tablespace 目錄改由手動維運流程建立。
- [ ] 4.5 正式資料需使用 PVC 或外部 PostgreSQL，不得存在 container writable layer。
- [ ] 4.6 SQL schema 初始化需由 DBA 或手動維運流程執行，不得混在 app container 啟動流程。
- [ ] 4.7 PostgreSQL Kustomize manifests 需拆分 Service、StatefulSet、Secret 範本與備份 CronJob。
- [ ] 4.8 PostgreSQL 備份 CronJob 需使用獨立 ServiceAccount，並可透過 IRSA annotation 取得物件儲存權限。
- [ ] 4.9 PostgreSQL 備份設定不得含正式 access key 或 secret key。
- [ ] 4.10 PostgreSQL data PVC 需有明確 disk name，AWS EBS CSI StorageClass 需帶 `Name` tag 範本。
- [ ] 4.11 PostgreSQL image 需提供 `deployments/postgres/Makefile`，並支援 `make build`。
- [ ] 4.12 PostgreSQL image Makefile 需支援 Docker Hub login / push，且不得保存 Docker Hub 密碼。
- [ ] 4.13 PostgreSQL image 需定位為公用 PostgreSQL 18 + ulid extension image，不得包含 Opscenter 專案初始化動作。
- [ ] 4.14 PostgreSQL image build 需可明確指定 `linux/amd64` 或 `linux/arm64`，避免 Kubernetes node 架構與 image manifest 不相容。
- [ ] 4.15 PostgreSQL image multi-arch build 需支援指定 buildx builder，避免在 QEMU 模擬環境編譯 Rust extension 時發生不穩定。

## 需求 5：Redis 與依賴服務

系統需要明確規劃 Redis 來源，避免 API 啟動後找不到 cache 服務。

### 驗收條件

- [ ] 5.1 開發 / 測試環境可使用 in-cluster Redis。
- [ ] 5.2 正式環境可切換外部 Redis。
- [ ] 5.3 Redis 密碼需使用 Kubernetes Secret。
- [ ] 5.4 API Deployment 需以環境變數注入 Redis 連線資訊。

## 需求 6：設定、Secret 與安全

Kubernetes 部署需把設定與敏感資料分離。

### 驗收條件

- [ ] 6.1 非敏感設定放 ConfigMap。
- [ ] 6.2 敏感設定放 Secret。
- [ ] 6.3 `JWT_SECRET`、DB password、Redis password、OIDC / SAML secret 不得進入 image。
- [ ] 6.4 metrics basic auth 密碼需由 Secret 提供。
- [ ] 6.5 container securityContext 需避免 privileged 與 root 權限，除非 PostgreSQL image 初始化階段有明確需求。
- [ ] 6.6 可選擇使用 `vault-agent` mutating webhook 從 Vault 或 AWS Secrets Manager 提取 Secret。
- [ ] 6.7 使用 `vault-agent` Pod injection 時，Namespace 必須標記 `vault-agent.io/admission-webhooks: "enabled"`。
- [ ] 6.8 使用 `vault-agent` Pod injection 時，Pod template labels 必須標記 `inject-vault-agent: "true"`。
- [ ] 6.9 使用 `vault-agent` Pod injection 時，Pod template annotations 必須包含 `inject-vault-agent: "true"`、backend、path 與必要 keys。
- [ ] 6.10 使用 `vault-agent` Secret sync 時，Secret labels 與 annotations 需標記 `inject-vault-agent: "true"`，並配置 backend、path 與必要 keys。
- [ ] 6.11 使用 Vault Agent 時，Vault token、role id、secret id 或 Kubernetes auth token 不得寫入 image。
- [ ] 6.12 若以 Secret sync 提供完整 `config.yaml`，需使用獨立 `opscenter-api-config-secret`，不得與環境變數 Secret `opscenter-api-secret` 共用名稱。

## 需求 7：Health、Metrics 與營運

Kubernetes 需要能判斷服務健康狀態，Prometheus 需要能抓取 metrics。

### 驗收條件

- [ ] 7.1 API liveness 使用 `/api/v1/healthz/live`。
- [ ] 7.2 API readiness 使用 `/api/v1/healthz/ready`。
- [ ] 7.3 Web liveness / readiness 可使用 `/` 或獨立 static health path。
- [ ] 7.4 metrics 使用 ServiceMonitor 或受限 internal Ingress。
- [ ] 7.5 access log 需排除 health 與 metrics，避免探針洗版。

## 需求 8：部署驗收

部署規劃需有可執行的驗收流程。

### 驗收條件

- [ ] 8.1 image build 可在本機或 CI 執行。
- [ ] 8.2 manifests 可通過 dry-run 或 kubeconform 類驗證。
- [ ] 8.3 Web 能載入 `index.html` 與靜態資源。
- [ ] 8.4 API health ready 回傳 DB / Redis 檢查結果。
- [ ] 8.5 Ingress 同網域 `/api/v1` 呼叫不需要 CORS。
- [ ] 8.6 PostgreSQL extension `ulid` 與 `generate_ulid()` 驗證通過。

## 需求 9：Helm 與 Kustomize 目錄

部署產物需同時保留 Helm 與 Kustomize 兩種路徑，方便正式環境與本機測試採不同部署方式。

### 驗收條件

- [ ] 9.1 Helm chart 需放在 `deployments/helm`。
- [ ] 9.2 Kustomize base / overlays 需放在 `deployments/kustomize`。
- [ ] 9.3 Helm 與 Kustomize 需共用相同 port 規劃：Web `8080`，API `9998`。
- [ ] 9.4 Helm 與 Kustomize 的 Secret 範本只能使用 placeholder。
- [ ] 9.5 Kustomize app base 與 PostgreSQL base 需拆開，允許使用不同 namespace。
- [ ] 9.6 Kustomize PostgreSQL base 需拆細為多個資源檔，不能把 Service、StatefulSet、備份 CronJob 全塞進單一 YAML。
- [ ] 9.7 PostgreSQL 掛載磁碟需在 Kustomize base 中保留可追蹤的 volume name、PVC label / annotation 與 StorageClass tag。
