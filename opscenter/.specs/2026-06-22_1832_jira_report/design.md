# Jira Report 設計

## 設計來源

本文件參考舊設計：

- `.kiro/specs/2026-06-01_10-22_oncall-ticket-system/requirements.md` 需求 15
- `.kiro/specs/2026-06-01_10-22_oncall-ticket-system/design-backend.md` `jira_issues` 資料表與 API 草案
- `.kiro/specs/2026-06-01_10-22_oncall-ticket-system/design-frontend.md` `/projects/:id/jira` 路由與 feature 目錄

本文件為 Jira Report 後續實作準則。若與舊文件有落差，以本文件為準。

## 文件與現況對齊

目前程式碼與契約檢查結果：

- `opscenter-server/Docs/openapi.json` 尚未提供 `/projects/{id}/jira/import`、`/projects/{id}/jira/report`、`/projects/{id}/jira/export`。
- `opscenter-server/internal` 尚未有 `internal/jira` bounded context。
- 目前 server router 尚未註冊 `/api/v1/projects/:id/jira/*` API。
- `cmd/import_ticket_data` 是 Ticket Excel 匯入工具，負責寫入 `tickets`，不得作為 Jira CSV 匯入工具重用。
- `internal/report` 內含 `op_jira_issue_count` 等固定報表指標，但資料來源是既有報表查詢，不等同本 spec 的 `jira_issues`。
- 舊文件的 `jira_issues` schema 與 API 草案只作為來源，本文件已修正 primary key、unique key、API contract 與匯出權限。

後續實作時，若程式碼與本文件不一致，需先回到本 spec 新增或修正 task，再修改程式碼。

## 架構定位

Jira Report 是獨立 bounded context，建議後端建立：

```text
opscenter-server/internal/jira/
├── domain.go
├── repository.go
├── service.go
├── parser.go
├── query_service.go
└── delivery.go
```

前端建立：

```text
opscenter-frontend/src/features/jira/
├── api/
├── components/
├── pages/
├── charts/
├── types.ts
└── queryKeys.ts
```

路由：

```text
/projects/:id/jira
```

Jira Report 不依賴 `tickets`，也不修改 `tickets`。

## 資料表設計

### jira_issues

```sql
CREATE TABLE jira_issues (
  id CHAR(26) DEFAULT generate_ulid(),
  issue_key              VARCHAR(64) NOT NULL,
  created_date           DATE NOT NULL,
  project_id             CHAR(26) NOT NULL REFERENCES projects(id),
  summary                TEXT,
  issue_id               VARCHAR(64),
  issue_type             VARCHAR(64),
  status                 VARCHAR(64),
  project_key            VARCHAR(32),
  project_name           VARCHAR(255),
  project_type           VARCHAR(64),
  project_lead           VARCHAR(255),
  project_desc           TEXT,
  project_url            VARCHAR(512),
  priority               VARCHAR(64),
  resolution             VARCHAR(64),
  assignee               VARCHAR(255),
  reporter               VARCHAR(255),
  creator                VARCHAR(255),
  updated_at             TIMESTAMPTZ,
  last_viewed_at         TIMESTAMPTZ,
  resolved_at            TIMESTAMPTZ,
  due_date               DATE,
  votes                  INT,
  labels                 TEXT[],
  description            TEXT,
  environment            TEXT,
  watchers               TEXT,
  original_estimate      INT,
  remaining_estimate     INT,
  time_spent             INT,
  work_ratio             INT,
  agg_original_estimate  INT,
  agg_remaining_estimate INT,
  agg_time_spent         INT,
  security_level         VARCHAR(64),
  cf_epic_status         VARCHAR(64),
  cf_epic_color          VARCHAR(32),
  cf_story_points        NUMERIC(6,1),
  cf_epic_name           VARCHAR(255),
  cf_epic_link           VARCHAR(64),
  cf_level               VARCHAR(64),
  imported_at            TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  PRIMARY KEY (id, created_date),
  UNIQUE (project_id, issue_key, created_date)
) PARTITION BY RANGE (created_date);
```

設計修正說明：

- 舊稿寫 `PRIMARY KEY (id, issue_key, created_date)`，但實務上 `id` 已唯一，去重不應依賴 primary key。
- 本設計使用 `PRIMARY KEY (id, created_date)` 滿足 partition key 要求，並使用 `UNIQUE (project_id, issue_key, created_date)` 做同專案去重。
- `project_id` 對應 Opscenter 主專案，只代表資料歸屬，不代表 Jira Issue 與 Opscenter Ticket 有關聯。

