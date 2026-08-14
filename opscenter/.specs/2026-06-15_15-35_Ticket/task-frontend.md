# Ticket Frontend Tasks

## 文件定位

本文件只追蹤 Ticket 工作區前端工作。前端不得假造未出現在 OpenAPI 或 server implementation 的 API。

## 0. 前置契約

- [x] 0.1 讀取 OpenAPI 並建立 Ticket API 缺口清單
  - 檢查 `opscenter-server/Docs/openapi.json`
  - 若缺少 Ticket CRUD / Activity / Attachment paths，標記 Ticket CRUD UI task blocked
  - Project / Sub Project / Project Member API 可先使用現有 OpenAPI
  - 完成：已確認 Ticket CRUD / Activity / Attachment paths 已存在，並在 `design-frontend.md` 更新 API 缺口清單
  - 補充：後端 task `0.4` 已完成，metadata options 可使用 `GET /api/v1/ticket-metadata/options`
  - 補充：後端 task `0.5` 已完成，新增專案成員候選人可使用 `GET /api/v1/projects/{id}/user-options`
  - _Design: API 缺口_

- [x] 0.2 建立 Ticket feature 目錄骨架
  - `src/features/ticket/api`
  - `src/features/ticket/pages`
  - `src/features/ticket/components`
  - `src/features/ticket/types`
  - 不建立假 API client
  - 完成：已確認 `api`、`components`、`types` 既有骨架存在，並新增可追蹤的 `pages/.gitkeep`
  - _Design: 文件定位_

- [x] 0.3 建立 Ticket i18n namespace
  - `ticket.json` 補繁中 / 簡中 / 英文
  - 共用文案使用 `common`
  - 完成：已確認 `ticket` namespace 已掛入 `i18n.ts`，並補齊三語 Ticket 工作區、列表、建立、詳情、子專案、成員、附件、狀態與驗證文案
  - 完成：三語 `ticket.json` 鍵集合一致，共 245 個 leaf keys
  - _Design: i18n_

## 1. Project Workspace Shell

- [x] 1.1 建立 Project Workspace layout
  - 顯示 project name / key / status / visibility
  - tabs 或 navigation：Tickets / 子專案 / 成員
  - 支援 breadcrumbs
  - 串 `GET /api/v1/projects/:id`
  - 完成：已新增 `ProjectWorkspaceLayout`、workspace project API / hooks，並將既有 `/projects/:id/tickets` route 接上工作區外框
  - 完成：layout 使用真實 `GET /api/v1/projects/:id` 與 `GET /api/v1/projects`，失敗時顯示錯誤狀態，不顯示假專案
  - _Design: 專案工作區骨架_

- [x] 1.2 建立 workspace route
  - `/projects/:id/tickets`
  - `/projects/:id/sub-projects`
  - `/projects/:id/members`
  - 完成：已建立 `ProjectTicketsPage`、`ProjectSubProjectsPage`、`ProjectMembersPage`，三個 route 皆掛載同一個 `ProjectWorkspaceLayout`
  - 完成：此 task 不串接子專案、成員或 Ticket list 資料，避免提前完成後續 `2.x`、`3.x`、`4.x`
  - _Design: 路由_

- [x] 1.3 修正 `Tickets` 側欄 fallback 入口
  - `Tickets` 側欄入口在沒有 currentProject 時會導向 `/projects`
  - `/projects` 不得使用 placeholder shell，也不得新增不在設計中的獨立專案列表頁
  - 串 `GET /api/v1/projects`
  - 有可存取專案時直接導向第一個專案的 `/projects/:id/tickets`
  - API 失敗或沒有可存取專案時顯示錯誤 / 空狀態，不得顯示假專案
  - 完成：已新增 `ProjectsEntryRedirectPage` 並替換 `/projects` index route 的 placeholder
  - 完成：fallback route 使用真實 `GET /api/v1/projects`，有資料即導向 `/projects/:id/tickets`
  - 完成：Ticket 列表進入後避免 Data Grid pagination model 以相同值重複 setState，並避免工作區 layout 重複寫入相同 currentProject
  - _Design: 路由_

- [x] 1.4 調整 Ticket 工作區導航結構
  - `Tickets` 側欄項目改為父選單
  - 子表單包含 `Ticket 列表`、`子專案`、`專案成員`
  - 移除 `ProjectWorkspaceLayout` 頁首 tabs
  - 主專案下拉仍保留，切換主專案時維持目前子表單 route
  - 完成：已在 `AppSidebar` 建立 Tickets 子選單，子表單分別導向 `/projects/:id/tickets`、`/projects/:id/sub-projects`、`/projects/:id/members`
  - 完成：已移除工作區頁首 tabs，避免子專案與專案成員在頁內 tabs 呈現
  - 完成：已補 common 三語側欄子表單文案
  - _Design: 路由 / 專案工作區骨架_

- [x] 1.5 調整主專案下拉位置
  - Ticket 列表、子專案、專案成員共用 `ProjectWorkspaceLayout`
  - 主專案下拉選單需移到 status chip 後方
  - 桌面版下拉選單與「啟用」狀態 chip 間距固定為 20px
  - 窄螢幕可換行，避免下拉選單擠壓專案名稱與狀態
  - 完成：已將 `ProjectWorkspaceLayout` 的主專案下拉移至狀態 chip 後方，桌面版以 8px flex gap 加 12px left margin 對齊 20px 間距
  - _Design: 專案工作區骨架_

- [x] 1.6 調整 Tickets 側欄子選單順序
  - 子選單預設 seed 順序需為：子專案、專案成員、資訊來源、事件類型、Ticket 列表
  - 補充：本 task 完成當時排序由前端 `AppSidebar` 固定陣列控制；後續依 Menu spec 改為由 `form_nodes.sort_order` 決定最終顯示順序，不回頭修改本已完成 task 狀態
  - 資訊來源與事件類型目前先建立側欄入口與待實作 route，避免點擊後 route not found
  - 完成：已調整 `ticketChildItems` 順序，新增資訊來源與事件類型側欄項目、三語文案與 placeholder route
  - 對齊：`.kiro/specs/2026-06-18_17-45_Menu` 已接管側欄可見性與排序需求
  - _Design: 路由_

## 2. 子專案管理

- [x] 2.1 建立子專案 API client
  - `GET /api/v1/projects/:id/sub-projects`
  - `POST /api/v1/projects/:id/sub-projects`
  - `GET /api/v1/sub-projects/:sid`
  - `PUT /api/v1/sub-projects/:sid`
  - `DELETE /api/v1/sub-projects/:sid`
  - response shape 需依 OpenAPI，不得假造欄位
  - 完成：已新增 `features/ticket/api/subProjects.ts`，型別與 request / response shape 依 `Docs/openapi.json`
  - _Design: 子專案管理頁_

