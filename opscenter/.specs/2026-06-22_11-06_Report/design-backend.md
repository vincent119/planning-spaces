# Report Backend Design

## Bounded Context

Report 後端位於 `internal/report`，只讀取 Ticket 與 Project 相關資料並管理 `report_templates`。Report 不直接修改 `tickets`、`ticket_activities`、`ticket_types`、`ticket_resources`、`projects` 或 `sub_projects`。

建議分層：

```text
internal/report/
├── domain.go              # ReportMode、Config、Payload 型別
├── service.go             # 權限後的 use case
├── query_service.go       # 模式 A/B/C/D 查詢入口
├── repository.go          # report_templates CRUD
├── delivery.go            # Gin handler / Swagger 註解
└── *_test.go
```

## Data Models

### report_templates

```sql
CREATE TABLE report_templates (
  id CHAR(26) PRIMARY KEY DEFAULT generate_ulid(),
  project_id  CHAR(26) NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
  name        VARCHAR(128) NOT NULL,
  description TEXT,
  config      JSONB NOT NULL,
  created_by  CHAR(26) NOT NULL REFERENCES users(id),
  created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at  TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  deleted_at  TIMESTAMPTZ
);

CREATE INDEX ON report_templates (project_id) WHERE deleted_at IS NULL;
CREATE INDEX ON report_templates (created_by) WHERE deleted_at IS NULL;
CREATE UNIQUE INDEX ux_report_templates_project_name_active
  ON report_templates (project_id, name)
  WHERE deleted_at IS NULL;

COMMENT ON TABLE report_templates IS '報表範本資料';
COMMENT ON COLUMN report_templates.config IS 'ReportTemplateConfig JSON：report_mode、維度、指標、圖表與統計口徑';
```

範本刪除採軟刪除，避免已產出報表或審計紀錄失去來源。

## Domain Types

```go
type ReportMode string

const (
  ReportModeA ReportMode = "A"
  ReportModeB ReportMode = "B"
  ReportModeC ReportMode = "C"
  ReportModeD ReportMode = "D"
  ReportModeE ReportMode = "E" // 值班統計
  ReportModeG ReportMode = "G" // MS-指標導向月報
)

type ReportTemplateConfig struct {
  ReportMode    ReportMode `json:"report_mode"`
  XAxis         string     `json:"x_axis,omitempty"`
  YAxis         string     `json:"y_axis,omitempty"`
  Metrics       []string   `json:"metrics,omitempty"`
  ChartType     string     `json:"chart_type"`
  PersonBasis   string     `json:"person_basis,omitempty"`
  TitleTemplate string     `json:"title_template,omitempty"`
  Indicators    []string   `json:"indicators,omitempty"`
}

type ReportDateRange struct {
  DateFrom string `json:"date_from"`
  DateTo   string `json:"date_to"`
  Timezone string `json:"timezone,omitempty"`
}

type ReportChartPayload struct {
  Title      string             `json:"title"`
  ReportMode string             `json:"report_mode"`
  ChartType  string             `json:"chart_type"`
  XAxis      ReportAxis         `json:"x_axis"`
  Series     []ReportSeries     `json:"series"`
  Table      *ReportDetailTable `json:"table,omitempty"`
  Meta       ReportMeta         `json:"meta"`
}

type ReportAxis struct {
  Type   string   `json:"type"`
  Labels []string `json:"labels"`
}

type ReportSeries struct {
  Name  string    `json:"name"`
  Stack string    `json:"stack,omitempty"`
  Data  []float64 `json:"data"`
}

type ReportDetailTable struct {
  Columns []ReportTableColumn `json:"columns"`
  Rows    []map[string]any    `json:"rows"`
}

type ReportMeta struct {
  ProjectID   string `json:"project_id"`
  DateFrom    string `json:"date_from"`
  DateTo      string `json:"date_to"`
  Timezone    string `json:"timezone"`
  PersonBasis string `json:"person_basis,omitempty"`
  Empty       bool   `json:"empty"`
  GeneratedAt string `json:"generated_at"`
}

type DailyShiftExecutionPayload struct {
  Title      string                       `json:"title"`
  ReportMode string                       `json:"report_mode"` // E
  Matrix     DailyShiftExecutionMatrix    `json:"matrix"`
  Meta       ReportMeta                   `json:"meta"`
}

type DailyShiftExecutionMatrix struct {
  Columns []DailyShiftExecutionColumn `json:"columns"`
  Rows    []DailyShiftExecutionRow    `json:"rows"`
}

type DailyShiftExecutionColumn struct {
  Date    string `json:"date"`
  Weekday string `json:"weekday"`
}

type DailyShiftExecutionRow struct {
  ID          string         `json:"id"`
  Group       string         `json:"group"`
  Label       string         `json:"label"`
  RowType     string         `json:"row_type"` // group, shift, person
  Shift       string         `json:"shift,omitempty"`
  PersonID    string         `json:"person_id,omitempty"`
  PersonName  string         `json:"person_name,omitempty"`
  Level       int            `json:"level"`
  Values      map[string]int `json:"values"` // key = YYYY-MM-DD
}
```

