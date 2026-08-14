# Ticket Spec Requirements

## 文件定位

此 spec 接續 `2026-06-01_10-22_oncall-ticket-system`，專門收斂 Ticket 工作區、Ticket CRUD、活動紀錄、附件、子專案管理與專案成員管理。

既有 Auth、Admin、選單權限、排程、報表、全域設定仍以原 spec 為準。本 spec 不重寫那些模組，只在 Ticket 工作區需要引用時標註依賴。

## 目前已知契約狀態

- 需求來源：原 spec `requirements.md` 需求 3、需求 4、需求 7、需求 8 與附件相關補充。
- 後端設計來源：原 spec `design-backend.md` Ticket 章節。
- OpenAPI 來源：`opscenter-server/Docs/openapi.json`。
- 目前 `Docs/openapi.json` 已有 Project、Sub Project、Project Member、Ticket CRUD、Ticket Activity、Ticket Attachment API paths。
- 目前 `Docs/openapi.json` 已有 `GET /api/v1/ticket-metadata/options`，但資訊來源 contract 需由 `ticket_sources` 收斂為 project scoped `ticket_resources`。
- 目前 `Docs/openapi.json` 已有 `ticket_resources` 管理 API paths：`GET/POST /api/v1/projects/{id}/ticket-resources`、`GET/PUT/DELETE /api/v1/ticket-resources/{rid}`。
- 目前 `Docs/openapi.json` 已有 `ticket_types` 管理 API paths：`GET/POST /api/v1/ticket-types`、`GET/PUT/DELETE /api/v1/ticket-types/{tid}`。
- 目前 `Docs/openapi.json` 已有 `GET /api/v1/projects/{id}/user-options`，但專案成員需以原 spec `design-backend.md` 的 `project_members` 表為目標契約；候選來源與 response schema 需由後端 task 0.6 重新補強，不得直接等同 Admin 使用者管理資料。
- 目前 server 已存在 `internal/ticket` 與 `internal/attachment` route / service implementation。
- 目前 `Docs/openapi.json` 已列出 Ticket paths、query / path parameters、JSON requestBody schema 與 response DTO schema；前端仍需依本 spec 的 `design.md` contract 與 server DTO 檢查語意，不得自行假造欄位。
- 前端不得假造未出現在 OpenAPI、`design.md` contract 或後端實作中的 API；API 失敗不得顯示成功。

## 需求 1：Ticket 工作區資訊架構

使用者需要在專案脈絡中管理 Ticket，而不是在 Admin 專案管理頁中處理子專案或成員。

### 驗收條件

- [ ] 1.1 Ticket 工作區入口包含 `/projects/:id/tickets`、`/projects/:id/tickets/new`、`/tickets/:id`。
- [ ] 1.2 子專案管理放在 Ticket / 專案工作區，不放在 `/admin/projects`。
- [ ] 1.3 專案成員管理放在 Ticket / 專案工作區，不放在 `/admin/projects`。
- [ ] 1.4 Ticket 工作區需顯示目前專案脈絡，讓使用者知道自己正在操作哪個 project。
- [ ] 1.5 Ticket 工作區不得顯示後端未回傳的假欄位或假統計。

## 需求 2：Ticket 列表

使用者需要快速搜尋、篩選與掃描專案內 Ticket。

### 驗收條件

- [ ] 2.1 列表需以 Data Grid 顯示 Ticket，並支援 i18n 內建文案。
- [ ] 2.2 欄位至少包含標題、狀態、優先級、事件類型、子專案、資訊來源、指派人、建立人、更新時間、操作。
- [ ] 2.3 篩選至少包含狀態、優先級、事件類型、子專案、資訊來源、指派人、關鍵字與建立日期區間。
- [ ] 2.4 列表需串真實 API；若 OpenAPI 尚未提供 Ticket list contract，前端 task 必須先標記為 blocked 或依後端補契約後再實作。
- [ ] 2.5 空狀態需清楚顯示沒有 Ticket，並依權限顯示建立按鈕。
- [ ] 2.6 Ticket 列表建立日期區間預設為使用者目前時區的當天，送出查詢時轉成 `created_from` / `created_to` 傳給後端，不得只在前端本地過濾。
- [ ] 2.7 Ticket 列表操作欄需提供「複製建立」入口，點擊後不得直接建立資料，而是導向建立 Ticket 頁並預填可複製欄位，等待操作員確認後送出。
- [ ] 2.8 Ticket 列表操作欄需提供刪除入口；刪除按鈕只對 Ticket 建立者本人且狀態為 `open` 的 Ticket 顯示，且需二次確認後才呼叫刪除 API。