- [x] 2.2 建立子專案列表頁
  - 搜尋 key / 名稱
  - 狀態篩選
  - Data Grid i18n
  - 空狀態
  - 完成：已在 `/projects/:id/sub-projects` 串接 `GET /api/v1/projects/:id/sub-projects`，以 Data Grid 顯示子專案代碼、名稱、描述、狀態與更新時間
  - 完成：搜尋與狀態篩選為本地篩選，因目前後端 list sub-projects contract 未提供 query filters
  - 完成：本 task 未加入新增、編輯、刪除操作，保留給 `2.3`、`2.4`
  - _Design: 子專案管理頁_

- [x] 2.3 建立子專案新增 / 編輯 Dialog
  - 欄位：key、name、description、is_active
  - 成功後 invalidate sub-project query
  - API 失敗不得顯示成功
  - 完成：已在 `/projects/:id/sub-projects` 加入新增與編輯 Dialog，欄位包含 key、name、description、is_active
  - 修正：子專案代碼驗證與後端契約對齊為 2 到 8 個英文字母或數字，儲存時仍轉為大寫，避免 `test22` 被前端錯誤擋下
  - 完成：新增串 `POST /api/v1/projects/:id/sub-projects`，編輯串 `PUT /api/v1/sub-projects/:sid`，成功後 invalidate sub-project query
  - 完成：API 失敗只顯示錯誤 Toast，不會顯示成功訊息；刪除流程保留給 `2.4`
  - 驗證：`ticket.json` 三語鍵集合一致；`npm run typecheck` 通過
  - _Design: 子專案管理頁_

- [x] 2.4 建立子專案刪除流程
  - 二次確認 Dialog
  - 後端拒絕時顯示錯誤
  - 完成：已在子專案列表操作欄加入刪除按鈕，點擊後顯示二次確認 Dialog
  - 完成：確認刪除串 `DELETE /api/v1/sub-projects/:sid`，成功後 invalidate sub-project query
  - 完成：後端拒絕或 API 失敗時只顯示錯誤 Toast，不會從前端假造刪除成功
  - 驗證：`ticket.json` 三語鍵集合一致；`npm run typecheck` 通過
  - _Design: 子專案管理頁_

- [x] 2.5 修正子專案空列表 API 回傳防護
  - `GET /api/v1/projects/:id/sub-projects` 若回傳空資料、`data: null` 或缺少 `data`，前端需正規化為空陣列，避免 React Query 顯示 `data is undefined`
  - 子專案頁 Data Grid pagination model 需避免相同值重複 setState
  - 完成：`listSubProjects` 已保證回傳 `SubProject[]`，空資料統一為 `[]`
  - 完成：子專案列表分頁狀態加入相同值保護
  - 驗證：`npm run typecheck`
  - _Design: 子專案管理頁_

## 3. 專案成員管理

- [x] 3.1 建立專案成員 API client
  - `GET /api/v1/projects/:id/members`
  - `GET /api/v1/projects/:id/user-options`
  - `POST /api/v1/projects/:id/members`
  - `PUT /api/v1/projects/:id/members/:uid`
  - `DELETE /api/v1/projects/:id/members/:uid`
  - 完成：已新增 `features/ticket/api/members.ts`，型別依 `Docs/openapi.json` 與後端 `project-members` DTO
  - 完成：`user-options` 支援 keyword、limit、offset、exclude_members query 參數
  - 完成：新增 / 更新 / 刪除只回傳後端 envelope data，不假造成員資料
  - 驗證：`npm run typecheck` 通過
  - _Design: 專案成員管理頁_

- [x] 3.2 建立專案成員列表頁
  - 搜尋使用者
  - Data Grid i18n
  - 完成：已在 `/projects/:id/members` 串接 `GET /api/v1/projects/:id/members`，以 Data Grid 顯示使用者名稱、真實姓名、信箱與加入時間
  - 修正：早期版本曾顯示全域角色與狀態，已由 `3.5` 移除；專案成員不得當成 Admin 用戶管理表的前端投影
  - 完成：搜尋為本地篩選，因目前後端 list members contract 未提供 query filters
  - 完成：本 task 未加入新增、編輯或移除操作，保留給 `3.3`、`3.4`
  - 驗證：`ticket.json` 三語鍵集合一致；`npm run typecheck` 通過
  - _Design: 專案成員管理頁_

- [x] 3.3 建立新增 / 編輯成員 Dialog
  - 成員搜尋需串 `GET /api/v1/projects/:id/user-options`
  - 新增成員候選人查詢預設帶 `exclude_members=true`
  - 不得使用 `/api/v1/admin/users` 或假資料作為候選人來源
  - 完成：已在 `/projects/:id/members` 加入新增成員 Dialog，候選人來源只串 `GET /api/v1/projects/:id/user-options?exclude_members=true`
  - 完成：新增串 `POST /api/v1/projects/:id/members`
  - 完成：成功後 invalidate members query；API 失敗只顯示錯誤 Toast，不會顯示成功
  - 完成：本 task 未加入移除成員流程，保留給 `3.4`
  - 驗證：`ticket.json` 三語鍵集合一致；`npm run typecheck` 通過
  - _Design: 專案成員管理頁_

- [x] 3.4 建立移除成員流程
  - 二次確認 Dialog
  - 移除後 invalidate members query
  - 完成：已在專案成員列表操作欄加入移除按鈕，點擊後顯示二次確認 Dialog
  - 完成：確認移除串 `DELETE /api/v1/projects/:id/members/:uid`，成功後 invalidate members query
  - 完成：後端拒絕或 API 失敗時只顯示錯誤 Toast，不會從前端假造移除成功
  - 驗證：`ticket.json` 三語鍵集合一致；`npm run typecheck` 通過
  - _Design: 專案成員管理頁_

- [x] 3.5 移除專案成員頁 Admin 用戶管理欄位
  - 專案成員以原 spec `design-backend.md` 的 `project_members` 表為準，不得在列表顯示 `global_role` 或系統帳號狀態
  - 完成：已從 `/projects/:id/members` Data Grid 移除全域角色與用戶帳號狀態欄位
  - 完成：前端 Project Member 型別不再宣告 `global_role`、`is_active` 為專案成員顯示欄位
  - 完成：此 task 修正 `3.2` 早期顯示全域角色與狀態的設計落差；不代表後端 Project Member contract 已完成
  - _Requirements: 6.5, 6.6_
  - _Design: 專案成員管理頁_

