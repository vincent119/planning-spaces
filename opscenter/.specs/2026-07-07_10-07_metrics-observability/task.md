# Metrics Observability Task

## 1. Config 與設定驗證

- [x] 1.1 修正 metrics 設定註解與環境變數名稱
  - 將範例設定中的 `METRICS_PASSWORND` 修正為 `METRICS_PASSWORD`。
  - 確認設定讀取仍以設定檔加環境變數覆寫為主，不把密碼寫入 `system_settings`。
  - _Requirements: 2.6, 5.5_

- [x] 1.2 實作 `metrics.enabled` 與 `metrics.path` 驗證
  - `metrics.path` 空值時預設為 `/metrics`。
  - `metrics.path` 必須以 `/` 開頭。
  - `metrics.path` 不可與 health、swagger、API root 路徑衝突。
  - _Requirements: 1.1, 1.2, 5.1, 5.2, 5.4_

- [x] 1.3 實作 metrics Basic Auth 設定驗證
  - `metrics.basic_auth.enabled=true` 時必須有 username。
  - production 環境啟用 Basic Auth 時，password 不可為空。
  - _Requirements: 2.1, 2.5, 5.3_

- [x] 1.4 補齊 metrics config 單元測試
  - 驗證 enabled、path normalize、路徑衝突、Basic Auth 空密碼與環境變數覆寫。
  - _Requirements: 6.4_

## 2. Metrics 模組與 Prometheus Registry

- [x] 2.1 建立 `internal/metrics` 模組骨架
  - 建立 registry、collector、delivery、middleware 的內部分層。
  - 使用自有 registry，避免直接污染 Prometheus default registry。
  - _Requirements: 1.4, 6.1_

- [x] 2.2 接入 Prometheus client 套件
  - 使用 `github.com/prometheus/client_golang/prometheus` 與 `promhttp`。
  - 確認依賴版本與 `go.mod`、`go.sum` 更新。
  - _Requirements: 1.4_

- [x] 2.3 註冊 Go runtime 與 process collectors
  - 輸出 Go runtime、process、GC、goroutine、memory 等標準指標。
  - _Requirements: 4.1, 4.2_

- [x] 2.4 註冊 App SLI 基礎指標
  - `opscenter_app_up`
  - `opscenter_build_info{version,commit,env}`
  - _Requirements: 8.4, 8.5, 8.6_

- [x] 2.5 補齊 registry 測試
  - 驗證重複註冊不 panic。
  - 驗證 disabled metrics 不建立不必要 handler。
  - _Requirements: 6.1, 6.4_

## 3. Metrics Endpoint 與 Basic Auth

- [x] 3.1 建立 root-level metrics endpoint
  - metrics 路由掛在設定的 `metrics.path`。
  - 不掛在 `/api/v1` 底下。
  - _Requirements: 1.1, 1.3_

- [x] 3.2 實作 metrics disabled 行為
  - `metrics.enabled=false` 時不註冊 endpoint。
  - 存取時應回傳明確的 404。
  - _Requirements: 1.2, 6.2_

- [x] 3.3 實作 Prometheus text exposition response
  - response 格式符合 Prometheus scrape 規範。
  - content type 由 `promhttp` 管理。
  - _Requirements: 1.4, 1.5_

- [x] 3.4 實作 Basic Auth 保護
  - 啟用時未帶或錯誤認證回 401。
  - 認證比對需使用 constant-time compare。
  - _Requirements: 2.1, 2.2, 2.3, 2.4_

- [x] 3.5 補齊 endpoint 測試
  - 覆蓋 enabled、disabled、Basic Auth 成功、Basic Auth 失敗、Prometheus response。
  - _Requirements: 6.1, 6.2, 6.3_

## 4. HTTP Middleware 與 Access Log 排除

- [x] 4.1 實作 HTTP request metrics middleware
  - 記錄 request total、duration、request size、response size。
  - label 使用 `method`、`route`、`status`。
  - _Requirements: 3.1, 3.2, 3.3, 3.4_

