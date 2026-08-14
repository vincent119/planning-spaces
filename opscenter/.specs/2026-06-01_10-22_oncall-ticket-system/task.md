# Backend Task Plan

## Scope

本文件先拆解 OnCall Ticket System 的 **Backend MVP** 任務。MVP 目標是打通 Ticket 主流程：使用者與權限、專案 / 子專案、表單樹權限、Ticket CRUD、狀態流轉、指派 / 協作、留言 / 活動記錄、搜尋篩選、附件管理、Global Setting、基礎觀測與部署。

MVP 不實作：SLA、通知、完整排班 / 假勤、Jira CSV 匯入、報表設計器、API Key、SSO。這些保留在 Phase 2 / Phase 3。

## 狀態校正

本文件中的 Ticket Core、Search / Filter / Dashboard Basic、Attachment Storage 歷史勾選，代表當時後端實作與測試曾完成，不能代表目前 Ticket 功能已端到端完成。

目前已確認 `opscenter-server/internal/ticket` 與 `opscenter-server/internal/attachment` 有 route 與 service implementation，但 `opscenter-server/Docs/openapi.json` 尚未列出 Ticket CRUD、Ticket Activity、Ticket Attachment paths。Ticket 的 OpenAPI 契約同步、前端頁面設計與前端實作，已移到 `../2026-06-15_15-35_Ticket` 以新的未完成 task 追蹤。

後續不得在本文件已完成項目中追加新需求後仍維持完成狀態；新增或補強工作必須建立新的未完成 task。

---

## 1. Backend 專案骨架與基礎設施

- [x] 1.1 建立 `opscenter-server/` Go module 與 DDD 目錄結構
  - `cmd/server`
  - `internal/{auth,user,project,form,ticket,setting,scheduler,infra}`
  - `pkg`
  - `configs`
  - `sql`
  - `web/dist`
  - _Requirements: 12.1, 12.2, 非功能需求部署架構_

- [x] 1.2 建立 Gin HTTP server、route group 與 graceful shutdown
  - listen port 預設 `9898`
  - `/api/v1` API prefix
  - static SPA fallback 預留 `web/dist`
  - _Requirements: 12.1, 12.2_

- [x] 1.3 建立統一 API response 與錯誤處理
  - response format：`code`, `message`, `data`, `trace_id`
  - domain / validation / auth / permission / conflict 錯誤對應 HTTP status
  - _Requirements: 12.1_

- [x] 1.4 建立 config 載入與環境變數覆寫
  - DB / Redis / JWT / storage / CORS / timezone
  - business timezone 固定載入 `Asia/Taipei`
  - _Requirements: 非功能需求時區、安全性_

- [x] 1.5 建立 logger、request id / trace id middleware
  - 每個 request 產生或接收 `X-Request-ID`
  - response 回傳 `trace_id`
  - _Requirements: 12.1, 非功能需求安全性_

- [x] 1.5a 移除 Gin 內建 access log，統一使用 zlogger 結構化 request log
  - API request 完成時只由 `middleware.AccessLogWithSkips` 輸出結構化 log
  - 不再同時輸出 `[GIN]` 文字格式 access log，避免同一 request 產生兩種格式
  - _Requirements: 非功能需求安全性；Design: 可觀測性_

- [x] 1.6 建立 DB / Redis 連線初始化
  - PostgreSQL 使用 GORM
  - Redis 用於 refresh token / cache 預留
  - _Requirements: 非功能需求擴展性_

- [x] 1.7 建立 health endpoints
  - `GET /api/v1/healthz/live`
  - `GET /api/v1/healthz/ready` 檢查 DB / Redis
  - _Requirements: 12.1, 非功能需求可用性_

---

## 2. Database Schema 與 Seed

- [x] 2.1 建立 ULID extension / wrapper 檢查 SQL
  - migration / app 啟動時確認 `generate_ulid()` 存在
  - 所有主鍵使用 `CHAR(26)` + `DEFAULT generate_ulid()`
  - _Requirements: 非功能需求識別碼_

- [x] 2.2 建立使用者與認證相關 tables
  - `users`
  - `user_mfa_devices`
  - refresh token 儲存策略使用 Redis
  - _Requirements: 1.1, 1.3, 1.8, 1.9_

- [x] 2.3 建立專案與成員 tables
  - `projects`
  - `sub_projects`
  - `project_members`
  - _Requirements: 1.2, 4.1-4.8_