- [x] 3.6 對齊獨立專案成員新增 contract
  - 等待後端 task `0.6` 補強 Project Member contract，並對齊原 spec `project_members` / `ops_user` 目標契約
  - 依新 contract 調整新增成員 Dialog 的候選來源與 request body
  - 不得改用 `/api/v1/admin/users`
  - 不得使用假候選人或在 API 失敗時顯示成功
  - 完成：`ProjectUserOption` 型別已依後端 `0.6` / OpenAPI 收斂為 `id`、`username`、`email`、`full_name`
  - 補充：後端 `0.7` 已確認候選來源為 `ops_user`，前端 email 型別可為空，無 email 時顯示 `-`
  - 完成：新增成員 Dialog 仍只串 `GET /api/v1/projects/:id/user-options?exclude_members=true`；目前後端 request body 仍要求 `role`，前端只送出隱藏預設值，不顯示角色下拉
  - 完成：Ticket namespace 移除 `member.global_role`，候選人相關文案改為成員候選人，避免被解讀為 Admin 用戶管理
  - 驗證：三語 `ticket.json` 鍵集合一致；`npm run typecheck` 通過
  - _Requirements: 6.5, 6.6_
  - _Design: Project Member Contract_

- [x] 3.7 移除專案成員頁角色選擇 UI
  - 後端成員候選來源為 `ops_user`，`ops_user` 沒有角色欄位，專案成員頁不得顯示角色篩選、角色欄位或角色下拉選單
  - 新增成員 Dialog 只顯示成員候選人搜尋，不提供角色選擇
  - 列表操作只保留移除成員，不提供編輯角色入口
  - 完成：已從 `/projects/:id/members` 移除角色篩選、專案角色欄與角色編輯 Dialog
  - 完成：`design-frontend.md` 已同步移除專案成員角色下拉與角色欄規劃
  - 限制：新增成員流程仍是舊的候選人搜尋模式，需由 `3.8` 改為建立專案維運人員
  - 驗證：`npm run typecheck`
  - _Design: 專案成員管理頁_

- [x] 3.8 改造新增專案維運人員 Dialog
  - 等待後端 task `0.9` 補強新增專案維運人員 contract，並等待後端 task `0.10` 完成交易流程實作
  - 新增 Dialog 改為表單輸入 `user_name`、`description`，不再使用候選人搜尋
  - 送出時呼叫目前主專案的新增成員 API，由後端同 transaction 建立 `ops_user` 與 `project_members`
  - 不顯示角色下拉、Email、MFA、phone、global_role、啟用狀態或 Admin 用戶欄位
  - 成功後 invalidate members query；失敗只顯示後端錯誤，不得從前端假造成功
  - 完成：`ProjectMembersPage` 新增 Dialog 已改為 `使用者名稱`、`描述` 表單，不再呼叫 `user-options`
  - 完成：`addProjectMember` payload 改為 `user_name`、`description`，成功 response 使用後端回傳的 `ProjectMember`
  - 完成：已補三語 i18n 與前端 `user_name` 必填、長度、格式驗證
  - 驗證：`npm run typecheck`
  - _Design: 專案成員管理頁_

- [x] 3.9 修正無專案成員時顯示錯誤
  - 問題：專案沒有啟用成員時，Ticket 列表與專案成員頁不應顯示「專案成員載入失敗」
  - `GET /api/v1/projects/:id/members` 若回傳空資料、`data: null` 或缺少 `data`，前端需正規化為空陣列
  - 不得用假資料填入指派人或成員清單
  - 完成：`listProjectMembers` 已保證回傳 `ProjectMember[]`，空資料統一為 `[]`
  - 驗證：`npm run typecheck`
  - _Design: 專案成員管理頁 / Ticket 列表_

- [x] 3.10 移除專案成員列表未支援欄位
  - 問題：`ops_user` 目前沒有真實姓名與 Email 欄位，專案成員列表不應顯示真實姓名或信箱欄
  - 專案成員列表只顯示使用者名稱、加入時間與操作
  - 搜尋只搜尋畫面可見的使用者名稱
  - 完成：`ProjectMembersPage` 已移除真實姓名與信箱欄位
  - 完成：`design-frontend.md` 已同步修正列表設計
  - 驗證：`npm run typecheck`
  - _Design: 專案成員管理頁_

- [x] 3.11 移除新增專案維運人員描述欄位
  - 問題：專案成員列表與 `ops_user` 目前 UI 不顯示描述，新增 Dialog 不應提供不可見欄位
  - 新增專案維運人員 Dialog 只保留 `使用者名稱`
  - 送出 payload 只包含 `user_name`
  - 完成：`ProjectMembersPage` 已移除描述輸入欄位
  - 完成：`CreateProjectMemberPayload` 已移除必填 `description`
  - 完成：三語 `ticket.json` 移除 `member.description` 文案
  - 修正：後端新增專案維運人員遇到 `ops_user.user_name` 唯一鍵衝突時改回專用 409 message `project member username already exists`，不再顯示 generic `resource conflict`
  - 修正：前端已將 `project member username already exists` 映射到三語 i18n 文案，不直接顯示後端英文訊息
  - 驗證：`npm run typecheck`
  - _Design: 專案成員管理頁_

## 3A. 資訊來源與事件類型管理

- [x] 3A.1 補強資訊來源與事件類型管理 API contract
  - 資訊來源頁依使用者指定對應 `ticket_resources`
  - 事件類型頁對應 `ticket_types`
  - 目前 `GET /api/v1/ticket-metadata/options` 只可供選項讀取，不可作為 CRUD 管理 API
  - 後端 OpenAPI paths 與 DTO 已由 server task 4.2 / 4.3 補齊，可進行前端串接
  - 資訊來源 contract 已確認需廢棄 `ticket_sources`，收斂為 `ticket_resources`
  - `ticket_resources` 需新增 `is_active`，新增預設 `true`，刪除改為 soft delete
  - 資訊來源 list / options 不得回傳或顯示 `is_active=false` 資料
  - 完成：已在 `requirements.md` 補目前 OpenAPI 契約狀態與 6A.7 / 6A.8 更新
  - 完成：已在 `design.md` 補 `Ticket Resource Management Contract` 與 `Ticket Type Management Contract`
  - 完成：已在 `design-frontend.md` 將 API 缺口改為已確認 API contract，明確列出 route、權限、soft delete 與欄位規則
  - _Design: 資訊來源管理頁 / 事件類型管理頁_