## MS-指標導向月報

本節對應需求 15。模式 `G` 不新增資料表、查詢來源或獨立聚合器，而是重用模式 A 的固定 OP 聚合與序列化結果。

- `metrics` 與 `indicators` 只能各有一項，且必須一致。
- 唯一允許指標為 `op_jira_issue_count`、`op_alert_notification_count`、`op_payment_domain_change_count`。
- 兩欄皆省略時，後端正規化為 `op_jira_issue_count`；任一欄含一般 Ticket 指標、多項指標或與另一欄不一致時回傳輸入錯誤。
- 後端一律覆寫 `person_basis = created_by`，圖表、明細表及 CSV 共用模式 A 的月內週區間聚合結果。
- 後端不以專案代碼作安全性或資料隔離判斷；模式 G 只在前端的 MS 專案控制列提供，Report 權限及主專案範圍驗證沿用既有流程。

## API Contract

所有 API 前綴為 `/api/v1`。

### OpenAPI 現況與缺口

已檢查 `opscenter-server/Docs/openapi.json` 與 `opscenter-server/docs/openapi.json`，目前沒有下列 Report paths：

- `/api/v1/projects/{id}/report-templates`
- `/api/v1/projects/{id}/report-templates/{template_id}`
- `/api/v1/projects/{id}/report-templates/{template_id}/execute`
- `/api/v1/projects/{id}/reports/preview`
- `/api/v1/projects/{id}/reports/builtin-monthly`
- `/api/v1/projects/{id}/reports/export`

後續 server task 需補 delivery route、Swagger 註解與 OpenAPI schema；不得只新增 path 而缺少 requestBody / response DTO。OpenAPI 必須明確描述：

| 類型 | 缺口 |
| ---- | ---- |
| requestBody | 範本建立 / 更新、preview、builtin-monthly、execute |
| response DTO | `ReportTemplateResponse`、`ReportChartPayload`、範本列表、刪除結果 |
| 錯誤 response | 400、401、403、404、409、500，統一 `httpx.APIResponse[struct{}]` |
| CSV response | `text/csv` 或 `application/octet-stream`，需含 `Content-Disposition` |
| 權限說明 | Report read / create / update / delete 與 project visibility |

### Envelope

JSON API 使用既有 `httpx.APIResponse[T]`：

```json
{
  "code": 0,
  "message": "ok",
  "data": {},
  "trace_id": "01K..."
}
```

錯誤 response：

```json
{
  "code": 400,
  "message": "invalid report request",
  "trace_id": "01K..."
}
```

CSV export 不包 JSON envelope；成功時直接回傳檔案內容。CSV 失敗時回傳 JSON error envelope，前端需依 `Content-Type` 判斷，不得把錯誤 JSON 下載成 CSV。

### 範本

| Method | Path | 權限 | 說明 |
| ------ | ---- | ---- | ---- |
| GET | `/projects/:id/report-templates` | Report read + project visible | 範本列表 |
| POST | `/projects/:id/report-templates` | Report create 或 PM+ | 建立範本 |
| GET | `/projects/:id/report-templates/:template_id` | Report read + project visible | 範本詳情 |
| PUT | `/projects/:id/report-templates/:template_id` | Report update 或 PM+ | 更新範本 |
| DELETE | `/projects/:id/report-templates/:template_id` | Report delete 或 PM+ | 軟刪除範本 |