- [x] 2.4 建立表單樹、群組與業務授權來源 tables
  - `form_nodes`
  - `groups`
  - `group_members`
  - `group_form_permissions`
  - `casbin_rule`
  - `form_audit_logs`
  - _Requirements: 2.1-2.10_

- [x] 2.5 建立 Ticket 核心 tables
  - `ticket_types`
  - `ticket_sources`
  - `tickets`
  - `ticket_collaborators`
  - `ticket_activities`
  - _Requirements: 3.1-3.10, 6.1-6.8, 7.1-7.6, 8.1-8.7_

- [x] 2.6 建立附件 tables
  - `attachments`
  - metadata 不回傳 `storage_key`
  - _Requirements: 3.11-3.13_

- [x] 2.7 建立系統設定與排程日誌 tables
  - `system_settings`
  - `scheduler_logs`
  - MVP 僅做設定與清理任務基礎，不做 SLA 排程
  - _Requirements: Global Setting, 非功能需求安全性_

- [x] 2.8 建立初始 seed
  - default admin user
  - default form nodes：`ticket` / `ticket/create`
  - default permission group：`engineers`
  - default group form permission：`engineers` 可讀 / 新增 / 修改 `ticket/create`
  - default ticket types：Daily / Issue
  - default ticket sources：TG Alert / Mail / Signal / Signal 項目群 / WhatsApp / WhatsApp 項目群 / Zabbix Alert / 支付or業務項目域名更換
  - default priorities：P1-P4 目前由 `tickets.priority` constraint 與 `ticket_types.allowed_priorities` 管理，不建立獨立 seed
  - default system settings
  - _Requirements: 1.1, 2.1-2.10, 3.1, Global Setting_

- [x] 2.9 建立預設系統管理選單樹與 OIDC 權限群組 seed
  - 新增系統管理選單樹：`system`
  - 新增子節點：`system/users`、`system/roles`、`system/menus`、`system/projects`、`system/logs`、`system/schedulers`、`system/settings`
  - 新增或更新權限群組：`ops_admin`、`ops_op_admin`
  - `ops_admin` 對 `system` 子樹具備 `read/create/update/delete`
  - `ops_op_admin` 僅對 `system/logs` 具備 `read`
  - 寫入業務來源表 `group_form_permissions`，並由後端同步或 seed 對應 `casbin_rule`
  - 不建立 `casbin_rule` 前端管理介面；前端只管理 `groups`、`group_members`、`group_form_permissions`
  - SQL 必須冪等，可重複執行
  - _Requirements: 1.17, 2.17；Design: 預設選單樹 seed, 預設選單權限 seed_

- [x] 2.10 補齊權限群組 API 狀態與來源欄位契約
  - `GET /api/v1/admin/groups?status=active|inactive` 依群組啟用狀態篩選；未傳 `status` 時回傳全部群組
  - `PUT /api/v1/admin/groups/:id` 接受選用 `is_active`，供前端權限群組啟用 / 停用流程使用
  - `GET /api/v1/admin/groups/:id/permissions` 每筆權限需回傳 `source`
  - 目前來源表資料回傳 `direct`；若該筆設定 `override_parent=true` 則回傳 `override`
  - 未完成有效權限展開 API 前，不回傳前端自行推測的 `inherited`
  - _Requirements: 2.12, 2.14；Design: 群組 CRUD, 群組表單權限_

- [x] 2.11 修正表單節點新增時空 id 寫入問題
  - `form_nodes.id` 由 PostgreSQL `DEFAULT generate_ulid()` 產生
  - 後端新增表單節點時，若 domain node ID 為空，不得把空字串 `id` 寫入 INSERT
  - 新增成功 response 必須回傳資料庫產生的有效 ULID，避免前端後續刪除送出 `/admin/forms/`
  - 補 repository 測試確認 create 使用 DB default 回填 id
  - _Requirements: 2.1；Design: 表單樹管理, ULID 主鍵規範_

- [x] 2.12 修正權限群組刪除與停用語意分離
  - `PUT /api/v1/admin/groups/:id` 搭配 `is_active=false` 負責停用群組
  - `DELETE /api/v1/admin/groups/:id` 必須真正刪除群組，不得只更新 `is_active=false`
  - 刪除群組時需清除該群組同步出的 `casbin_rule` policy
  - `group_members` 與 `group_form_permissions` 依 FK cascade 刪除，`form_audit_logs` 保留 before snapshot
  - 刪除後群組不得因狀態篩選為「全部狀態」而繼續顯示在前端列表
  - _Requirements: 2.12；Design: 群組 CRUD, Casbin policy 同步_

---

## 3. Auth / User / MFA

