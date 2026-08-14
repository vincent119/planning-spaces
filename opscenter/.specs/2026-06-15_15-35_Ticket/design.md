# Ticket Spec Design

## 文件定位

本文件描述 Ticket bounded context 的整體設計與邊界。前端頁面細節放在 `design-frontend.md`，實作拆解放在 `task.md` 與 `task-frontend.md`。

## 與既有 spec 的關係

- 原 spec：`../2026-06-01_10-22_oncall-ticket-system`
- 本 spec 只處理 Ticket 工作區。
- Auth、global role、form permission、admin menu、排程、報表、Jira 匯入仍由原 spec 管理。
- 共用 API 契約以 `opscenter-server/Docs/openapi.json` 為準。

## Bounded Context

Ticket 工作區包含：

- Ticket list / create / detail / update
- Ticket status transition
- Ticket assignment / collaborators
- Ticket activities / comments
- Ticket attachments
- Project workspace context
- Sub Project management
- Project Member management

Ticket 工作區不包含：

- Admin Project 基本資料管理
- Admin Role / User / Menu management
- 排班 / 假勤編輯
- 報表設計器
- Jira 匯入流程

## Route Map

```text
/projects/:id/tickets             Ticket 列表
/projects/:id/tickets/new         建立 Ticket
/tickets/:id                      Ticket 詳情
/projects/:id/sub-projects        子專案管理
/projects/:id/members             專案成員管理
/projects/:id/sources             資訊來源管理
/projects/:id/ticket-types        事件類型管理
```

## API Dependency

目前 OpenAPI 已確認的 Project workspace API：

```text
GET    /api/v1/projects
POST   /api/v1/projects
GET    /api/v1/projects/{id}
PUT    /api/v1/projects/{id}
DELETE /api/v1/projects/{id}

GET    /api/v1/projects/{id}/sub-projects
POST   /api/v1/projects/{id}/sub-projects
GET    /api/v1/sub-projects/{sid}
PUT    /api/v1/sub-projects/{sid}
DELETE /api/v1/sub-projects/{sid}

GET    /api/v1/projects/{id}/members
POST   /api/v1/projects/{id}/members
PUT    /api/v1/projects/{id}/members/{uid}
DELETE /api/v1/projects/{id}/members/{uid}
GET    /api/v1/projects/{id}/user-options

GET    /api/v1/projects/{id}/ticket-resources
POST   /api/v1/projects/{id}/ticket-resources
GET    /api/v1/ticket-resources/{rid}
PUT    /api/v1/ticket-resources/{rid}
DELETE /api/v1/ticket-resources/{rid}
```

目前 OpenAPI 已確認的 Ticket API：

```text
GET/POST /api/v1/tickets
GET/PUT/DELETE /api/v1/tickets/{id}
GET /api/v1/projects/{id}/tickets
POST /api/v1/tickets/{id}/status
POST /api/v1/tickets/{id}/assign
POST /api/v1/tickets/{id}/collaborators
DELETE /api/v1/tickets/{id}/collaborators/{uid}
POST /api/v1/tickets/{id}/activities
GET/POST /api/v1/tickets/{id}/attachments
GET /api/v1/attachments/{id}/content
DELETE /api/v1/tickets/{id}/attachments/{aid}
GET /api/v1/ticket-metadata/options
GET/POST /api/v1/ticket-types
GET/PUT/DELETE /api/v1/ticket-types/{tid}
```

以上契約以 `opscenter-server/Docs/openapi.json` 為準。前端不得使用未列在 OpenAPI 或未由 server implementation 支援的 Ticket API。

## Project Member Contract

專案成員以原 spec `../2026-06-01_10-22_oncall-ticket-system/design-backend.md` 的 `project_members` 表為目標契約，不是 Admin 使用者管理表的前端投影。

契約規則：

