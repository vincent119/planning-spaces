# Ticket Backend Tasks

## 文件定位

本文件只追蹤 Ticket spec 後端工作。已在原 spec 完成且經 OpenAPI 可見的 Project / Sub Project / Project Member API 不在此重做，除非發現契約缺口。

## 0. 契約整理

- [x] 0.1 對齊 Ticket API 與 OpenAPI
  - 已確認 server 有 `internal/ticket` 與 `internal/attachment` route / service implementation
  - 檢查 Ticket / Attachment delivery 是否缺 swagger 註解
  - 若 swagger 註解已存在但 OpenAPI 未列出，修 generator 掃描範圍
  - 重新執行 `make openapi`
  - 確認 `Docs/openapi.json` 列出 Ticket CRUD、Activity、Attachment paths
  - 此 task 完成前，不得把舊 spec 的 Ticket Core 歷史勾選解讀成 Ticket 端到端完成
  - 完成：已補 Ticket / Attachment delivery swagger 註解並重新產生 `Docs/openapi.json`
  - _Requirements: 2.4, 3.1-3.6, 4.1-4.6_

- [x] 0.2 建立 Ticket API contract 文件段落
  - request / response 欄位不得只存在程式碼
  - 補齊 list filters、pagination、sort contract
  - 補齊 error code 與權限失敗行為
  - 完成：已在 `design.md` 補齊 Ticket API Contract 段落
  - _Design: API Dependency_

- [x] 0.3 補強 OpenAPI JSON schema 精確度
  - `cmd/openapi` 需輸出 JSON body requestBody，不只輸出 `body` parameter
  - Ticket create / update / status / assign / collaborator / comment request schema 需可由 OpenAPI 讀取
  - Ticket list / detail / activity / attachment metadata response schema 需可由 OpenAPI 讀取
  - Project / Sub Project / Project Member response schema 也需避免只剩 generic object
  - 完成：已補 `cmd/openapi` Go struct schema 解析、package-aware DTO resolution、JSON requestBody 與 `httpx.APIResponse[T]` response schema，並重新產生 `Docs/openapi.json`
  - _Design: API 缺口_

- [x] 0.4 建立 Ticket metadata options API contract
  - 需確認或新增 Ticket type options API
  - 需確認或新增 Ticket source options API
  - 需確認 priority options contract 是否由 ticket type 決定或固定枚舉
  - 前端建立 Ticket 頁不得硬編假選項
  - 完成：已新增 `GET /api/v1/ticket-metadata/options`，回傳 `ticket_types`、project scoped `ticket_resources`、`priorities`；priority 為全域 `P1` 到 `P4`，可用與預設值依 ticket type 的 `allowed_priorities` / `default_priority`
  - 注意：`ticket_sources` 已被後續設計判定為錯誤分叉；目前 schema 與對外契約已由 4.4 / 4.6 收斂為 project scoped `ticket_resources`
  - 完成：已補 OpenAPI schema 巢狀 DTO 解析並重新產生 `Docs/openapi.json`
  - _Design: API 缺口_

- [x] 0.5 確認或建立 user search API contract
  - 專案成員新增需要可選擇使用者
  - Ticket 指派 / mention / collaborator 也可能需要使用者搜尋或專案成員選項
  - 若既有 users API 可用，需記錄 path 與 response shape；若不可用，需新增後端 contract
  - 完成：已新增 `GET /api/v1/projects/{id}/user-options`，供新增專案成員 Dialog 搜尋 active users，支援 `keyword`、`limit`、`offset`、`exclude_members`
  - 完成：已確認 Ticket 指派 / mention / collaborator 候選人應使用既有 `GET /api/v1/projects/{id}/members`，避免顯示非專案成員
  - 完成：已重新產生 `Docs/openapi.json`，確認 `user-options` response schema 包含 `id`、`username`、`email`、`full_name`、`global_role`、`is_active`
  - 修正：專案成員後續確認需以原 spec `design-backend.md` 的 `project_members` 表為目標契約，`active users`、`global_role`、`is_active` 不再視為最終契約；需由 `0.6` 重新補強
  - _Design: API 缺口_