- [x] 3.1 實作密碼登入
  - `POST /api/v1/auth/login`
  - bcrypt / argon2 password hash
  - inactive user 拒絕登入
  - _Requirements: 1.3, 1.8_

- [x] 3.2 實作 JWT access token 與 refresh token
  - access token 8 小時
  - refresh token 7 天，Redis 儲存與撤銷
  - `POST /api/v1/auth/refresh`
  - `POST /api/v1/auth/logout`
  - _Requirements: 1.8_

- [ ] 3.2a 實作閒置時間自動登出
  - 新增 `security.session.timeout` seed，預設 28800 秒，供管理端設定會話閒置逾時秒數
  - 新增已登入 session policy API，只回傳 `idle_timeout_seconds`，不得暴露 system_settings metadata
  - 前端登入後監聽使用者活動，超過 `idle_timeout_seconds` 未操作時清除 token、清除目前使用者狀態並導回登入頁
  - 設定值小於等於 0 時停用前端閒置自動登出；設定缺值或無效時使用預設 28800 秒
  - 後端測試需覆蓋 policy 解析、401 保護、最小 response 與 seed 冪等；前端測試需使用 fake timer 覆蓋自動登出、活動續期、停用設定、API fallback 與登入 / MFA / SSO 頁面不啟動 timer
  - 手動驗收需以短 timeout 設定確認受保護頁閒置後導回登入，並顯示閒置逾時 Toast
  - 目前已完成後端 Go 測試與前端 build 驗證；前端 fake timer 單元測試待建立前端測試框架後補齊
  - _Requirements: 1.8；Design: 系統設定, 前端登入流程狀態機_

- [x] 3.3 實作 auth middleware
  - 驗證 `Authorization: Bearer`
  - 注入 current user / role / trace context
  - _Requirements: 12.1, 非功能需求安全性_

- [x] 3.4 實作 MFA 裝置管理
  - `POST /api/v1/auth/mfa/setup`
  - `POST /api/v1/auth/mfa/verify`
  - `GET /api/v1/auth/mfa/devices`
  - `DELETE /api/v1/auth/mfa/devices/:id`
  - _Requirements: 1.9_

- [x] 3.5 實作 Admin 使用者管理 API
  - 使用者列表 / 新增 / 修改 / 停用
  - request / response 必須包含 `phone`，對應 `users.phone`
  - 使用者列表 / 單筆 response 必須包含 `has_mfa`，只要存在已驗證且啟用中的 MFA 裝置即為 `true`
  - 不在 users table 儲存班制歸屬
  - _Requirements: 1.1-1.7, 1.13, 1.13a_

- [x] 3.6 實作個人資料查詢與更新 API
  - 目前登入使用者可查詢自己的 `username`、`full_name`、`email`、`phone`、`global_role`
  - 目前登入使用者可更新自己的 `full_name`、`email`、`phone`
  - `username` 與 `global_role` 不允許透過個人資料 API 修改
  - API request / response 必須包含 `phone`，對應 `users.phone`
  - _Requirements: 1.13_

- [x] 3.7 實作 self-service 修改密碼 API
  - `PUT /api/v1/auth/password`
  - 從 Access Token 取得目前登入使用者，不接受指定 `user_id`
  - request 必須包含 `current_password` 與 `new_password`
  - 後端必須以目前登入使用者的 `password_hash` 驗證目前密碼正確後才更新
  - 新密碼最小長度由 `system_settings` 的 `security.login.password_length` 定義，設定缺值或無效時預設 4
  - 後端需以 `security.login.password_length` 作為最終驗證來源，通過後重新產生 password hash
  - 不得沿用 Admin 使用者更新 API 跳過目前密碼驗證
  - 成功與失敗需回傳可供前端 Toast 顯示的 `message`
  - 成功需記錄 `security_audit_logs`，不得記錄明文密碼或 hash
  - _Requirements: 1.14, 1.15_

- [x] 3.8 實作已登入密碼政策 API
  - `GET /api/v1/auth/password-policy`
  - 需 Bearer Access Token，不公開給未登入使用者
  - 只回傳 `{ "min_length": number }`，不得回傳 `system_settings` key、category、description、設定來源或其他安全設定
  - `min_length` 來源為 `system_settings.security.login.password_length`，設定缺值、停用或無效時預設 4
  - 不做前端可解密的 response 混淆或加密；安全性由權限控制、最小揭露與 `PUT /auth/password` 後端最終驗證承擔
  - _Requirements: 1.14, 1.16_