- `project_members` 為專案成員與專案角色資料來源，核心欄位為 `project_id`、`user_id`、`role`、`joined_at`。
- `project_members.user_id` 指向 `ops_user(id)`；成員列表、Ticket 指派與協作者候選都必須以 `ops_user` 為資料來源，不得 join Admin `users` 顯示帳號管理資料。
- 前端專案成員列表不得顯示 Admin 使用者管理欄位，例如 `global_role`、MFA、phone、remark 或系統帳號狀態。
- 前端專案成員列表只顯示 project member contract 欄位：使用者識別資訊、專案角色、加入時間與操作。
- 新增專案成員不是挑既有候選人，而是在目前 project 下建立一位 `ops_user`，再建立 `project_members` 關聯。
- `POST /api/v1/projects/{id}/members` request body 只接受 `user_name` 與 `description`；`project_id` 來自 path，`ops_user.id` 由後端產生。
- `POST /api/v1/projects/{id}/members` 後端需以同一 transaction 寫入 `ops_user` 與 `project_members`，成功回傳 `MemberResponse`。
- `project_members.role` 若資料表仍必填，由後端使用內部預設值 `engineer`；前端不得提供角色下拉或要求使用者理解角色欄位。
- `ops_user.user_name` 必填、長度上限 128，只允許英文字母、數字、底線、點、短橫與 `@`；重複 `user_name` 或同專案重複加入需回 409。
- `ops_user.is_active` 為運維人員顯示狀態，新增成員預設 `true`。
- 移除專案成員為 soft delete 語意：`DELETE /api/v1/projects/{id}/members/{uid}` 將 `ops_user.is_active` 改為 `false`，不硬刪 `project_members` 或 `ops_user`。
- 專案成員列表、候選查詢與專案權限查詢需排除 `ops_user.is_active=false` 的資料；前端不得顯示停用成員。
- Ticket 指派、mention、協作者候選人仍只能使用 `GET /api/v1/projects/{id}/members` 的既有專案成員清單。
- `GET /api/v1/projects/{id}/user-options` 不作為新增專案成員主流程；若保留，只能供其他搜尋場景使用。
- 前端不得改用 `/api/v1/admin/users`，也不得用假資料模擬候選人。

## Project User Option Contract

`GET /api/v1/projects/{id}/user-options`

此 API 不作為新增專案成員主流程。新增專案成員應使用 `POST /api/v1/projects/{id}/members` 建立 `ops_user` 與 `project_members` 關聯。Ticket 指派、mention、協作者候選人應使用 `GET /api/v1/projects/{id}/members`，因為那些操作只能選擇既有專案成員。

Query filters：

| 參數 | 型別 | 說明 |
| --- | --- | --- |
| `keyword` | string | 搜尋 `username`、`email`、`full_name` |
| `limit` | int | 每頁筆數，預設 50，上限 100 |
| `offset` | int | 起始位移 |
| `exclude_members` | bool | `true` 時排除已在此 project 內的成員 |

Response `data`：

```json
{
  "items": [],
  "total": 0,
  "limit": 50,
  "offset": 0
}
```

`items[]` 欄位：

```text
id
username
email
full_name
```

契約規則：

- 候選來源為 `ops_user`，不得直接等同 Admin users table。
- 需要登入，且目前使用者需具備該專案 `project_manager` 權限；`admin` / `op_admin` 可依既有 project service 跨專案管理規則通過。
- `exclude_members=true` 時，已存在於 `project_members` 的使用者不得出現在結果中。
- 不回傳 `password_hash`、MFA 狀態、remark、phone 或其他 Admin 使用者管理欄位。
- 前端新增成員 Dialog 不使用此 API 搜尋候選人，也不得退回挑選既有候選人的舊流程。
- 前端不得使用 `/api/v1/admin/users` 作為 Ticket 工作區新增成員來源，因為該 API 需要 Admin 權限且語意屬於系統使用者管理。

## Ticket API Contract

所有 Ticket API 使用既有 `httpx.APIResponse` envelope。成功回應的有效資料在 `data`，錯誤回應包含 `code`、`message` 與 `trace_id`。前端只可依 `data` 內實際存在的欄位顯示資料，不得由 UI 假造顯示名稱、統計或狀態。

### Create Ticket

`POST /api/v1/tickets`

Request body：

```json
{
  "title": "支付域名切換異常",
  "description": "Markdown 原文描述",
  "ticket_type_id": "01K...",
  "priority": "p2",
  "project_id": "01K...",
  "sub_project_id": "01K...",
  "ticket_resource_id": "01K...",
  "external_ref": "JIRA-123",
  "assignee_id": "01K..."
}
```

欄位規則：

- `title`、`description`、`ticket_type_id`、`priority`、`project_id`、`sub_project_id`、`ticket_resource_id` 為必填。
- `external_ref`、`assignee_id` 為選填。
- `created_by` 由登入者決定，不接受 request body 指定。
- 成功後回傳 `DetailResponse`，包含 `ticket`、`activities`、`comments`、`attachments`。