- [x] 3A.2 建立資訊來源列表頁 shell
  - 路由：`/projects/:id/sources`
  - 顯示目前主專案脈絡
  - 搜尋名稱 / 類型、子專案篩選、類型篩選
  - Data Grid 舊版欄位：名稱、類型、子專案、狀態、細節摘要、更新時間、操作
  - 預設只顯示 `is_active=true` 資訊來源
  - API 失敗不得顯示假資料
  - 完成：已新增 `ProjectTicketSourcesPage`，route `/projects/:id/sources` 改接真實列表頁
  - 完成：已新增 `ticketResources` API client，列表使用 `GET /api/v1/projects/{id}/ticket-resources`
  - 完成：頁面支援搜尋名稱 / 類型、子專案篩選、類型篩選與 Data Grid i18n；新增 / 編輯 / 刪除仍留給 3A.3 / 3A.4，不做假操作
  - 修正：`sub_project_id` 依資料庫 schema 為選填，前端列表可顯示未指定子專案資料
  - _Depends on: 3A.1_

- [x] 3A.3 建立資訊來源新增 / 編輯 Dialog 設計實作
  - 舊版欄位：名稱、類型、子專案、details key / value 列表
  - 舊版 details key 不可重複，空 key 不送出
  - 新增 / 編輯成功後重新讀取列表
  - API 失敗不得顯示成功
  - 完成：`ProjectTicketSourcesPage` 已新增舊版資訊來源 Form Dialog，包含名稱、類型、子專案、啟用狀態與 details key / value 編輯
  - 完成：新增串 `POST /api/v1/projects/{id}/ticket-resources`，編輯串 `PUT /api/v1/ticket-resources/{rid}`
  - 完成：舊版表單送出會驗證必填欄位與 details key 不可重複，空 key 不送出；成功後重新讀取資訊來源列表
  - 注意：`details` 已由 3A.4a 標記為待移除，後續 UI 需改一般欄位
  - 修正：子專案欄位改為選填，Dialog 提供「不指定子專案」選項；API client 會將 `sub_project_id=null` 正規化為空值
  - _Depends on: 3A.1_

- [x] 3A.4 建立資訊來源刪除流程
  - 使用確認 Dialog
  - 刪除需呼叫後端 soft delete，將 `ticket_resources.is_active=false`
  - 成功後重新讀取列表，UI 不顯示停用資料
  - 後端拒絕刪除時保留資料並顯示錯誤
  - 不得以前端本地假刪除取代 API 結果
  - 完成：`ProjectTicketSourcesPage` row action 已開啟刪除確認 Dialog
  - 完成：確認後呼叫 `DELETE /api/v1/ticket-resources/{rid}`，成功後重新讀取列表
  - 完成：刪除失敗顯示後端錯誤並保留資料，不做本地假刪除
  - _Depends on: 3A.1_

- [x] 3A.4a 重構資訊來源管理 UI 欄位
  - `ticket_resources` 表單移除 `details` key / value 編輯器
  - 新增 / 編輯 Dialog 欄位改為：代碼、名稱、類型、說明、子專案、啟用、排序
  - 列表欄位改為：代碼、名稱、類型、說明、子專案、狀態、排序、更新時間、操作
  - API payload 改為 `code`、`name`、`resource_type`、`description`、`sub_project_id`、`is_active`、`sort_order`
  - API 失敗不得顯示成功，不得用舊 `details` 假資料填補
  - 完成：`ticketResources` API client 已改用 `code`、`resource_type`、`description`、`sort_order` 型別與 payload，不再送出 `type` / `details`
  - 完成：`ProjectTicketSourcesPage` 列表、搜尋、篩選、Dialog 與驗證已改為正式欄位
  - 完成：三語 `ticket.json` 已補資訊來源代碼、說明、排序與驗證文案，移除來源頁舊 `details` 文案依賴
  - 驗證：`npm run typecheck`
  - _Depends on: server 4.5_

- [x] 3A.4b Ticket 建立 / 列表 / 詳情改用 `ticket_resource_id`
  - 建立 Ticket 表單資訊來源欄位由 `ticket_source_id` 改為 `ticket_resource_id`
  - Ticket 列表 filter 由 `ticket_source_id` 改為 `ticket_resource_id`
  - Ticket row/detail 顯示改用 `ticket_resource_name` / `ticket_resource_code` / `ticket_resource_type`
  - 資訊來源選項必須依目前主專案讀取 project scoped `ticket_resources`
  - 移除前端對 metadata options `ticket_sources` 的依賴
  - 完成：`features/ticket/api/tickets.ts` 已改為 `ticket_resource_id` request / query / response contract，metadata options 改為 `ticket_resources` 並帶入 `project_id`
  - 完成：建立 Ticket 頁送出 `ticket_resource_id`，來源下拉改讀 `ticket_resources`
  - 完成：Ticket 列表 filter 改送 `ticket_resource_id`，row 顯示改讀 `ticket_resource_name` / `ticket_resource_code`
  - 完成：Ticket 詳情 subtitle 與側欄資訊來源改讀 `ticket_resource_name`
  - 修正：當使用者停留在已不存在的 project URL 時，`ProjectWorkspaceLayout` 會在 project 404 且有可用專案時導向同分頁的第一個可用專案，避免持續對不存在 project 發送請求
  - 驗證：`npm run typecheck`
  - _Depends on: server 4.4_

- [x] 3A.5 建立事件類型列表頁 shell
  - 路由：`/projects/:id/ticket-types`
  - 顯示目前主專案脈絡，但資料為系統層級事件類型
  - 搜尋代碼 / 名稱、狀態篩選、系統內建篩選
  - Data Grid 欄位：代碼、名稱、狀態、系統內建、升級、SLA、預設優先級、允許優先級、指派必填、排序、操作
  - API 失敗不得顯示假資料
  - 完成：已新增 `ticketTypes` API client，列表使用 `GET /api/v1/ticket-types`
  - 完成：已新增 `ProjectTicketTypesPage`，route `/projects/:id/ticket-types` 改接真實列表頁
  - 完成：頁面支援搜尋代碼 / 名稱、狀態篩選、系統內建篩選與 Data Grid i18n；新增 / 編輯 / 刪除仍留給 3A.6 / 3A.7，不做假操作
  - 修正：事件類型 workspace breadcrumb 正確顯示「事件類型」，切換主專案時維持 `/ticket-types` 路由
  - 修正：事件類型 API client 與 Data Grid 對既有資料缺值做防護，代碼 / 名稱缺值時顯示 `-`，不顯示空白欄
  - _Depends on: 3A.1_

- [x] 3A.6 建立事件類型新增 / 編輯 Dialog 設計實作
  - 欄位：代碼、名稱、描述、啟用、支援升級、套用 SLA、允許優先級、預設優先級、指派必填、排序
  - 預設優先級必須包含在允許優先級中
  - 系統內建事件類型核心欄位 disabled
  - 新增 / 編輯成功後重新讀取列表
  - API 失敗不得顯示成功
  - 完成：事件類型列表頁已開啟新增 / 編輯 Dialog，送出時呼叫 `POST /api/v1/ticket-types` 或 `PUT /api/v1/ticket-types/{tid}`
  - 完成：前端驗證代碼、名稱、允許優先級與預設優先級；系統內建類型核心流程欄位鎖定，只允許更新顯示與管理欄位
  - 完成：成功後重讀列表並顯示成功通知；API 失敗顯示後端錯誤，不做假成功
  - _Depends on: 3A.1_

