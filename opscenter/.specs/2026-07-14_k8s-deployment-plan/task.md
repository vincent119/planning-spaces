# K8s 部署規劃 Task

## 1. 前端 Web Image

- [x] 1.1 建立前端 multi-stage Dockerfile
  - build stage 使用 Node LTS。
  - runtime stage 使用 Nginx。
  - _Requirements: 1.1, 1.2, 1.5_

- [x] 1.2 建立 Nginx SPA 設定
  - 支援 `try_files $uri $uri/ /index.html`。
  - 補 `/healthz` static health endpoint。
  - _Requirements: 1.3, 7.3_

- [x] 1.3 確認前端 API base URL
  - 使用 `/api/v1` 相對路徑。
  - 不寫死內部 Service DNS 或外部 domain。
  - _Requirements: 1.4, 3.5_

- [x] 1.4 建立前端 Docker build / push Makefile 指令
  - `docker-build` 使用 Dockerfile 預設值建置本機 image。
  - `docker-login` 執行 registry login，但不保存密碼。
  - `docker-push` 支援本機 image 重新標記後推送。
  - `docker-build-push` 串接 build 與 push。
  - `docker-print-config` 顯示 image repository 與 tag 參數。
  - Makefile 使用 `/bin/sh`，可在 Linux GitLab runner 執行。
  - 操作文件補在 `opscenter-frontend/README.md` 與 `deployments/README.md`。
  - _Requirements: 1.8, 1.9, 1.10_

## 2. 後端 API Image

- [x] 2.1 建立後端 multi-stage Dockerfile
  - 2026-08-07 修正：後端獨立 image 使用 `api_only` build tag，避免乾淨 Docker context 因缺少 `web/dist` 導致 Go embed 編譯失敗
  - build stage 編譯 `cmd/server`。
  - runtime stage 使用最小化 image。
  - runtime stage 安裝 `vips`、`libwebp`、`libheif`、`libheif-aom`，支援 WebP 與 AVIF 輸出能力檢查。
  - _Requirements: 2.1, 2.9_

- [x] 2.2 整理後端 runtime 設定注入
  - 確認 ConfigMap / Secret 對應環境變數。
  - Secret 不寫入 image。
  - _Requirements: 2.2, 2.3, 6.1, 6.2, 6.3_

- [x] 2.3 補 API probe 與 graceful shutdown 設定
  - liveness 使用 `/api/v1/healthz/live`。
  - readiness 使用 `/api/v1/healthz/ready`。
  - terminationGracePeriodSeconds 需符合後端 graceful shutdown。
  - 程式端 health endpoint 與 graceful shutdown 已存在；Deployment probe 已接入 `deployments/kustomize/base/api-deployment.yaml`。
  - _Requirements: 2.4, 2.5, 7.1, 7.2_

- [x] 2.4 建立後端 Docker build / push Makefile 指令
  - `docker-build` 使用 Dockerfile 預設值建置本機 image。
  - `docker-login` 執行 registry login，但不保存密碼。
  - `docker-push` 支援本機 image 重新標記後推送。
  - `docker-build-push` 串接 build 與 push。
  - `docker-print-config` 顯示 image repository 與 tag 參數。
  - Makefile 使用 `/bin/sh`，可在 Linux GitLab runner 執行。
  - 操作文件補在 `opscenter-server/README.md` 與 `deployments/README.md`。
  - _Requirements: 2.6, 2.7, 2.8_

## 3. Ingress 與 Service

- [x] 3.1 建立 Web / API Service manifests
  - `opscenter-web` 對應 Nginx port。
  - `opscenter-api` 對應 Gin API port。
  - _Requirements: 3.1, 3.2, 3.3_

- [x] 3.2 建立 Ingress path routing
  - `/api/*` 導向 API。
  - `/` 導向 Web。
  - API path 優先於 Web fallback。
  - AWS ALB 使用 `ingressClassName: alb` 與 `ssl-redirect` annotation。
  - API / Web health check 放在各自 Service annotation。
  - _Requirements: 3.1, 3.2, 3.3, 3.5, 3.6, 3.7, 3.8_

- [x] 3.3 規劃 metrics 存取方式
  - 優先使用 ServiceMonitor。
  - 若使用 Ingress，需限制來源與 auth。
  - _Requirements: 3.4, 7.4_

## 4. PostgreSQL Image 與 DB 初始化

