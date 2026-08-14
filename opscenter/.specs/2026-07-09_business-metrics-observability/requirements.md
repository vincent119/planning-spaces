# Business Metrics Observability 需求

## 文件定位

本 spec 接續 `../2026-07-07_10-07_metrics-observability` 的第 9 組後續規劃，專門定義 Ticket、排班、報表、儲存與圖片轉換的業務 metrics。

原始設計稿 `../2026-06-01_10-22_oncall-ticket-system` 不在本次修改範圍。

## 背景

目前 metrics 基礎設施已涵蓋：

- HTTP request metrics。
- Runtime / DB / Redis metrics。
- Scheduler、Webhook、Notification、Auth、MFA、SSO、Security Audit metrics。

但業務域 metrics 尚未完成正式口徑。若直接實作，容易產生下列問題：

- label 使用 ticket id、user id、title 等高基數或敏感資料。
- Ticket、排班、報表各自定義不同 result / status 口徑，Grafana 難以聚合。
- 指標名稱未先固定，後續更名會破壞既有 dashboard 與 alert。
- 事件觸發點不清楚，導致重複計數或漏計。

因此本階段先完成需求與設計，不直接改程式。

## 範圍

### 包含

- Ticket domain metrics 規劃。
- Schedule domain metrics 規劃。
- Report domain metrics 規劃。
- Storage / Image conversion metrics 規劃。
- 指標命名、metric type、label 白名單。
- 禁用 label 與敏感資訊邊界。
- 後端接線位置與測試驗收規劃。

### 不包含

- 不實作 Go 程式碼。
- 不新增 migration。
- 不新增 Grafana dashboard。
- 不新增 Prometheus alert rule。
- 不修改初始設計稿。
- 不新增前端 UI。

## 需求 1：共用 Metrics 規範

業務 metrics 必須沿用 `opscenter` namespace，並維持低基數 label。

### 驗收條件

- [ ] 1.1 所有業務 metrics 名稱需以 `opscenter_` 開頭。
- [ ] 1.2 Counter 名稱需以 `_total` 結尾。
- [ ] 1.3 Histogram 需包含單位後綴，例如 `_seconds`、`_bytes`。
- [ ] 1.4 Gauge 僅用於目前狀態或目前數量。
- [ ] 1.5 所有 label 需有白名單。
- [ ] 1.6 禁止 label 包含 `ticket_id`、`user_id`、`username`、`email`、`title`、`description`、`external_ref`、`trace_id`、`request_id`、URL、檔名、object key、錯誤文字。

## 需求 2：Ticket Metrics

系統需要規劃 Ticket 建立、狀態流轉、目前未結案、處理時間與附件上傳 metrics。

### 驗收條件

- [ ] 2.1 需規劃 Ticket 建立數 counter。
- [ ] 2.2 需規劃 Ticket 狀態轉換 counter。
- [ ] 2.3 需規劃目前未結案 Ticket gauge。
- [ ] 2.4 需規劃 Ticket resolution duration histogram。
- [ ] 2.5 需規劃 Ticket 附件上傳 counter 與 bytes histogram。
- [ ] 2.6 Ticket label 可使用 `project`、`ticket_type`、`priority`、`status`、`from_status`、`to_status`、`shift`、`result`。
- [ ] 2.7 Ticket metrics 不得使用 ticket id、title、description、external ref、建立者、指派人員、處理人員作為 label。

## 需求 3：Schedule Metrics

系統需要規劃排班週期、展開、確認、驗證失敗與排班矩陣更新 metrics。

### 驗收條件

- [ ] 3.1 需規劃排班週期目前數量 gauge。
- [ ] 3.2 需規劃排班週期建立 / 展開 counter。
- [ ] 3.3 需規劃排班展開耗時 histogram。
- [ ] 3.4 需規劃排班確認 counter。
- [ ] 3.5 需規劃排班驗證失敗 counter。
- [ ] 3.6 需規劃排班格子更新 counter。
- [ ] 3.7 Schedule label 可使用 `mode`、`status`、`result`、`rule`、`shift`、`leave_type`、`source`。
- [ ] 3.8 Schedule metrics 不得使用 period id、schedule id、user id、人名、週期名稱、錯誤文字作為 label。

## 需求 4：Report Metrics

系統需要規劃報表 preview、query、export 與範本操作 metrics。

### 驗收條件

- [ ] 4.1 需規劃 report preview counter 與 duration histogram。
- [ ] 4.2 需規劃 report query duration histogram。
- [ ] 4.3 需規劃 report export counter 與 duration histogram。
- [ ] 4.4 需規劃 report template save / delete counter。
- [ ] 4.5 Report label 可使用 `dataset`、`report_mode`、`group_by`、`series`、`format`、`result`。
- [ ] 4.6 Report metrics 不得使用 template id、template name、查詢條件原文、使用者、日期值作為 label。

## 需求 5：Storage / Image Metrics

系統需要規劃附件上傳、圖片轉換、儲存失敗與暫存檔清理 metrics。

### 驗收條件

- [ ] 5.1 需規劃 storage upload counter。
- [ ] 5.2 需規劃 storage upload bytes histogram。
- [ ] 5.3 需規劃 image conversion counter。
- [ ] 5.4 需規劃 image conversion duration histogram。
- [ ] 5.5 需規劃 temp file cleanup counter。
- [ ] 5.6 需規劃 temp files current gauge。
- [ ] 5.7 Storage label 可使用 `backend`、`content_type`、`format`、`result`、`reason`。
- [ ] 5.8 Storage metrics 不得使用 bucket name、object key、local path、filename、upload url、原始錯誤文字作為 label。

## 需求 6：驗證與文件同步

後續實作前，需要先確定 metrics 輸出不含高基數或敏感資訊。

### 驗收條件

- [ ] 6.1 每個 domain 需有 metrics recorder 測試。
- [ ] 6.2 每個 domain 需有 prometheus gather 測試，確認 metric name 與 label。
- [ ] 6.3 需有敏感 label 掃描測試。
- [ ] 6.4 需有 `go test ./...` 驗收。
- [ ] 6.5 若實作內容超出本 spec，需先補文件再執行。