- [x] 0.6 補強 Project Member contract 對齊原後端設計
  - 專案成員以原 spec `design-backend.md` 的 `project_members` 表為目標契約，不是 Admin 用戶管理表的前端投影
  - `project_members` 核心欄位為 `project_id`、`user_id`、`role`、`joined_at`
  - 原後端設計中 `project_members.user_id` 指向 `ops_user(id)`；需檢查目前 server implementation / migration 若仍指向 `users(id)`，必須修正或在設計文件提出一致化決策
  - 重新定義 `GET /api/v1/projects/{id}/members` response schema，移除或不得依賴 `global_role`、MFA、phone、remark、系統帳號狀態等 Admin 用戶管理欄位
  - 重新定義 `GET /api/v1/projects/{id}/user-options` 候選來源與 response schema；不得讓前端直接使用 `/api/v1/admin/users`
  - `POST /api/v1/projects/{id}/members` 與 `PUT /api/v1/projects/{id}/members/{uid}` 維持以 `project_members.user_id` 作為成員識別；文件與 OpenAPI 必須一致
  - 重新產生 `Docs/openapi.json`
  - 完成：已移除 `MemberResponse` 與 `UserOptionResponse` 的 `global_role`、`is_active`
  - 完成：repository 不再 select `u.global_role`、`u.is_active` 作為 response 欄位；`users.is_active` 僅保留為候選人內部篩選條件
  - 修正：此 task 第一版只收斂 response 欄位但仍 join `users`，資料來源仍錯；已由 `0.7` 補正為 `ops_user`
  - 完成：已重新產生 `Docs/openapi.json`，確認 `/projects/{id}/members` 與 `/projects/{id}/user-options` schema 不含 `global_role`、`is_active`
  - 驗證：`go test ./internal/project ./internal/ticket ./internal/auth` 通過
  - _Requirements: 6.2, 6.5, 6.6_
  - _Design: Project Member Contract_

- [x] 0.7 修正 Project Member 實際資料來源為 ops_user
  - 修正 `GET /api/v1/projects/{id}/members`，由 `project_members` join `ops_user`，不得 join Admin `users`
  - 修正 `GET /api/v1/projects/{id}/user-options`，候選來源改為 `ops_user`
  - 修正新增成員存在檢查，改查 `ops_user`
  - 新增 migration `0019_align_project_members_ops_user.sql`，將 `project_members.user_id`、`ticket_collaborators.user_id` FK 對齊 `ops_user(id)`；`tickets.assignee_id` 因 `tickets` 為分區表，不新增 PostgreSQL 不支援的 `NOT VALID` FK，維持 application layer 驗證
  - 修正 Ticket 指派驗證、assignee 顯示與 collaborator 顯示的 join 來源為 `ops_user`
  - 完成：前端 Project Member email 型別改為 optional，`ops_user` 無 email 時顯示 `-`
  - 驗證：`go test ./internal/project ./internal/ticket ./internal/auth` 通過
  - _Requirements: 6.2, 6.5, 6.6_
  - _Design: Project Member Contract_

- [x] 0.8 修正 Project Member ops_user 授權對應
  - 問題：`project_members.user_id` 已對齊 `ops_user(id)`，但專案存取檢查仍以登入者 `users.id` 查 `project_members.user_id`，導致 Ticket 列表與專案成員頁載入失敗
  - 專案 API 的 `RequireProjectAccess`、`ListProjects` 與 Project Member 授權查詢需使用 `CurrentUser.Username` 對應 `ops_user.user_name`
  - Ticket list repository 的專案可見範圍也需使用 `project_members -> ops_user` 對應，不得直接以 `users.id` 比對 `project_members.user_id`
  - 保留直接以 `user_id` 比對的相容路徑，避免既有測試資料或 migration 過渡期資料失效
  - 完成：已補 `ActorUsername` 傳遞到 project service / repository / middleware 與 auth project role service
  - 完成：Ticket list scope 已改用 `EXISTS project_members LEFT JOIN ops_user` 判斷專案可見範圍
  - 驗證：`go test ./internal/project ./internal/ticket ./internal/auth`；`npm run typecheck`
  - _Requirements: 6.2, 6.5, 6.6_
  - _Design: Project Member Contract_