- [x] 3A.7 建立事件類型停用 / 刪除流程
  - 系統內建事件類型不顯示刪除按鈕
  - 非系統內建事件類型刪除需二次確認
  - 後端拒絕刪除時保留資料並顯示錯誤
  - 停用成功後不得出現在建立 Ticket 與列表篩選選項
  - 完成：事件類型列表中 `is_system=true` 不渲染刪除按鈕，非系統內建類型顯示刪除動作
  - 完成：刪除前顯示確認 Dialog，確認後呼叫 `DELETE /api/v1/ticket-types/{tid}`
  - 完成：成功後重讀事件類型列表並顯示成功通知；API 失敗顯示後端錯誤，不做本地假刪除
  - 修正：事件類型 API client 會將後端既有資料的 `allowed_priorities=null` 正規化為空陣列，避免列表或編輯 Dialog 崩潰；送出時仍依 contract 驗證至少一個允許優先級
  - 修正：後端 `ListTicketTypes` repository 改為顯式掃描欄位，避免事件類型列表只顯示 boolean 欄位而 `code` / `name` / `default_priority` 變成空值；已補 repository 測試
  - 修正：事件類型前端 query key 已改版並在 rows 過濾缺 id / code / name 的舊快取資料，刪除前也會檢查 id，避免送出 `/ticket-types/` 造成 route not found
  - _Depends on: 3A.1_

## 4. Ticket 列表

- [x] 4.1 建立 Ticket list API client
  - OpenAPI 已列出 list paths、query parameters 與 `ListResponse` schema
  - response type 需依 OpenAPI schema 與 `design.md` contract，不得假造欄位
  - 事件類型、資訊來源與 priority 篩選選項使用 `GET /api/v1/ticket-metadata/options`
  - 完成：已新增 `features/ticket/api/tickets.ts`，包含 `listTickets`、`listProjectTickets` 與 `getTicketMetadataOptions`
  - 完成：Ticket list params 支援 keyword、status、priority、sub_project_id、assignee_id、ticket_type_id、ticket_resource_id、created_from、created_to、sort、pagination 等 OpenAPI 既有參數
  - 修正：3A.4b 已將資訊來源 filter 從 `ticket_source_id` 收斂為 `ticket_resource_id`
  - 完成：Ticket response、ListResponse 與 metadata options 型別依 `Docs/openapi.json` 與 `design.md` contract 建立，不新增 UI 假造欄位
  - 完成：本 task 不建立 create / detail / update client，保留給 `5.1`、`6.1` 與後續操作 task
  - 驗證：`npm run typecheck` 通過
  - _Depends on: 0.1_

- [x] 4.2 建立 Ticket 列表頁 shell
  - 可先建立無資料 shell
  - 不顯示假 Ticket
  - Data Grid 欄位依 `design-frontend.md`
  - 完成：已在 `/projects/:id/tickets` 建立 Ticket 列表頁 shell，包含搜尋列、篩選控制外觀、刷新按鈕與新增 Ticket 入口
  - 完成：Data Grid 欄位依 `design-frontend.md` 建立 ID、標題、狀態、優先級、類型、子專案、資訊來源、指派人、更新時間與操作欄
  - 完成：本 task rows 固定為空陣列，不呼叫 list API、不顯示假 Ticket；真實資料、篩選選項與 pagination 保留給 `4.3`
  - 驗證：`ticket.json` 三語鍵集合一致；`npm run typecheck` 通過
  - _Design: Ticket 列表頁_

- [x] 4.3 串接 Ticket 列表資料
  - keyword / status / priority / type / sub-project / assignee filters
  - pagination
  - loading / error / empty states
  - 完成：已在 `/projects/:id/tickets` 串接 `GET /api/v1/projects/:id/tickets`，使用目前 route project id，不顯示假 Ticket
  - 完成：keyword、status、priority、type、sub-project、source、assignee filters 會帶入後端既有 query parameters
  - 完成：priority / type / source 選項串 `GET /api/v1/ticket-metadata/options`，sub-project 與 assignee 選項依目前主專案重新讀取
  - 完成：Data Grid 使用 server pagination；切換主專案時不沿用上一個專案的 Ticket rows
  - 完成：已補 loading、API error 與 filtered empty states
  - 驗證：`ticket.json` 三語鍵集合一致；`npm run typecheck` 通過
  - _Blocked by: 4.1_

- [x] 4.4 優化 Ticket 列表篩選列自適應寬度
  - 問題：狀態、優先級、事件類型、子專案、資訊來源、指派人下拉與操作按鈕在寬度不足時擠在同一列，導致欄位文字重疊
  - 篩選列需支援 flex wrap，搜尋欄、下拉欄位與操作按鈕各自有合理最小寬度
  - 操作按鈕可換行，不得遮住或壓縮下拉選單
  - 完成：Ticket 列表篩選列已改為可換行 flex 版型，搜尋欄與下拉欄位使用自適應寬度，按鈕組在窄寬度時換行
  - _Design: Ticket 列表頁_

- [x] 4.5 新增 Ticket 列表建立日期區間篩選
  - 搜尋列在關鍵字後方新增日期起、日期迄欄位
  - 進入 `/projects/:id/tickets` 時，日期區間預設為使用者目前時區的當天
  - 預設日期區間需以 `YYYY-MM-DD` 帶入 `created_from` / `created_to` 並執行初次列表 API 查詢
  - 日期起點不可晚於日期迄點；驗證失敗不得送出 API
  - 清空日期區間時不得帶 `created_from` / `created_to`
  - 搜尋、日期或任一篩選變更送出查詢時，pagination 重設為第一頁
  - 三語 i18n 需補日期欄位、錯誤提示與清空日期文案
  - 完成：Ticket 列表篩選列已新增日期起 / 日期迄欄位，初始值為 `Asia/Taipei` 當天
  - 完成：列表 API query key 與 request 已帶入 `created_from` / `created_to`；清空日期後不帶日期參數
  - 完成：日期起晚於日期迄時停用查詢並顯示欄位錯誤
  - 完成：三語 `ticket.json` 已補日期欄位、清空日期與日期區間錯誤文案
  - 驗證：`npm run typecheck`
  - 驗證：三語 `ticket.json` 鍵集合一致
  - _Requirements: 2.3, 2.6_
  - _Design: Ticket 列表頁 / Ticket API Contract_