## 需求 3：建立與編輯 Ticket

值班工程師需要快速建立 Ticket 並填寫必要資訊。

### 驗收條件

- [ ] 3.1 建立 Ticket 需檢查 `ticket/create` 的 `create` 權限；沒有權限不得顯示假成功。
- [ ] 3.2 必填欄位包含標題、描述、事件類型、優先級、專案、子專案、資訊來源。
- [ ] 3.3 外部單號為選填自由文字。
- [ ] 3.4 Ticket 建立後狀態為 `Open`，不提供草稿狀態。
- [ ] 3.5 已關閉 Ticket 不可編輯。
- [ ] 3.6 建立與更新成功後需重新讀取列表或詳情資料。
- [ ] 3.7 事件類型、資訊來源與優先級選項需由真實 API 取得，不得在前端硬編假選項；資訊來源必須使用目前 project scope 的 `ticket_resources`。
- [ ] 3.8 從列表複製建立 Ticket 時，建立頁需預填標題、事件類型、優先級、子專案、資訊來源、指派人；描述欄只在原 Ticket 有描述內容時才帶入原描述，不得以列資訊摘要填入描述欄；不得複製附件、貼圖、storage key 或直接建立 Ticket。
- [ ] 3.9 刪除 Ticket 採 soft delete，前後端都只能允許 Ticket 建立者本人刪除狀態為 `open` 的 Ticket；刪除成功後需重新讀取列表資料，不得顯示已刪除 Ticket。

## 需求 4：Ticket 詳情

使用者需要在單一頁面看到 Ticket 內容、狀態、活動歷程、留言與附件。

### 驗收條件

- [ ] 4.1 詳情頁路由為 `/tickets/:id`。
- [ ] 4.2 詳情頁需顯示 Ticket 基本欄位、狀態、優先級、事件類型、子專案、資訊來源、指派人、協作者、建立時間、更新時間。
- [ ] 4.3 詳情頁需顯示活動紀錄時間軸。
- [ ] 4.4 詳情頁需支援新增留言；留言內容以 Markdown 原文儲存。
- [ ] 4.5 留言輸入需使用 MarkdownEditor，支援編輯 / 預覽切換、錯誤提示與送出中狀態。
- [ ] 4.6 Markdown 顯示需避免 XSS，使用安全 renderer 或純文字 fallback，不得直接注入未消毒 HTML。
- [ ] 4.7 詳情頁需支援附件列表與圖片預覽，附件內容透過系統 API 取得，不暴露儲存路徑。
- [ ] 4.8 API 失敗時需顯示錯誤狀態，不得顯示假資料。
- [ ] 4.9 留言需支援圖片附件；使用者在留言區拖曳或貼上圖片並送出後，該圖片需顯示在對應留言下方，不得只出現在全域附件列表。
- [ ] 4.10 留言圖片附件仍需透過系統附件 API 取得與預覽，不得暴露 storage key、S3 URL 或本機路徑。

## 需求 5：狀態流轉、指派與協作

使用者需要依權限更新 Ticket 狀態與負責人。

### 驗收條件

- [ ] 5.1 狀態流轉需依後端允許狀態顯示操作，不得在前端硬編未確認的轉換。
- [ ] 5.2 指派負責人需串後端真實 API。
- [ ] 5.3 協作者新增與移除需串後端真實 API。
- [ ] 5.4 關閉 Ticket 需限制 Project Manager / Admin。
- [ ] 5.5 所有操作成功後需重新讀取詳情與活動紀錄。
- [ ] 5.6 指派、mention 與協作者候選人需使用 `GET /api/v1/projects/:id/members` 的專案成員清單，不得搜尋或顯示非專案成員。

## 需求 6：子專案與專案成員

Ticket 工作區需要管理子專案與專案成員，供 Ticket 分類與權限使用。

### 驗收條件

