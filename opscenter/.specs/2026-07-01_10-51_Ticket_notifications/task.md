# Ticket 通知機制任務

## 1. 資料模型與 migration

- [x] 1.1 新增 `ops_user.linked_user_id`
  - 回填 `LOWER(ops_user.user_name) = LOWER(users.username)` 可匹配資料
  - 無法匹配者保留 NULL
  - 完成：新增 `0045_ticket_notifications_data_model.sql`，補欄位、FK、索引與 username 對應回填
  - _Requirements: 2.10, 3.7; Design: 帳號模型_

- [x] 1.2 新增 `notification_events`
  - 包含 event type、project、ticket、activity、actor、idempotency key、payload
  - 建立 pending 查詢索引
  - 完成：建立通知 outbox 與 pending / project / ticket 查詢索引；Ticket 採分區複合主鍵，`ticket_id` 由 `ticket_created_at` 輔助定位並由應用層驗證
  - _Requirements: 1.1-1.6; Design: notification_events_

- [x] 1.3 新增 `user_notifications`
  - 包含 recipient、title、body、link_path、read_at
  - 建立 unread count 與列表查詢索引
  - 完成：建立站內通知資料表、唯一收件人約束與 unread / recipient / project 查詢索引
  - _Requirements: 2.1-2.9; Design: user_notifications_

- [x] 1.4 新增 `notification_recipient_skips`
  - 記錄無法產生站內通知的解析原因
  - 完成：建立解析略過紀錄表與 event / reason 查詢索引
  - _Requirements: 2.10, 3.8-3.9; Design: notification_recipient_skips_

- [x] 1.5 新增 `project_webhooks` 與 `webhook_deliveries`
  - Webhook project scope
  - Webhook auth type 與加密 auth config
  - delivery status、attempt、next attempt、response excerpt
  - 完成：建立 project scope Webhook 設定、加密欄位、事件類型 JSON、delivery 狀態與 due / webhook / event / status 查詢索引
  - _Requirements: 4.1-5.10, 6.8; Design: project_webhooks, webhook_deliveries, Webhook outbound 認證_

- [x] 1.6 新增通知與 Webhook global settings seed
  - `notification.retention_days`
  - `webhook.delivery_retention_days`
  - `webhook.max_attempts`
  - `webhook.timeout_seconds`
  - `webhook.require_https`
  - `webhook.allow_private_network_in_dev`
  - `webhook.jwt_ttl_seconds`
  - 完成：`0045_ticket_notifications_data_model.sql` 已 seed 通知保留、Webhook 保留、重試、timeout、HTTPS、private network 與 JWT TTL 設定
  - _Requirements: 4.10, 5.5, 5.8, 9.5; Design: 設定項_

## 2. 後端通知事件

- [x] 2.1 建立 `internal/notification` bounded context
  - domain、repository、service、delivery 基礎結構
  - 完成：新增 `internal/notification` 的 domain、repository、service、delivery，保留後續 API 與 worker 擴充點
  - _Requirements: 1.1-1.6; Design: 後端模組設計_

- [x] 2.2 實作 Ticket event publisher
  - Ticket transaction 成功才寫入 `notification_events`
  - 支援 idempotency key
  - 完成：新增 `PublishTicketEvent`，以 Ticket activity id 組成 idempotency key，並在同一個 GORM transaction 寫入 `notification_events`
  - _Requirements: 1.3, 1.5; Design: 事件流程_

- [x] 2.3 將 Ticket 建立、狀態變更、指派、協作者與留言操作接上 event publisher
  - 不直接呼叫 Webhook
  - 不阻塞 Ticket API
  - 完成：Ticket 建立、狀態變更、指派、協作者新增、留言新增與 mention 會寫入通知 outbox；Webhook 只建立 delivery，不在 Ticket API 中投遞
  - _Requirements: 1.1, 1.6; Design: 事件類型_

- [x] 2.4 實作 recipient resolver
  - 建立者、被指派人、協作者、專案管理者、mention
  - 去重、排除操作者、skipped reason
  - 完成：dispatcher 解析 users、ops_user.linked_user_id、project_members 與 ticket_collaborators，並記錄無法送達、停用、自我通知與重複收件原因
  - _Requirements: 3.1-3.9; Design: 收件人解析_

- [x] 2.5 實作 notification dispatcher
  - 由 pending event 展開 `user_notifications`
  - 建立符合事件篩選的 `webhook_deliveries`
  - 成功後標記 `processed_at`
  - 完成：新增 pending event dispatcher，逐筆以 `FOR UPDATE SKIP LOCKED` 鎖定事件，展開站內通知、建立 Webhook delivery，成功後更新 processed_at
  - _Requirements: 1.6, 2.1, 5.1; Design: 事件流程_

