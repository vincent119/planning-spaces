# Business Metrics Observability Task

## 1. 共用規範與基礎設計

- [x] 1.1 確認 business metrics label 白名單
  - 整理共用 label、result、reason、status enum。
  - 確認禁止 label 清單。
  - _Requirements: 1.1, 1.5, 1.6_

- [x] 1.2 設計 metrics recorder 邊界
  - Ticket、Schedule、Report、Storage 各自透過 interface 接線。
  - domain service 不直接依賴 Prometheus 套件。
  - _Requirements: 1.1, 1.4_

- [x] 1.3 補 label normalize helper 規劃
  - 空值轉 `unknown`。
  - result、reason、status 超出白名單時轉 `unknown` 或固定分類。
  - _Requirements: 1.5, 1.6_

- [x] 1.4 補敏感資訊掃描測試規劃
  - 檢查 metrics output 不含 ticket id、user id、token、secret、URL、filename。
  - _Requirements: 6.2, 6.3_

### 1. 驗收紀錄

- 2026-07-09：新增 business metrics 共用 label 白名單、禁止 label 清單、result / reason / status enum 正規化 helper。
- 2026-07-09：新增敏感 metrics output token 偵測 helper，供後續 domain metrics 測試共用。
- 2026-07-09：現有 metrics recorder label normalize 改用共用 helper。
- 2026-07-09：執行 `go test ./internal/metrics`，結果通過。
- 2026-07-09：執行 `go test ./...`，結果通過。

## 2. Ticket Metrics 規劃

- [x] 2.1 設計 Ticket create counter
  - `opscenter_tickets_created_total{project, ticket_type, priority, shift, source}`。
  - _Requirements: 2.1, 2.6, 2.7_

- [x] 2.2 設計 Ticket status transition counter
  - `opscenter_tickets_status_changes_total{project, from_status, to_status}`。
  - _Requirements: 2.2, 2.6, 2.7_

- [x] 2.3 設計 Ticket open current collector
  - `opscenter_tickets_open_current{project, priority, shift}`。
  - 使用查詢型 gauge，避免手動增減不一致。
  - _Requirements: 2.3, 2.6, 2.7_

- [x] 2.4 設計 Ticket resolution duration histogram
  - `opscenter_tickets_resolution_duration_seconds{project, priority, shift}`。
  - _Requirements: 2.4, 2.6, 2.7_

- [x] 2.5 設計 Ticket attachment metrics
  - 上傳次數與 bytes histogram。
  - _Requirements: 2.5, 2.6, 2.7_

- [x] 2.6 補 Ticket metrics 測試規劃
  - recorder、collector、敏感 label 測試。
  - _Requirements: 6.1, 6.2, 6.3_

### 2. 驗收紀錄

- 2026-07-09：新增 Ticket metrics recorder，包含 created、status changes、resolution duration、attachment upload。
- 2026-07-09：新增 Ticket open current provider collector，採查詢型 gauge，label 僅使用 project、priority、shift。
- 2026-07-09：補 Ticket status 白名單，涵蓋 open、in_progress、pending、escalated、resolved、closed、cancelled。
- 2026-07-09：新增 Ticket metrics gather 測試、label normalize 測試與敏感 label 掃描測試。
- 2026-07-09：執行 `go test ./internal/metrics`，結果通過。
- 2026-07-09：執行 `go test ./...`，結果通過。

## 3. Schedule Metrics 規劃

- [x] 3.1 設計 schedule periods current collector
  - `opscenter_schedule_periods_current{mode, status}`。
  - _Requirements: 3.1, 3.7, 3.8_

- [x] 3.2 設計 schedule period created counter
  - `opscenter_schedule_periods_created_total{mode, source, result, reason}`。
  - _Requirements: 3.2, 3.7, 3.8_

- [x] 3.3 設計 schedule generation counter 與 duration
  - `opscenter_schedule_generation_total{mode, result, reason}`。
  - `opscenter_schedule_generation_duration_seconds{mode, result}`。
  - _Requirements: 3.2, 3.3, 3.7, 3.8_

- [x] 3.4 設計 schedule confirm counter
  - `opscenter_schedule_confirm_total{mode, result, reason}`。
  - _Requirements: 3.4, 3.7, 3.8_

- [x] 3.5 設計 schedule validation failed counter
  - `opscenter_schedule_validation_failed_total{mode, rule}`。
  - rule 使用固定規則代碼，不使用錯誤文字。
  - _Requirements: 3.5, 3.7, 3.8_

- [x] 3.6 設計 schedule cell update counter
  - `opscenter_schedule_cell_updates_total{shift, leave_type, source, result, reason}`。
  - _Requirements: 3.6, 3.7, 3.8_

- [x] 3.7 補 Schedule metrics 測試規劃
  - generation、confirm、validation failure、cell update、敏感 label 測試。
  - _Requirements: 6.1, 6.2, 6.3_

### 3. 驗收紀錄