- [x] 0.9 補強新增專案維運人員 contract
  - 新增專案成員不是挑既有候選人，而是在目前專案建立一位運維人員
  - `POST /api/v1/projects/{id}/members` 需在同一 transaction 內新增 `ops_user`，再以新 `ops_user.id` 建立 `project_members`
  - request body 需改為接收 `user_name`、`description`；`project_id` 來自 path，不接受 body 指定
  - `ops_user.id` 由後端產生；前端不得送入
  - 後端需定義 `user_name` 驗證、同專案重複資料與衝突行為；衝突需回 409
  - OpenAPI 需同步更新 requestBody / response schema
  - 舊的 `GET /api/v1/projects/{id}/user-options` 不作為新增成員主流程；若保留，需明確標示用途，避免前端誤用
  - 完成：新增 `CreateProjectMemberRequest`，`POST /api/v1/projects/{id}/members` contract 改為 `user_name`、`description`
  - 完成：OpenAPI 已重新產生，新增成員 response 為 `MemberResponse`
  - 完成：新增 SQL migration `0020_add_ops_user_username_unique_index.sql`，以 `ops_user.user_name` 唯一索引支撐 409 衝突語意
  - 完成：`design.md` 已移除新增成員候選人主流程，明確記錄新增成員需建立 `ops_user` 與 `project_members`
  - _Requirements: 6.2, 6.5, 6.6_
  - _Design: Project Member Contract_

- [x] 0.10 實作新增專案維運人員交易流程
  - 依 `0.9` contract 修改 `POST /api/v1/projects/{id}/members`
  - service / repository 需以 database transaction 同步建立 `ops_user` 與 `project_members`
  - 不得再要求前端提供既有 `user_id` 或從 `user-options` 選擇候選人
  - `user_name` 需做必填、長度、格式與唯一性驗證；違反唯一性或同專案重複加入需回 409
  - `project_members.role` 若資料表仍為必填，後端需定義內部預設值，不暴露給前端 UI
  - 成功 response 需回傳新增後的 project member view，且來源為 `project_members -> ops_user`
  - 補單元測試與 repository 測試，覆蓋成功交易、`ops_user` 建立失敗 rollback、`project_members` 建立失敗 rollback、重複 `user_name` 衝突
  - 重新產生 `Docs/openapi.json`
  - 完成：`CreateProjectMember` service 驗證 `user_name` 必填、長度與格式，並以內部預設角色 `engineer` 建立成員
  - 完成：repository 使用 transaction 寫入 `ops_user` 與 `project_members`
  - 完成：repository 測試已覆蓋成功 commit、`ops_user` 建立失敗 rollback、`project_members` 建立失敗 rollback、`ops_user.user_name` 唯一鍵衝突與 `project_members` 唯一鍵衝突
  - 完成：已重新產生 `Docs/openapi.json`
  - _Requirements: 6.2, 6.5, 6.6_
  - _Design: Project Member Contract_

- [x] 0.11 修正專案維運人員移除為狀態停用
  - 問題：專案成員刪除顯示 404，且刪除語意應為 soft delete，不應硬刪 `project_members` 或 `ops_user`
  - `ops_user` 新增 `is_active` 狀態欄位，新增成員預設 `true`
  - `DELETE /api/v1/projects/{id}/members/{uid}` 需將對應 `ops_user.is_active` 改為 `false`
  - 專案成員列表、候選查詢與專案權限查詢不得回傳或使用 `is_active=false` 的運維人員
  - 完成：新增 migration `0021_add_ops_user_active_status.sql`
  - 完成：repository `DeleteMember` 已改為停用 `ops_user`，不刪除 `project_members`
  - 完成：補 repository 測試覆蓋 soft delete 與找不到成員回 `ErrMemberNotFound`
  - _Requirements: 6.2, 6.5, 6.6_
  - _Design: Project Member Contract_

