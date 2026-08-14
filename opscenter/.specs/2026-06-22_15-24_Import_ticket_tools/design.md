# Import Ticket Tools 設計

## 設計目標

Import Ticket Tools 用於將既有 Excel 開單紀錄匯入 Opscenter Ticket 模組。設計重點是 CLI 先預覽、再驗證、最後以 `-commit` 提交，避免 Excel 欄位差異或資料不完整造成錯誤 Ticket。

初始匯入來源為 `~/Downloads/202606-Jira開單紀錄表.xlsx`。匯入分頁採動態偵測，接受 `開單記錄-MMDD` 與 `開單紀錄-MMDD`，例如 `開單記錄-0601`、`開單紀錄-011`。

本次檔案中符合範例範圍的分頁為：

- `開單記錄-061`
- `開單記錄-068`
- `開單記錄-0615`
- `開單記錄-0622`

使用者原始描述的 `開單紀錄-061 to 0622` 在檔案中不是實際分頁名稱，因此實作時不得用該字串作為固定分頁名稱。後續實作必須依兩種支援的命名規則掃描活頁簿分頁，並讓使用者選擇要匯入的分頁。

## 系統位置

建立後端 CLI：

- `opscenter-server/cmd/import_ticket_data`

初始版本不建立前端 UI，不新增 Tickets 選單入口。

若後續需要服務化或 UI 化，再從 CLI 邏輯抽出 `internal/ticketimport` 共用模組。

## CLI 使用設計

預覽：

```bash
go run ./cmd/import_ticket_data \
  -file ~/Downloads/202606-Jira開單紀錄表.xlsx \
  -project-key MS \
  -actor-username admin
```

實際匯入：

```bash
go run ./cmd/import_ticket_data \
  -file ~/Downloads/202606-Jira開單紀錄表.xlsx \
  -project-key MS \
  -actor-username admin \
  -commit
```

提交並明確開啟自動建立主資料與保存來源檔：

```bash
go run ./cmd/import_ticket_data \
  -file ~/Downloads/202606-Jira開單紀錄表.xlsx \
  -project-key MS \
  -actor-username admin \
  -auto-create-sub-projects=true \
  -auto-create-resources=true \
  -save-source-file=true \
  -commit
```

指定分頁：

```bash
go run ./cmd/import_ticket_data \
  -file ~/Downloads/202606-Jira開單紀錄表.xlsx \
  -project-key MS \
  -sheets 開單記錄-0601,開單記錄-0622
```

## 匯入流程

```text
讀取 Excel
  -> CLI 讀取分頁與表頭
  -> 建立記憶體預覽結果
  -> 逐列正規化與驗證
  -> CLI 顯示預覽統計與錯誤明細
  -> 使用 -commit 確認提交
  -> 寫入 Ticket
  -> 輸出匯入結果
```

## API 設計

初始版本不提供 HTTP API。以下 API 契約暫不實作，保留作為後續 UI 化時的延伸設計。

### 建立預覽

```http
POST /api/v1/projects/:project_id/ticket-imports/preview
Content-Type: multipart/form-data
```

表單欄位：

| 欄位 | 型別 | 必填 | 說明 |
| --- | --- | --- | --- |
| file | file | 是 | `.xlsx` 檔案 |
| sheet_names | string array | 否 | 指定分頁；未提供時自動使用符合 `開單記錄-MMDD` 或 `開單紀錄-MMDD` 的分頁 |
| default_ticket_type | string | 否 | 預設事件類型代碼，預設 `daily` |
| default_priority | string | 否 | 預設優先級；未提供時使用事件類型預設 |
| sub_project_required | boolean | 否 | 子專案找不到時是否阻擋匯入 |
| auto_create_sub_projects | boolean | 否 | 找不到子專案時是否自動建立，預設 true |
| auto_create_resources | boolean | 否 | 找不到資訊來源時是否自動建立，預設 true |
| resource_type | string | 否 | 自動建立資訊來源時使用的 type，預設 message |
| save_source_file | boolean | 否 | 提交時是否保存原始 Excel 檔，預設 true |

回應：