- [x] 3.9 實作公開 Auth 設定查詢 API
  - `GET /api/v1/auth/config`
  - 不需要登入，供登入頁判斷是否顯示 OIDC 入口
  - response 必須包含 `oidc.enabled` 與 `oidc.login_url`
  - `oidc.enabled` 來源為後端 `config.yaml` / ENV 載入後的 `OIDC.Enabled`
  - `oidc.login_url` 預設為 `/api/v1/auth/oidc/login`
  - 不得回傳 `client_secret`、`provider_url` 或其他敏感 OIDC 設定
  - 完整 OIDC login / callback 流程仍屬 D8 Phase 3
  - _Requirements: 1.11, 1.16_

- [x] 3.10 實作未驗證 MFA 裝置自動清理
  - `GET /api/v1/auth/mfa/devices` 只回傳已驗證且啟用的 MFA 裝置
  - 新增 repository / service 方法清理 `is_verified = FALSE AND is_active = TRUE` 且逾時的 MFA 裝置
  - 逾時門檻由 `system_settings.security.mfa.pending_device_retention_minutes` 定義，缺值、停用或無效時預設 30 分鐘
  - 清理採軟刪除，將 `is_active` 設為 `FALSE`，不得影響已驗證裝置
  - 服務啟動時執行一次清理，並註冊排程 `clean_unverified_mfa_devices`
  - 清理成功 / 失敗需寫入 scheduler log 或結構化日誌
  - _Requirements: 1.9_

---

## 4. 表單樹與 RBAC 權限

- [x] 4.1 實作全域角色與專案角色權限判斷
  - `admin`
  - `op_admin`
  - `member`
  - project role：Project Manager / Engineer / Viewer
  - _Requirements: 1.1, 1.2, 1.11_

- [x] 4.2 實作表單樹 CRUD
  - `GET /api/v1/admin/forms`
  - `GET /api/v1/admin/forms/:id`
  - `POST /api/v1/admin/forms`
  - `PUT /api/v1/admin/forms/:id`
  - `DELETE /api/v1/admin/forms/:id`
  - _Requirements: 2.1-2.5_

- [x] 4.3 實作群組與群組成員 CRUD
  - `GET/POST /api/v1/admin/groups`
  - `GET/PUT/DELETE /api/v1/admin/groups/:id`
  - `GET/POST /api/v1/admin/groups/:id/members`
  - `DELETE /api/v1/admin/groups/:id/members/:uid`
  - _Requirements: 2.6, 2.7_

- [x] 4.4 實作群組表單權限 API
  - `GET /api/v1/admin/groups/:id/permissions`
  - `POST /api/v1/admin/groups/:id/permissions`
  - `DELETE /api/v1/admin/groups/:id/permissions/:pid`
  - `group_form_permissions` 作為業務授權來源表，異動後需同步 Casbin policy
  - _Requirements: 2.6-2.10_

- [x] 4.5 實作 Casbin 授權服務與 middleware
  - 封裝 `Enforce(userID, formPath, action)`
  - 使用 PostgreSQL adapter table `casbin_rule` 持久化 policy
  - handler 不自行查表判斷表單權限
  - 專案內操作需同時滿足 Project Role 與 Form Permission
  - _Requirements: 1.11, 2.8, 2.9_

- [x] 4.6 實作有效權限查詢
  - `GET /api/v1/forms/tree`
  - `GET /api/v1/forms/permissions?path=...`
  - 透過 Casbin 逐一檢查 `read/create/update/delete`
  - _Requirements: 2.8-2.10_

- [x] 4.7 實作表單樹與群組權限審計日誌
  - 記錄操作人、時間、異動前後內容
  - 記錄 Casbin policy 同步摘要
  - _Requirements: 2.10_

- [x] 4.8 對齊一般工作區讀取 API 與表單 read 權限
  - 所有對應側邊欄、專案工作區 tab 或一般工作入口的讀取 API，必須先檢查對應 `form_nodes.full_path` 的有效 `read` 權限
  - 不得以「任一工作區 read 權限」、單純 `op_admin` 角色或 project member 身分取代指定表單節點 read 權限
  - 事件類型讀取使用 `tickets/ticket-types:read`
  - SLA 讀取使用 `tickets/sla:read`
  - Dashboard 讀取使用 `dashboard:read`
  - 子專案、資訊來源、專案成員讀取分別使用 `tickets/sub-projects:read`、`tickets/sources:read`、`tickets/members:read`
  - Ticket metadata options 使用 `tickets/list:read`
  - 排班週期、人員班別、班別設定讀取分別使用 `schedule/periods:read`、`schedule/assignments:read`、`schedule/shifts:read`
  - 保留例外：個人 profile、個人通知、公開設定、health、metrics、登入 / MFA / SSO 流程與僅限 Admin 的安全管理 API
  - _Requirements: 2.8, 2.9, 2.15, 2.18；Design: 一般工作區讀取 API 權限對應_