- [x] 4.6 Ticket 列表複製建立 Ticket
  - 操作欄在查看按鈕前新增複製建立入口
  - 點擊後導向 `/projects/:id/tickets/new`，透過 route state 帶入可複製欄位
  - 建立頁需預填標題、事件類型、優先級、子專案、資訊來源與指派人；描述欄只在原 Ticket 有描述時帶入原描述
  - 不得直接呼叫 create API，不得複製附件 / 貼圖 / storage key / 詳情連結
  - 完成：列表複製按鈕已改為複製建立，導向新增 Ticket 並帶入 route state；建立頁套用草稿後提示操作員確認
  - 修正：描述欄不再填入列資訊摘要，僅在原 Ticket 有描述內容時帶入原描述
  - 驗證：`npm run typecheck`、`npm run build` 通過
  - _Requirements: 2.7, 3.8_
  - _Design: Ticket 列表頁 / 建立 Ticket 頁_

- [x] 4.7 Ticket 列表刪除操作
  - 操作欄最後新增刪除按鈕
  - 僅 Ticket 建立者本人且狀態為 `open` 時可看到刪除按鈕
  - 刪除前需二次確認，確認文案包含 Ticket 標題
  - 確認後呼叫 `DELETE /api/v1/tickets/:id`，成功後重新讀取目前列表
  - 失敗顯示後端錯誤或刪除失敗 toast
  - 完成：列表操作欄最後已新增刪除按鈕，依 `created_by` 與 `status=open` 顯示，確認後呼叫 delete API 並重新讀取列表
  - 驗證：`npm run typecheck`、`npm run build` 通過
  - _Requirements: 2.8, 3.9_
  - _Design: Ticket 列表頁_

## 5. 建立 Ticket

- [x] 5.1 建立 Ticket create API client
  - OpenAPI 已列出 `POST /api/v1/tickets` requestBody 與 response schema
  - OpenAPI 已列出 `GET /api/v1/ticket-metadata/options`，用於建立表單的事件類型、資訊來源與 priority 選項
  - request / response type 需依 OpenAPI schema 與 `design.md` contract，不得假造欄位
  - 完成：已在 `features/ticket/api/tickets.ts` 新增 `CreateTicketPayload`、`TicketDetail`、`TicketActivity`、`TicketAttachment` 與 `createTicket`
  - 完成：`createTicket` 串 `POST /api/v1/tickets`，成功回傳後端 `DetailResponse` 的 `data`
  - 完成：`getTicketMetadataOptions` 已由 `4.1` 建立，建立表單可沿用該 client，不硬編事件類型、資訊來源或 priority 選項
  - 完成：本 task 不建立表單 UI、不送假 API，表單 shell 與串接保留給 `5.2`、`5.3`
  - 驗證：`npm run typecheck` 通過
  - _Depends on: 0.1_

- [x] 5.2 建立 Ticket create form shell
  - 欄位與 layout 依設計
  - 描述欄位使用共用 MarkdownEditor
  - 事件類型、資訊來源與 priority 下拉選單只吃 metadata options API 回傳資料，不硬編假選項
  - submit 在 5.1 API client 完成前不得送假 API
  - 完成：已新增 `/projects/:id/tickets/new` 建立頁 shell，使用專案工作區 layout，並掛入 router
  - 完成：建立頁讀取 metadata options、子專案與專案成員真實 API 作為下拉選項；沒有資料時不顯示假選項
  - 完成：描述欄位使用共用 `MarkdownEditor` 初版，提供編輯 / 預覽與純文字 fallback
  - 完成：submit button 目前維持 disabled，不送假 API；實際建立與錯誤處理保留給 `5.3`
  - 驗證：`ticket.json` 三語鍵集合一致；`npm run typecheck` 通過
  - _Design: 建立 Ticket 頁_

- [x] 5.3 串接建立 Ticket
  - 成功後導向詳情頁
  - 失敗顯示後端錯誤
  - 完成：建立頁送出時呼叫 `createTicket`，payload 依 `CreateTicketPayload` 與目前專案 `projectId` 組成
  - 完成：送出前驗證標題、描述、事件類型、priority、子專案、資訊來源；若事件類型要求指派人，會驗證指派人員
  - 完成：送出中禁用表單控制與按鈕，避免重複提交
  - 完成：成功後顯示成功通知、invalidate project tickets / detail query，並導向 `/tickets/:id`
  - 完成：失敗時顯示後端錯誤訊息，不顯示假成功
  - 修正：建立頁子專案選項只列出目前主專案啟用子專案；選擇資訊來源時若後端回傳 `sub_project_id`，會自動帶入對應子專案欄位
  - 修正：建立頁子專案下拉若目前主專案沒有可用子專案，顯示明確空選項文案，不留成無內容下拉
  - 驗證：`ticket.json` 三語鍵集合一致；`npm run typecheck` 通過
  - _Blocked by: 5.1_

## 6. Ticket 詳情

- [x] 6.1 建立 Ticket detail API client
  - OpenAPI 已列出 `GET /api/v1/tickets/{id}` 與 `DetailResponse` schema
  - response type 需依 OpenAPI schema 與 `design.md` contract，不得假造欄位
  - 完成：已在 `features/ticket/api/tickets.ts` 新增 `getTicketDetail(ticketId)`
  - 完成：client 串 `GET /api/v1/tickets/{id}`，使用既有 `TicketDetail` / `Ticket` / `TicketActivity` / `TicketAttachment` 型別
  - 完成：型別只包含 OpenAPI 與 `design.md` 已定義欄位，不新增假欄位
  - 驗證：`npm run typecheck` 通過
  - _Depends on: 0.1_

- [x] 6.2 建立 Ticket 詳情頁 shell
  - 標題與狀態列
  - 主內容 / 側欄布局
  - 活動紀錄 / 留言 / 附件區塊 shell
  - 不顯示假資料
  - 完成：已新增 `/tickets/:id` 詳情頁 shell，並掛入 router
  - 完成：頁首顯示 route ticket id、空狀態 status / priority、禁用的狀態切換 / 指派 / 編輯操作
  - 完成：主內容包含描述、活動紀錄 / 留言 tabs、附件區塊；目前只顯示未載入空狀態，不顯示假資料
  - 完成：側欄保留狀態、優先級、事件類型、子專案、資訊來源、指派人、協作者、建立人與時間欄位的 shell
  - 完成：留言區塊先共用禁用版 `MarkdownEditor` shell；實際送出與資料串接保留給 `6.3`、`7.3`
  - 驗證：`ticket.json` 三語鍵集合一致；`npm run typecheck` 通過
  - _Design: Ticket 詳情頁_

