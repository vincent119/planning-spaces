# Metrics Observability 設計

## 文件定位

本文件描述後端如何把既有 `metrics` 設定接到 Prometheus runtime 行為。需求來源為本 spec 的 `requirements.md`。

原始設計稿 `../2026-06-01_10-22_oncall-ticket-system` 不在本次修改範圍。

## 現況

目前已存在：

- `config.Config.Metrics`
- `EndpointConfig`
- `BasicAuthConfig`
- `defaults().Metrics`
- `METRICS_PASSWORD` 環境變數覆寫
- `configs/config.yaml` 與 runtime `config/config.yaml` 中的 `metrics` 區塊

目前缺口：

- `router.NewRouter()` 沒有註冊 `cfg.Metrics.Path`。
- `go.mod` 沒有 `github.com/prometheus/client_golang`。
- 沒有 Prometheus registry / handler。
- 沒有 HTTP middleware 記錄 request metrics。
- 沒有 DB / Redis pool collector。
- `router.go` skip list 寫死 `/api/v1/metrics`，但 config 預設為 `/metrics`。
- runtime `config/config.yaml` 註解拼錯為 `METRICS_PASSWORND`，程式實際讀 `METRICS_PASSWORD`。

## 設計目標

- 讓 `metrics.enabled` 成為真實 runtime 開關。
- 讓 `metrics.path` 成為唯一 endpoint 路徑來源。
- 第一版完成可 scrape、可保護、可驗證的 Prometheus endpoint。
- 避免高基數 label。
- 避免 metrics scrape 與 health probe 汙染 HTTP request metrics 與 access log。
- 不改動 S3 設定與前端。

## 套件選型

使用官方 Prometheus Go client：

```text
github.com/prometheus/client_golang/prometheus
github.com/prometheus/client_golang/prometheus/promhttp
```

原因：

- 與初始設計稿套件堆疊一致。
- 支援 Go runtime / process collector。
- 可使用自訂 registry，避免全域 registry 汙染測試。
- `promhttp.HandlerFor()` 可直接掛 Gin route。

## 後端模組設計

新增模組：

```text
opscenter-server/internal/metrics
```

建議檔案：

```text
collector.go
delivery.go
middleware.go
middleware_test.go
delivery_test.go
```

### Registry

提供建構函式：

```go
type Dependencies struct {
	Config config.EndpointConfig
	AppEnv string
	DB     *db.DB
	Redis  *cache.Redis
}

func NewRegistry(deps Dependencies) (*prometheus.Registry, error)
```

Registry 內容：

- Go runtime collector
- process collector
- HTTP request counter
- HTTP request duration histogram
- DB pool collector
- Redis pool collector

使用自訂 registry，不使用 default registry，避免測試重複註冊與跨 package side effect。

### Delivery

提供 router 註冊：

```go
func Register(r *gin.Engine, deps Dependencies)
```

行為：

- `Config.Enabled = false` 時直接 return。
- 使用 `Config.Path` 註冊 route。
- handler 使用 `promhttp.HandlerFor(registry, promhttp.HandlerOpts{})`。
- Basic Auth middleware 只套在 metrics route。

此 route 掛在 root engine，不掛在 `/api/v1` group。預設 endpoint 是：

```text
GET /metrics
```

原因：

- 初始設計稿規劃為 `GET /metrics`。
- Prometheus 常見 scrape path 是 root-level `/metrics`。
- 不應與業務 API auth middleware 混在一起。

## Basic Auth 設計

新增 metrics 專用 middleware：

```go
func BasicAuth(cfg config.BasicAuthConfig) gin.HandlerFunc
```

規則：

- `cfg.Enabled = false` 時不套用。
- 使用 `c.Request.BasicAuth()` 解析帳密。
- 使用 `crypto/subtle.ConstantTimeCompare` 比對 username 與 password。
- 失敗回 `401`，並設定 `WWW-Authenticate` header。
- 不輸出帳號或密碼到 log。

### 密碼空值策略

生產環境：

- `metrics.enabled = true`
- `metrics.basic_auth.enabled = true`
- `metrics.basic_auth.password` trim 後為空

上述情境 `config.Load()` 應回錯，讓服務 fail fast。

非生產環境：

- 可允許空密碼以降低本機開發阻力。
- 測試需覆蓋生產環境 fail fast。

## Config 設計

沿用既有 struct：

```go
type EndpointConfig struct {
	Enabled   bool
	Path      string
	BasicAuth BasicAuthConfig
}
```

新增驗證：

- `metrics.path` 空值時回預設 `/metrics`。
- `metrics.path` 必須以 `/` 開頭。
- `metrics.path` 不得為 `/api/v1/healthz/live`。
- `metrics.path` 不得為 `/api/v1/healthz/ready`。
- `metrics.path` 不得為 `/swagger/*any` 或 `/swagger/doc.json`。
- 生產環境 Basic Auth 啟用時 password 不得為空。