建立 / 更新 request：

```json
{
  "name": "月報模式 C",
  "description": "人員與子專案堆疊圖",
  "config": {
    "report_mode": "C",
    "chart_type": "stacked_bar",
    "person_basis": "created_by",
    "title_template": "{{year}}年{{from}}-{{to}}運維處理事件數量"
  }
}
```

範本 response：

```json
{
  "id": "01KVREPORTTEMPLATE000000001",
  "project_id": "01KVPROJECT00000000000001",
  "name": "月報模式 C",
  "description": "人員與子專案堆疊圖",
  "config": {
    "report_mode": "C",
    "chart_type": "stacked_bar",
    "person_basis": "created_by",
    "title_template": "{{year}}年{{from}}-{{to}}運維處理事件數量"
  },
  "created_by": "01KTBB9FCWG7YQRGK7YFN061AY",
  "created_at": "2026-06-22T03:30:00Z",
  "updated_at": "2026-06-22T03:30:00Z"
}
```

### 預覽與執行

| Method | Path | 權限 | 說明 |
| ------ | ---- | ---- | ---- |
| POST | `/projects/:id/reports/preview` | Report read + project visible | 依 request config 即時預覽 |
| POST | `/projects/:id/reports/builtin-monthly` | Report read + project visible | 執行內建月報 A / B / C |
| POST | `/projects/:id/reports/daily-shift-execution` | Report read + project visible | 執行值班統計 |
| POST | `/projects/:id/report-templates/:template_id/execute` | Report read + project visible | 執行已儲存範本 |
| GET | `/projects/:id/reports/export` | Report read + project visible | 依查詢條件匯出 CSV |

### Project Scope Filter

Report 查詢以 path 上的 `:id` 作為主專案範圍。額外的 `sub_project_id` 為選填篩選條件：

- 未提供 `sub_project_id` 或值為空字串：查詢整個主專案。
- 提供 `sub_project_id`：後端必須驗證該子專案隸屬於 path `:id` 主專案，且使用者可見該主專案後，查詢限定 `tickets.sub_project_id = sub_project_id`。
- `preview`、`builtin-monthly`、`template execute`、`daily-shift-execution` 與 `export` 必須使用相同語意。
- `sub_project_id` 不得只由前端過濾；後端 SQL / query builder 必須套用條件，避免匯出與 API 結果超出畫面範圍。
- `meta` 可回傳 `sub_project_id`，讓前端確認目前 payload 的查詢範圍。

`/reports/preview` request：

```json
{
  "date_range": {
    "date_from": "2026-06-01",
    "date_to": "2026-06-30",
    "timezone": "Asia/Taipei"
  },
  "sub_project_id": "",
  "config": {
    "report_mode": "D",
    "x_axis": "person",
    "y_axis": "sub_project",
    "metrics": ["ticket_count"],
    "chart_type": "stacked_bar",
    "person_basis": "created_by"
  }
}
```

`/reports/builtin-monthly` request：

```json
{
  "mode": "C",
  "year": 2026,
  "month": 6,
  "person_basis": "created_by",
  "sub_project_id": ""
}
```

`/reports/daily-shift-execution` request：

```json
{
  "date_range": {
    "date_from": "2026-06-01",
    "date_to": "2026-06-30",
    "timezone": "Asia/Taipei"
  },
  "metric_groups": [
    "alert_notification"
  ],
  "include_shift_summary": true,
  "include_person_detail": true
}
```

`/reports/daily-shift-execution` response 使用 `DailyShiftExecutionPayload`。報表中心在單一「值班統計」模式下，由「數值」選擇並送出單一 `metric_group`，讓畫面分別呈現「值班統計－告警通知」、「值班統計－域名更換」、「值班統計－支付域名更換」。查無資料時 `matrix.rows=[]` 且 `meta.empty=true`，不得回傳假列。後端仍回傳既有 shift row 與每日 values；總計及指定班別列隱藏是前端呈現邏輯，不變更 API 或資料庫。

### 值班統計查詢規則

資料來源原則：