- [x] 6.3 串接 Ticket 詳情資料
  - loading / error / not found states
  - 完成：詳情頁使用 `getTicketDetail` 串 `GET /api/v1/tickets/{id}`，query key 為 `['ticket', 'detail', ticketId]`
  - 完成：載入中顯示 skeleton，404 顯示 not found，其他錯誤顯示可刷新錯誤狀態
  - 完成：成功時顯示 Ticket 標題、狀態、priority、描述、活動紀錄、留言、附件 metadata 與側欄資訊
  - 完成：Markdown 內容以純文字 / `white-space: pre-wrap` 顯示，不注入 HTML
  - 完成：狀態切換、指派、編輯與留言送出仍維持禁用，實際操作保留給 `7.1`、`7.2`、`7.3`
  - 驗證：`ticket.json` 三語鍵集合一致；`npm run typecheck` 通過
  - _Blocked by: 6.1_

## 7. Ticket 操作

- [x] 7.1 串接狀態流轉
  - 只顯示後端允許操作
  - 成功後 invalidate detail / activities
  - 完成：已新增 `transitionTicketStatus(ticketId, payload)`，串 `POST /api/v1/tickets/{id}/status`
  - 完成：詳情頁依 `design.md` 狀態流轉矩陣顯示可切換狀態，不顯示非法狀態
  - 完成：`escalated` 只在 metadata options 顯示該 ticket type 支援 escalation 時出現
  - 完成：依目前登入者全域角色與 `/projects/{id}/members` 專案角色過濾操作；viewer 不顯示送出操作，`closed` 目標需 project_manager 或全域管理角色
  - 完成：`resolved` 要求解決摘要，`cancelled` 要求取消原因
  - 完成：成功後更新 detail cache，並 invalidate detail / project tickets；失敗顯示後端錯誤，不改判成功
  - 驗證：`ticket.json` 三語鍵集合一致；`npm run typecheck` 通過
  - _Blocked by: Backend task 2.1_

- [x] 7.2 串接指派與新增協作者
  - 候選人使用 `GET /api/v1/projects/:id/members` 的既有專案成員清單
  - 不得使用 `user-options` 顯示非專案成員作為指派或協作者候選人
  - 成功後 invalidate detail / activities
  - 完成：已新增 `assignTicket`、`addTicketCollaborator`、`removeTicketCollaborator` API client
  - 完成：詳情頁指派候選人使用既有專案成員清單，不使用 `user-options`
  - 完成：詳情頁新增協作者候選人使用既有專案成員清單，不使用 `user-options`
  - 完成：指派與新增協作者成功後 invalidate detail / project tickets；失敗顯示後端錯誤
  - 完成：Project Manager 或全域管理角色才啟用指派 / 新增協作者；已關閉 Ticket 禁用操作
  - 補充：後端 `2.4` 已補 `DetailResponse.collaborators`，協作者清單與移除 UI 已由 `7.2a` 完成
  - 驗證：`ticket.json` 三語鍵集合一致；`npm run typecheck` 通過
  - _Blocked by: Backend task 2.2_

- [x] 7.2a 串接協作者清單與移除 UI
  - 後端 task `2.4` 已補 `DetailResponse.collaborators`
  - 顯示目前協作者清單
  - 移除協作者時呼叫 `DELETE /api/v1/tickets/{id}/collaborators/{uid}`
  - 成功後 invalidate detail / project tickets
  - 完成：詳情頁側欄已從 `detail.collaborators` 顯示目前協作者清單，空清單顯示空狀態
  - 完成：新增協作者候選人會排除既有協作者，避免重複新增
  - 完成：移除協作者串 `DELETE /api/v1/tickets/{id}/collaborators/{uid}`，成功後 invalidate detail / project tickets，失敗顯示後端錯誤
  - 驗證：`ticket.json` 三語鍵集合一致；`npm run typecheck`
  - _Backend: task 2.4_

- [x] 7.2b 修正詳情頁頁首指派入口
  - 問題：Ticket 詳情頁頁首「指派」按鈕只有視覺狀態，沒有導向實際指派控制，使用者點擊後無反應
  - 頁首指派按鈕需在有權限且 Ticket 未關閉時可用
  - 點擊後需定位到側欄既有指派人下拉與送出按鈕，不新增第二套指派流程
  - 成功指派仍沿用 `POST /api/v1/tickets/{id}/assign`
  - 完成：頁首「指派」已接上側欄指派區定位，避免按鈕無作用

- [x] 7.3 串接留言
  - mention 候選人使用 `GET /api/v1/projects/:id/members` 的既有專案成員清單
  - Markdown 原文送出
  - 成功後清空輸入並更新 timeline
  - 完成：已新增 `addTicketComment(ticketId, payload)` API client，串 `POST /api/v1/tickets/{id}/activities`
  - 完成：詳情頁留言區使用共用 `MarkdownEditor` 輸入 Markdown 原文，送出 payload 包含 `content`、`is_internal=false`、`mentioned_user_ids`
  - 完成：mention 候選人只使用目前 Ticket 所屬專案的 `GET /api/v1/projects/:id/members` 啟用成員，不使用 `user-options`
  - 完成：送出成功後清空留言內容與 mention 選取，並 invalidate detail query 更新 timeline / comments
  - 完成：後端拒絕或 API 失敗時只顯示錯誤 Toast，不顯示假成功
  - 驗證：`ticket.json` 三語鍵集合一致；`npm run typecheck` 通過
  - _Blocked by: Backend task 2.3_

- [x] 7.3a 串接 Ticket 詳情編輯
  - 問題：Ticket 詳情頁頁首「編輯」按鈕為禁用狀態，未接 `PUT /api/v1/tickets/{id}`
  - 新增詳情頁編輯 Dialog，欄位包含標題、描述、事件類型、優先級、資訊來源與外部單號
  - 指派維持獨立 `assign` API，不混入編輯表單
  - 已關閉 Ticket 不可編輯；全域管理角色或 Project Engineer 以上才顯示可編輯操作
  - 事件類型切換後需依 allowed priorities 收斂優先級
  - 成功後更新 detail cache 並 invalidate detail / project tickets；失敗顯示後端錯誤
  - 完成：已新增 `updateTicket(ticketId, payload)` API client 並接上詳情頁編輯 Dialog