- [x] 4.9 修正一般讀取 API 的二次授權與前端相依呼叫
  - `GET /api/v1/projects/:id/sla` 在 `tickets/sla:read` 通過後，只確認專案存在，不得再要求 project member 或任一工作區 read 權限
  - `GET /api/v1/projects/:id/dashboard` 在 `dashboard:read` 通過後，只確認專案存在，不得再要求 project member 或任一工作區 read 權限
  - Ticket 列表頁只有在使用者具備 `schedule/shifts:read` 時，才呼叫 `GET /api/v1/schedule/current-shift-group` 並顯示班別篩選
  - 避免使用者已具備 Ticket 列表或 SLA read 權限時，因非主要頁面相依 API 權限不足而出現 403
  - _Requirements: 2.8, 2.9, 2.18；Design: 一般工作區讀取 API 權限對應_

---

## 5. Project / SubProject

- [x] 5.1 實作專案 CRUD
  - `GET/POST /api/v1/projects`
  - `GET/PUT/DELETE /api/v1/projects/:id`
  - _Requirements: 4.1-4.3_

- [x] 5.2 實作子專案 CRUD
  - `GET/POST /api/v1/projects/:id/sub-projects`
  - `GET/PUT/DELETE /api/v1/sub-projects/:sid`
  - _Requirements: 4.2, 4.3_

- [x] 5.3 實作專案成員與角色管理
  - `GET/POST /api/v1/projects/:id/members`
  - `PUT/DELETE /api/v1/projects/:id/members/:uid`
  - _Requirements: 1.2, 4.4-4.7_

- [x] 5.4 實作專案歸屬權限 middleware
  - 非 Admin 僅能存取自己有角色的專案
  - _Requirements: 1.4-1.7, 4.8_

---

## 6. Ticket Core

- [x] 6.1 實作 Ticket 建立 API
  - `POST /api/v1/tickets`
  - 驗證表單 `create` 權限
  - 必填：標題、描述、事件類型、優先級、專案、子專案、資訊來源
  - 建立時寫入 `ticket_activities.created`
  - _Requirements: 3.1, 8.1, 8.6_

- [x] 6.2 實作 Ticket 詳情與列表 API
  - `GET /api/v1/tickets/:id`
  - `GET /api/v1/projects/:id/tickets`
  - 包含活動紀錄、留言、附件 metadata
  - _Requirements: 3.2, 3.9, 8.1_

- [x] 6.3 實作 Ticket 更新 API
  - `PUT /api/v1/tickets/:id`
  - 一次 API 多欄位異動寫入同一筆 `field_updated`
  - `field_changes` 包含 before / after
  - _Requirements: 3.3, 8.6, 8.7_

- [x] 6.4 實作 Ticket 狀態流轉
  - Open → In Progress → Pending → Resolved → Closed
  - Reopen Closed / Resolved
  - 關閉權限限制 Project Manager / Admin
  - 狀態變更寫入 `status_changed`
  - _Requirements: 6.1-6.8_

- [x] 6.5 實作 Ticket 指派與協作
  - 指派負責人
  - 新增 / 移除 collaborators
  - 寫入 `assigned` / `collaborator_added` / `collaborator_removed`
  - _Requirements: 7.1-7.6_

- [x] 6.6 實作留言 API
  - 新增留言
  - Markdown 原文儲存
  - 寫入 `comment_added`
  - _Requirements: 8.1-8.5_

- [x] 6.7 實作 Ticket 軟刪除或刪除策略
  - 刪除需寫入 `deleted`
  - 若採軟刪除，列表預設排除
  - _Requirements: 8.6_

---

## 7. Search / Filter / Dashboard Basic

- [x] 7.1 實作 Ticket 基本搜尋與篩選
  - 關鍵字
  - status
  - priority
  - project / sub_project
  - assignee
  - created date range
  - _Requirements: 9.1-9.5_

- [x] 7.2 實作列表分頁與排序
  - page / page_size
  - sort by created_at / updated_at / priority
  - _Requirements: 9.1, 非功能需求效能_

- [x] 7.3 建立必要 indexes
  - project + status
  - project + created_at
  - assignee
  - priority
  - ticket activity by ticket
  - _Requirements: 9.1-9.5, 非功能需求效能_