- 日期欄位：依 `Asia/Taipei` 將 `date_from` 到 `date_to` 展開為每日欄位。
- 指標群組：使用白名單，不允許前端任意字串進 SQL。
- Ticket 類指標：優先由 `tickets`、`ticket_types`、`ticket_resources`、`sub_projects` 與 `ticket_activities` 聚合。
- 班別：需依排班資料或後端可驗證的時間規則推導，不在 Ticket 表新增班別快照欄位。
- 人員：可依 `ticket_activities.actor_id` 或 `tickets.created_by` 聚合；API 需明確定義使用口徑。

初始白名單：

| metric_group | 顯示名稱 | 初始來源 |
| ---- | ---- | ---- |
| `jira_notification` | Jira 通知數 | 待確認 Jira/通知來源；來源不足時回空 |
| `alert_notification` | 告警通知數 | `ticket_resources.resource_type = alert` 或指定資訊來源 |
| `domain_change` | 域名更換 | `ticket_types` / `ticket_resources` / 子專案規則待確認 |
| `payment_domain_change` | 支付域名更換數 | `ticket_types` / `ticket_resources` / 子專案規則待確認 |

來源不足的群組不能由前端硬補；後端需在 `meta` 或 row warning 中標示資料來源尚未設定。

`/report-templates/:template_id/execute` request：

```json
{
  "date_range": {
    "date_from": "2026-06-01",
    "date_to": "2026-06-30",
    "timezone": "Asia/Taipei"
  }
}
```

### DTO 驗證規則

| 欄位 | 規則 |
| ---- | ---- |
| `report_mode` | `A`、`B`、`C`、`D`、`E` |
| `chart_type` | `bar`、`stacked_bar` |
| `person_basis` | `created_by`、`actor`，未傳時預設 `created_by` |
| `date_from` / `date_to` | `YYYY-MM-DD`，`date_from <= date_to` |
| `timezone` | 目前只接受空值或 `Asia/Taipei` |
| `x_axis` | 白名單：`day`、`week`、`month`、`person`、`sub_project` |
| `y_axis` | 白名單：`shift`、`person`、`task_title`、`sub_project` |
| `metrics` | 白名單：`ticket_count`、`ticket_type`、`source`、`sub_project` |
| `indicators` | 白名單：`ticket_type`、`source`、`sub_project`、`op_jira_issue_count`、`op_alert_notification_count`、`op_payment_domain_change_count` |

### 空資料 Payload

查詢成功但無資料時回傳 HTTP 200：

```json
{
  "title": "2026年0601-0630運維處理事件數量",
  "report_mode": "C",
  "chart_type": "stacked_bar",
  "x_axis": {
    "type": "category",
    "labels": []
  },
  "series": [],
  "table": {
    "columns": [],
    "rows": []
  },
  "meta": {
    "project_id": "01KVPROJECT00000000000001",
    "date_from": "2026-06-01",
    "date_to": "2026-06-30",
    "timezone": "Asia/Taipei",
    "person_basis": "created_by",
    "empty": true,
    "generated_at": "2026-06-22T03:30:00Z"
  }
}
```

### 錯誤契約

| HTTP | `code` | `message` | 場景 |
| ---- | ------ | --------- | ---- |
| 400 | 400 | `invalid report request` | request body、日期、mode、維度或白名單驗證失敗 |
| 401 | 401 | `current user missing` | 未登入或 middleware 未注入使用者 |
| 403 | 403 | `permission denied` | Report 權限不足或專案不可操作 |
| 404 | 404 | `resource not found` | 專案或範本不存在，或不可見時依 Project 規則隱藏 |
| 409 | 409 | `resource conflict` | 範本名稱重複或狀態衝突 |
| 500 | 500 | `query report failed` | 查詢或系統錯誤 |

### CSV Response

成功：

```text
HTTP/1.1 200 OK
Content-Type: text/csv; charset=utf-8
Content-Disposition: attachment; filename="report_MS_2026-06-01_2026-06-30_C.csv"
```

錯誤時不得回傳 `text/csv`，需使用 JSON error envelope。

## Query Rules

### 共通規則

- `ticket_types`、`ticket_resources`、`sub_projects` 均以 id join，不用顯示文字當 key。
- 查詢欄位、維度、排序、指標使用白名單轉換，不直接拼接前端輸入。
- 日期區間轉換由共用 helper 處理，固定 `Asia/Taipei`。
- 複雜聚合使用 `db.Raw()` 與參數化 SQL。
- 查詢需包含 `project_id = ?`，避免跨專案資料外洩。