修正 config 註解：

```yaml
password: "" # ENV: METRICS_PASSWORD
```

## HTTP Middleware 設計

新增 middleware：

```go
func HTTPMetrics(registry *Registry, ignoredPaths map[string]struct{}) gin.HandlerFunc
```

記錄時機：

- 放在 request id / recovery 後。
- request 完成後記錄 method、route、status、duration。

label：

| label | 來源 | 說明 |
| --- | --- | --- |
| `method` | `c.Request.Method` | GET / POST 等 |
| `route` | `c.FullPath()` | Gin route pattern |
| `status` | `strconv.Itoa(c.Writer.Status())` | HTTP status code |

route label 規則：

- 優先使用 `c.FullPath()`。
- 空值時使用固定值 `unmatched`。
- 不使用 `c.Request.URL.Path`，避免 `ticket_id`、`user_id` 等進入 label。

忽略路徑：

- metrics path 本身。
- health liveness / readiness。
- swagger 文件。
- 靜態資源與 SPA fallback 可先不統計，或以 route `static` 歸類；第一版建議只統計已匹配的 API route。

## Metric 命名

使用 `opscenter` namespace，避免與 Go runtime 或其他 middleware 名稱衝突。

### HTTP

```text
opscenter_http_requests_total{method, route, status}
opscenter_http_errors_total{method, route, status}
opscenter_http_requests_in_flight
opscenter_http_request_duration_seconds_bucket{method, route, status, le}
opscenter_http_request_duration_seconds_sum{method, route, status}
opscenter_http_request_duration_seconds_count{method, route, status}
opscenter_http_request_size_bytes_bucket{method, route, le}
opscenter_http_request_size_bytes_sum{method, route}
opscenter_http_request_size_bytes_count{method, route}
opscenter_http_response_size_bytes_bucket{method, route, status, le}
opscenter_http_response_size_bytes_sum{method, route, status}
opscenter_http_response_size_bytes_count{method, route, status}
```

`opscenter_http_errors_total` 只記錄 4xx 與 5xx，保留 `status` label 方便計算 client error 與 server error。`opscenter_http_requests_in_flight` 不加 route label，避免長時間 request 導致 label 集合失控。

### App

```text
opscenter_app_up
opscenter_build_info{version, commit, env}
```

`opscenter_app_up` 在 process 啟動後固定為 `1`。`opscenter_build_info` 用 gauge，值固定為 `1`，label 僅允許低基數部署資訊，不加入 hostname、pod name、instance id。

### DB

```text
opscenter_db_pool_open_connections
opscenter_db_pool_in_use_connections
opscenter_db_pool_idle_connections
opscenter_db_pool_wait_count_total
opscenter_db_pool_wait_duration_seconds
```

資料來源使用 `sql.DB.Stats()`。若目前 `db.DB` 尚未暴露 `Stats()`，需新增只讀方法，不讓 metrics 直接碰 private field。

### Redis

```text
opscenter_redis_pool_hits_total
opscenter_redis_pool_misses_total
opscenter_redis_pool_timeouts_total
opscenter_redis_pool_total_connections
opscenter_redis_pool_idle_connections
opscenter_redis_pool_stale_connections_total
```

資料來源使用 `redis.Client.PoolStats()`。

## 系統流程 Metrics 設計

本段屬於第一版追加範圍，用來補足系統內部非 HTTP 的核心流程觀測。這些 metrics 不直接暴露業務敏感資料，label 需維持低基數。

### Scheduler

```text
opscenter_scheduler_runs_total{task_key, status}
opscenter_scheduler_run_duration_seconds_bucket{task_key, status, le}
opscenter_scheduler_run_duration_seconds_sum{task_key, status}
opscenter_scheduler_run_duration_seconds_count{task_key, status}
opscenter_scheduler_last_success_timestamp_seconds{task_key}
opscenter_scheduler_lock_busy_total{task_key}
```

label 規則：

- `task_key` 使用 scheduler 既有 task key。
- `status` 僅允許 `success`、`failed`、`skipped`。
- 不加入 `locked_by`，避免 instance id 或 hostname 成為 label。

接線位置：

- 任務開始與結束由 scheduler service 記錄。
- lock busy 時遞增 `opscenter_scheduler_lock_busy_total`。
- 任務成功時更新 `opscenter_scheduler_last_success_timestamp_seconds`。

### Webhook

```text
opscenter_webhook_deliveries_total{event_type, result}
opscenter_webhook_delivery_duration_seconds_bucket{event_type, result, le}
opscenter_webhook_delivery_duration_seconds_sum{event_type, result}
opscenter_webhook_delivery_duration_seconds_count{event_type, result}
opscenter_webhook_retry_total{event_type}
opscenter_webhook_dead_letter_current
```