- [x] 4.1 建立 PostgreSQL Dockerfile
  - 基於 PostgreSQL 18.x。
  - 安裝 `pgx_ulid` / `ulid` extension 所需檔案。
  - 不包含正式密碼或正式資料。
  - _Requirements: 4.1, 4.2, 4.3_

- [x] 4.2 確認 PostgreSQL image 不包含 Opscenter tablespace 初始化
  - PostgreSQL image 不建立 `/var/lib/postgresql/data/tablespaces/opscenter`。
  - PostgreSQL image 不建立 `/var/lib/postgresql/data/tablespaces/opscenter_temp`。
  - repository 不提供 Opscenter tablespace 初始化腳本。
  - tablespace 目錄由 DBA 或維運人員依目標環境手動建立。
  - _Requirements: 4.4, 4.5, 4.13_

- [x] 4.3 建立 PostgreSQL StatefulSet / PVC 規劃檔
  - 開發 / 測試可使用單 pod StatefulSet。
  - 正式環境需保留外部 PostgreSQL 或 Operator 路線。
  - PostgreSQL bootstrap superuser 使用 `postgres`，應用帳號由 schema / role SQL 或 DBA 維運流程建立。
  - PostgreSQL readiness / liveness probe 需明確使用 `POSTGRES_USER` 與 `POSTGRES_DB`，不得退回容器使用者 `root`。
  - _Requirements: 4.5_

- [x] 4.4 建立 DB schema / migration 手動執行規劃
  - 由 DBA 或維運人員手動審核並執行 `opscenter-server/sql/*.sql`。
  - `0008_database_roles_and_grants.sql` 需獨立審核與授權確認。
  - 不建立獨立 Kubernetes Job。
  - 不在 API container 啟動時自動 migration。
  - 需紀錄已執行 SQL 版本、執行時間與結果。
  - _Requirements: 4.6_

- [x] 4.5 補 PostgreSQL image 驗證
  - 驗證 `CREATE EXTENSION IF NOT EXISTS ulid;`。
  - 驗證 `generate_ulid()`。
  - 驗證 `btree_gist` 可用。
  - _Requirements: 8.6_

- [x] 4.6 建立 PostgreSQL image Makefile
  - `deployments/postgres/Makefile` 提供 `make build`。
  - `make build` 會先檢查 Docker daemon 是否可連線。
  - `make doctor` 可單獨檢查 Docker daemon 狀態。
  - 預設建置 `opscenter-postgres:18`。
  - 支援覆寫 `IMAGE`、`TAG`、`PG_MAJOR`、`PGRX_VERSION`、`PGX_ULID_REPO`、`PGX_ULID_REF`、`PLATFORM`、`PLATFORMS`、`BUILDX_BUILDER`、`CARGO_BUILD_JOBS`。
  - `make build` 預設使用 `PLATFORM=linux/amd64`。
  - `make buildx-push` 預設使用 `PLATFORMS=linux/amd64,linux/arm64`。
  - `CARGO_BUILD_JOBS` 預設為 `1`，降低 buildx multi-arch 編譯 `pgrx` 的記憶體峰值。
  - `BUILDX_BUILDER` 可指定原生或遠端 builder，避免 Apple Silicon 本機 QEMU 編譯不穩。
  - `PGX_ULID_REF` 預設為 `master`，符合 upstream repository 分支名稱。
  - PostgreSQL image 定位為公用 PostgreSQL 18 + ulid extension image。
  - _Requirements: 4.1, 4.2, 4.11, 4.13, 4.14, 4.15_

- [x] 4.7 建立 PostgreSQL image Docker Hub login / push 指令
  - `make login` 執行 Docker Hub login。
  - `make push REMOTE_TAG=tagname` 推送 `vincent119/postgres:tagname`。
  - `make push LOCAL_IMAGE=opscenter-postgres:18 REMOTE_TAG=tagname` 支援指定本機來源 image。
  - `make build-push` 串接 build 與 push。
  - `make buildx-push` 支援直接建置並推送指定 platform manifest。
  - `make buildx-list` 支援查看目前 buildx builder。
  - `make inspect-remote` 支援檢查 Docker Hub image manifest。
  - Makefile 不保存 Docker Hub 密碼。
  - _Requirements: 4.12, 4.14, 4.15_

## 5. Redis 與依賴服務

- [x] 5.1 建立 Redis dev manifest
  - 開發 / 測試環境可使用 in-cluster Redis。
  - _Requirements: 5.1_