```json
{
  "data": {
    "preview_id": "01...",
    "file_name": "202606-Jira開單紀錄表.xlsx",
    "sheets": ["開單記錄-0601", "開單記錄-0622"],
    "summary": {
      "total_rows": 100,
      "valid_rows": 96,
      "warning_rows": 2,
      "error_rows": 2,
      "duplicate_rows": 0
    },
    "rows": [
      {
        "row_id": "01...",
        "sheet_name": "開單記錄-0601",
        "row_number": 2,
        "action": "create",
        "status": "valid",
        "external_ref": "QIEZ-1658",
        "title": "定時更換果醬支付域名-0900",
        "will_create_sub_project": false,
        "will_create_resource": false,
        "errors": [],
        "warnings": []
      }
    ]
  }
}
```

### 提交匯入

```http
POST /api/v1/projects/:project_id/ticket-imports/:preview_id/commit
Content-Type: application/json
```

Request：

```json
{
  "mode": "insert_only",
  "skip_invalid": false
}
```

模式：

| mode | 行為 |
| --- | --- |
| insert_only | 已存在的 Jira 單號略過 |
| upsert | 已存在的 Jira 單號更新可更新欄位 |

Response：

```json
{
  "data": {
    "import_id": "01...",
    "status": "completed",
    "created_count": 90,
    "updated_count": 0,
    "skipped_count": 8,
    "failed_count": 2
  }
}
```

### 查詢匯入批次

```http
GET /api/v1/projects/:project_id/ticket-imports/:import_id
```

用途：

- 重新查看匯入結果。
- 追蹤每列錯誤。
- 支援後續非同步匯入。

### 查詢匯入歷史

```http
GET /api/v1/projects/:project_id/ticket-imports
```

支援參數：

| 參數 | 說明 |
| --- | --- |
| date_from | 匯入建立時間起 |
| date_to | 匯入建立時間迄 |
| status | 批次狀態 |
| page | 分頁頁碼 |
| page_size | 每頁筆數 |

## 資料表設計

初始版本不新增任何資料表。

匯入工具只使用既有資料表：

- `projects`
- `sub_projects`
- `ticket_types`
- `ticket_resources`
- `tickets`
- `ticket_activities`
- storage/attachment 既有資料表或檔案儲存設定，用於保存原始 Excel 檔
- `users`

去重使用既有 `tickets.external_ref` 欄位。若同一主專案已存在相同 `external_ref` 且未軟刪除，CLI 預設略過該列，不新增重複 Ticket。

## 欄位正規化設計

| Excel 欄位 | 正規化欄位 | 說明 |
| --- | --- | --- |
| 日期 | opened_at | Excel serial date 轉換為日期 |
| 已知問題 | title | 去除前後空白 |
| 資訊來源 | ticket_resource_id | 依名稱或代碼對應 `ticket_resources` |
| 處理方式 | description | 與備註合併，不建立獨立活動紀錄 |
| Project | project_id | 存在時用於校驗，不存在時使用 URL 主專案 |
| 填寫Jira項目選擇 | sub_project_id | 依子專案名稱或代碼對應；留白時寫入 NULL |
| Jira單號 | external_ref | 有值時用於去重；留白時為空 |
| 填寫人 | created_by / imported_reporter_text | 不對應 `ops_user`；需對應既有 `users` 後寫入 `tickets.created_by`，並保留為描述中的外部文字 |
| 目前進度 | status | 依狀態對應表轉換 |
| 備註 | description | 與處理方式合併，不建立獨立活動紀錄 |

Ticket 列表與詳情查詢使用 `LEFT JOIN sub_projects`，並將子專案欄位的 NULL 正規化為空字串，確保未關聯子專案的 Ticket 仍會回傳。

描述合併格式建議：

```text
處理方式：
{處理方式}

備註：
{備註}

匯入資訊：
填寫人：{填寫人}
來源分頁：{sheet_name}
Excel 列號：{row_number}
Jira 單號：{external_ref}
```

`填寫人` 不查詢也不建立 `ops_user`。若欄位有值，需查詢既有 `users`：