- [x] 7.4 實作 MVP 儀表板統計 API
  - open tickets
  - 今日新增
  - 今日解決
  - P1/P2 待處理
  - 不顯示 SLA 違反
  - _Requirements: 13.1, 13.2, 13.3_

- [x] 7.5 Ticket 列表支援開單人篩選
  - Status: Complete
  - 專案與跨專案 Ticket 列表接受 `creator_id` query parameter
  - repository 依 `tickets.created_by` 執行參數化篩選
  - Ticket metadata 回傳指定專案實際開單人去重清單，不沿用指派人員候選來源
  - 2026-08-07 修正：開單人去重改用 `GROUP BY`，避免 PostgreSQL 拒絕 `SELECT DISTINCT` 搭配未列於 SELECT 清單的顯示名稱排序表達式
  - 補 repository SQL 與 delivery query parsing 測試並更新 OpenAPI
  - Verify: `go test ./internal/ticket`、`make openapi`、`git diff --check`
  - _Requirements: 3.20_

---

## 8. Attachment Storage

- [x] 8.1 建立 storage backend interface 與可實際存取的 S3 adapter
  - Local implementation
  - 使用 AWS SDK for Go v2 實作 private bucket GetObject / PutObject / DeleteObject
  - 支援 IRSA、Instance Role、static credentials 與 MinIO custom endpoint
  - 不使用 pre-signed URL
  - 完成定義：正式組裝路徑可建立 AWS SDK-backed S3 backend；只有 interface、fake client 或 unavailable backend 不視為完成
  - storage key：`yyyy/mm/dd/{attachment_ulid}.{ext}`，日期以 `Asia/Taipei`
  - `{ext}` 僅允許 `avif` 或 `webp`
  - _Requirements: 3.11, 3.12_

- [x] 8.2 使用 govips 實作圖片轉檔與暫存清理
  - 使用 `github.com/davidbyttow/govips/v2/vips`，移除正式執行路徑的 `ffmpeg` converter
  - 建置與執行環境提供 `libvips`、WebP 所需的 `libwebp`、AVIF 所需的 `libheif`、`pkg-config`，並使用 `CGO_ENABLED=1`
  - 程序啟動 / graceful shutdown 統一管理 `vips.Startup` / `vips.Shutdown`
  - `vips.Startup` 前註冊 `govips` / `libvips` log bridge，統一透過 `zlogger` 結構化輸出，包含 `subsystem=attachment`、`component=libvips`、`vips_domain`、`vips_level`
  - 啟動時驗證 WebP / AVIF save capability，不支援時應用程式啟動失敗
  - 依 Global Setting `storage.image.output_format` 轉成 `avif` 或 `webp`
  - 原始上傳 content-type 白名單：jpeg / png / gif / webp
  - 解碼驗證實際圖片格式，不只信任 header / 副檔名
  - 套用正確圖片方向、移除 EXIF / GPS metadata，限制寬高與總像素數
  - 動態 GIF 必須保留動畫；encoder 不支援時回傳 `422`
  - 設定 libvips concurrency / cache 上限並完成容器 memory limit 壓力測試
  - 使用 `storage.image.temp_dir` 建立每次上傳獨立暫存工作目錄
  - 轉檔成功、失敗、request cancel、panic recovery 皆需清理暫存檔
  - 啟動時清理超過 `storage.image.temp_retention_minutes` 的殘留暫存目錄，並提供 `clean_image_conversion_temp` 共用操作供 task 9 排程註冊
  - _Requirements: 3.11, 3.12, 非功能需求安全性_

- [x] 8.3 實作附件上傳 API
  - 驗證原始檔案大小 ≤ 10MB
  - 附件數量 < 20
  - 上傳前一律轉檔後才寫入 Local / S3
  - metadata 寫入原始 `filename`，但 `content_type` / `size_bytes` / `storage_key` 使用轉檔後結果
  - 寫入 `attachment_added`
  - _Requirements: 3.11, 8.6_

- [x] 8.4 實作附件內容串流 API
  - `GET /api/v1/attachments/:id/content`
  - 驗證 JWT + Ticket / Project 存取權
  - 回傳轉檔後 `Content-Type`：`image/avif` 或 `image/webp`
  - metadata API 不暴露 `storage_key`
  - _Requirements: 3.12_

- [x] 8.5 實作附件刪除 API
  - 僅上傳者可刪，已關閉 Ticket 不可刪
  - 同步刪除 Local / S3 實體檔
  - 寫入 `attachment_deleted`
  - _Requirements: 3.13, 8.6_