- [ ] 6.1 子專案列表與 CRUD 使用 `GET/POST /api/v1/projects/:id/sub-projects`、`GET/PUT/DELETE /api/v1/sub-projects/:sid`。
- [ ] 6.2 專案成員列表與 CRUD 使用 `GET/POST /api/v1/projects/:id/members`、`PUT/DELETE /api/v1/projects/:id/members/:uid`。
- [ ] 6.3 專案成員 role 使用 server 既有值：`project_manager`、`engineer`、`viewer`。
- [ ] 6.4 不得在前端偽造 `owner`、`is_active`、`sub_project_count` 等後端未回傳欄位。
- [ ] 6.5 專案成員使用原 spec `design-backend.md` 定義的 `project_members` 表，欄位以 `project_id`、`user_id`、`role`、`joined_at` 為核心；不是 Admin 使用者管理表的前端投影。
- [ ] 6.6 新增專案成員候選來源需由後端重新定義；不得直接使用 `/api/v1/admin/users`，也不得顯示 `global_role`、MFA、phone、remark 或系統帳號狀態等 Admin 使用者管理欄位。

## 需求 6A：事件類型與資訊來源管理

Ticket 工作區需要管理建立 Ticket 時使用的事件類型與資訊來源，讓專案管理者可以維護分類資料。

### 驗收條件

- [ ] 6A.1 事件類型頁路由為 `/projects/:id/ticket-types`，顯示 `ticket_types` 資料。
- [ ] 6A.2 事件類型欄位至少包含代碼、名稱、啟用狀態、是否系統內建、是否支援升級、是否套用 SLA、預設優先級、允許優先級、是否必填指派人、排序、更新時間與操作。
- [ ] 6A.3 系統內建事件類型不得刪除；不可修改會破壞流程語意的核心欄位。
- [ ] 6A.4 資訊來源頁路由為 `/projects/:id/sources`，顯示 `ticket_resources` 資料。
- [ ] 6A.5 資訊來源欄位至少包含代碼、名稱、類型、說明、所屬子專案、啟用狀態、排序、更新時間與操作。
- [ ] 6A.6 資訊來源需依目前主專案 scope 讀取，不得顯示其他主專案資料。
- [ ] 6A.7 前端需使用 OpenAPI 已提供的 `ticket_types` 與 `ticket_resources` 管理 API；不得使用 metadata options API 假裝 CRUD 成功。
- [ ] 6A.8 `ticket_sources` 為錯誤分叉設計，需廢棄；Ticket 建立、列表、詳情、metadata options 與資訊來源管理都必須收斂為 `ticket_resources`。
- [ ] 6A.9 `ticket_resources` 需新增 `is_active BOOLEAN NOT NULL DEFAULT TRUE`，新增資訊來源預設啟用。
- [ ] 6A.10 刪除資訊來源採 soft delete，將 `ticket_resources.is_active` 改為 `false`；列表與 Ticket 建立選項不得顯示停用資料。
- [ ] 6A.11 `ticket_resources.sub_project_id` 為選填欄位，資料庫不建立 `REFERENCES sub_projects(id)` 外鍵；若 API request 提供子專案 ID，後端需以 service 驗證其屬於目前主專案且啟用。
- [ ] 6A.12 `ticket_resources` 不使用 `details JSONB`；欄位需收斂為 `code`、`name`、`resource_type`、`description`、`project_id`、`sub_project_id`、`is_active`、`sort_order`、`created_at`、`updated_at`。
- [ ] 6A.13 `資訊來源` 與 `事件類型` 都需要獨立頁面，不得只放在 Ticket 建立或列表下拉選項中。

## 需求 7：附件

Ticket 附件需安全上傳、轉檔與存取。

### 驗收條件

- [ ] 7.1 圖片附件單檔上限 10MB，每張 Ticket 最多 20 個附件。
- [ ] 7.2 前端只透過系統 API 顯示附件，不使用 Local / S3 直接 URL。
- [ ] 7.3 附件上傳、刪除與預覽需顯示 loading / error 狀態。
- [ ] 7.4 已關閉 Ticket 不可新增或刪除附件。
- [ ] 7.5 AttachmentUpload 需支援點擊選擇檔案、拖曳上傳與貼上截圖。
- [ ] 7.6 拖曳與貼上流程需共用同一套檔案驗證，不得繞過 content-type、大小與數量限制。
- [ ] 7.7 附件需可選擇關聯到留言 activity；關聯到留言的附件需同時保留 Ticket scope，並在該留言下方顯示。
- [ ] 7.8 刪除留言附件時仍採附件 soft delete；刪除後該留言下方不得再顯示該附件。