label 規則：

- `event_type` 使用系統定義的低基數事件類型。
- `result` 僅允許 `success`、`failed`、`skipped`。
- 不加入 webhook id、URL、project id 或錯誤訊息。

### Notification

```text
opscenter_notifications_created_total{type}
opscenter_notifications_unread_current
```

`type` 使用站內通知定義的通知類型。`unread_current` 第一版可用全站總數，不依 user 分 label。

### Auth

```text
opscenter_auth_login_total{source, result}
opscenter_auth_mfa_challenges_total{result}
opscenter_auth_sso_login_total{provider, protocol, result}
opscenter_auth_token_refresh_total{result}
opscenter_security_audit_events_total{event_type, result}
```

label 規則：

- `source` 可用 `password`、`oidc`、`saml`。
- `result` 使用 `success`、`failed`、`mfa_required`、`mfa_setup_required` 等固定集合。
- `provider` 使用 SSO provider key；若 provider 數量由管理端開放大量新增，需評估是否只保留 `protocol`。
- 不加入 username、email、user_id、ip、token、secret。

## 業務域 Metrics 後續規劃

本段先保留規劃，不列入第一版必做實作。原因是 Ticket、排班與報表 metrics 涉及業務口徑，需要先確認 label 與聚合語意，避免日後 Grafana 解讀錯誤。

### Ticket

```text
opscenter_tickets_created_total{project, type, priority, shift}
opscenter_tickets_status_changes_total{project, from_status, to_status}
opscenter_tickets_open_current{project, priority, shift}
opscenter_tickets_resolution_duration_seconds_bucket{project, priority, le}
opscenter_tickets_resolution_duration_seconds_sum{project, priority}
opscenter_tickets_resolution_duration_seconds_count{project, priority}
opscenter_tickets_attachment_upload_total{project, result}
```

label 邊界：

- `project` 使用 project key 或 project id 需二選一。第一版建議使用 project key，較方便 Grafana 篩選，但需確認 project key 不會高頻變動。
- 不得使用 ticket id、title、external ref。
- `shift` 僅允許現行值班班制設定中的有效班別。

### 排班

```text
opscenter_schedule_periods_current{status}
opscenter_schedule_generation_total{mode, result}
opscenter_schedule_generation_duration_seconds_bucket{mode, result, le}
opscenter_schedule_validation_failed_total{rule}
```

label 邊界：

- `status` 使用排班週期固定狀態集合。
- `mode` 使用 `4week`、`8week` 等低基數模式。
- `rule` 使用程式定義的驗證規則代碼，不使用人員姓名或 user id。

### 報表

```text
opscenter_report_preview_total{report_type, result}
opscenter_report_query_duration_seconds_bucket{report_type, result, le}
opscenter_report_export_total{format, result}
```

label 邊界：

- `report_type` 使用報表模式或範本類型，不使用範本名稱。
- `format` 使用 `csv`、`excel` 等固定集合。

### 儲存與圖片轉換

```text
opscenter_storage_upload_total{backend, result}
opscenter_storage_upload_bytes_bucket{backend, result, le}
opscenter_image_conversion_total{format, result}
opscenter_image_conversion_duration_seconds_bucket{format, result, le}
opscenter_temp_files_current
```

label 邊界：

- `backend` 使用 `local`、`s3`。
- `format` 使用 `webp`、`avif`。
- 不加入 bucket name、object key、file name。

## Router 接線

`server.NewRouter()` 需調整：

1. 建立 metrics registry。
2. 依 `deps.Config.Metrics.Enabled` 註冊 `deps.Config.Metrics.Path`。
3. HTTP request metrics middleware 使用同一 registry。
4. log / gzip skip path 不再硬寫 `/api/v1/metrics`，改用 `deps.Config.Metrics.Path`、`deps.Config.Health.LivenessPath`、`deps.Config.Health.ReadinessPath`。

建議順序：

```go
r := gin.New()
r.Use(gin.Recovery())
r.Use(gin.LoggerWithConfig(gin.LoggerConfig{SkipPaths: observabilitySkipPaths(deps.Config)}))
r.Use(middleware.RequestID())
r.Use(metrics.HTTPMetrics(...))
r.Use(middleware.AccessLogWithSkips(observabilitySkipPaths(deps.Config)))
metrics.Register(r, ...)
```

`observabilitySkipPaths()` 需至少包含：

```text
metrics.path
health.liveness_path
health.readiness_path
/swagger/*any
/swagger/doc.json
```

若 `metrics.enabled = false`，仍可把 `metrics.path` 放入 skip list；該 route 不存在時不影響行為。

若要避免 scrape / probe access log 雜訊，`AccessLog` 需要支援 skip path。可新增：