- [x] 8.6 附件寫入改用 Ticket 表單操作權限
  - Status: Complete
  - Depends: 8.3, 8.5
  - Context: `op_member` 已具備 `tickets/list:update`，但附件上傳與刪除仍查詢 `project_members` 的 engineer role，導致沒有專案成員資料時回傳 403
  - Boundary: Allowed Changes 為附件需求與後端設計、`internal/attachment` service 與測試；Forbidden 為附件儲存格式、S3 / Local backend、Ticket API schema、前端權限節點與資料庫 schema
  - 上傳與刪除附件需檢查 `tickets/list:update`，不得呼叫 `RequireProjectAccess`
  - 列出附件與讀取內容維持 `tickets/list:read`
  - 刪除仍需驗證原上傳者，已關閉 Ticket 仍不可上傳或刪除
  - 驗收案例：使用者有 `tickets/list:update` 且沒有 `project_members` 時，可以新增圖片附件
  - 驗收案例：使用者缺少 `tickets/list:update` 時，上傳與刪除回傳 403
  - Verify: `go test ./internal/attachment ./internal/project ./internal/server`、`git diff --check`
  - Implementation Notes: 附件上傳與刪除改用 `tickets/list:update`，附件清單與內容維持 `tickets/list:read`；移除附件服務對 `project_members` 的依賴，並補齊允許與拒絕情境測試。`go test ./internal/attachment`、`go test ./internal/project ./internal/server` 均通過。
  - _Requirements: 3.10, 3.13, 3.17; Design: 一般工作區讀取 API 權限對應, Attachment Storage_

---

## 9. Global Setting / Admin Logs / Scheduler Logs

- [x] 9.1 實作 Global Setting CRUD
  - `GET /api/v1/admin/global-settings`
  - `POST /api/v1/admin/global-settings`
  - `GET /api/v1/admin/global-settings/:key`
  - `PUT /api/v1/admin/global-settings/:key`
  - `DELETE /api/v1/admin/global-settings/:key`
  - _Requirements: Global Setting, 非功能需求安全性_

- [x] 9.2 實作設定讀取優先級
  - `system_settings` > ENV > `config.yaml`
  - sensitive setting API 隱藏值
  - _Requirements: 非功能需求安全性_

- [x] 9.3 實作安全 / 系統 / 登入 / 排程日誌查詢基礎
  - MVP 至少支援登入日誌與系統操作審計
  - 匯出 CSV 可延後到後續 task
  - _Requirements: 2.10, 非功能需求安全性_

- [x] 9.4 實作 scheduler log 寫入與查詢
  - `GET /api/v1/system/schedulers`
  - `POST /api/v1/system/schedulers/:name/trigger`
  - `GET /api/v1/system/schedulers/:name/logs`
  - MVP 僅清理日誌類任務，不做 SLA checker
  - _Requirements: Global Setting, Phase boundary_

- [x] 9.5 實作排程任務定義 CRUD 與啟停 API
  - `POST /api/v1/system/schedulers`
  - `PUT /api/v1/system/schedulers/:name`
  - `PATCH /api/v1/system/schedulers/:name/enabled`
  - 新增、更新、啟停需寫入 `schedulers` 表，不得只回傳假成功
  - 啟停 API 回應需穩定，前端不得因 response body 缺失而例外
  - _Requirements: Global Setting, Phase boundary_

- [x] 9.6 補強 Admin Activity Logs schema / API 欄位契約
  - `system_audit_logs` 需新增並持久化 `username`、`method`、`path`、`status_code`、`result`、`duration_ms`
  - `GET /api/v1/admin/logs/activity` response item 需直接回傳上述欄位，不得只藏於 `detail`
  - 查詢需支援 `username`、`module`、`ip`、`method`、`status_code`、`result`、`date_range`
  - 系統操作 audit 寫入點需保存 HTTP method、path、status code、耗時與結果；不得讓前端推測或偽造
  - 補 repository / service / delivery 測試與 migration
  - _Requirements: 非功能需求安全性；Design: 日誌查詢（Admin）_

- [x] 9.7 補齊 Admin 非 GET 操作審計寫入覆蓋
  - 管理端 `/admin/*` 與 `/system/schedulers*` 非 GET 操作需統一寫入 `system_audit_logs`
  - 覆蓋 users / roles / schedulers / settings / forms / groups / permissions
  - 成功與失敗都需記錄 `username`、`module`、`action`、`method`、`path`、`ip_address`、`status_code`、`result`、`duration_ms`
  - 不得記錄 password、secret、token、MFA secret 等敏感 request body
  - 避免 Global Setting handler 與 middleware 重複寫入
  - 補 admin audit middleware 測試
  - _Requirements: 非功能需求安全性；Design: Admin 操作審計寫入_

