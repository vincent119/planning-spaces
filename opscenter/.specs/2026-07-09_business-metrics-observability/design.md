# Business Metrics Observability 設計

## 文件定位

本文件定義業務 metrics 的設計口徑。需求來源為本 spec 的 `requirements.md`。

本階段只做規劃設計，不實作程式碼。

## 設計原則

- 使用 `opscenter` namespace。
- 指標名稱一旦釋出，不任意更名。
- label 必須低基數、可預期、可白名單驗證。
- result / status / reason 使用固定 enum，不使用錯誤文字。
- 不把個資、業務內容、DB id、ULID、URL、檔名放入 label。
- counter 只增不減。
- gauge 只用於目前狀態或目前數量。
- duration 使用 seconds histogram。
- bytes 使用 bytes histogram。

## 共用 Label 規則

### 允許

| label | 用途 | 備註 |
| --- | --- | --- |
| `project` | 專案維度 | 使用 project key，不使用 project id |
| `result` | 操作結果 | 固定為 `success`、`failure`、`skipped` |
| `reason` | 失敗分類 | 固定 enum，不使用錯誤文字 |
| `status` | 業務狀態 | 固定 enum |
| `source` | 操作來源 | 固定 enum，例如 `api`、`scheduler`、`import` |

### 禁止

以下欄位不得出現在任何 business metrics label：

```text
ticket_id
period_id
schedule_id
template_id
user_id
username
email
title
description
external_ref
trace_id
request_id
ip
url
raw_path
query
file_name
object_key
bucket
error
```

### 失敗原因 enum

第一版統一使用：

```text
validation
permission
not_found
conflict
configuration
dependency
timeout
storage
unknown
```

handler / service 可把內部錯誤映射到上述分類。Prometheus label 不直接輸出錯誤內容。

## Ticket Metrics 設計

### 指標

```text
opscenter_tickets_created_total{project, ticket_type, priority, shift, source}
opscenter_tickets_status_changes_total{project, from_status, to_status}
opscenter_tickets_open_current{project, priority, shift}
opscenter_tickets_resolution_duration_seconds_bucket{project, priority, shift, le}
opscenter_tickets_resolution_duration_seconds_sum{project, priority, shift}
opscenter_tickets_resolution_duration_seconds_count{project, priority, shift}
opscenter_tickets_attachment_upload_total{project, content_type, result, reason}
opscenter_tickets_attachment_upload_bytes_bucket{project, content_type, result, le}
opscenter_tickets_attachment_upload_bytes_sum{project, content_type, result}
opscenter_tickets_attachment_upload_bytes_count{project, content_type, result}
```

### Label 白名單

| label | 來源 | 限制 |
| --- | --- | --- |
| `project` | project key | 不使用 id |
| `ticket_type` | ticket type code | 固定來源於事件類型設定 |
| `priority` | Ticket priority | `P1`、`P2`、`P3`、`P4` |
| `shift` | duty shift code | 只能用現行值班班制設定中的班別代碼 |
| `source` | 建立來源 | `api`、`import`、`copy` |
| `from_status` | 舊狀態 | 固定 Ticket 狀態 |
| `to_status` | 新狀態 | 固定 Ticket 狀態 |
| `content_type` | 附件分類 | `image`、`document`、`other` |
| `result` | 結果 | `success`、`failure` |
| `reason` | 失敗分類 | 共用 reason enum |

### 接線點

- Ticket create 成功後記錄 `opscenter_tickets_created_total`。
- Ticket status transition 成功後記錄 `opscenter_tickets_status_changes_total`。
- Ticket close / resolve 成功時計算 `created_at` 到完成時間，記錄 resolution duration。
- `open_current` 使用 collector 查詢 DB，目前狀態為 open / processing 類狀態的 Ticket。
- 附件上傳成功或失敗時記錄 attachment metrics。

### 注意事項