### 年份分區

每年建立一個 partition：

```sql
CREATE TABLE jira_issues_2026 PARTITION OF jira_issues
  FOR VALUES FROM ('2026-01-01') TO ('2027-01-01');

CREATE INDEX idx_jira_issues_2026_project_created
  ON jira_issues_2026 (project_id, created_date);

CREATE INDEX idx_jira_issues_2026_assignee
  ON jira_issues_2026 (project_id, assignee);

CREATE INDEX idx_jira_issues_2026_project_key
  ON jira_issues_2026 (project_id, project_key);
```

後續每年需新增對應 partition。若匯入資料跨年度，service 必須確認目標 partition 存在，否則回傳明確錯誤，不得 panic。

## CSV 欄位映射

| CSV 欄位 | DB 欄位 |
| --- | --- |
| 概要 | summary |
| 问题关键字 | issue_key |
| 问题ID | issue_id |
| 问题类型 | issue_type |
| 状态 | status |
| 项目关键字 | project_key |
| 项目名称 | project_name |
| 项目类型 | project_type |
| 项目主管 | project_lead |
| 项目描述 | project_desc |
| 项目URL | project_url |
| 优先级 | priority |
| 解决结果 | resolution |
| 经办人 | assignee |
| 报告人 | reporter |
| 创建者 | creator |
| 创建日期 | created_date |
| 已更新 | updated_at |
| 最近查看的 | last_viewed_at |
| 已解决 | resolved_at |
| 到期日 | due_date |
| 表决 | votes |
| 标签 | labels |
| 描述 | description |
| 环境 | environment |
| 管理关注列表 | watchers |
| 初始预估 | original_estimate |
| 剩余的估算 | remaining_estimate |
| 耗费时间 | time_spent |
| 工作量比率 | work_ratio |
| Σ 原预估时间 | agg_original_estimate |
| Σ 预估剩余时间 | agg_remaining_estimate |
| Σ 耗费时间 | agg_time_spent |
| 安全级别 | security_level |
| 自定义字段(Epic状态) | cf_epic_status |
| 自定义字段(Epic颜色) | cf_epic_color |
| 自定义字段(Story Point) | cf_story_points |
| 自定义字段(史诗名称) | cf_epic_name |
| 自定义字段(史诗链接) | cf_epic_link |
| 自定义字段(等级) | cf_level |

## Parser 設計

Parser 職責：

- 僅接受 CSV。
- 支援 UTF-8 與 UTF-8 BOM。
- 第一列作為 header。
- 不執行公式。
- 對每列建立 `JiraIssueImportRow`，保留 row number 與錯誤訊息。
- 必填欄位缺少時標記 row error。

日期解析：

- `创建日期` 支援 `YYYY-MM-DD`、`YYYY/MM/DD`、`YYYY-MM-DD HH:mm:ss`、`YYYY/MM/DD HH:mm:ss`。
- `创建日期` 與 timestamp 欄位需支援 Jira 簡中匯出的中文月份與上午/下午格式，例如 `31/五月/26 7:48 下午`。
- 日期與時間以系統業務時區 `Asia/Taipei` 解析。
- `created_date` 只保存 date。
- `updated_at`、`last_viewed_at`、`resolved_at` 保存 timestamp。

數值解析：

- 空字串保存為 NULL。
- `votes`、估算欄位、耗費欄位、工作量比率轉整數。
- `cf_story_points` 轉 decimal。
- 無法解析時該列 error，不寫入該列。

## 匯入流程

```text
HTTP multipart upload
  -> 驗證專案可見性與權限
  -> 解析 CSV header
  -> 逐列轉換 JiraIssue
  -> 逐列 insert
  -> ON CONFLICT (project_id, issue_key, created_date) DO NOTHING
  -> 回傳匯入 summary 與失敗列
```

匯入模式初始只支援 `insert_only`。

## API 設計

### 匯入 Jira CSV

```http
POST /api/v1/projects/:id/jira/import
Content-Type: multipart/form-data
```

Form data：

| 欄位 | 型別 | 必填 | 說明 |
| --- | --- | --- | --- |
| file | file | 是 | Jira CSV |

Response：