---

## 10. Testing / Quality Gate

- [x] 10.1 建立 unit test 基礎
  - config
  - response / error mapping
  - permission merge
  - ticket state machine
  - _Requirements: 1.11, 6.1-6.8_

- [x] 10.2 建立 repository integration test
  - 使用測試 PostgreSQL
  - schema / seed 可重建
  - _Requirements: 非功能需求效能與安全性_

- [x] 10.3 建立 API handler test
  - auth
  - project
  - form permission
  - ticket CRUD
  - attachment content
  - _Requirements: 12.1, 12.2_

- [x] 10.4 建立 lint / format / test Makefile targets
  - `make fmt`
  - `make lint`
  - `make test`
  - `make run`
  - _Requirements: 開發規範_

- [x] 10.5 建立 OpenAPI 產出流程
  - Swagger / OpenAPI 3.0
  - _Requirements: 12.3_

- [x] 10.6 修正系統管理專案新增 / 編輯 Dialog 成功後未關閉
  - 問題：React Query `onSuccess` 期間 mutation 仍可能處於 pending 狀態，既有 `closeDialog()` 會因 `submitting` guard 直接 return
  - 修正：新增 / 編輯成功流程使用強制關閉，手動取消仍維持送出中不可關閉
  - 驗證：`npm run typecheck`
  - _Design: 專案管理（Admin）_

---

## 11. Docker / Deployment

- [x] 11.1 建立 backend Dockerfile stage 或 monorepo Dockerfile backend stage
  - Go build
  - runtime `TZ=Asia/Taipei`
  - expose `9898`
  - _Requirements: 非功能需求部署架構_

- [ ] 11.2 建立 local development compose
  - PostgreSQL
  - Redis
  - backend
  - _Requirements: 非功能需求擴展性_

- [x] 11.3 建立 migration / seed 執行說明
  - 手動執行 SQL
  - ULID extension prerequisite
  - _Requirements: 非功能需求識別碼_

---

## Deferred Tasks

- [ ] D1 Phase 2：SLA 管理與 SLA checker
- [ ] D2 Phase 2：站內通知 / Email 通知
- [ ] D3 Phase 2：自訂報表基礎查詢
- [ ] D4 Phase 2：Jira CSV 匯入與統計報表
- [ ] D5 Phase 2：非同步 import / export job
- [ ] D6 Phase 3：完整排班 / 假勤管理
- [ ] D7 Phase 3：報表設計器與範本
- [ ] D8 Phase 3：SSO / OIDC / SAML
  - [ ] D8.1 建立 OIDC 群組對應資料模型與 seed
    - 新增 `oidc_group_mappings`
    - seed 預設 mapping：`ops_admin -> admin`、`ops_op_admin -> op_admin`、`ops_member -> member`、`ops_op_member -> member`
    - `target_type` 支援 `global_role` 與 `permission_group`
    - 不修改 `roles.code`，也不得用外部群組名稱取代內部 `users.global_role`
    - _Requirements: 1.17；Design: oidc_group_mappings_
  - [ ] D8.2 實作 OIDC callback 群組同步
    - 從 OIDC claim 讀取外部群組，claim key 預設 `groups` 且可由設定調整
    - 多個 `global_role` mapping 命中時取最高權限序：`admin > op_admin > member`
    - `permission_group` mapping 命中時同步 `group_members`，僅同步已存在且啟用的 `groups`
    - 未命中任何 mapping 的新使用者預設 `global_role = member`，不得自動加入權限群組
    - 同步結果需寫入登入日誌或安全審計摘要，不得記錄 OIDC token、client secret 或敏感 claim
    - _Requirements: 1.11, 1.16, 1.17；Design: OIDC 登入同步規則_
  - [ ] D8.3 補 OIDC 群組對應測試
    - 覆蓋 `ops_admin`、`ops_op_admin`、`ops_member`、`ops_op_member`
    - 覆蓋多群組最高權限選擇
    - 覆蓋未知群組 fallback 到 `member`
    - 覆蓋 `permission_group` 不存在時不得建立假群組或偽造成功
    - _Requirements: 1.17_
- [ ] D9 Phase 3：API Key
- [ ] D10 Phase 3：Webhook 通知