### List Tickets

`GET /api/v1/tickets`

`GET /api/v1/projects/{id}/tickets`

Query filters：

| 參數 | 型別 | 說明 |
| --- | --- | --- |
| `project_id` | string | 僅全域列表使用，限制專案範圍 |
| `keyword` | string | 標題、描述或外部單號關鍵字 |
| `status` | string[] | 可重複帶入多個狀態 |
| `priority` | string[] | 可重複帶入多個優先級 |
| `sub_project_id` | string | 子專案 ID |
| `assignee_id` | string | 指派人 ID |
| `ticket_type_id` | string | 事件類型 ID |
| `ticket_resource_id` | string | 資訊來源 ID，對應 `ticket_resources.id` |
| `created_from` | string | 建立日期起點，格式 `YYYY-MM-DD` |
| `created_to` | string | 建立日期終點，格式 `YYYY-MM-DD` |
| `view` | string | 預設視圖，僅全域列表使用 |

日期區間契約：

- 前端 Ticket 列表進入頁面時預設查詢使用者目前時區的當天。
- 台灣使用者以 `Asia/Taipei` 日界線計算當天日期，再以 `YYYY-MM-DD` 傳給 `created_from` / `created_to`。
- `created_from` 與 `created_to` 用於查詢 `tickets.created_at`。
- 後端以 `Asia/Taipei` 解讀日期；`created_from` 包含當日 00:00:00，`created_to` 包含當日完整日期，實作上以小於隔日 00:00:00 查詢。
- 若使用者清空日期區間，前端不得帶 `created_from` / `created_to`，後端回傳不限制建立日期的結果。
- 後端需接受 `created_from <= created_to` 的查詢；若區間格式錯誤或起點晚於終點，需回 400，不得忽略錯誤後回傳不正確資料。

Pagination：

| 參數 | 型別 | 說明 |
| --- | --- | --- |
| `page` | int | 頁碼 |
| `page_size` | int | 每頁筆數 |
| `limit` | int | 每頁筆數，保留給 offset 模式 |
| `offset` | int | 起始位移 |

Sort：

| 參數 | 型別 | 說明 |
| --- | --- | --- |
| `sort_by` | string | 排序欄位，目前前端只可使用 OpenAPI / 後端已支援欄位 |
| `sort_direction` | string | `asc` 或 `desc` |

Response `data`：

```json
{
  "items": [],
  "total": 0,
  "page": 1,
  "page_size": 20,
  "limit": 20,
  "offset": 0
}
```

### Ticket Response

列表、詳情與操作成功時回傳的 Ticket 欄位：

```text
id
title
description
ticket_type_id
ticket_type_code
ticket_type_name
priority
status
project_id
project_key
project_name
sub_project_id
sub_project_key
sub_project_name
ticket_resource_id
ticket_resource_code
ticket_resource_name
ticket_resource_type
external_ref
assignee_id
assignee_username
assignee_full_name
created_by
creator_username
creator_full_name
resolved_at
closed_at
resolution_summary
cancel_reason
created_at
updated_at
```

顯示名稱欄位如 `project_name`、`ticket_type_name`、`assignee_full_name` 可能為空；前端需顯示空狀態或 fallback，不得自行假造名稱。

### Detail Ticket

`GET /api/v1/tickets/{id}`

Response `data`：

```json
{
  "ticket": {},
  "activities": [],
  "comments": [],
  "attachments": [],
  "collaborators": []
}
```

- `activities` 用於系統事件時間軸。
- `comments` 用於留言列表，留言內容保留 Markdown 原文。
- `attachments` 只回傳 metadata，不包含 `storage_key`。
- 留言若有圖片附件，該留言 response 需包含 `attachments` metadata 陣列；這些附件仍屬於同一 Ticket，且不包含 `storage_key`。
- `collaborators` 用於目前協作者清單，欄位包含 `ticket_id`、`user_id`、`username`、`email`、`full_name`、`added_at`。前端需以此欄位顯示與移除協作者，不得從 activity 推測目前協作者。

### Ticket Metadata Options

`GET /api/v1/ticket-metadata/options`

此 API 提供建立 Ticket 與列表篩選需要的全域選項。前端不得硬編事件類型、資訊來源或優先級選項。

Response `data`：

```json
{
  "ticket_types": [],
  "ticket_resources": [],
  "priorities": []
}
```

`ticket_types[]` 欄位：