```go
func AccessLogWithSkips(paths []string) gin.HandlerFunc
```

`AccessLogWithSkips` 規則：

- path 比對使用 request raw path。
- metrics path 與 health path 直接 return，不寫 `http request` log。
- 只跳過 access log，不移除 request id、recovery、handler 錯誤處理。
- 若 handler 內部需要記錄錯誤，仍由 handler 自行用結構化日誌記錄。
- health readiness 會高頻呼叫 DB / Redis 連線檢查；成功檢查必須靜默，不寫 `Database ping ok` 或 `Redis ping ok` 類成功 log。
- health readiness 依賴檢查失敗時可保留必要錯誤 log，避免完全失去故障線索。

第一版可直接讓 metrics route 在 access log 前註冊並返回，但 Gin global middleware 仍會套用；因此較乾淨做法是讓 `AccessLog` 支援 skip，並同步使用 Gin logger 的 `SkipPaths`。

## 與 Health / OTel 的關係

Metrics：

- 提供 Prometheus scrape。
- 使用 pull model。
- 不需要登入 token。
- 可用 Basic Auth 保護。

Health：

- 只回服務是否可用。
- 不輸出 runtime 細節。
- 不與 metrics 合併。

OTel：

- 後續用於 trace / span。
- 本次不實作。
- `otel` config 保持不變。

## 安全性

- metrics endpoint 不回傳 secrets。
- Basic Auth 密碼只可從設定或 ENV 讀取，不寫入 log。
- label 禁止使用高基數與個資。
- DB / Redis collector 不輸出連線字串、host、username、database name。
- Prometheus text 可能暴露 runtime 狀態，生產環境需啟用 Basic Auth 或透過內網保護。

## 測試規劃

### Config test

- `METRICS_PASSWORD` 可覆寫密碼。
- production + metrics basic auth password 空值會失敗。
- non-production 可允許空密碼。
- `metrics.path` 非 `/` 開頭會失敗。
- `metrics.path` 空值回預設 `/metrics`。

### Delivery test

- enabled 時 `GET /metrics` 存在。
- disabled 時 `GET /metrics` 不存在。
- Basic Auth 未帶回 `401`。
- Basic Auth 錯誤回 `401`。
- Basic Auth 正確回 `200`。
- response 包含 Prometheus text content type。

### Middleware test

- API request 後輸出 `opscenter_http_requests_total`。
- 4xx / 5xx request 後輸出 `opscenter_http_errors_total`。
- 並行 request 時 `opscenter_http_requests_in_flight` 會增加並在完成後下降。
- request / response size histogram 有輸出。
- route label 使用 Gin route pattern。
- 不包含實際 ULID / ticket id。
- metrics path 不被統計。

### Collector test

- DB nil 不 panic。
- Redis nil 不 panic。
- registry 可重複建立，不產生 duplicate registration panic。
- build info 不包含 hostname、pod name、instance id。
- app up 固定輸出 `1`。

### 系統流程 metrics test

- scheduler 任務成功、失敗、skipped 會遞增 `opscenter_scheduler_runs_total`。
- scheduler lock busy 會遞增 `opscenter_scheduler_lock_busy_total`。
- scheduler 成功會更新 `opscenter_scheduler_last_success_timestamp_seconds`。
- webhook 投遞成功、失敗與 retry 會更新對應 metrics。
- notification 建立與 unread current 會更新對應 metrics。
- auth login / MFA / SSO / refresh token 流程會更新對應 metrics。
- 測試輸出不得包含 username、email、user_id、ip、token、secret。

## 實作順序建議

1. 補 config 驗證與 config 註解修正。
2. 新增 `internal/metrics` registry、Basic Auth、delivery。
3. 新增 HTTP request metrics middleware。
4. 補 app SLI metrics：in-flight、error total、request / response size、build info、app up。
5. 接入 `server.NewRouter()`，移除硬寫 `/api/v1/metrics`。
6. 補 DB / Redis pool collector。
7. 補 scheduler metrics。
8. 補 webhook / notification metrics。
9. 補 auth metrics。
10. 補測試並執行 `go test ./...`。
11. 另開後續 task 實作 Ticket / 排班 / 報表 / 儲存 metrics。

## 風險與控制

- `metrics.enabled = true` 且 production 密碼空白會造成啟動失敗；這是安全性設計，需要部署前設定 `METRICS_PASSWORD`。
- 若 route label 使用 raw path，Prometheus time series 會暴增；測試需防止此回歸。
- 若使用全域 Prometheus registry，測試容易 duplicate registration；第一版使用自訂 registry。
- 若把 metrics 掛進 `/api/v1` 並套 auth middleware，Prometheus scrape 會被 access token 阻擋；第一版使用 root-level route。