### 模式 A

- 依月份切成週區間。
- 每個 indicator 回傳一組 chart payload 或同 payload 中多個 chart section；MVP 若回傳單一 payload，需在 `series` 中明確標註 indicator。
- 人員維度依 `person_basis` 決定。
- 固定 OP 指標需支援 `op_jira_issue_count`、`op_alert_notification_count`、`op_payment_domain_change_count`。
- 固定 OP 指標 payload 使用既有 `ReportChartPayload`：`x_axis.labels` 為值班人員、`series[0].name` 為 `總計`、`series[0].data` 為各人員整月總計。
- 固定 OP 指標 `table.columns` 必須包含 `person`、`total` 與月內週區間欄位；`table.rows` 每位值班人員一列。
- 週區間欄位由後端產生，範例：`6/1-6/4`、`6/5-6/11`、`6/12-6/18`、`6/19-6/25`、`6/26-6/30`。

固定 OP 指標統計口徑：

| indicator | 顯示名稱 | 統計口徑 |
| ---- | ---- | ---- |
| `op_jira_issue_count` | OP Jira 開單數量統計 | 依 `tickets.external_ref` 去除前後空白後是否非空統計 |
| `op_alert_notification_count` | OP 告警通知數量統計 | 依 `ticket_resources.resource_type = alert` 或設定的告警資訊來源統計 |
| `op_payment_domain_change_count` | OP 支付or業務項目域名更換統計 | 依 `ticket_resources.code = business_domain_change` 統計 |

#### 固定 OP 月報 Excel 基準修正

`TicketFact` 增加報表判斷所需的穩定欄位：

```go
type TicketFact struct {
    ExternalRef          string
    TicketResourceCode   string
    ExcelOpenTicketImport bool
}
```

Repository 查詢需以既有 join 取得 `t.external_ref` 與 `tr.code`，並保留有效班別與人員欄位。`ExcelOpenTicketImport` 由同 Ticket 的 `created` activity 及 `content LIKE 'imported from Excel 開單記錄-%'` 判斷；此內容由既有 importer 寫入，是目前可用的匯入 provenance。不得改以 `tickets.description` 判斷。查詢仍必須參數化限制 `project_id`、`deleted_at` 與 `Asia/Taipei` 日期邊界，不得以 MS 專案 id 或 Excel 筆數寫死條件。

`factMatchesFixedOPIndicator` 的本次調整如下：

| indicator | 判斷規則 |
| ---- | ---- |
| `op_jira_issue_count` | `fact.ExcelOpenTicketImport && strings.TrimSpace(fact.ExternalRef) != ""` |
| `op_payment_domain_change_count` | `fact.ExcelOpenTicketImport && fact.TicketResourceCode == "business_domain_change"` |
| `op_alert_notification_count` | 不修改既有規則 |

聚合需分成「名冊」與「計數」兩個階段：Jira 與支付域名的名冊從日期範圍內全部 Excel 開單記錄 facts 建立，計數才套用 indicator predicate。如此可保留支付域名更換數量為 0 的值班人員。告警通知名冊仍只使用符合既有告警 predicate 的 facts。顯示名稱使用有效班別加人員，將早班／中班／晚班正規化為 `早`／`中`／`晚`；若缺少班別，使用明確的未歸屬班別標籤，不從 `description` 擷取填寫人。

週區間沿用月初到首個星期四、其後星期五至星期四的切法；當最後一段只有單一天時，併入前一段。2026 年 7 月的輸出固定驗證為五段：`7/1-7/2`、`7/3-7/9`、`7/10-7/16`、`7/17-7/23`、`7/24-7/31`。

標題由後端依月份與 indicator 產生：

- `YYYY年MM月 OP Jira開單數量統計`
- `YYYY年MM月 OP 支付or業務項目域名更換統計`

圖表、明細表與 CSV 必須共用同一聚合結果。單元測試驗證欄位映射、predicate、班別人員標籤、零值名冊、週區間與標題；整合測試可使用 MS 專案 2026 年 7 月的 663 與 492 作為環境基準，但不得將基準數字納入正式邏輯。模式 B／C／D／E 與告警通知不在本次變更範圍。