- `users.username = 填寫人` 或 `users.full_name = 填寫人`。
- 若 `填寫人` 帶有 `早-`、`中-`、`晚-` 等班別前綴，先保留原文到描述，再以去除前綴後的人名查詢 `users`。
- 僅接受 `users.is_active = TRUE`。
- 找不到使用者時列為 row error，不寫入該列。
- 若欄位為空，才使用 `-actor-username` 查到的 `users.id` 作為 fallback 建立者。

提交時 `tickets.created_by` 與初始 `ticket_activities.actor_id` 使用該列解析出的建立者，避免整批資料都落到 CLI 操作者。

## 日期解析

Excel `日期` 欄目前為 serial date，例如 `46174`。後端必須依 Excel 日期序列轉換為日期。

設計規則：

- 優先依 Excel serial date 解析。
- 若欄位為文字，支援 `YYYY-MM-DD`、`YYYY/MM/DD`。
- 解析失敗時列為阻擋錯誤。
- 時區使用系統設定時區，預設 `Asia/Taipei`。

## 來源資料差異處理

符合 `開單記錄-MMDD` 或 `開單紀錄-MMDD` 的分頁可能有 `Project` 欄位，也可能沒有。

本次檢查到 `開單記錄-061` 與 `開單記錄-068` 有 `Project` 欄位，`開單記錄-0615` 與 `開單記錄-0622` 沒有 `Project` 欄位。這是來源檔差異，不應成為固定規則。

設計規則：

- 如果 Excel 有 `Project` 欄位，必須與 URL 的 `project_id` 對應主專案一致。
- 如果 Excel 沒有 `Project` 欄位，使用 URL 的 `project_id`。
- 如果 Excel `Project` 與 URL 主專案不一致，預覽列為阻擋錯誤。

## 後端服務設計

建議分層：

- `delivery`：處理 multipart、JSON request、權限。
- `service`：管理預覽、驗證、提交。
- `parser`：讀取 xlsx 與分頁。
- `mapper`：欄位正規化與主資料對應。
- `repository`：儲存匯入批次、匯入列、寫入 Ticket。

### Parser

Parser 職責：

- 只讀取 `.xlsx`。
- 不執行公式或巨集。
- 支援指定分頁。
- 未指定分頁時，掃描並選取符合 `^開單記錄-\d{4}$` 或相容舊格式 `^開單記錄-\d{3,4}$` 的分頁。
- 自動偵測第一列為表頭。
- 忽略完全空白列。
- 回傳 `sheet_name`、`row_number`、`raw_data`。

### Mapper

Mapper 職責：

- 將 Excel 欄位名稱映射到系統欄位。
- 正規化日期、狀態、資訊來源、子專案。
- 依設定判斷缺少的子專案與資訊來源是否可自動建立。
- 建立錯誤與警告清單。
- 判斷列動作為 create、update 或 skip。

### Auto Create Master Data

匯入工具允許在提交時自動建立缺少的子專案與資訊來源。

預覽階段不得直接寫入主資料，而是標示：

- `will_create_sub_project=true`
- `will_create_resource=true`

提交階段依列資料建立主資料，且必須先再次查詢，避免 dry-run 到 commit 期間其他程序已建立相同資料。

#### 自動建立子專案

使用 Excel `填寫Jira項目選擇` 建立 `sub_projects`：

- `project_id`：目前主專案。
- `key`：由 Excel 值正規化產生，轉大寫，只保留英數、底線與連字號，並符合後端 check constraint。
- `name`：Excel 原始顯示值。
- `description`：空字串。
- `is_active`：`true`。

若正規化後 `key` 不合法或與既有資料衝突，該列提交失敗並輸出明確錯誤。

#### 自動建立資訊來源

使用 Excel `資訊來源` 建立 `ticket_resources`：

- `project_id`：目前主專案。
- `sub_project_id`：若列資料已對應子專案且來源非主專案共用，則帶入；否則可為空。
- `type`：CLI `-resource-type`，預設 `message`。
- `name`：Excel 原始顯示值。
- `description`：可為空，若 schema 沒有此欄位則不寫入。
- `is_active`：`true`。