```text
id
code
name
description
supports_escalation
applies_sla
allowed_priorities
default_priority
assignee_required
is_system
sort_order
```

`ticket_resources[]` 欄位：

```text
id
code
name
resource_type
description
sort_order
```

`priorities[]` 欄位：

```text
code
name
sort_order
```

契約規則：

- 後端只回傳啟用中的 `ticket_types` 與目前 project 可用的 `ticket_resources`。
- `priorities` 為全域優先級 contract，目前固定為 `P1`、`P2`、`P3`、`P4`。
- Ticket type 可用優先級以 `ticket_types[].allowed_priorities` 為準。
- Ticket type 預設優先級以 `ticket_types[].default_priority` 為準。
- 建立 Ticket 頁選取事件類型後，priority 下拉選項需依該 type 的 `allowed_priorities` 過濾，預設值使用 `default_priority`。
- API 需登入。事件類型與優先級為系統層級參照資料；資訊來源改為 project scoped，metadata options 必須依目前 project scope 回傳 `ticket_resources`。

### Ticket Resource Management Contract

`ticket_resources` 是 Ticket 系統唯一資訊來源資料表。

- `ticket_sources` 已判定為錯誤分叉設計，需由後續 migration / service task 移除主流程依賴。
- Ticket 建立、列表、詳情、metadata options 與資訊來源管理都必須收斂到 `ticket_resources`。
- `ticket_resources` 為 project scoped 資訊來源管理資料。

Routes：

```text
GET    /api/v1/projects/{id}/ticket-resources
POST   /api/v1/projects/{id}/ticket-resources
GET    /api/v1/ticket-resources/{rid}
PUT    /api/v1/ticket-resources/{rid}
DELETE /api/v1/ticket-resources/{rid}
```

Request body：

```json
{
  "code": "tg_alert",
  "name": "TG Alert",
  "resource_type": "alert",
  "description": "Telegram 告警來源",
  "sub_project_id": "01K...",
  "is_active": true
}
```

Schema 欄位：

```text
id
code
name
resource_type
description
project_id
sub_project_id
is_active
sort_order
created_at
updated_at
```

契約規則：

- List API 只回傳目前 project 下啟用中的 `ticket_resources`。
- `GET /api/v1/projects/{id}/ticket-resources` 需具備專案 Viewer 權限。
- Create / Update / Delete 需具備所屬專案 Manager 權限。
- `project_id` 來自 route 或既有資料，不接受 request body 覆寫。
- `code`、`name`、`resource_type` 為必填。
- `description` 為選填文字欄位。
- 不使用 `details JSONB`；資訊來源管理不再提供 key / value JSON 編輯器。
- `ticket_resources.sub_project_id` 依目前資料庫 schema 為 nullable 欄位，且不建立 `REFERENCES sub_projects(id)` 外鍵。
- `sub_project_id` 選填；未提供時代表資訊來源只掛在主專案，不指定子專案。
- 若 request 提供 `sub_project_id`，後端 service 必須檢查該子專案屬於目前 project，且子專案需為啟用狀態。
- `ticket_resources.is_active` 型別為 `BOOLEAN NOT NULL DEFAULT TRUE`。
- `ticket_resources.sort_order` 型別為 `INT NOT NULL DEFAULT 0`，用於下拉與列表排序。
- `ticket_resources.code` 在同一個 project 內需唯一。
- 既有資料在 migration 後視為啟用資料。
- 新增資訊來源預設 `is_active=true`。
- 刪除資訊來源採 soft delete，不硬刪資料，只更新 `is_active=false` 與 `updated_at`。
- 資訊來源列表與 Ticket 建立選項不得回傳 `is_active=false` 資料。
- 前端不得使用全域 `ticket_sources` 或舊 metadata 欄位假裝資訊來源 CRUD 已完成。
- 成功刪除回傳 `{ "deleted": true }`。
- 400 表示 request body 或欄位驗證失敗；401 表示未登入；403 表示權限不足；404 表示 project、sub project 或 resource 不存在；409 表示資料狀態衝突。

### Ticket Type Management Contract

`ticket_types` 用於事件類型管理，是系統層級參照資料；入口雖位於 Ticket 工作區，但資料不依 project scope 過濾。

Routes：

```text
GET    /api/v1/ticket-types
POST   /api/v1/ticket-types
GET    /api/v1/ticket-types/{tid}
PUT    /api/v1/ticket-types/{tid}
DELETE /api/v1/ticket-types/{tid}
```