#### 三個固定 OP 指標人員顯示一致性

`buildModeAFixedOPIndicator` 必須對 `op_alert_notification_count`、`op_jira_issue_count`、`op_payment_domain_change_count` 共用 `fixedOPPersonLabel` 與 `fixedOPPersonShiftRank`。告警不得再經由 indicator 特例退回純人名，也不得跳過班別排序。

名冊來源與人員顯示需分離處理：告警名冊仍只收錄符合既有告警 predicate 的 facts；Jira 與支付域名仍使用 Excel 開單記錄名冊。統一標籤不得擴大告警名冊或改變任何指標數字。

聚合先產生單一有序 `labels`，再由該 labels 同步建立：

- `x_axis.labels`
- `series[0].data` 的人員 index
- `table.rows` 的列序與 `person`
- CSV 的人員列序

後端測試需針對三個指標驗證 `labels[index] == table.rows[index].person`，且 series data 長度與 labels 相等。前端不得為修正告警顯示而重新組合班別或人員。

### 模式 B

- 以 Ticket 標題或任務內容聚合。
- 橫軸為人員。
- 堆疊或表格列需保留 Ticket id，方便前端後續連回明細。

### 模式 C

- 整月彙總。
- 橫軸為人員。
- stack 為子專案或業務項目。
- `person_basis = actor` 時需 join `ticket_activities` 並限制 activity 類型，避免同一張 Ticket 多次活動造成錯誤計數；預設可先支援 `created`、`field_updated`、`status_changed`、`comment_added` 等白名單。

### 模式 D

- 依 `ReportTemplateConfig` 動態選擇查詢器。
- MVP 可先限制可用維度組合，超出白名單回 400。
- 後續完整設計器再擴充更多維度，不改 payload 結構。

## Audit And Logging

- 範本新增、修改、刪除寫入 `system_audit_logs`，detail 包含 `template_id`、`project_id`、異動前後摘要。
- 報表查詢失敗需記錄 `request_id`、`project_id`、`report_mode`、`date_from`、`date_to`、`user_id`。
- 不記錄 JWT、cookie、S3 secret 或資料庫連線字串。

## OpenAPI

Report delivery 需補 Swagger 註解並由既有 `make openapi` 產出 `Docs/openapi.json`。OpenAPI schema 必須包含 requestBody 與 response DTO，不得只列 path。

## 模式 D 品質強化後端契約

### V2 Config 與 Metadata

- `TemplateConfigV2` 新增選填 `title_template`，只允許純文字及 `{{project_name}}`、`{{date_from}}`、`{{date_to}}`；未知變數、未閉合語法與超過長度限制需回 `400 invalid_title_template`。
- dataset metadata 維持穩定 code，列舉型 filter 可新增 `options`；大量動態選項使用專案範圍內、具 read 權限的分頁 options endpoint，不把任意資料全量塞入 metadata。
- config validation error 需回傳穩定 code；若擴充 response data，格式為 `{ "field": "query.filters", "validation_code": "invalid_filter_values" }`，前端不得解析自由文字 message。

### POST CSV Export

新增：

```text
POST /api/v1/projects/:id/reports/export
Content-Type: application/json
Accept: text/csv
權限：Report read + project visible
```

request body 與 preview 共用 date range、`sub_project_id` 與 config 白名單驗證。成功 response 維持 `text/csv` 與 `Content-Disposition`；失敗使用 JSON error envelope。舊 GET export 在相容期只接受既有簡單 query，不接受完整 V2 config，並於 OpenAPI 標記 deprecated。

### CSV Formula Injection 防護

所有 CSV 文字欄位在唯一序列化邊界套用 `sanitizeCSVCell`：移除判斷用前置空白後，若首字元為 `=`、`+`、`-`、`@`，輸出值前加單引號。數值型欄位維持數值格式，不以字串轉義掩蓋資料型別錯誤。需覆蓋 Ticket 標題、人員名稱、子專案、事件類型、資訊來源、動態 table 欄位與每日矩陣 label。

### 保護行為