- 2026-07-09：新增 Schedule metrics recorder，包含 period created、generation、confirm、validation failed、cell update。
- 2026-07-09：新增 Schedule periods current provider collector，採查詢型 gauge，label 僅使用 mode、status。
- 2026-07-09：補 Schedule status 白名單，涵蓋 draft、confirmed、locked。
- 2026-07-09：新增 Schedule metrics gather 測試、label normalize 測試與敏感 label 掃描測試。
- 2026-07-09：執行 `go test ./internal/metrics`，結果通過。
- 2026-07-09：執行 `go test ./...`，結果通過。

## 4. Report Metrics 規劃

- [x] 4.1 設計 report preview counter 與 duration
  - preview 次數與耗時。
  - _Requirements: 4.1, 4.5, 4.6_

- [x] 4.2 設計 report query duration
  - service query 耗時，不輸出 SQL 或查詢條件。
  - _Requirements: 4.2, 4.5, 4.6_

- [x] 4.3 設計 report export counter 與 duration
  - format 只允許 `csv`、`excel`。
  - _Requirements: 4.3, 4.5, 4.6_

- [x] 4.4 設計 report template operation counter
  - template save / delete 次數。
  - 不使用 template id 或 template name。
  - _Requirements: 4.4, 4.5, 4.6_

- [x] 4.5 補 Report metrics 測試規劃
  - preview、query、export、template、敏感 label 測試。
  - _Requirements: 6.1, 6.2, 6.3_

### 4. 驗收紀錄

- 2026-07-09：新增 Report metrics recorder，包含 preview、query、export、template save/delete。
- 2026-07-09：Report metrics label 僅使用 dataset、report_mode、group_by、series、format、result、reason。
- 2026-07-09：新增 Report metrics gather 測試、label normalize 測試與敏感 label 掃描測試。
- 2026-07-09：執行 `go test ./internal/metrics`，結果通過。
- 2026-07-09：執行 `go test ./...`，結果通過。

## 5. Storage / Image Metrics 規劃

- [x] 5.1 設計 storage upload counter
  - `opscenter_storage_upload_total{backend, content_type, result, reason}`。
  - _Requirements: 5.1, 5.7, 5.8_

- [x] 5.2 設計 storage upload bytes histogram
  - `opscenter_storage_upload_bytes{backend, content_type, result}`。
  - _Requirements: 5.2, 5.7, 5.8_

- [x] 5.3 設計 image conversion counter 與 duration
  - conversion 次數與耗時。
  - format 必須 normalize。
  - _Requirements: 5.3, 5.4, 5.7, 5.8_

- [x] 5.4 設計 temp file cleanup metrics
  - cleanup counter 與 temp files current gauge。
  - _Requirements: 5.5, 5.6, 5.7, 5.8_

- [x] 5.5 補 Storage metrics 測試規劃
  - upload、conversion、cleanup、敏感 label 測試。
  - _Requirements: 6.1, 6.2, 6.3_

### 5. 驗收紀錄

- 2026-07-09：新增 Storage metrics recorder，包含 upload counter、upload bytes histogram。
- 2026-07-09：新增 Image metrics recorder，包含 conversion counter 與 duration histogram。
- 2026-07-09：新增 temp file cleanup counter 與 temp files current provider collector。
- 2026-07-09：Storage / Image metrics label 僅使用 backend、content_type、format、result、reason。
- 2026-07-09：補 local_path、path、upload_url 為禁止 business metric label。
- 2026-07-09：新增 Storage / Image metrics gather 測試、label normalize 測試與敏感 label 掃描測試。
- 2026-07-09：執行 `go test ./internal/metrics`，結果通過。
- 2026-07-09：執行 `go test ./...`，結果通過。

## 6. 整合驗收規劃

- [x] 6.1 補 domain metrics gather 測試規劃
  - 確認每個新增指標可被 Prometheus gather。
  - _Requirements: 6.1, 6.2_

- [x] 6.2 補敏感資訊驗收規劃
  - 確認 metrics output 不含禁止 label 與敏感值。
  - _Requirements: 1.6, 6.3_

- [x] 6.3 補 `go test ./...` 驗收規劃
  - 後續實作完成後需全量測試通過。
  - _Requirements: 6.4_

- [x] 6.4 文件同步檢查
  - 後續實作若新增指標或 label，需先更新本 spec。
  - _Requirements: 6.5_

### 6. 驗收紀錄

- 2026-07-09：新增 Business domain metrics 整合 gather 測試，涵蓋 Ticket、Schedule、Report、Storage / Image 指標。
- 2026-07-09：新增 Business domain metrics 整體敏感 token 掃描測試。
- 2026-07-09：確認本 spec 已先涵蓋本次新增指標與 label，未新增超出文件範圍的指標。
- 2026-07-09：執行 `go test ./internal/metrics`，結果通過。
- 2026-07-09：執行 `go test ./...`，結果通過。