- [x] 4.2 實作 in-flight 與 HTTP error 指標
  - `opscenter_http_requests_in_flight`
  - `opscenter_http_errors_total`
  - 4xx、5xx 都納入 error counter，但 status label 需可區分。
  - _Requirements: 8.1, 8.2_

- [x] 4.3 收斂 route label cardinality
  - 優先使用 Gin `c.FullPath()`。
  - 找不到 route 時使用固定值 `unmatched`。
  - 不把原始 URL、query string、path id 放進 label。
  - _Requirements: 3.5, 8.3_

- [x] 4.4 排除 metrics、health、swagger 類 endpoint 的 metrics 與 access log
  - metrics endpoint 不產生 HTTP metrics 自我遞迴。
  - metrics 與 health endpoint 不寫入 access log。
  - swagger 類靜態文件不進入 access log。
  - _Requirements: 3.6, 7.1, 7.2, 7.3, 7.4, 7.5_

- [x] 4.5 重構 access log skip path 設定
  - 將 skip path 統一由 router 依 config 組合。
  - 支援動態 metrics path。
  - 保留一般 API request access log。
  - _Requirements: 7.6, 7.7, 7.8_

- [x] 4.6 補齊 HTTP middleware 與 access log 測試
  - 驗證 route label 不爆量。
  - 驗證 metrics、health 不寫 access log。
  - 驗證一般 API 仍寫 access log。
  - _Requirements: 6.5, 6.6, 7.1, 7.2, 7.7_

- [x] 4.7 修正 health readiness 依賴成功檢查 log
  - DB / Redis readiness 成功檢查不得寫入成功 log。
  - 依賴檢查失敗時仍保留必要錯誤 log。
  - _Requirements: 7.8, 7.9_

## 5. Infrastructure Collectors

- [x] 5.1 暴露 DB pool stats 指標
  - active、idle、max open、wait count、wait duration。
  - _Requirements: 4.3_

- [x] 5.2 暴露 Redis pool stats 指標
  - hits、misses、timeouts、total conns、idle conns、stale conns。
  - Redis 未啟用或 client 為 nil 時不可 panic。
  - _Requirements: 4.4, 4.5_

- [x] 5.3 補齊 infrastructure collector 測試
  - 驗證 DB stats、Redis stats、nil Redis client。
  - _Requirements: 4.6, 6.4_

## 6. Scheduler Metrics

- [x] 6.1 補 scheduler run counter
  - 依 task key 與 status 記錄執行次數。
  - label 不包含 instance id、locked_by、錯誤文字。
  - _Requirements: 9.1, 9.9_

- [x] 6.2 補 scheduler duration histogram
  - 依 task key 與 status 記錄執行耗時。
  - _Requirements: 9.2_

- [x] 6.3 補 scheduler last success timestamp
  - 每個 task key 成功時更新最後成功時間。
  - _Requirements: 9.3_

- [x] 6.4 補 scheduler lock busy counter
  - 多 Pod 分散式鎖取得失敗或 busy 時記錄。
  - _Requirements: 9.4_

- [x] 6.5 補 scheduler metrics 測試
  - 驗證 success、failed、skipped、lock busy 的計數與 label。
  - _Requirements: 6.4, 9.1, 9.4_

## 7. Webhook 與 Notification Metrics

- [x] 7.1 補 webhook delivery 指標
  - 記錄投遞次數、狀態、耗時。
  - label 只允許 webhook id 或穩定代碼，不包含 URL、payload、token。
  - _Requirements: 9.5, 9.9_

- [x] 7.2 補 webhook retry 與 dead letter 指標
  - 記錄 retry 次數與 dead letter 數量。
  - _Requirements: 9.6_

- [x] 7.3 補 notification 指標
  - 記錄站內通知建立數。
  - 暴露 unread notification gauge。
  - _Requirements: 9.7_

- [x] 7.4 補 webhook 與 notification metrics 測試
  - 驗證成功、失敗、retry、dead letter、unread gauge。
  - 驗證指標不輸出敏感資訊。
  - _Requirements: 6.4, 9.5, 9.6, 9.7, 9.9_

## 8. Auth 與 Security Metrics