- 不在 label 放 Ticket 標題、外部單號或指派人。
- `open_current` 是查詢型 gauge，不在每次狀態變更手動增減，避免資料不一致。
- `shift` 取 Ticket 上已保存的班別，不在 metrics 記錄時計算歷史班別。

## Schedule Metrics 設計

### 指標

```text
opscenter_schedule_periods_current{mode, status}
opscenter_schedule_periods_created_total{mode, source, result, reason}
opscenter_schedule_generation_total{mode, result, reason}
opscenter_schedule_generation_duration_seconds_bucket{mode, result, le}
opscenter_schedule_generation_duration_seconds_sum{mode, result}
opscenter_schedule_generation_duration_seconds_count{mode, result}
opscenter_schedule_confirm_total{mode, result, reason}
opscenter_schedule_validation_failed_total{mode, rule}
opscenter_schedule_cell_updates_total{shift, leave_type, source, result, reason}
```

### Label 白名單

| label | 來源 | 限制 |
| --- | --- | --- |
| `mode` | 週期模式 | `4week`、`8week` |
| `status` | 週期狀態 | `draft`、`confirmed`、`closed` |
| `source` | 操作來源 | `api`、`scheduler` |
| `shift` | 班別代碼 | 只允許現行值班班制中的早 / 中 / 晚或設定允許班別 |
| `leave_type` | 假別 | `work`、`regular_leave`、`rest_day`、`public_holiday`、`other` |
| `rule` | 驗證規則 | 固定 rule code |
| `result` | 結果 | `success`、`failure`、`skipped` |
| `reason` | 失敗分類 | 共用 reason enum |

### 驗證規則代碼

第一版建議固定：

```text
missing_assignment
min_staff
weekly_regular_leave
leave_balance
public_holiday_balance
invalid_period
permission
conflict
```

### 接線點

- 建立排班週期後記錄 `periods_created_total`。
- 展開 / 重新整理班表時記錄 generation counter 與 duration。
- 確認週期時記錄 confirm counter。
- 驗證失敗時依 rule code 記錄 `validation_failed_total`。
- 排班格子更新成功或失敗時記錄 `cell_updates_total`。
- `periods_current` 使用 collector 依 status 查詢 DB。

### 注意事項

- 不把 period name、period id、user id、人名放進 label。
- 驗證失敗訊息可在 log 或 API response 顯示，但 metrics 僅放 rule code。
- 國定假日名稱不進 label。

## Report Metrics 設計

### 指標

```text
opscenter_report_preview_total{dataset, report_mode, group_by, series, result, reason}
opscenter_report_preview_duration_seconds_bucket{dataset, report_mode, result, le}
opscenter_report_preview_duration_seconds_sum{dataset, report_mode, result}
opscenter_report_preview_duration_seconds_count{dataset, report_mode, result}
opscenter_report_query_duration_seconds_bucket{dataset, report_mode, result, le}
opscenter_report_query_duration_seconds_sum{dataset, report_mode, result}
opscenter_report_query_duration_seconds_count{dataset, report_mode, result}
opscenter_report_export_total{dataset, format, result, reason}
opscenter_report_export_duration_seconds_bucket{dataset, format, result, le}
opscenter_report_export_duration_seconds_sum{dataset, format, result}
opscenter_report_export_duration_seconds_count{dataset, format, result}
opscenter_report_templates_saved_total{dataset, result, reason}
opscenter_report_templates_deleted_total{dataset, result, reason}
```

### Label 白名單

| label | 來源 | 限制 |
| --- | --- | --- |
| `dataset` | 報表資料集 | `ticket` 等固定 dataset code |
| `report_mode` | 報表模式 | 固定 enum，例如 `daily_oncall_execution` |
| `group_by` | 人員基準 | `assignee`、`creator`、`shift` |
| `series` | 系列維度 | 固定 enum，不用顯示名稱 |
| `format` | 匯出格式 | `csv`、`excel` |
| `result` | 結果 | `success`、`failure` |
| `reason` | 失敗分類 | 共用 reason enum |

### 接線點