- POST export 必須重新執行 Report read、專案可見性、子專案歸屬與 config 白名單驗證。
- preview、template execute、GET 相容匯出與 POST export 的日期及專案範圍語意一致。
- 不在 log、trace 或 error message 記錄完整 config filter values；僅記錄 request id、project id、dataset、日期範圍與結果狀態。
- 同步匯出維持既有大小限制；超限回明確錯誤，後續再銜接非同步 Job。

## V3 多區塊報表範本後端設計

### 文件定位與已知契約

本節對應需求 8。V3 沿用 `report_templates`、範本 CRUD、project scope、Report 表單權限、V2 dataset metadata、query planner 與 CSV 防護；新增 config version 3、layout validation、multi-block preview 與 revision conflict，不改寫 V1、V2 parser 與既有單圖 execute response。

### Bounded Context

包含：V3 config domain、layout / block / parameter validation、V3 template persistence、revision、multi-block preview orchestration、partial result、audit 摘要、資料 block CSV。

不包含：新資料集、跨資料集 join、任意公式、HTML sanitization、即時協作、PDF renderer、非同步查詢 Job。

### V3 Domain Contract

```go
type TemplateConfigV3 struct {
    Version    int                         `json:"version"`
    Parameters []ReportParameterDefinition `json:"parameters"`
    Layout     ReportLayoutConfig          `json:"layout"`
    Blocks     []ReportBlockConfig         `json:"blocks"`
}

type ReportLayoutConfig struct {
    Columns   int                `json:"columns"`
    RowHeight int                `json:"row_height"`
    Gap       int                `json:"gap"`
    Items     []ReportLayoutItem `json:"items"`
}

type ReportLayoutPreviewResponse struct {
    Revision int                        `json:"revision"`
    Blocks   []ReportBlockPreviewResult `json:"blocks"`
}
```

`ReportBlockConfig` 以 custom unmarshal 或明確 DTO discriminator 驗證 block type，不接受任意 map 直接進 domain。資料 block query 轉換成既有 V2 planner input；text block 不執行 query。

### Persistence 與 Revision

- 優先沿用 `report_templates.config` JSON 欄位與 `config_version = 3`，不為每個 block 建立資料表。
- `report_templates` 新增或確認 `revision` 整數欄位；create 從 1 開始，每次成功 update 原子加 1。
- update 條件使用 `WHERE id = ? AND project_id = ? AND revision = ?`；affected rows 為 0 時重新確認 not found 或回 `ErrConflict`。
- 不修改 V1、V2 config JSON；只有使用者明確另存 V3 時建立新 config。

### Validation

- blocks 最多 20，資料 blocks 最多 12；block id 必填、長度受限且在範本內唯一。
- layout columns 固定 12；每個資料與文字 block 必須各有一筆 layout item，且不得有孤兒 item。
- `x >= 0`、`y >= 0`、`w / h > 0`、`x + w <= 12`，並符合 block type min / max。
- 任兩個矩形不得相交；後端只驗證 canonical layout，不替前端猜測碰撞結果。
- parameter code 與 type 使用白名單；binding 必須引用已宣告 parameter。
- text content 以 rune 計算長度，拒絕 HTML 標記並以純文字回傳；前端仍不得使用 HTML renderer。
- 資料 query 經 `NormalizeTemplateConfigV2` 等價白名單驗證，category / series limit 不得放寬。

### API Contract

新增且不改變既有 execute payload：

| Method | Path | 權限 | 說明 |
| --- | --- | --- | --- |
| POST | `/projects/:id/reports/layout-preview` | reports read | 以 V3 config 與 runtime parameters 預覽多個 block |
| POST | `/projects/:id/report-templates/:template_id/execute-layout` | reports read | 執行已儲存 V3 範本 |
| POST | `/projects/:id/report-templates/:template_id/blocks/:block_id/export` | reports read | 匯出已儲存資料 block CSV |

範本 CRUD request / response 增加選填 `revision`；V3 update 必填 revision，V1、V2 既有 client 在相容期維持原行為。若無法在同一 endpoint 安全區分，需建立明確 V3 update endpoint，不得弱化 V1、V2 相容性。

preview request：