```json
{
  "data": {
    "total_rows": 120,
    "created_rows": 110,
    "skipped_rows": 8,
    "failed_rows": 2,
    "row_errors": [
      {
        "row_number": 10,
        "issue_key": "QIEZ-1648",
        "message": "创建日期不可為空"
      }
    ]
  }
}
```

### 查詢 Jira 報表

```http
GET /api/v1/projects/:id/jira/report
```

Query：

| 參數 | 說明 |
| --- | --- |
| date_from | 创建日期起，`YYYY-MM-DD` |
| date_to | 创建日期迄，`YYYY-MM-DD` |
| jira_project_key | Jira 项目关键字 |
| status | Jira 状态 |
| priority | Jira 优先级 |
| assignee | Jira 经办人 |

Response：

```json
{
  "data": {
    "title": "Jira 執行統計",
    "chart": {
      "x_axis": {
        "type": "category",
        "labels": ["Chenmin", "Terry", "Kevin"]
      },
      "series": [
        {
          "name": "Issue 數量",
          "type": "bar",
          "data": [14, 33, 23]
        }
      ]
    },
    "table": {
      "columns": [
        { "field": "assignee", "header_name": "经办人", "type": "string" },
        { "field": "count", "header_name": "總計", "type": "number" }
      ],
      "rows": [
        { "id": "Chenmin", "assignee": "Chenmin", "count": 14 }
      ]
    },
    "meta": {
      "project_id": "01...",
      "date_from": "2026-06-01",
      "date_to": "2026-06-30",
      "empty": false,
      "generated_at": "2026-06-22T10:00:00Z"
    }
  }
}
```

### 匯出 Jira 報表 CSV

```http
GET /api/v1/projects/:id/jira/export
```

Query 同 `/jira/report`。

成功：

```http
Content-Type: text/csv; charset=utf-8
Content-Disposition: attachment; filename="jira_report_2026-06-01_2026-06-30.csv"
```

失敗：

```http
Content-Type: application/json
```

不得在失敗時回傳 CSV。

## DTO Contract

### JiraImportSummaryResponse

```ts
interface JiraImportSummaryResponse {
  data: {
    total_rows: number;
    created_rows: number;
    skipped_rows: number;
    failed_rows: number;
    row_errors: JiraImportRowError[];
  };
}
```

### JiraImportRowError

```ts
interface JiraImportRowError {
  row_number: number;
  issue_key?: string;
  message: string;
}
```

欄位說明：

| 欄位 | 說明 |
| --- | --- |
| row_number | CSV 原始列號，第一列 header 不列入資料列統計 |
| issue_key | 該列可解析出的 Jira Issue Key；若 key 本身缺失，可省略 |
| message | 可顯示給使用者的錯誤訊息 |

### JiraReportQuery

```ts
interface JiraReportQuery {
  date_from: string;
  date_to: string;
  jira_project_key?: string;
  status?: string;
  priority?: string;
  assignee?: string;
}
```

Query 規則：

- `date_from`、`date_to` 必填，格式為 `YYYY-MM-DD`。
- `date_from` 不得晚於 `date_to`。
- 空字串與未提供的選填欄位一律視為不套用該 filter。
- 後端不得直接拼接 query string 到 SQL。

### JiraReportPayload

```ts
interface JiraReportPayload {
  data: {
    title: string;
    chart: {
      x_axis: {
        type: "category";
        labels: string[];
      };
      series: Array<{
        name: string;
        type: "bar";
        data: number[];
      }>;
    };
    table: {
      columns: Array<{
        field: string;
        header_name: string;
        type: "string" | "number" | "datetime";
      }>;
      rows: Array<{
        id: string;
        assignee: string;
        count: number;
      }>;
    };
    meta: {
      project_id: string;
      date_from: string;
      date_to: string;
      timezone: string;
      empty: boolean;
      generated_at: string;
    };
  };
}
```

無資料時：

- `chart.x_axis.labels` 回傳空陣列。
- `chart.series[0].data` 回傳空陣列。
- `table.rows` 回傳空陣列。
- `meta.empty` 回傳 `true`。
- 前端不得自行產生假 rows 或假 chart data。

### JiraReportCsvExport

成功時回傳 CSV stream：

```http
HTTP/1.1 200 OK
Content-Type: text/csv; charset=utf-8
Content-Disposition: attachment; filename="jira_report_2026-06-01_2026-06-30.csv"
```

錯誤時回傳 JSON error envelope：