Request body：

```json
{
  "code": "ops_event",
  "name": "維運事件",
  "description": "維運事件分類",
  "supports_escalation": true,
  "applies_sla": true,
  "allowed_priorities": ["P1", "P2", "P3"],
  "default_priority": "P2",
  "assignee_required": false,
  "is_active": true,
  "sort_order": 10
}
```

Response `data` 欄位：

```text
id
code
name
description
supports_escalation
applies_sla
allowed_priorities
default_priority
assignee_required
is_system
is_active
sort_order
created_at
updated_at
```

契約規則：

- List API 回傳全部 `ticket_types`，包含啟用、停用與系統內建狀態；前端可自行做列表篩選。
- `GET /api/v1/ticket-metadata/options` 仍只回傳啟用中的事件類型，供 Ticket 建立與列表篩選使用。
- 管理 API 需登入，且只有 `admin` / `op_admin` 可操作。
- Create API 建立自訂事件類型，`is_system` 由後端固定為 `false`，前端不得送出或假設可設定。
- Create / Update 若未提供 `is_active`，後端預設為 `true`。
- `code` 必填，允許小寫英文字母、數字、底線與短橫，長度上限 64。
- `name` 必填，長度上限 64。
- `allowed_priorities` 至少一個值，且只能包含 `P1`、`P2`、`P3`、`P4`。
- `default_priority` 必須包含在 `allowed_priorities`。
- `is_system=true` 的事件類型不可刪除。
- `is_system=true` 的事件類型不可修改核心流程欄位：`code`、`supports_escalation`、`applies_sla`、`allowed_priorities`、`default_priority`、`assignee_required`。
- `is_system=true` 的事件類型仍可更新顯示資訊與管理欄位：`name`、`description`、`is_active`、`sort_order`。
- Delete API 採 soft delete，將 `is_active=false`，成功回傳 `{ "deleted": true }`。
- 停用後不得出現在 Ticket 建立與列表篩選的 metadata options。
- 400 表示 request body 或欄位驗證失敗；401 表示未登入；403 表示權限不足；404 表示事件類型不存在；409 表示系統內建限制或狀態衝突。

### Update / Delete Ticket

`PUT /api/v1/tickets/{id}`

Request body 與 create 相同欄位，但全部為 optional pointer 語意；未送出的欄位不得被清空。

`DELETE /api/v1/tickets/{id}`

刪除權限：

- 僅允許 Ticket 建立者本人執行。
- 僅允許刪除狀態為 `open` 的 Ticket。
- 刪除仍採 soft delete，並寫入 `ticket_activities.action_type = deleted`。

成功回應：

```json
{
  "deleted": true
}
```

刪除策略由後端實作決定；前端只依成功 response 更新列表，不得直接從 UI 假設資料庫已硬刪。

### Status / Assign / Collaborators

`POST /api/v1/tickets/{id}/status`

```json
{
  "status": "resolved",
  "resolution_summary": "已修復",
  "cancel_reason": ""
}
```

成功回傳 `TicketResponse`。前端只應顯示後端允許的狀態操作；若狀態流轉 API 回傳 `409`，需顯示後端錯誤，不得在前端改判成功。

允許狀態流轉：

| 目前狀態 | 可切換狀態 |
| --- | --- |
| `open` | `in_progress`、`cancelled` |
| `in_progress` | `pending`、`escalated`、`resolved`、`cancelled` |
| `pending` | `in_progress`、`resolved` |
| `escalated` | `in_progress`、`pending`、`resolved`、`cancelled` |
| `resolved` | `closed`、`open` |
| `closed` | `open` |
| `cancelled` | 無 |

狀態流轉規則：

- `resolved` 需要 `resolution_summary`。
- `cancelled` 需要 `cancel_reason`。
- 切換到 `closed` 需要 Project Manager 或 Admin 權限。
- 其他狀態流轉需要專案 Engineer 以上權限。
- `escalated` 僅在後端確認 ticket type 支援 escalation 時允許。
- 成功流轉需寫入 `ticket_activities.action_type = status_changed`。

`POST /api/v1/tickets/{id}/assign`

```json
{
  "assignee_id": "01K..."
}
```

成功回傳 `TicketResponse`。

指派規則：

- `assignee_id` 必須是該 Ticket 所屬主專案的有效成員。
- 指派需要 Project Manager 或 Admin 權限。
- 已關閉 Ticket 不可指派。
- 成功指派需寫入 `ticket_activities.action_type = assigned`。