- [x] 5.2 規劃正式 Redis 接線
  - 支援外部 Redis。
  - 密碼由 Secret 注入。
  - _Requirements: 5.2, 5.3, 5.4_

## 6. ConfigMap / Secret

- [x] 6.1 建立 ConfigMap 範本
  - 包含非敏感 app、db、redis、metrics、health 設定。
  - _Requirements: 6.1_

- [x] 6.2 建立 Secret 範本
  - 只提交 `secret.example.yaml`。
  - 正式 Secret 不進 git。
  - _Requirements: 6.2, 6.3, 6.4_

- [x] 6.3 補 container securityContext 規劃
  - Web / API 避免 root 與 privileged。
  - PostgreSQL 依官方 image 行為設定。
  - _Requirements: 6.5_

- [x] 6.4 規劃 Vault Agent 啟用條件
  - 保留不使用 Vault Agent 時的 Kubernetes Secret 路線。
  - 使用 Vault Agent 時，Namespace 需標記 `vault-agent.io/admission-webhooks: "enabled"`。
  - Vault / AWS Secrets Manager 來源需依環境區分。
  - _Requirements: 6.6, 6.7_

- [x] 6.5 規劃 Vault Agent Pod injection
  - Pod template labels 需標記 `inject-vault-agent: "true"`。
  - Pod template annotations 需包含 `inject-vault-agent: "true"`。
  - annotations 需包含 backend、path 與必要 keys。
  - 注入 key 需對應 DB、Redis、JWT、metrics、OIDC / SAML、Webhook 與 storage secrets。
  - _Requirements: 6.8, 6.9_

- [x] 6.6 規劃 Vault Agent Secret sync
  - Secret labels 與 annotations 需標記 `inject-vault-agent: "true"`。
  - Secret annotations 需包含 backend、path 與必要 keys。
  - 使用 `envFrom.secretRef` 接入 API Deployment。
  - 完整 `config.yaml` 使用獨立 `opscenter-api-config-secret` 掛載到 `/app/config`。
  - `opscenter-api-secret` 僅保留給敏感環境變數。
  - 標註 Secret 更新後既有 Pod 不會自動重新載入 env，需 rollout restart。
  - _Requirements: 6.10, 6.12_

- [x] 6.7 驗證 Vault Agent Secret 安全邊界
  - Vault token 不得寫入 image。
  - AppRole `role_id` / `secret_id` 不得寫入 image。
  - Kubernetes auth token 不得寫入 image。
  - secret path 需依環境分層，不同 namespace 不共用正式 secret。
  - _Requirements: 6.11_

## 7. Health、Metrics 與營運

- [x] 7.1 建立 ServiceMonitor 或 metrics scrape manifest
  - metrics endpoint 不公開給一般使用者。
  - 使用 basic auth 或內網限制。
  - ServiceMonitor 已接入 `deployments/kustomize/base/service-monitor.yaml`；Ingress 不導流 `/metrics` 到 API。
  - _Requirements: 7.4_

- [x] 7.2 驗證 health / metrics access log 排除
  - health 與 metrics 不應產生 access log 噪音。
  - 已補測試確認 health / metrics 進入 observability skip paths。
  - _Requirements: 7.5_

## 8. 部署驗收

- [ ] 8.1 驗證 image build
  - `opscenter-web`、`opscenter-api`、`opscenter-postgres` 均可 build。
  - 已驗證 `npm run build` 與 `go test ./...` 通過；Docker daemon 未啟動，image build 尚未完成。
  - _Requirements: 8.1_

- [ ] 8.2 驗證 manifests
  - 使用 dry-run 或 kubeconform 類工具檢查 manifests。
  - 已補本機離線驗收腳本 `deployments/validation/verify-local.rb`；`kubeconform` 未安裝，`kubectl dry-run` 因無可用 API discovery 尚未完成。
  - _Requirements: 8.2_

- [ ] 8.3 驗證 Web / API / Ingress
  - `/` 可載入前端。
  - `/api/v1/healthz/ready` 通過。
  - 同網域 API 呼叫不需要 CORS。
  - 已驗證前端 build 產出 `dist/index.html` 與 `dist/assets`；API readiness 與 Ingress 需部署後驗收。
  - _Requirements: 8.3, 8.4, 8.5_

