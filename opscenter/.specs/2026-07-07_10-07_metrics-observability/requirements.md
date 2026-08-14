# Metrics Observability 需求

## 文件定位

本 spec 接續初始設計稿 `../2026-06-01_10-22_oncall-ticket-system` 已規劃的 `GET /metrics` 與 Prometheus Metrics，補齊目前設定檔已存在但 runtime 尚未實作的缺口。

原始設計稿為初始設計基準，本文件不修改原始 spec。

## 背景

目前後端設定檔已有下列設定：

```yaml
metrics:
  enabled: true
  path: "/metrics"
  basic_auth:
    enabled: true
    username: "metrics"
    password: "" # ENV: METRICS_PASSWORD
```

現況只完成設定模型與環境變數覆寫：

- `config.Config.Metrics`
- `EndpointConfig`
- `BasicAuthConfig`
- `METRICS_PASSWORD`

但尚未完成：

- `/metrics` route 註冊。
- Prometheus handler。
- HTTP request metrics middleware。
- DB / Redis runtime metrics。
- Basic Auth 套用。
- 設定檔註解拼字一致性。

因此目前不能視為 metrics 功能已完成。

## 範圍

### 包含

- 依 `metrics.enabled` 控制是否啟用 metrics endpoint。
- 依 `metrics.path` 註冊 scrape endpoint，第一版預設 `/metrics`。
- `metrics.basic_auth.enabled = true` 時，metrics endpoint 需使用 Basic Auth。
- Prometheus 格式輸出。
- HTTP request total 與 latency metrics。
- Go runtime、process、DB pool、Redis pool 基礎 metrics。
- 應用程式 SLI metrics，例如 in-flight request、HTTP error、build info。
- Scheduler、Notification、Webhook、Auth 的系統流程 metrics。
- Ticket、排班、報表、儲存與圖片轉換的後續業務 metrics 規劃。
- 設定驗證與測試。
- Metrics 與 Health endpoint 不寫 access log。

### 不包含

- 不處理 S3 設定檔內容。
- 不修改初始設計稿。
- 不新增 Grafana Dashboard。
- 不新增 Prometheus scrape config。
- 不實作 OpenTelemetry tracing。
- 不在第一版實作完整 Ticket domain metrics，例如 SLA、Ticket lifecycle counter；該部分先完成規劃，後續另開實作 task。
- 不將 metrics 管理放進前端 UI。

## 需求 1：Metrics Endpoint

系統需要依設定暴露 Prometheus metrics endpoint。

### 驗收條件

- [ ] 1.1 `metrics.enabled = true` 時，後端需註冊 `metrics.path` 指定的 endpoint。
- [ ] 1.2 `metrics.enabled = false` 時，不得註冊 metrics endpoint。
- [ ] 1.3 預設路徑為 `/metrics`，不得硬寫成 `/api/v1/metrics`。
- [ ] 1.4 metrics response content type 需符合 Prometheus text exposition format。
- [ ] 1.5 metrics endpoint 不需套用一般 access token 驗證。
- [ ] 1.6 metrics endpoint 不得被 SPA fallback 接走。

## 需求 2：Basic Auth

Metrics endpoint 需要支援 Basic Auth，以避免公開暴露 runtime 資訊。

### 驗收條件

- [ ] 2.1 `metrics.basic_auth.enabled = true` 時，未帶 Basic Auth 需回 `401`。
- [ ] 2.2 帳號或密碼錯誤時需回 `401`。
- [ ] 2.3 帳號密碼正確時需回 `200`。
- [ ] 2.4 Basic Auth 比對需使用 constant-time compare。
- [ ] 2.5 生產環境中，若 `metrics.enabled = true` 且 `metrics.basic_auth.enabled = true` 但密碼空白，啟動需失敗。
- [ ] 2.6 設定檔註解需統一為 `METRICS_PASSWORD`。

## 需求 3：HTTP Request Metrics