## 1. Ticket Core 補強

- [x] 1.1 驗證 Ticket create contract
  - 必填：title、description、ticket_type_id、priority、project_id、sub_project_id、ticket_resource_id
  - created_by 來自登入者，不接受 request body 指定
  - 建立時同 transaction 寫入 `ticket_activities.created`
  - 完成：已補 service 必填欄位測試、actor 來源測試與 repository create transaction 測試
  - 修正：Create reference 驗證中的 `assignee_id` 已改查目前專案的啟用 `project_members -> ops_user`，不得再查 Admin `users` 表，避免建立 Ticket 選擇專案維運人員時回 404
  - 驗證：`go test ./internal/ticket`
  - _Requirements: 3.1, 3.15_

- [x] 1.2 驗證 Ticket list contract
  - 支援 project scope
  - 支援 keyword、status、priority、ticket_type_id、sub_project_id、assignee_id filters
  - 支援 pagination
  - response 欄位需足夠支援前端列表，不要求前端 JOIN 或假造顯示名稱
  - 完成：已補 `external_ref` keyword 搜尋、query/filter normalize 測試與 list response 顯示欄位測試
  - _Requirements: 2.1-2.4_

- [x] 1.3 驗證 Ticket detail contract
  - 回傳基本欄位與顯示用 summary
  - 不在 detail response 暴露附件 storage key
  - 完成：已補 detail response 顯示欄位測試、relation 載入測試與附件 metadata 不含 storage_key 驗證
  - _Requirements: 4.1-4.6_

- [x] 1.4 驗證 Ticket update / delete contract
  - 已關閉 Ticket 不可編輯
  - delete 採 soft delete
  - delete 寫入 `ticket_activities.deleted`
  - 完成：已驗證 update 多欄位單筆 `field_updated`、no-op update 不寫入、closed ticket reject、soft delete 與 `deleted` activity
  - _Requirements: 3.6, 3.7_

- [x] 1.5 回復 Ticket 刪除權限為建立者且 open
  - `DELETE /api/v1/tickets/:id` 僅允許 Ticket 建立者本人
  - 只能刪除狀態為 `open` 的 Ticket
  - 保留 soft delete 與 `ticket_activities.deleted`
  - 完成：service 刪除權限已回復為建立者本人且 Ticket 狀態為 `open`
  - 驗證：`go test ./internal/ticket/...` 通過
  - _Requirements: 3.9_

## 2. Ticket 操作

- [x] 2.1 驗證狀態流轉 API
  - 後端回傳或文件列出允許轉換
  - 關閉限制 Project Manager / Admin
  - 寫入 `status_changed`
  - 完成：已在 `design.md` 補允許狀態流轉矩陣，並驗證 service / repository / delivery 測試覆蓋狀態流轉、關閉權限與 `status_changed`
  - _Requirements: 5.1, 5.4_

- [x] 2.2 驗證指派與協作者 API
  - 指派負責人
  - 新增 / 移除 collaborators
  - 寫入對應 activity
  - 完成：已補指派 / 協作者 API 契約規則，並驗證 project member 檢查與 `assigned` / `collaborator_added` / `collaborator_removed` activity 覆蓋
  - _Requirements: 5.2, 5.3_