- Preview API 進入後記錄開始時間，結束時記錄 preview counter 與 duration。
- Query service 內記錄 query duration。
- Export API 依格式記錄 export counter 與 duration。
- 範本儲存與刪除成功或失敗時記錄 template counter。

### 注意事項

- 不放 template id、template name、date range、project id、user id。
- report_mode 必須由後端 enum 收斂，不能直接使用前端任意字串。
- SQL 錯誤映射到 `dependency` 或 `unknown`，不輸出 SQL error。

## Storage / Image Metrics 設計

### 指標

```text
opscenter_storage_upload_total{backend, content_type, result, reason}
opscenter_storage_upload_bytes_bucket{backend, content_type, result, le}
opscenter_storage_upload_bytes_sum{backend, content_type, result}
opscenter_storage_upload_bytes_count{backend, content_type, result}
opscenter_image_conversion_total{format, result, reason}
opscenter_image_conversion_duration_seconds_bucket{format, result, le}
opscenter_image_conversion_duration_seconds_sum{format, result}
opscenter_image_conversion_duration_seconds_count{format, result}
opscenter_temp_file_cleanup_total{result, reason}
opscenter_temp_files_current
```

### Label 白名單

| label | 來源 | 限制 |
| --- | --- | --- |
| `backend` | 儲存後端 | `local`、`s3` |
| `content_type` | 檔案分類 | `image`、`document`、`other` |
| `format` | 圖片輸出格式 | `webp`、`jpeg`、`png`、`original` |
| `result` | 結果 | `success`、`failure` |
| `reason` | 失敗分類 | 共用 reason enum |

### 接線點

- attachment upload service 記錄 upload counter 與 bytes。
- image conversion service 記錄 conversion counter 與 duration。
- temp cleanup scheduler 記錄 cleanup counter。
- temp files current 使用 collector 查詢暫存目錄或 DB 中尚未清理資料。

### 注意事項

- 不輸出 bucket、object key、本機 path、filename。
- `content_type` 只取分類，不使用完整 MIME type 作 label。
- 若 `storage.image.output_format` 允許設定值，metrics label 需先 normalize，不允許任意字串。

## 後端介面設計

建議延續現有 metrics registry 模式，新增 domain recorder 介面。

```go
type TicketMetricsRecorder interface {
	RecordTicketCreated(project, ticketType, priority, shift, source string)
	RecordTicketStatusChanged(project, fromStatus, toStatus string)
	RecordTicketResolution(project, priority, shift string, durationSeconds float64)
	RecordTicketAttachmentUpload(project, contentType, result, reason string, bytes int64)
}
```

Schedule、Report、Storage 各自定義 recorder interface，不讓 domain service 直接依賴 Prometheus 套件。

collector 類 gauge 由 metrics package 依 repository provider 查詢，避免 domain service 手動維護目前數量。

## 測試設計

### 單元測試

- recorder 呼叫後，Prometheus gather 可取得預期 counter / histogram。
- result / reason / status 維持固定 enum。
- 空 label 會被 normalize 為 `unknown`。
- bytes / duration 小於零時需歸零或拒絕記錄。

### 敏感資訊測試

metrics output 不得包含：

```text
token
secret
password
ticket_id
user_id
username
email
title
description
external_ref
object_key
filename
raw_path
query
```

### 整合測試

- Ticket create / status transition 可產生對應 metrics。
- Schedule generation / confirm / validation failure 可產生對應 metrics。
- Report preview / export 可產生對應 metrics。
- Attachment upload / image conversion 可產生對應 metrics。
- `go test ./...` 通過。

## 實作順序建議

1. Ticket metrics。
2. Schedule metrics。
3. Report metrics。
4. Storage / Image metrics。
5. 敏感資訊與 label 白名單共用測試。

原因：

- Ticket 是最核心且最常查的業務指標。
- Schedule 近期變動多，第二順位補觀測可以降低排班問題排查成本。
- Report 與 Storage 依賴前兩者資料或附件流程，排在後面風險較低。