```json
{
  "code": 403,
  "message": "沒有 Jira 報表匯出權限",
  "trace_id": "..."
}
```

後端必須在匯出 API 重新驗證權限。無權限回 `403`，查詢參數錯誤回 `400`，不得回傳 `text/csv`。

## 後端查詢設計

統計依 `assignee` 與 Jira 專案聚合，前端以 `assignee` 作為 X 軸，Jira 專案名稱作為堆疊 series。

```sql
SELECT
  COALESCE(NULLIF(TRIM(assignee), ''), '未指派') AS assignee,
  COALESCE(NULLIF(TRIM(project_name), ''), NULLIF(TRIM(project_key), ''), '未分類') AS jira_project_name,
  COUNT(*) AS count
FROM jira_issues
WHERE project_id = $1
  AND created_date >= $2
  AND created_date <= $3
  AND ($4 = '' OR project_key = $4)
  AND ($5 = '' OR status = $5)
  AND ($6 = '' OR priority = $6)
  AND ($7 = '' OR assignee = $7)
GROUP BY
  COALESCE(NULLIF(TRIM(assignee), ''), '未指派'),
  COALESCE(NULLIF(TRIM(project_name), ''), NULLIF(TRIM(project_key), ''), '未分類')
ORDER BY assignee ASC, jira_project_name ASC;
```

查詢必須使用參數化 SQL，不得拼接 query string。
報表 payload 需將相同 `assignee` 的不同 Jira 專案組成 stacked bar series；缺少某專案資料的 assignee 補 `0`，不得由前端自行推導假資料。

## 前端頁面設計

### `/projects/:id/jira`

頁面區塊：

```text
Jira 匯入與統計報表
├── 主專案摘要
├── 匯入 CSV
│   ├── 選擇檔案
│   ├── 匯入按鈕
│   └── 匯入結果摘要 / 錯誤列
├── 報表查詢條件
│   ├── 日期區間
│   ├── Jira 项目关键字
│   ├── 状态
│   ├── 优先级
│   └── 经办人
├── 長條圖
├── 明細表
└── 匯出 CSV
```

設計原則：

- 不顯示假資料。
- 尚未查詢時顯示空狀態。
- 匯入與報表查詢需用 toast 顯示錯誤。
- table / grid 需支援 i18n。
- 欄位寬度需適合中文與英文，避免重疊。
- 匯出 CSV 按鈕需依權限控制：有權限時顯示可點擊按鈕；無權限時隱藏或停用，且不得讓使用者誤以為可匯出。

## 權限設計

初始權限：

- `Project Viewer` 可查詢 Jira 報表與匯出。
- `Project Member` 可匯入 Jira CSV。
- `Project Manager` 可管理後續若新增的匯入歷史。

若選單權限已啟用，建議新增 form permissions：

| full_path | 權限 |
| --- | --- |
| `jira` | read |
| `jira/import` | create |
| `jira/report` | read |
| `jira/export` | read |

前端匯出按鈕需檢查 `jira/export` read 權限或等價後端回傳的 effective permission。後端仍需檢查專案可見性與角色，不得只依前端選單隱藏。

## OpenAPI

需補以下 paths：

- `POST /api/v1/projects/{id}/jira/import`
- `GET /api/v1/projects/{id}/jira/report`
- `GET /api/v1/projects/{id}/jira/export`

OpenAPI 必須包含：

- multipart requestBody
- report query parameters
- import summary response schema
- report payload response schema
- CSV success response
- JSON error response

## 測試策略

後端：

- CSV parser header 對應測試。
- UTF-8 BOM 測試。
- 日期與數字解析測試。
- 缺少 `问题关键字` / `创建日期` 測試。
- duplicate `project_id + issue_key + created_date` skip 測試。
- report 聚合測試。
- export CSV content type 測試。
- 權限測試。

前端：

- TypeScript typecheck。
- build。
- API error 顯示。
- 空資料狀態。
- i18n key 檢查。

## 與既有 Report 的關係

Jira Report 不併入 `ReportChartPayload` 的通用報表設計器主流程，原因：

- 資料來源是 `jira_issues`，不是 Opscenter `tickets`。
- 匯入與去重規則獨立。
- 舊需求明確要求 Jira 資料與系統內 Ticket 完全獨立。

若後續需要在 Report Center 顯示 Jira Report，可只做入口連結，不應讓通用 Report 查詢混讀 `jira_issues` 與 `tickets`。