- [x] 2.4 補強 Ticket Detail collaborators contract
  - `GET /api/v1/tickets/{id}` 需回傳目前協作者清單，或新增等價查詢 API
  - response 欄位需包含 user id、username、full_name、email 與 joined/created 時間
  - OpenAPI 需同步列出 schema，避免前端從 activity 推測目前協作者
  - 前端移除協作者 UI 需等待此 contract 完成
  - 完成：已新增 `TicketCollaborator` / `CollaboratorResponse`，`DetailResponse.collaborators` 回傳目前協作者清單
  - 完成：repository 透過 `ticket_collaborators` join `users` 查詢目前協作者，service 組入 Ticket detail
  - 完成：`make openapi` 已重新產生 `Docs/openapi.json`，`GET /api/v1/tickets/{id}` schema 已包含 `collaborators`
  - 驗證：`go test ./internal/ticket` 通過
  - _Requirements: 5.3_

- [x] 2.3 驗證留言 API
  - Markdown 原文儲存
  - 寫入 `comment_added`
  - 完成：已補留言 API 契約規則，並驗證 Markdown 原文、空白內容拒絕、mention 成員檢查、route actor 與 `comment_added` activity
  - _Requirements: 4.4_

## 3. 附件

- [x] 3.1 驗證附件 OpenAPI contract
  - 上傳 multipart contract
  - metadata list contract
  - content streaming endpoint
  - delete endpoint
  - 完成：已補 `cmd/openapi` 解析 `@Accept` / `@Produce` / `@Param`，重新產生 `Docs/openapi.json`，並驗證附件上傳 multipart file、metadata JSON、binary streaming、delete path parameters
  - _Requirements: 7.1-7.4_

- [x] 3.2 驗證附件權限與狀態限制
  - 已關閉 Ticket 不可新增或刪除附件
  - 目前後端規則為僅上傳者可刪除，Admin / Project Manager 不具備刪除他人附件的例外
  - 完成：已補附件權限與狀態契約，並驗證上傳 / 刪除需要 Engineer、列表 / 內容需要 Viewer、closed ticket 禁止新增與刪除、刪除僅允許上傳者，以及 repository 交易內重複檢查 closed / uploader
  - _Requirements: 7.3, 7.4_

- [x] 3.3 修正附件活動寫入 JSONB 參數型別
  - 問題：上傳附件成功寫入 metadata 後，新增 `ticket_activities.action_type = attachment_added` 時，PostgreSQL 無法推斷 `jsonb_build_object('attachment_id', ?)` 的 bind 參數型別，回傳 `SQLSTATE 42P18`
  - `ticket_activities.field_changes.attachment_id` 應明確以 `text` 寫入 JSONB，避免 prepared statement 型別推斷失敗
  - 完成：`internal/attachment/repository.go` 已將 attachment id 參數 cast 為 `?::text`
  - 完成：同步更新 attachment repository SQL mock 測試

- [x] 3.4 建立留言附件 schema 與 API contract
  - `attachments` 新增 nullable `activity_id CHAR(26)`，用於關聯留言 activity
  - `POST /api/v1/tickets/{id}/attachments` multipart request 支援 optional `activity_id`
  - service 需驗證 `activity_id` 屬於同一 Ticket 且 activity 為 `comment_added`
  - Attachment metadata response 需回傳 nullable `activity_id`
  - Detail response 的 `comments[]` 需包含該留言的 `attachments` metadata 陣列
  - 列表、內容串流、刪除權限沿用現有附件規則；soft delete 後留言下方不得顯示
  - 需更新 SQL、domain、repository、service、delivery、OpenAPI 與測試
  - _Requirements: 4.9, 4.10, 7.7, 7.8_

## 4. 資訊來源與事件類型管理