## 3. 站內通知 API

- [x] 3.1 實作 `GET /api/v1/notifications`
  - 支援 unread / all、limit、cursor
  - 只回目前登入者資料
  - 完成：新增站內通知列表 API，使用 `filter=unread|all`、`limit`、`cursor`，repository 查詢固定限制 `recipient_user_id`
  - _Requirements: 2.1, 2.5, 7.1_

- [x] 3.2 實作 `GET /api/v1/notifications/unread-count`
  - 回傳目前登入者未讀數
  - 完成：新增未讀數 API，只統計目前登入者且未軟刪除的未讀通知
  - _Requirements: 2.2, 7.1_

- [x] 3.3 實作 `PATCH /api/v1/notifications/{id}/read`
  - 只能標記自己的通知
  - 完成：新增單筆已讀 API，更新條件包含 notification id 與目前登入者 id，完成後回傳該通知
  - _Requirements: 2.3, 7.1_

- [x] 3.4 實作 `POST /api/v1/notifications/mark-read`
  - 支援 selected、all_visible、all_unread
  - 完成：新增批次已讀 API；`selected` / `all_visible` 需提供 ids，`all_unread` 標記目前登入者全部未讀通知
  - _Requirements: 2.4, 2.9_

- [x] 3.5 補站內通知 OpenAPI
  - request / response schema
  - 401 / 403 / 404 error response
  - 完成：補 delivery godoc 註解並重新產生 `docs/openapi.json`，包含列表、未讀數、單筆已讀與批次已讀 schema
  - _Requirements: 2.1-2.9_

## 4. Webhook API 與安全

- [x] 4.1 實作 Webhook URL validator
  - HTTPS、private network、metadata、redirect 規則
  - 完成：新增 Webhook URL policy，限制 HTTPS、localhost、private/link-local/metadata IP 與危險 hostname；測試投遞 client 不跟隨 redirect
  - _Requirements: 6.1-6.4; Design: SSRF 防護_

- [x] 4.2 實作 Webhook secret 產生、加密保存與 rotate
  - 明文只在 create / rotate response 顯示一次
  - 日誌不得輸出 secret
  - 完成：新增 `WEBHOOK_SECRET_ENCRYPTION_KEY` 設定，Webhook payload secret 以 AES-GCM 加密保存，create / rotate 只回傳本次明文
  - _Requirements: 4.3, 6.5-6.6_

- [x] 4.3 實作 Webhook outbound auth 設定
  - 支援 none、Basic Auth、JWT Auth
  - Basic username / password 加密保存
  - JWT Auth 每次投遞產生短效 Bearer JWT
  - 列表與詳情不得回傳認證 secret
  - 完成：支援 none / basic / jwt auth config；Basic password 與 JWT secret 加密保存，response 只回傳遮罩狀態與 TTL
  - _Requirements: 4.8-4.10, 6.8; Design: Webhook outbound 認證_

- [x] 4.4 實作 Webhook CRUD API
  - project scope
  - 權限限制 admin、op_admin、project_manager
  - 完成：新增 `/projects/:id/webhooks` CRUD，統一透過 project manager 權限檢查，admin / op_admin 沿用專案服務跨專案管理規則
  - _Requirements: 4.1-4.7, 7.2_

- [x] 4.5 實作 Webhook 測試投遞 API
  - 使用測試 payload
  - 依 Webhook auth type 帶入對應 Authorization header
  - 不建立真實 Ticket
  - 完成：新增測試投遞 API，送出 test payload，不建立 Ticket 或 delivery 紀錄，依 auth type 帶入 Basic / Bearer JWT
  - _Requirements: 4.5, 4.8-4.10_

- [x] 4.6 實作 Webhook delivery 查詢與手動重試 API
  - delivery Data Grid 所需欄位
  - failed delivery 可手動重試
  - 完成：新增 delivery 列表 API 與手動 retry API；retry 只允許 failed / cancelled 回到 pending，實際投遞留給後續 worker
  - _Requirements: 5.2, 5.9, 7.3_

- [x] 4.7 補 Webhook OpenAPI
  - CRUD、test、delivery、retry schema
  - 不回傳 secret 明文
  - 補 auth_type / auth_config request schema
  - 完成：補 Webhook CRUD、test、delivery、retry godoc 並重新產生 OpenAPI
  - _Requirements: 4.1-5.10, 6.8_

## 5. Webhook worker

- [x] 5.1 實作 webhook signer
  - `X-Opscenter-Signature`
  - timestamp + raw body
  - 完成：新增 payload HMAC-SHA256 簽章，使用 `X-Opscenter-Timestamp` 與 `X-Opscenter-Signature`，簽章基礎為 timestamp + raw body
  - _Requirements: 5.4; Design: Webhook 簽章_