- [ ] 8.4 驗證 PostgreSQL
  - `ulid` extension、tablespace 維運流程、schema SQL、seed SQL 通過。
  - 已提供 `deployments/postgres/verify.sql`；需由 DBA 或維運人員在目標 DB 執行。
  - _Requirements: 8.6_

## 9. Helm / Kustomize / 部署目錄落地

- [x] 9.1 建立部署目錄骨架
  - 建立 `deployments/helm`。
  - 建立 `deployments/kustomize/base`。
  - 建立 `deployments/kustomize/overlays/dev`。
  - 建立 `deployments/kustomize/overlays/prod`。
  - 建立 `opscenter-frontend/deploy`。
  - _Requirements: 1.6, 9.1, 9.2_

- [x] 9.2 建立 Helm chart 檔案落點
  - `Chart.yaml`。
  - `values.yaml`。
  - `templates/api-deployment.yaml`。
  - `templates/api-service.yaml`。
  - `templates/web-deployment.yaml`。
  - `templates/web-service.yaml`。
  - `templates/ingress.yaml`。
  - `templates/configmap.yaml`。
  - `templates/secret.example.yaml`。
  - `templates/servicemonitor.yaml`。
  - `templates/postgres-statefulset.yaml`。
  - `templates/redis.yaml`。
  - _Requirements: 9.1, 9.3, 9.4_

- [x] 9.3 建立 Kustomize base / overlay 檔案落點
  - `base/api-deployment.yaml`。
  - `base/api-service.yaml`。
  - `base/web-deployment.yaml`。
  - `base/web-service.yaml`。
  - `base/ingress.yaml`。
  - `base/configmap.yaml`。
  - `base/secret.example.yaml`。
  - `base/service-monitor.yaml`。
  - `base/redis.yaml`。
  - `base/kustomization.yaml`。
  - `postgres/base/kustomization.yaml`。
  - `postgres/base/storageclass.yaml`。
  - `postgres/base/service.yaml`。
  - `postgres/base/statefulset.yaml`。
  - `postgres/base/secret.example.yaml`。
  - `overlays/dev/kustomization.yaml`。
  - `overlays/prod/kustomization.yaml`。
  - _Requirements: 9.2, 9.3, 9.4_

- [x] 9.4 建立 PostgreSQL 與 Nginx 部署檔案落點
  - `deployments/postgres/Dockerfile`。
  - `opscenter-frontend/deploy/nginx.conf`。
  - _Requirements: 1.6, 1.7, 4.1, 4.4_

- [x] 9.5 拆分 Kustomize PostgreSQL namespace
  - `deployments/kustomize/base` 只保留 app namespace 物件。
  - `deployments/kustomize/postgres/base` 放 PostgreSQL 資源。
  - `deployments/kustomize/postgres/overlays/dev` 使用 `opscenter-db-dev`。
  - `deployments/kustomize/postgres/overlays/prod` 使用 `opscenter-db`。
  - app overlay 的 `DB_HOST` 使用跨 namespace service DNS。
  - _Requirements: 4.7, 9.2, 9.5_

- [x] 9.6 拆細 PostgreSQL base 並加入 IRSA 備份 CronJob
  - `service.yaml` 與 `statefulset.yaml` 分離。
  - `backup-serviceaccount.yaml` 保留 IRSA role arn placeholder。
  - `backup-cronjob.yaml` 保留每日備份排程與 S3 placeholder。
  - 備份 image 使用 `opscenter-postgres-backup:18`，需包含 `pg_dump`、`gzip`、`aws`。
  - overlay 可 patch 備份 namespace、schedule、bucket、prefix、region 與 IRSA role arn。
  - _Requirements: 4.8, 4.9, 9.6_

- [x] 9.7 補 PostgreSQL 掛載磁碟 tag name
  - PostgreSQL data volumeMount / PVC template 使用 `postgres-data`。
  - PVC metadata 加上 `opscenter.openai.io/disk-name=opscenter-postgres-data`。
  - PVC annotation 加上 `opscenter.openai.io/aws-ebs-name-tag=opscenter-postgres-data`。
  - 新增 `opscenter-postgres-gp3` StorageClass，透過 EBS CSI `tagSpecification_1` 設定 `Name=opscenter-postgres-data`。
  - dev / prod overlay 分別 patch 為 `opscenter-dev-postgres-data` 與 `opscenter-prod-postgres-data`。
  - Backup CronJob 的暫存 volume 使用 `postgres-backup-tmp`，避免 generic `tmp`。
  - _Requirements: 4.10, 9.7_