- [x] 4.1 補強 `ticket_resources` schema 與 soft delete contract
  - `ticket_resources` 新增 `is_active BOOLEAN NOT NULL DEFAULT TRUE`
  - 既有資料 migration 需回填 `is_active=true`
  - 新增資訊來源預設 `is_active=true`
  - 刪除資訊來源改為 soft delete，更新 `is_active=false` 與 `updated_at`
  - 資訊來源列表與 Ticket 建立選項不得回傳 `is_active=false` 資料
  - 重新產生 `Docs/openapi.json`
  - 完成：`ticket_resources.is_active`、註解與 active/project 查詢索引已整合至 `sql/0026_create_ticket_schema.sql`
  - 完成：已在 `design.md` 補齊舊版 Ticket Resource Contract
  - 注意：`ticket_sources` / `ticket_resources` 平行存在的舊 contract 已被 4.4 / 4.5 / 4.6 取代，目標為只保留 `ticket_resources`
  - _Requirements: 6A.4-6A.11_

- [x] 4.2 建立 `ticket_resources` 管理 API
  - `GET /api/v1/projects/{id}/ticket-resources`
  - `POST /api/v1/projects/{id}/ticket-resources`
  - `GET /api/v1/ticket-resources/{rid}`
  - `PUT /api/v1/ticket-resources/{rid}`
  - `DELETE /api/v1/ticket-resources/{rid}`
  - 舊版 DTO 欄位曾包含 `id`、`type`、`name`、`details`、`project_id`、`sub_project_id`、`is_active`、`created_at`、`updated_at`
  - 注意：4.5 已將正式 DTO 收斂為 `id`、`code`、`name`、`resource_type`、`description`、`project_id`、`sub_project_id`、`is_active`、`sort_order`、`created_at`、`updated_at`
  - List API 預設只回傳啟用資料
  - 完成：已在 project 模組新增 `TicketResource` domain / repository / service / delivery
  - 完成：已新增五個管理 API route，刪除採 soft delete 更新 `is_active=false`
  - 完成：已補 delivery swagger 註解、service 測試、handler 測試與 repository SQL mock 測試
  - 修正：`ticket_resources.sub_project_id` 依資料庫 schema 改為選填且不依 DB 外鍵約束；後端僅在有值時檢查子專案屬於目前 project 且啟用
  - _Requirements: 6A.4-6A.11_

- [x] 4.3 建立 `ticket_types` 管理 API
  - `GET /api/v1/ticket-types`
  - `POST /api/v1/ticket-types`
  - `GET /api/v1/ticket-types/{tid}`
  - `PUT /api/v1/ticket-types/{tid}`
  - `DELETE /api/v1/ticket-types/{tid}`
  - 系統內建事件類型不可刪除
  - 系統內建事件類型核心欄位不可修改
  - `default_priority` 必須包含在 `allowed_priorities`
  - 停用後不得出現在建立 Ticket 與列表篩選 metadata options
  - 重新產生 `Docs/openapi.json`
  - 完成：已在 ticket 模組新增 `TicketType` domain / repository / service / delivery
  - 完成：已新增五個管理 API route，刪除採 soft delete 更新 `is_active=false`
  - 完成：已補 admin/op_admin 管理權限、內建事件類型保護、priority contract 驗證與 handler/service 測試
  - _Requirements: 6A.1-6A.3, 6A.11_

- [x] 4.4 收斂 Ticket 資訊來源主流程到 `ticket_resources`
  - 廢棄 `ticket_sources` 作為 Ticket create / list / detail / metadata options 的資料來源
  - Ticket create / update request 欄位由 `ticket_source_id` 改為 `ticket_resource_id`
  - Ticket response 欄位由 `ticket_source_id` / `ticket_source_code` / `ticket_source_name` 改為 `ticket_resource_id` / `ticket_resource_code` / `ticket_resource_name` / `ticket_resource_type`
  - Ticket list filter 由 `ticket_source_id` 改為 `ticket_resource_id`
  - repository 查詢需 join `ticket_resources`，並限制 resource 屬於 Ticket project 且 `is_active=true`
  - metadata options 需回傳目前 project scope 的 `ticket_resources`
  - 需補資料 migration：已刪表空庫不再保留 `ticket_source_id` 過渡欄位；最終 schema 由 4.5 / 4.6 整合至 `0026`
  - 重新產生 `Docs/openapi.json`
  - 完成：Ticket create / update / list / detail / metadata options 已改用 `ticket_resource_id` 與 project scoped `ticket_resources`
  - 完成：Ticket response 回傳 `ticket_resource_id` / `ticket_resource_code` / `ticket_resource_name` / `ticket_resource_type`
  - 完成：已重新產生 `opscenter-server/Docs/openapi.json` 並通過 `go test ./internal/ticket`
  - _Requirements: 3.7, 6A.8, 6A.12_