- [x] 5.2 實作 webhook outbound auth header builder
  - Basic Auth Authorization header
  - JWT Auth 短效 Bearer token
  - 不記錄 Authorization header
  - 完成：worker 送出時依 auth_type 套用 Basic / Bearer JWT；JWT claims 包含 webhook、project、delivery、event 資訊，日誌不輸出 Authorization
  - _Requirements: 4.8-4.10, 6.8; Design: Webhook outbound 認證_

- [x] 5.3 實作 delivery worker
  - due delivery 查詢
  - HTTP timeout
  - 狀態轉換
  - 完成：新增 due delivery worker，使用 `FOR UPDATE SKIP LOCKED` 鎖定到期 delivery，投遞後更新 delivered / retrying / failed
  - _Requirements: 5.1-5.8_

- [x] 5.4 實作 retry backoff
  - 立即、1 分鐘、5 分鐘、15 分鐘、1 小時
  - 支援 429 `Retry-After`
  - 完成：依 attempt_count 計算 backoff，429 `Retry-After` 優先且最大延遲限制為 1 小時，達最大嘗試次數後標記 failed
  - _Requirements: 5.6-5.8_

- [x] 5.5 實作 response excerpt 截斷
  - 不保存完整敏感 response
  - 完成：只保存最多 2048 bytes response excerpt，讀取後裁切與 trim，不保存完整 response body
  - _Requirements: 5.10, 6.5_

- [x] 5.6 補 worker 可觀測日誌
  - event id、delivery id、webhook id、status、attempt、trace id
  - 不記錄 secret 或 Authorization header
  - 完成：worker 日誌包含 delivery id、webhook id、event id、event type、attempt、status、HTTP status；不記錄 secret、URL auth 或 Authorization header
  - _Requirements: 6.8, 9.1-9.4_

## 6. 前端站內通知

- [x] 6.1 改造 Header 通知鈴鐺
  - 未讀數由 API 取得
  - 移除固定 `3`
  - 完成：Header 改用站內通知鈴鐺元件，未讀數由 `/notifications/unread-count` 輪詢取得，移除原本固定 `3`
  - _Requirements: 2.2, 2.7, 8.1_

- [x] 6.2 實作通知面板
  - unread / all
  - 空狀態
  - Ticket 連結
  - 完成：新增通知 Popover，支援未讀 / 全部頁籤、空狀態、分頁載入與 Ticket 詳情導向
  - _Requirements: 8.1-8.4_

- [x] 6.3 實作通知已讀操作
  - 單筆已讀
  - 全部已讀
  - invalidate unread count
  - 完成：支援點擊通知或單筆按鈕標記已讀，以及標記全部未讀通知為已讀，完成後刷新通知列表與未讀數
  - _Requirements: 2.3-2.4, 8.3_

- [x] 6.4 補站內通知 i18n
  - 繁中、簡中、英文
  - 不得漏 key
  - 完成：補齊站內通知標題、頁籤、空狀態、操作、事件類型與 aria 文案，覆蓋繁中、簡中、英文
  - _Requirements: 8.8_

## 7. 前端 Webhook 管理

- [x] 7.1 新增 `/projects/:id/webhooks` 路由與專案工作區入口
  - 僅具備管理權限時顯示
  - 完成：新增 `/projects/:id/webhooks` 前端路由與 `tickets/webhooks` 側欄 mapping；實際顯示仍依後端有效選單樹與權限回傳
  - _Requirements: 8.5, 7.2_

- [x] 7.2 實作 Webhook 設定 Data Grid
  - 名稱、URL host、事件類型、啟用狀態、最後投遞狀態、更新時間
  - 完成：新增 Webhook 設定 Data Grid，顯示名稱、host、事件類型、認證、啟用狀態、最後投遞、更新時間與操作
  - _Requirements: 4.2, 8.6_

- [x] 7.3 實作新增 / 編輯 Webhook Dialog
  - URL 驗證
  - 認證方式選擇：無、Basic Auth、JWT Auth
  - Basic Auth credential 輸入與遮罩提示
  - JWT Auth TTL 設定與 secret 狀態提示
  - 事件類型多選
  - 啟用狀態
  - 完成：新增 Webhook 表單 Dialog，支援 URL 格式檢查、事件多選、啟用切換、none / Basic / JWT auth 與既有 secret 保留提示
  - _Requirements: 4.2, 4.4, 4.8-4.10, 8.7_

- [x] 7.4 實作 secret rotate 與一次性顯示
  - 支援 payload signature secret 與 JWT auth secret rotate
  - 旋轉後提示使用者更新外部系統
  - 完成：建立與 payload secret rotate 後以 Dialog 一次性顯示 secret；JWT auth secret 可於編輯 Dialog 輸入新值後儲存更新
  - _Requirements: 4.3, 4.10, 6.6, 8.7_