若後端 `ticket_resources` schema 與上述欄位不同，實作必須以實際 schema 為準，但需求語意仍是建立可在 UI 顯示且可被 Ticket 引用的資訊來源。

### Commit

Commit 職責：

- 使用記憶體中的正規化資料寫入 Ticket。
- 依 `external_ref` 檢查重複資料。
- 依設定建立缺少的子專案與資訊來源。
- 保存原始 Excel 檔。
- 初始版本只支援新增與略過，不做 upsert。
- 寫入 Ticket 活動紀錄。
- 輸出匯入統計。

## 交易設計

預設提交行為：

- `skip_invalid=false` 時，只要有阻擋錯誤，不允許提交。
- `skip_invalid=true` 時，只寫入 valid rows，invalid rows 標記為 skipped。

寫入 Ticket 時逐列使用交易處理 `tickets` 與 `ticket_activities`，單列失敗不影響其他列，並在 CLI 輸出 failed 明細。

初始實作同步完成，不建立非同步 job。

保存原始 Excel 檔只在 `-commit` 時執行。若保存原始檔失敗，預設視為阻擋錯誤，不應在無來源檔可追溯的情況下繼續匯入。若後續需要允許忽略保存失敗，必須新增明確參數。

## 權限控制

後端 API 必須檢查：

- 使用者已登入。
- 使用者可存取 `project_id`。
- 使用者具備 Ticket 建立權限。

若後續選單權限支援更細的操作碼，可新增 `ticket_import:create` 或等價權限；在此之前，不得只依前端 UI 判斷權限。

## 稽核設計

初始版本不新增稽核寫入。CLI 會輸出操作人、主專案、檔案名稱、分頁名稱、總列數、新增筆數、略過筆數與失敗筆數。

若後續需要在系統內保存匯入歷史或稽核紀錄，需另開需求，不混入本次 CLI 匯入。

## CLI 輸出設計

CLI 預覽輸出：

```text
import ticket data preview
file=202606-Jira開單紀錄表.xlsx
sheets=4
rows=120
valid=118
skipped=1
errors=1
dry_run=true
auto_create_sub_projects=true
auto_create_resources=true
save_source_file=true
```

欄位：

| 欄位 | 說明 |
| --- | --- |
| 分頁 | Excel 分頁 |
| 列號 | Excel 列號 |
| Jira 單號 | `external_ref` |
| 標題 | Ticket 標題 |
| 資訊來源 | 對應結果 |
| 子專案 | 對應結果 |
| 狀態 | 正規化狀態 |
| 動作 | 新增、更新、略過 |
| 驗證 | valid、warning、error |
| 將建立子專案 | 找不到子專案且提交時會自動建立 |
| 將建立資訊來源 | 找不到資訊來源且提交時會自動建立 |
| 訊息 | 錯誤或警告摘要 |

### 錯誤呈現

錯誤分成三層：

- 檔案層級：檔案格式、大小、分頁不存在。
- 欄位層級：必填欄位缺少、表頭不符。
- 列層級：日期錯誤、主資料對應失敗、重複資料。

列層級錯誤必須可在 Data Grid 展開查看完整訊息。

## OpenAPI 契約

實作後必須補強 OpenAPI：

- `POST /api/v1/projects/{project_id}/ticket-imports/preview`
- `POST /api/v1/projects/{project_id}/ticket-imports/{preview_id}/commit`
- `GET /api/v1/projects/{project_id}/ticket-imports`
- `GET /api/v1/projects/{project_id}/ticket-imports/{import_id}`

OpenAPI 必須明確描述 requestBody、response schema、錯誤格式，不得只產生空泛 path。

## 已確認決策

- 需要保存原始 Excel 檔。
- 允許匯入時自動建立缺少的子專案。
- 允許匯入時自動建立缺少的資訊來源。
- `填寫人` 不對應 `ops_user`，但需對應既有 `users` 後寫入 `tickets.created_by`；同時保留為外部文字。
- `處理方式` 與 `備註` 合併到描述，不拆成獨立活動紀錄。
- 不需要支援 Excel 欄位映射模板，只支援固定欄位名稱與已知欄位差異。