- [x] 4.5 重構 `ticket_resources` schema 欄位
  - 移除 `details JSONB`
  - 新增 / 調整欄位：`code`、`name`、`resource_type`、`description`、`project_id`、`sub_project_id`、`is_active`、`sort_order`、`created_at`、`updated_at`
  - `code` 在同一 project 內唯一
  - `sub_project_id` 選填且不建立 DB 外鍵；有值時由 service 驗證屬於目前 project 且啟用
  - 常用來源 `tg_alert`、`mail`、`signal`、`signal_project_group`、`whatsapp`、`whatsapp_project_group`、`zabbix_alert`、`business_domain_change` 需改為 project scoped `ticket_resources` 預設資料或專案初始化資料
  - 更新 project module DTO / repository / service / delivery / tests
  - 重新產生 `Docs/openapi.json`
  - 完成：移除無必要的 `sql/0023_converge_ticket_resource_main_flow.sql`
  - 完成：最終 schema 已整合至 `sql/0026_create_ticket_schema.sql`，將 `type` / `details` 收斂為 `resource_type` / `description`，並新增 `code`、`sort_order` 與 project scoped 預設資訊來源 seed
  - 完成：`tickets.ticket_source_id` 不再建立；Ticket 主流程只使用 `ticket_resource_id`
  - 完成：Project 資訊來源 API request / response 已改為 `code`、`resource_type`、`description`、`sort_order`；不再暴露 `details` / `source_type`
  - 完成：Ticket metadata options 與 Ticket response 已讀取 `ticket_resources.code` / `resource_type`
  - 完成：已重新產生 `opscenter-server/Docs/openapi.json` 並通過 `go test ./internal/...`
  - _Requirements: 6A.4-6A.13_

- [x] 4.6 移除 `ticket_sources` schema 與程式依賴
  - 移除或標記廢棄 `ticket_sources` table migration 依賴
  - 移除 ticket repository / metadata options 對 `ticket_sources` 的查詢
  - 移除 OpenAPI 中 `ticket_sources` response 欄位
  - 保留 migration 需考慮既有部署資料，不可直接破壞舊環境升級
  - 完成：依刪表重建策略，Ticket 相關 SQL 已收斂到 `sql/0026_create_ticket_schema.sql`
  - 完成：已移除舊 Ticket SQL：`0005_create_ticket_core_tables.sql`、`0006_create_attachment_tables.sql`、`0010_add_ticket_query_indexes.sql`、`0011_update_attachment_image_storage_constraints.sql`、`0022_add_ticket_resources_active_status.sql`、`0024_refactor_ticket_resources_schema.sql`、`0025_drop_ticket_sources_schema.sql`
  - 完成：`0013_add_design_schema_gap_tables.sql` 與 `0019_align_project_members_ops_user.sql` 已移除 Ticket schema 補丁，避免與 `0026` 重複建立或改欄位
  - 完成：`0026_create_ticket_schema.sql` 建立最終 Ticket schema，包含 `ticket_types`、`ticket_resources`、`services_level`、`tickets`、`ticket_collaborators`、`ticket_activities`、`attachments`、`ticket_attachments` view、`ticket_tags`、`ticket_affected_services`、`sla_configs`、相關索引與預設資訊來源 seed
  - 完成：後端 project / ticket 程式碼與 `Docs/openapi.json` 不含 `ticket_sources` / `ticket_source_id` 對外契約；`0025_drop_ticket_sources_schema.sql` 已依需求刪除
  - _Requirements: 6A.8_