系統需要記錄 HTTP request 數量與延遲。

### 驗收條件

- [ ] 3.1 需記錄 request counter。
- [ ] 3.2 需記錄 request duration histogram。
- [ ] 3.3 label 需使用低基數欄位，例如 method、route、status。
- [ ] 3.4 route label 需使用 Gin route pattern，例如 `/api/v1/tickets/:id`，不得使用完整 URL path。
- [ ] 3.5 不得使用 `user_id`、`ticket_id`、`trace_id`、`email` 等高基數 label。
- [ ] 3.6 metrics endpoint 本身不得被 HTTP request metrics 重複統計，避免 scrape 造成雜訊。

## 需求 4：Runtime / Infrastructure Metrics

系統需要提供最小可用的 runtime 與基礎設施 metrics。

### 驗收條件

- [ ] 4.1 需啟用 Go runtime collector。
- [ ] 4.2 需啟用 process collector。
- [ ] 4.3 需提供 DB pool 狀態，例如 open、idle、in use connections。
- [ ] 4.4 需提供 Redis pool 狀態，例如 hits、misses、timeouts、total connections、idle connections、stale connections。
- [ ] 4.5 DB / Redis metrics 不得包含 host、username、database name 或 password。
- [ ] 4.6 若 DB 或 Redis dependency 為 nil，metrics endpoint 不得 panic。

## 需求 5：設定驗證與相容性

Metrics 設定需要與既有 config loader、router 與測試保持一致。

### 驗收條件

- [ ] 5.1 `config.Load()` 需驗證 `metrics.path` 必須以 `/` 開頭。
- [ ] 5.2 `metrics.path` 不得與已知 API endpoint、health endpoint 或 swagger endpoint 衝突。
- [ ] 5.3 `METRICS_PASSWORD` 需覆寫 `metrics.basic_auth.password`。
- [ ] 5.4 現有 `EndpointConfig` / `BasicAuthConfig` 可沿用，不新增重複 config struct。
- [ ] 5.5 若 metrics path 被自訂，router 的 log / gzip skip 設定需同步使用該 path。

## 需求 6：測試與驗證

Metrics 功能需要有可自動化驗證，避免只有設定檔但沒有 runtime 行為。

### 驗收條件

- [ ] 6.1 後端測試需覆蓋 enabled 時 endpoint 存在。
- [ ] 6.2 後端測試需覆蓋 disabled 時 endpoint 不存在。
- [ ] 6.3 後端測試需覆蓋 Basic Auth 成功與失敗。
- [ ] 6.4 後端測試需覆蓋 `METRICS_PASSWORD` 環境變數覆寫。
- [ ] 6.5 後端測試需覆蓋 HTTP request metrics 有輸出 method、route、status labels。
- [ ] 6.6 後端測試需確認高基數 label 不存在。
- [ ] 6.7 `go test ./...` 需通過。

## 需求 7：Access Log 排除

Metrics 與 Health endpoint 屬於監控系統高頻探測流量，不應寫入一般 access log，避免污染業務請求觀測與增加日誌量。

### 驗收條件

- [ ] 7.1 `metrics.path` 指定的 endpoint 不得寫入 Gin access log。
- [ ] 7.2 `metrics.path` 指定的 endpoint 不得寫入自訂 `middleware.AccessLog()`。
- [ ] 7.3 `health.liveness_path` 不得寫入 Gin access log。
- [ ] 7.4 `health.liveness_path` 不得寫入自訂 `middleware.AccessLog()`。
- [ ] 7.5 `health.readiness_path` 不得寫入 Gin access log。
- [ ] 7.6 `health.readiness_path` 不得寫入自訂 `middleware.AccessLog()`。
- [ ] 7.7 log skip path 需由 `metrics.path`、`health.liveness_path`、`health.readiness_path` 組成，不得硬寫 `/api/v1/metrics`。
- [ ] 7.8 排除 access log 不代表跳過 handler 本身的錯誤處理；health 與 metrics handler 若內部發生錯誤，仍可由 handler 自行記錄必要錯誤。
- [ ] 7.9 health readiness 成功檢查不得寫入 DB / Redis 成功 log；只有依賴檢查失敗時才保留必要錯誤 log。