- [x] 8.1 補登入與 token 指標
  - 記錄 login success、login failure、token refresh success、token refresh failure。
  - label 僅允許結果、原因類別，不包含 username、user id、token。
  - _Requirements: 9.8, 9.9_

- [x] 8.2 補 MFA 指標
  - 記錄 MFA setup、verify、challenge 的 success/failure。
  - _Requirements: 9.8_

- [x] 8.3 補 SSO 指標
  - 記錄 SSO login success/failure。
  - label 可包含 provider key，但不可包含外部 identity、email、claim 原文。
  - _Requirements: 9.8, 9.9_

- [x] 8.4 補 security audit 指標
  - 針對安全事件寫入 audit 時同步記錄 counter。
  - label 需控制 cardinality。
  - _Requirements: 9.8, 9.9_

- [x] 8.5 補 auth metrics 測試
  - 驗證登入、MFA、SSO、token refresh、安全事件指標。
  - 驗證不輸出敏感欄位。
  - _Requirements: 6.4, 9.8, 9.9_

## 9. Business Metrics 後續規劃

- [x] 9.1 拆出 Ticket domain metrics 後續 task
  - 建立 ticket created、status transition、open ticket、resolution duration、attachment count 的規劃任務。
  - 本階段不直接實作 Ticket business metrics。
  - 已拆至 `../2026-07-09_business-metrics-observability`。
  - _Requirements: 10.1, 10.5, 10.6_

- [x] 9.2 拆出 schedule domain metrics 後續 task
  - 建立 period generated、schedule validation failure、confirm success/failure 的規劃任務。
  - 本階段不直接實作 schedule business metrics。
  - 已拆至 `../2026-07-09_business-metrics-observability`。
  - _Requirements: 10.2, 10.5, 10.6_

- [x] 9.3 拆出 report domain metrics 後續 task
  - 建立 report preview、query duration、export success/failure 的規劃任務。
  - 本階段不直接實作 report business metrics。
  - 已拆至 `../2026-07-09_business-metrics-observability`。
  - _Requirements: 10.3, 10.5, 10.6_

- [x] 9.4 拆出 storage domain metrics 後續 task
  - 建立 image upload、conversion、storage failure、temp file cleanup 的規劃任務。
  - 本階段不直接實作 storage business metrics。
  - 已拆至 `../2026-07-09_business-metrics-observability`。
  - _Requirements: 10.4, 10.5, 10.6_

## 10. 整合驗證

- [x] 10.1 執行後端單元測試
  - 執行 `go test ./...`。
  - 修正 metrics 相關測試失敗。
  - _Requirements: 6.1, 6.2, 6.3, 6.4_

- [x] 10.2 驗證 metrics endpoint 行為
  - 驗證 disabled、enabled、Basic Auth、Prometheus text response。
  - 驗證 `/metrics` 不掛在 `/api/v1`。
  - _Requirements: 1.1, 1.2, 1.3, 2.1, 2.2_

- [x] 10.3 驗證 access log 行為
  - 驗證 metrics endpoint 不寫 access log。
  - 驗證 health endpoint 不寫 access log。
  - 驗證一般 API request 仍寫 access log。
  - _Requirements: 7.1, 7.2, 7.7_

- [x] 10.4 驗證 label 與敏感資訊邊界
  - 檢查 metrics output 不包含 token、password、secret、URL payload、username、user id、query string。
  - 檢查高 cardinality label 沒有被引入。
  - _Requirements: 3.5, 8.3, 9.9_

- [x] 10.5 更新驗收紀錄
  - 在完成實作後補上測試指令與主要驗收結果。
  - _Requirements: 6.6_

### 驗收紀錄

- 2026-07-09：完成 scheduler、webhook、notification、auth、MFA、SSO、security audit metrics 接線。
- 2026-07-09：完成 unread notification gauge 與相關 collector 測試。
- 2026-07-09：完成低基數 label 檢查，metrics label 不包含 username、user id、token、secret、URL payload、query string。
- 2026-07-09：執行 `go test ./internal/metrics ./internal/scheduler ./internal/notification ./internal/auth ./internal/server`，結果通過。
- 2026-07-09：執行 `go test ./...`，結果通過。