- [x] 7.5 實作測試投遞
  - 顯示成功或失敗摘要
  - 完成：設定列提供測試投遞操作，顯示 delivered、event type、HTTP status、Authorization 狀態與錯誤摘要
  - _Requirements: 4.5, 8.7_

- [x] 7.6 實作投遞紀錄 Data Grid
  - 狀態、attempt、HTTP 狀態碼、錯誤摘要、下次重試時間
  - failed 可手動重試
  - 完成：新增選取 Webhook 後的 delivery Data Grid，支援狀態篩選、server pagination 與 failed / cancelled 手動重試
  - _Requirements: 5.2, 5.9, 8.6_

- [x] 7.7 補 Webhook i18n
  - 繁中、簡中、英文
  - 後端錯誤碼需映射為可讀文案
  - 完成：新增 `webhook` namespace，補齊繁中、簡中、英文文案，並將後端 Webhook 錯誤訊息映射為可讀文案
  - _Requirements: 8.8_

## 8. 審計、清理與驗證

- [x] 8.1 Webhook 設定變更寫入審計紀錄
  - 新增、修改、停用、刪除、rotate secret、測試投遞
  - 完成：Webhook create / update / delete / rotate secret / test delivery 會寫入 `system_audit_logs`，detail 僅保存非敏感摘要，不保存 secret、Basic password 或 JWT secret
  - _Requirements: 4.6, 9.1_

- [x] 8.2 實作通知與 delivery 保留清理
  - 依 global settings 清理
  - 完成：新增通知保留清理 service 與 `clean_expired_notifications` scheduler task，依 `notification.retention_days` 軟刪站內通知、依 `webhook.delivery_retention_days` 清理 Webhook delivery，並沿用 scheduler 分散式鎖
  - _Requirements: 9.5_

- [x] 8.3 後端單元與整合測試
  - event、resolver、API、Webhook validator、signer、retry
  - 完成：補 Webhook audit、測試投遞 audit、通知保留清理與 scheduler 清理任務測試；後端 `go test ./...` 通過
  - _Requirements: 1.1-9.5_

- [x] 8.4 前端型別檢查與建置
  - `npm run typecheck`
  - `npm run build`
  - 完成：`npm run typecheck` 與 `npm run build` 通過；build 僅有既有 bundle size 警告
  - _Requirements: 8.1-8.8_

- [ ] 8.5 手動驗收
  - 建立 Ticket 後收到站內通知
  - Header 未讀數更新
  - Webhook 測試投遞成功
  - Webhook 失敗後進入 retry / failed
  - 權限不足看不到 Webhook 管理入口
  - 待驗收：需使用瀏覽器與實際測試 Webhook endpoint 驗證端到端流程
  - _Requirements: 2.1-8.8_

## 9. Webhook JWT TTL 全域預設接線

- [x] 9.1 補 Webhook JWT TTL 預設契約文件
  - 明確區分 `config.yaml` 的 Webhook secret 與 `system_settings.webhook.jwt_ttl_seconds`
  - 說明全域設定只在單一 Webhook 未指定 TTL 時套用
  - 完成：需求 4.11 與設計稿已補上全域預設、單一 Webhook 快照與 fallback 規則
  - _Requirements: 4.11_

- [x] 9.2 後端建立 / 更新 Webhook 套用全域 TTL 預設
  - JWT Auth request 未帶 `jwt_ttl_seconds` 時讀取 `webhook.jwt_ttl_seconds`
  - 無效設定回退 300 秒
  - 實際 TTL 寫入加密 `auth_config_ciphertext`
  - 完成：`notification.Service` 已注入 setting resolver，建立 / 更新 JWT Webhook 時會套用全域 TTL 預設並快照保存
  - _Requirements: 4.11, 4.10, 6.8_

- [x] 9.3 前端允許 JWT TTL 留空使用全域預設
  - 新增 Webhook 表單不硬寫 300
  - 留空時 payload 不送 `jwt_ttl_seconds`
  - 編輯既有 Webhook 時仍顯示已保存 TTL
  - 完成：新增表單 TTL 預設為空白，空白時不送欄位；編輯既有 JWT Webhook 仍顯示後端回傳 TTL
  - _Requirements: 4.11, 8.7, 8.8_

- [x] 9.4 補驗證
  - 後端測試覆蓋全域 TTL 預設、明確 TTL 優先、無效全域設定回退
  - 前端型別檢查通過
  - 完成：已補後端單元測試並通過 `go test ./internal/notification`、`go test ./internal/scheduler`、`npm run typecheck`
  - _Requirements: 4.11, 8.7_