## 需求 8：應用程式 SLI Metrics

第一版除 HTTP request total 與 latency 外，需補充最小可用的應用程式 SLI 指標，讓維運能快速判斷目前版本、錯誤率與即時負載。

### 驗收條件

- [ ] 8.1 需提供目前同時處理中的 request 數。
- [ ] 8.2 需提供 HTTP 4xx / 5xx error counter。
- [ ] 8.3 需提供 request body size 與 response body size histogram。
- [ ] 8.4 需提供 build info gauge，至少包含 version、commit、env。
- [ ] 8.5 需提供 app up gauge，服務啟動後固定為 `1`。
- [ ] 8.6 build info label 不得包含 hostname、pod name、instance id 等高基數值。

## 需求 9：系統流程 Metrics

系統需要針對已存在的核心流程補充 metrics，優先涵蓋 scheduler、notification、webhook 與 auth。

### 驗收條件

- [ ] 9.1 Scheduler 需記錄任務執行總數，label 至少包含 `task_key` 與 `status`。
- [ ] 9.2 Scheduler 需記錄任務執行時間。
- [ ] 9.3 Scheduler 需記錄最近一次成功時間。
- [ ] 9.4 Scheduler 需記錄分散式鎖 busy 次數。
- [ ] 9.5 Webhook 需記錄投遞總數，label 至少包含 `event_type` 與 `result`。
- [ ] 9.6 Webhook 需記錄投遞耗時與 retry 次數。
- [ ] 9.7 Notification 需記錄站內通知建立總數與未讀通知數。
- [ ] 9.8 Auth 需記錄登入、MFA、SSO、token refresh 結果統計。
- [ ] 9.9 Auth / Notification / Webhook metrics 不得包含 username、email、user_id、ip、token、secret。

## 需求 10：業務域 Metrics 後續規劃

Ticket、排班、報表與儲存 metrics 需先完成指標命名與 label 邊界規劃，避免後續實作時產生高基數或語意不一致問題。

### 驗收條件

- [ ] 10.1 Ticket metrics 規劃需包含建立數、狀態轉換數、open current、處理時間、附件上傳數。
- [ ] 10.2 Ticket metrics label 可使用 project、type、priority、shift，但不得使用 ticket_id。
- [ ] 10.3 排班 metrics 規劃需包含週期狀態、展開結果、展開耗時、驗證失敗規則。
- [ ] 10.4 報表 metrics 規劃需包含 preview / query 次數、結果與耗時。
- [ ] 10.5 儲存 metrics 規劃需包含 upload 次數、upload bytes、圖片轉換次數與耗時、暫存檔數量。
- [ ] 10.6 業務域 metrics 第一版可先不實作，但文件需保留後續 task 拆分依據。

## 驗收條件總結

- [ ] 設定 `metrics.enabled = true` 且 Basic Auth 正確時，`GET /metrics` 回 Prometheus text。
- [ ] 設定 `metrics.enabled = false` 時，`GET /metrics` 不存在。
- [ ] 未授權 scrape 被拒絕。
- [ ] 輸出包含 Go runtime / process metrics。
- [ ] 輸出包含 HTTP request counter / histogram。
- [ ] 輸出包含應用程式 SLI metrics。
- [ ] 輸出包含 Scheduler / Notification / Webhook / Auth metrics。
- [ ] 輸出包含 DB / Redis pool metrics。
- [ ] Ticket / 排班 / 報表 / 儲存 metrics 已完成後續規劃。
- [ ] 不輸出高基數 label。
- [ ] Metrics 與 Health endpoint 不寫 access log。
- [ ] 不修改或依賴 S3 設定。