- [x] 7.4 建立 MarkdownEditor 共用元件
  - 編輯 / 預覽切換
  - 欄位錯誤提示
  - 送出中禁用重複提交
  - 預覽需使用安全 renderer 或純文字 fallback，避免 XSS
  - 建立 Ticket 描述與詳情頁留言共用同一元件
  - 補充：`5.2` 已建立初版共用 `MarkdownEditor` 並用於建立 Ticket 描述欄位；`7.3` 已在詳情頁留言送出流程使用同一元件；本 task 仍保留完整驗證與 renderer 強化
  - 完成：共用 `MarkdownEditor` 已提供編輯 / 預覽切換，建立 Ticket 描述與詳情頁留言共用同一元件
  - 完成：欄位錯誤提示支援 `error` / `helperText`，並補上 `aria-describedby` 關聯
  - 完成：送出中可透過 `disabled` 禁用輸入，既有建立 Ticket 與留言流程會在提交中傳入禁用狀態，避免重複提交
  - 完成：預覽改用 React 節點安全渲染 Markdown 常用格式，不使用 `dangerouslySetInnerHTML`，HTML / script 內容只會以文字呈現
  - 完成：支援標題、無序 / 有序清單、引用、程式碼區塊、行內 code、粗體與斜體
  - 驗證：`ticket.json` 三語鍵集合一致；`npm run typecheck` 通過
  - _Design: MarkdownEditor_

## 8. 附件

- [x] 8.1 建立附件 API client
  - upload / list / content / delete
  - 不暴露 storage key
  - 完成：已在 `features/ticket/api/tickets.ts` 新增 `listTicketAttachments(ticketId)`，串 `GET /api/v1/tickets/{id}/attachments`
  - 完成：已新增 `uploadTicketAttachment(ticketId, { file })`，使用 `multipart/form-data` 的 `file` 欄位串 `POST /api/v1/tickets/{id}/attachments`
  - 完成：已新增 `downloadTicketAttachmentContent(attachmentId)`，串 `GET /api/v1/attachments/{id}/content` 並以 `Blob` 回傳
  - 完成：已新增 `deleteTicketAttachment(ticketId, attachmentId)`，串 `DELETE /api/v1/tickets/{id}/attachments/{aid}`
  - 完成：附件型別只保留 OpenAPI metadata 欄位，不新增或暴露 `storage_key` / 直接儲存路徑
  - 驗證：`ticket.json` 三語鍵集合一致；`npm run typecheck` 通過
  - _Blocked by: Backend task 3.1_

- [x] 8.2 建立附件 UI
  - 上傳按鈕
  - 拖曳上傳 dropzone
  - 貼上截圖
  - metadata list
  - 圖片預覽
  - delete flow
  - 完成：詳情頁附件區已加入上傳按鈕與隱藏 file input，支援多選圖片
  - 完成：已加入拖曳 dropzone 與 paste 事件入口，會將檔案加入待上傳清單，但本 task 不送出 API
  - 完成：已強化 metadata list，顯示檔名、content type、大小、建立時間與操作按鈕
  - 完成：已加入圖片預覽 Dialog；本地待上傳圖片可預覽，後端既有附件內容載入保留給 `8.3`
  - 完成：已加入刪除確認 Dialog；確認送出保留給 `8.3`，避免假刪除
  - 驗證：`ticket.json` 三語鍵集合一致；`npm run typecheck` 通過
  - _Design: 附件_

- [x] 8.3 串接附件流程
  - 上傳進度 / loading / error
  - 已關閉 Ticket 禁用上傳與刪除
  - 完成：詳情頁附件列表已串 `listTicketAttachments(ticketId)`，並在載入中顯示 progress、失敗時顯示錯誤
  - 完成：上傳已串 `uploadTicketAttachment(ticketId, { file })`，依待上傳檔案逐檔送出，顯示逐檔完成比例與 loading 狀態
  - 完成：附件圖片預覽已串 `downloadTicketAttachmentContent(attachmentId)`，使用 Blob URL 顯示，不暴露 storage key 或直接儲存路徑
  - 完成：下載按鈕已串 `downloadTicketAttachmentContent(attachmentId)`，由瀏覽器下載 Blob
  - 完成：刪除確認已串 `deleteTicketAttachment(ticketId, attachmentId)`，成功後 invalidate attachments / detail
  - 完成：已關閉 Ticket 禁用選檔、拖曳 / 貼上加入待上傳、上傳送出與刪除；下載與預覽仍可用
  - 限制：目前 `apiClient` 使用 fetch，沒有上傳 byte progress callback；本 task 以逐檔完成比例呈現進度
  - 驗證：`ticket.json` 三語鍵集合一致；`npm run typecheck` 通過
  - _Blocked by: 8.1_

- [x] 8.4 建立附件檔案輸入驗證
  - 點擊、拖曳、貼上三種入口共用同一套驗證
  - content-type 白名單：JPG / PNG / GIF / WebP
  - 單檔小於等於 10MB
  - 每張 Ticket 附件數量小於等於 20
  - invalid 檔案需顯示錯誤，不得送出 API
  - 完成：點擊選檔、拖曳 drop、paste 三種入口都呼叫同一個 `addPendingAttachmentFiles` 與 `validateAttachmentFiles`
  - 完成：content type 白名單限制為 `image/jpeg`、`image/png`、`image/gif`、`image/webp`
  - 完成：單檔大小限制為 10MB；每張 Ticket 以目前後端附件數量加待上傳數量共同限制最多 20 個
  - 完成：invalid 檔案會顯示具體錯誤訊息，不加入待上傳清單
  - 完成：上傳 mutation 送出前再次驗證待上傳檔案，避免 invalid 檔案送 API
  - 驗證：`ticket.json` 三語鍵集合一致；`npm run typecheck` 通過
  - _Design: 附件_

- [x] 8.5 附件圖片直接預覽
  - 問題：既有附件列表只有眼睛按鈕，使用者無法在附件列直接確認圖片內容
  - 圖片附件列表需顯示縮圖；點擊縮圖或眼睛按鈕都開啟大圖預覽 Dialog
  - 縮圖與大圖仍只能透過 `GET /api/v1/attachments/{id}/content` 取得，不暴露 storage key、S3 URL 或本機路徑
  - Blob object URL 需在元件更新或卸載時釋放
  - 完成：詳情頁附件列表已加入圖片縮圖直接預覽，並補上預覽 Blob URL cleanup

- [x] 8.6 留言圖片附件 UI
  - 留言 MarkdownEditor 區域需支援拖曳圖片與貼上截圖
  - 待上傳圖片需顯示縮圖、檔名、大小與移除按鈕
  - 點擊「新增留言」後先呼叫 `POST /api/v1/tickets/{id}/activities` 建立留言，再逐張呼叫附件上傳 API 並帶入 `activity_id`
  - 每則留言下方需顯示該留言 response 的 `attachments` 縮圖；點擊縮圖開啟既有附件預覽 Dialog
  - 仍需保留全域附件列表作為 Ticket 附件總覽，但不得在前端猜測留言與附件關聯
  - 部分圖片上傳失敗時需顯示錯誤並重新讀取 Detail，不得顯示假成功
  - _Blocked by: Backend task 3.4_
  - _Requirements: 4.9, 4.10, 7.7, 7.8_