`POST /api/v1/tickets/{id}/collaborators`

```json
{
  "user_id": "01K..."
}
```

`DELETE /api/v1/tickets/{id}/collaborators/{uid}`

成功回應：

```json
{
  "updated": true
}
```

協作者規則：

- 新增協作者時，`user_id` 必須是該 Ticket 所屬主專案的有效成員。
- 新增與移除協作者需要 Project Manager 或 Admin 權限。
- 已關閉 Ticket 不可新增或移除協作者。
- 成功新增協作者需寫入 `ticket_activities.action_type = collaborator_added`。
- 成功移除協作者需寫入 `ticket_activities.action_type = collaborator_removed`。
- 新增已存在的協作者或移除不存在的協作者時，後端不重複寫入 activity；前端需依 API 成功後重新讀取詳情。
- 前端新增協作者候選人必須使用 `GET /api/v1/projects/{id}/members`；在後端詳情資料尚未提供目前協作者清單前，只能提供新增協作者，不得顯示或移除未確認的協作者。

### Ticket Activities / Comments

`POST /api/v1/tickets/{id}/activities`

目前此 endpoint 用於新增留言。

Request body：

```json
{
  "content": "Markdown 原文",
  "is_internal": false,
  "mentioned_user_ids": ["01K..."]
}
```

Response `data` 為 `ActivityResponse`：

```text
id
ticket_id
actor_id
actor_username
actor_full_name
action_type
field_changes
content
is_internal
mentioned_user_ids
attachments
created_at
```

留言規則：

- `content` 必填，後端會 trim 前後空白；空白內容回傳 `400`。
- `content` 以 Markdown 原文儲存，後端不做 HTML render；前端顯示 preview 或內容時必須自行處理安全渲染。
- `is_internal` 寫入 `ticket_activities.is_internal`，用於內部留言顯示權限。
- `mentioned_user_ids` optional；後端會正規化與去重，且所有提及使用者都必須是該 Ticket 所屬主專案的有效成員。
- 已關閉 Ticket 不可新增留言。
- 成功新增留言需寫入 `ticket_activities.action_type = comment_added`。
- 成功後回傳新建立的 `ActivityResponse`，前端需依回傳資料或重新讀取詳情更新留言列表。

留言附件規則：

- 留言圖片附件需要關聯到 `ticket_activities.id`，讓前端能在每則留言下方顯示該留言上傳的圖片。
- 後端 schema 建議在 `attachments` 新增 nullable `activity_id CHAR(26)`；一般 Ticket 附件為 `NULL`，留言附件填入 `comment_added` activity id。
- `attachments.activity_id` 不建立跨分區 foreign key，因 `ticket_activities` 為月份分區表；service / repository 需驗證 activity 屬於同一 Ticket 且 `action_type = comment_added`。
- Detail response 的 `comments[]` 每筆需包含 `attachments: MetadataResponse[]`，只包含未 soft delete 的附件。
- Ticket 全域 `attachments` 可繼續回傳全部未刪除附件；前端需避免在留言附件與附件總覽之間假造關聯。

### Attachments

`POST /api/v1/tickets/{id}/attachments`

- `multipart/form-data`
- 欄位名稱：`file`
- 僅支援圖片附件，限制依後端驗證為準。
- 前端點擊、拖曳、貼上截圖都必須使用同一個 upload contract。
- OpenAPI 需輸出 `requestBody.content["multipart/form-data"].schema.properties.file`，型別為 `string` / `binary`，且 `file` 為 required。
- 可選欄位：`activity_id`。提供時表示此附件關聯到指定留言 activity；後端需驗證 activity 屬於 path Ticket 且為留言。
- 成功回應 `200`，`data` 為 `MetadataResponse`；錯誤狀態至少包含 `400`、`401`、`403`、`404`、`409`、`422`、`500`。

`GET /api/v1/tickets/{id}/attachments`

Response `data` 為 `MetadataResponse[]`：

```text
id
ticket_id
filename
content_type
size_bytes
storage_backend
uploaded_by
activity_id
created_at
```

- OpenAPI response content 為 `application/json`。
- Metadata response 不得包含 `storage_key` 或任何 Local / S3 直接路徑。
- `activity_id` 為 nullable；一般 Ticket 附件為空，留言附件回傳留言 activity id。
- 錯誤狀態至少包含 `401`、`403`、`404`、`500`。