```json
{
  "config": { "version": 3 },
  "parameter_values": {
    "date_range": {
      "date_from": "2026-08-01",
      "date_to": "2026-08-31",
      "timezone": "Asia/Taipei"
    },
    "sub_project": "01..."
  },
  "block_ids": ["ticket-total", "daily-trend"]
}
```

`block_ids` 省略時執行所有資料 block；提供時只能是 config 中存在的 block。後端仍驗證完整 config，避免以未執行 block 夾帶不合法資料。

### Partial Result 與錯誤

HTTP 層錯誤用於 request、權限、project scope 或整份 config 無效。合法 config 的個別 query 失敗時 HTTP 仍回成功 envelope，block result 使用：

```json
{
  "block_id": "daily-trend",
  "status": "error",
  "payload": null,
  "error": { "code": "query_failed" }
}
```

允許 status：`success`、`empty`、`error`。不得把 SQL、filter values 或內部 error message 回傳前端。每個 block query 使用獨立 context；request 取消時停止尚未執行與執行中的 query。

### 查詢、效能與安全

- 單次最多執行 12 個資料 block，server concurrency 上限 4；不得為每個 block 無限制建立 goroutine。
- 相同 normalized query 與 parameter hash 可在單一 request 內去重，但 response 仍需複製對應 block id。
- 每個 block 沿用 project access、sub-project ownership、date range、preview category / series 與 CSV Formula Injection 防護。
- V3 config 不寫入 access log；audit 只記 template id、revision、block id / type 清單及 layout checksum。
- text block 只作資料傳輸，不由後端產生可執行 HTML。

### Audit Events

- `report_template_v3_created`
- `report_template_v3_updated`
- `report_template_v3_block_exported`

update audit 摘要記錄 before / after revision、block count、changed block ids、layout checksum，不記錄 query filter values、文字全文或使用者輸入的敏感資料。

### 風險與處理方式

- V3 JSON 增大：限制 block 數、文字長度與 request body，超限回 413 或明確 400 code。
- partial success 難以監控：metrics 分開記錄 request count、block query count、block error count 與 latency。
- revision migration 影響既有資料：migration 設 default 1，V1、V2 讀取不依賴 client 傳 revision。
- 重複 query 增加 DB 壓力：request-scope dedupe、concurrency 4 與既有 planner limit 共同保護。

## OP 專案開單數量年報

本節對應需求 14。年報新增獨立內部模式代碼 `F`，不改變既有 A 至 E 語意。年報 request 使用 `year` 與固定指標 `op_project_ticket_count`；後端依 `Asia/Taipei` 將整年轉為 UTC half-open range，並固定建立 12 個月份 bucket。

建議 API 沿用 Report 預覽與匯出入口：

```json
{
  "date_range": {
    "date_from": "2026-01-01",
    "date_to": "2026-12-31",
    "timezone": "Asia/Taipei"
  },
  "sub_project_id": "",
  "config": {
    "report_mode": "F",
    "indicators": ["op_project_ticket_count"],
    "chart_type": "stacked_bar",
    "person_basis": "created_by"
  }
}
```

查詢資料來源限定 `tickets` 與 `projects`。使用 `tickets.created_at` 計數，每張 Ticket 計一次；不得 join `ticket_activities` 造成倍增。年報範圍為所有 `projects.deleted_at IS NULL AND projects.status = 'active'` 的主專案，停用與封存主專案不納入；統計不受 URL 主專案或子專案篩選影響，入口仍以既有 Report `read` 權限保護。月份邊界與空月份由 domain 層依 `Asia/Taipei` 建立，SQL 只回傳實際聚合值。

`ReportChartPayload` 規則如下：

- `report_mode = F`，`chart_type = stacked_bar`。
- `x_axis.labels` 固定為 `YYYY/01` 至 `YYYY/12`。
- 每個 `series` 代表一個主專案，`data` 固定 12 筆；缺值補 `0`。
- `table.columns` 依序為 `project`、12 個月份、`total`。
- `table.rows` 先列主專案，再列 `row_type = grand_total` 的「總數」列。
- `meta.empty` 依全年實際 Ticket 總數是否為零決定；即使為空仍保留 12 月份與總數列。

主專案排序需穩定，以名稱及 id 作 deterministic fallback。圖表、表格與 CSV 必須由同一個聚合模型序列化，避免各自加總產生差異。