`GET /api/v1/attachments/{id}/content`

- 回傳附件 binary content。
- Browser 不直連 Local / S3，也不使用 storage key。
- 前端需以 authenticated fetch 取得 blob，並管理 object URL lifecycle。
- OpenAPI 成功 response content 為 `application/octet-stream`，schema 型別為 `string` / `binary`。
- HTTP response 需設定 `Content-Type`、`Cache-Control: private`、`Content-Disposition: inline`，可取得內容長度時設定 `Content-Length`。
- 錯誤狀態至少包含 `401`、`403`、`404`、`500`。

`DELETE /api/v1/tickets/{id}/attachments/{aid}`

成功回應：

```json
{
  "deleted": true
}
```

- OpenAPI 需包含 path parameters：`id` 為 Ticket ID，`aid` 為附件 ID。
- 錯誤狀態至少包含 `401`、`403`、`404`、`409`、`500`。

附件權限與狀態規則：

- 列表與內容串流需要該 Ticket 所屬主專案 Viewer 以上權限。
- 上傳附件需要該 Ticket 所屬主專案 Engineer 以上權限。
- 已關閉 Ticket 不可新增附件；service 會先檢查狀態，repository 也必須在交易內鎖定 Ticket 後再次檢查。
- 上傳成功需寫入 `ticket_activities.action_type = attachment_added`。
- 刪除附件需要該 Ticket 所屬主專案 Engineer 以上權限，且目前後端只允許附件上傳者本人刪除；Admin / Project Manager 不具備刪除他人附件的例外。
- 已關閉 Ticket 不可刪除附件；service 會先檢查狀態，repository 也必須在交易內鎖定附件與 Ticket 後再次檢查。
- 刪除 API 若 path 的 Ticket ID 與附件所屬 Ticket ID 不一致，需回傳 not found，不可刪除。
- 刪除成功採 soft delete，需寫入 `attachments.deleted_at`、`attachments.deleted_by`，並寫入 `ticket_activities.action_type = attachment_deleted`；metadata soft delete 成功後才刪除私有儲存後端物件。

### Error / Permission Contract

| HTTP | code | 使用情境 |
| --- | --- | --- |
| 400 | 400 | request body 格式錯誤、未知欄位、必要參數缺失 |
| 401 | 401 | 未登入或 current user 缺失 |
| 403 | 403 | global role、project role 或 form permission 不足 |
| 404 | 404 | Ticket、Project、Sub Project、Attachment 或參照資料不存在 |
| 409 | 409 | Ticket 狀態衝突，例如已關閉不可編輯、非法狀態流轉、附件數量限制 |
| 422 | 422 | 附件過大、圖片格式不支援、圖片內容驗證失敗 |
| 500 | 500 | 非預期伺服器錯誤 |

前端規則：

- 只有 2xx 且 API 回傳成功時才顯示成功 Toast。
- 401 交由 auth flow 處理。
- 403 顯示權限不足，不得改用假資料繼續操作。
- 404 顯示找不到資源或返回列表。
- 409 顯示狀態衝突並重新讀取詳情。
- 422 顯示欄位或附件驗證錯誤。
- 500 顯示一般錯誤並保留既有資料，不得顯示服務正常。

## 權限邊界

- 建立 Ticket 需具備 `ticket/create` 的 `create` form permission。
- 專案內 Ticket 操作需同時符合 project role 與 form permission。
- `Admin` 可略過 form permission，但仍需遵守前端資料契約，不得顯示假資料。
- Project Manager 可管理所屬專案 Ticket、成員與設定，可關閉 Ticket。
- Engineer 可建立、編輯 Ticket 與留言。
- Viewer 僅可查看 Ticket。

## 狀態機

狀態機沿用原 spec；前端只顯示後端允許的狀態操作。

```text
Open -> In Progress -> Resolved -> Closed
Open -> Cancelled
In Progress -> Escalated
Escalated -> Resolved
Resolved -> Open
Closed -> Open
```

若後端實際狀態機與此不同，以 OpenAPI / server implementation 為準，本文件需同步修正。

## 資料真實性規則

- 不得在前端假造 Ticket、sub-project、member、attachment 資料。
- 後端未回傳欄位不得在 UI 顯示為真實資料。
- 所有成功 Toast 必須在 API 成功後顯示。
- 所有 Data Grid 需套用共用 `getDataGridLocaleText`。
