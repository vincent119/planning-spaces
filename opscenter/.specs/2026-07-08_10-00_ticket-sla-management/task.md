# Ticket SLA 管理與 SLA Checker Task

## 1. 現況收斂與契約確認

- [x] 1.1 確認 Ticket SLA 來源欄位
  - 確認 `ticket_types.applies_sla` 實際 schema 與 API response
  - 確認 Ticket priority、status、created_at、resolved_at、closed_at 欄位
  - 確認 deleted / cancelled 狀態表示方式
  - 完成：`ticket_types.applies_sla` 已存在於 `0026_create_ticket_schema.sql`，後端 ticket type domain / repository / delivery 與前端 ticket type API / UI 已接線
  - 完成：`tickets` 已有 `priority`、`status`、`created_at`、`resolved_at`、`closed_at`
  - 完成：`status` DB constraint 已包含 `open`、`in_progress`、`pending`、`escalated`、`resolved`、`closed`、`cancelled`
  - 完成：軟刪除使用 `tickets.deleted_at` / `deleted_by`；SLA checker 後續必須排除 `deleted_at IS NOT NULL` 與 `status = cancelled`
  - _Requirements: 2.1-2.4；Design: SLA 規則_

- [x] 1.2 確認 Ticket activity action type 擴充方式
  - 新增 `sla_response_breached`
  - 新增 `sla_resolve_breached`
  - 確認 action type 是否有 DB constraint 或前端 enum
  - 完成：`ticket_activities.action_type` 有 DB CHECK constraint，目前只包含舊版 `sla_breached`，未包含 `sla_response_breached` / `sla_resolve_breached`
  - 完成：Go `ticket` domain 目前沒有 SLA action 常數；後續需新增常數、repository 寫入路徑與 migration 更新 CHECK constraint
  - 完成：前端 activity response 以 string 顯示，沒有獨立 action enum；後續仍需補 i18n 呈現，避免詳情頁直接顯示 raw action
  - _Requirements: 7.1-7.3；Design: Notification 整合_

- [x] 1.3 確認 notification event 擴充契約
  - 新增 `ticket.sla_response_breached`
  - 新增 `ticket.sla_resolve_breached`
  - 確認 system actor 契約
  - 確認 dispatcher 收件人規則可沿用或擴充
  - 完成：`notification_events.event_type` 有 DB CHECK constraint，目前只允許 `ticket.created`、`ticket.status_changed`、`ticket.assigned`、`ticket.collaborator_added`、`ticket.comment_added`、`ticket.mentioned`
  - 完成：Go `notification.knownEventType()` 與前端 Webhook 事件型別清單尚未包含 SLA breach event
  - 完成：`notification_events.actor_user_id` 目前 `NOT NULL REFERENCES users(id)`，`TicketEventInput.ActorUserID` 也必填；SLA checker 後續必須補 system actor 契約，不得假冒任一 admin
  - 完成：dispatcher 的 `EventTicketStatusChanged` 收件人規則已包含建立者、被指派人、協作者與專案管理者，可作為 SLA breach 第一版收件人規則基礎
  - 完成：Webhook delivery 會依 `project_webhooks.event_types` 篩選事件，後續只需讓 SLA breach event 進入 event type 白名單與 UI 選項
  - _Requirements: 7.4-7.9；Design: Notification 整合_

## 2. 資料模型與 Migration

- [x] 2.1 補齊 `sla_configs`
  - project scope
  - priority、response_minutes、resolve_minutes、is_active
  - constraint：priority 合法、時間為正整數、resolve >= response
  - unique project + priority
  - 現況：`0026_create_ticket_schema.sql` 已有簡版 `sla_configs`，migration 應以 `CREATE TABLE IF NOT EXISTS` / `ALTER TABLE ... ADD COLUMN IF NOT EXISTS` 相容補齊，不得盲目重建
  - 完成：新增 `0048_ticket_sla_management.sql`，以相容 ALTER 補 `is_active`、created / updated audit 欄位、`deleted_at`、時間驗證 constraint 與 active unique index
  - _Requirements: 1.1-1.8；Design: sla_configs_

- [x] 2.2 新增 `ticket_sla_states`
  - ticket_id、project_id、priority、applies_sla
  - config_source、response_due_at、resolve_due_at
  - responded_at、resolved_at、breached_at 欄位
  - status、excluded_reason、last_checked_at
  - 完成：新增 `ticket_sla_states`，包含 `ticket_created_at` 以支援 Ticket 月分區定位，並補 priority、source、status、minutes、due order 與 applies_sla 欄位完整性 constraint
  - _Requirements: 3.1-3.7；Design: ticket_sla_states_

- [x] 2.3 新增 SLA checker scheduler seed
  - task key：`sla_checker`
  - cron：`*/5 * * * *`
  - 使用 `ON CONFLICT (task_key) DO UPDATE`
  - 完成：`0048_ticket_sla_management.sql` 已 seed `sla_checker`，並使用 `ON CONFLICT (task_key) DO UPDATE`
  - _Requirements: 6.1-6.3；Design: Scheduler_

- [x] 2.4 補 due time 查詢索引
  - response due partial index
  - resolve due partial index
  - 確認分區 Ticket 關聯查詢不會退化成全表掃描
  - 完成：新增 response due 與 resolve due partial index，另補 `ticket_id, ticket_created_at` 與 project status 查詢索引
  - _Requirements: 6.4-6.5；Design: ticket_sla_states_

## 3. 後端 SLA 設定 API

- [x] 3.1 建立 `internal/sla` 模組
  - domain、repository、service、delivery
  - 不放入 ticket package 內直接膨脹
  - 完成：新增 `internal/sla` bounded context，包含 domain、repository、service、delivery 與對應測試
  - _Requirements: 1.1-1.8；Design: 後端模組_

- [x] 3.2 實作 SLA config repository
  - 查詢 project SLA config
  - upsert P1-P4 config
  - 讀取 default config fallback
  - 完成：repository 支援查詢專案 SLA config、整包 upsert P1-P4，service 會在專案設定缺漏時套用 default fallback
  - _Requirements: 1.1-1.8；Design: API 設計_

- [x] 3.3 實作 `GET /api/v1/projects/:id/sla`
  - 回傳 project config 與 default config
  - 專案成員以上可讀
  - 完成：已註冊 `GET /api/v1/projects/:id/sla`，使用 project viewer 權限檢查，回傳 `items` 與 `defaults`
  - _Requirements: 8.1, 8.3_

- [x] 3.4 實作 `PUT /api/v1/projects/:id/sla`
  - 整包更新 P1-P4
  - 驗證 resolve >= response
  - 限 `admin`、`op_admin`、`project_manager`
  - 寫入 audit
  - 完成：已註冊 `PUT /api/v1/projects/:id/sla`，要求 P1-P4 整包更新，驗證正整數與 `resolve_minutes >= response_minutes`
  - 完成：使用 project manager 權限檢查，`admin` / `op_admin` 由既有 project access 規則放行
  - 完成：成功更新後寫入 `system_audit_logs`，保留 before / after 設定快照
  - _Requirements: 8.2-8.3, 1.5-1.8_

## 4. Ticket 生命週期接線

- [x] 4.1 Ticket 建立後建立 SLA state
  - applies_sla=true 才建立追蹤狀態
  - due time 以 Ticket created_at 起算
  - 保存 config source 與 minutes snapshot
  - 完成：Ticket 建立成功後呼叫 SLA lifecycle，依 `ticket_types.applies_sla` 建立 `ticket_sla_states` 快照；不套 SLA 的 Ticket 不建立 state
  - _Requirements: 2.1-2.2, 3.1-3.5_

- [x] 4.2 Ticket 狀態進入 `in_progress` 時更新 responded_at
  - 只寫入第一次 responded_at
  - 已 response breach 不清除 breach
  - 完成：狀態流轉後呼叫 SLA lifecycle，`in_progress` 以 `COALESCE(responded_at, changed_at)` 寫入首次回應時間
  - _Requirements: 4.1, 4.3, 4.6_

- [x] 4.3 Ticket resolved / closed 時更新 responded_at 與 resolved_at
  - 尚未 responded_at 時補 responded_at
  - 設定 resolved_at
  - 若兩種 SLA 都已 met，狀態更新為 met
  - 完成：`resolved` / `closed` 會補 `responded_at` 與 `resolved_at`，未 breach 時標記 `met`，已 breach 時保留 `breached`
  - _Requirements: 4.2-4.3, 5.1-5.3_

- [x] 4.4 Ticket cancelled 時標記 excluded
  - 不再被 checker 處理
  - 不列入 breach
  - 完成：`cancelled` 會將 SLA state 標記為 `excluded`，`excluded_reason = ticket_cancelled`
  - _Requirements: 2.4, 10.6_

- [x] 4.5 Ticket priority 變更時重新計算未完成 SLA due time
  - 依新 priority 讀取目前 project SLA config
  - 保留既有 breach
  - 寫入 Ticket activity
  - 完成：Ticket priority / project 變更後重新讀取目前 SLA config，僅重算尚未完成且未 breach 的 due time，並寫入 `sla_recalculated` activity
  - _Requirements: 3.6-3.7_

- [x] 4.6 Ticket type applies_sla 變更接線
  - true -> false 時 excluded
  - false -> true 時補建 SLA state
  - 寫入 audit 或 Ticket activity
  - 完成：Ticket type 變更後重新讀取新事件類型 SLA 契約；true -> false 標記 excluded，false -> true 補建 / 恢復 SLA state，並寫入 `sla_recalculated` activity
  - _Requirements: 2.5-2.6_

## 5. SLA Checker

- [x] 5.1 接入 scheduler task catalog
  - 新增 `sla_checker`
  - 使用既有 scheduler runner
  - 使用既有分散式鎖
  - 完成：新增 `TaskSLAChecker` 與 catalog 預設 cron `*/5 * * * *`，透過既有 `TriggerScheduled` 與 `scheduler_locks` 執行
  - _Requirements: 6.1-6.3_

- [x] 5.2 實作 due SLA 查詢
  - 查詢 response_due_at 已過期且未 responded / breached
  - 查詢 resolve_due_at 已過期且未 resolved / breached
  - 使用批次限制
  - 完成：repository 以 due time 查詢 response / resolve 逾期 state，join `tickets` 排除 deleted / cancelled，並使用 `sla.checker_batch_limit`
  - _Requirements: 6.4-6.5_

- [x] 5.3 實作 response breach idempotent update
  - 條件更新 `response_breached_at IS NULL`
  - 成功更新才產生活動；通知事件接點保留給 6.x
  - 完成：以 `response_breached_at IS NULL`、`responded_at IS NULL` 與 due time 條件更新；成功更新後同 transaction 寫入 `sla_response_breached` activity
  - 完成：通知 event 產生點以 `BreachPublisher` 介面保留；實際 notification event type、收件人與 system actor outbox 契約仍依 6.x 執行
  - _Requirements: 4.4-4.6, 6.6, 10.4_

- [x] 5.4 實作 resolve breach idempotent update
  - 條件更新 `resolve_breached_at IS NULL`
  - 成功更新才產生活動；通知事件接點保留給 6.x
  - 完成：以 `resolve_breached_at IS NULL`、`resolved_at IS NULL` 與 due time 條件更新；成功更新後同 transaction 寫入 `sla_resolve_breached` activity
  - 完成：通知 event 產生點以 `BreachPublisher` 介面保留；實際 notification event type、收件人與 system actor outbox 契約仍依 6.x 執行
  - _Requirements: 5.4-5.6, 6.6, 10.5_

- [x] 5.5 寫入 scheduler log
  - scanned
  - response_breached
  - resolve_breached
  - failed
  - lock_result
  - 完成：scheduler log detail 寫入 `scanned`、`response_breached`、`resolve_breached`、`failed`、`skipped`、`batch_limit`，並沿用既有 lock detail
  - _Requirements: 6.7-6.8_

## 6. Notification 整合

- [x] 6.1 擴充 notification event type
  - `ticket.sla_response_breached`
  - `ticket.sla_resolve_breached`
  - 不重寫 notification dispatcher
  - 完成：新增 Go notification event constants、`knownEventType()` 白名單與 `notification_events` CHECK constraint
  - 完成：SLA checker 透過 `BreachPublisher` 轉寫既有 notification outbox，dispatcher 流程未重寫
  - _Requirements: 7.4-7.5, 7.9_

- [x] 6.2 擴充 SLA breach 收件人解析
  - 沿用 status changed 收件人規則
  - 建立者、被指派人、協作者、專案管理者
  - 排除無效帳號與重複收件人
  - 完成：`ticket.sla_response_breached` 與 `ticket.sla_resolve_breached` 共用 status changed recipient candidate 規則
  - 完成：system actor 事件不做 actor self skip，仍保留無效帳號、專案成員與重複收件人排除
  - _Requirements: 7.6_

- [x] 6.3 補 system actor 支援
  - checker 產生的事件不得假冒任一 admin
  - 若現有 schema 必填 actor，需補 system actor 契約
  - 完成：`notification_events.actor_user_id` 改為可 NULL，SLA breach event 允許空 actor
  - 完成：dispatcher 查到空 actor 時以「系統」產生通知內容，Webhook payload 保留空 actor_user_id
  - _Requirements: 7.7_

- [x] 6.4 Webhook 事件類型選項加入 SLA breach
  - Webhook 管理表單可選擇 SLA breach event type
  - delivery payload 走既有 Webhook worker
  - 完成：前端 Webhook 事件選項加入兩個 SLA breach event，既有新建預設訂閱不自動擴大
  - 完成：Webhook delivery 繼續透過 `project_webhooks.event_types` 與既有 worker 投遞
  - _Requirements: 7.8-7.9_

- [x] 6.5 補通知 i18n
  - SLA response breach
  - SLA resolve breach
  - 繁中、簡中、英文
  - 完成：站內通知 event chip 與 Webhook event label 已補繁中、簡中、英文文案
  - _Requirements: 7.4-7.9_

## 7. Ticket API 與前端顯示

- [x] 7.1 Ticket list response 加入 SLA summary
  - applies_sla
  - status
  - next_due_at
  - response_breached / resolve_breached
  - 完成：Ticket repository 查詢 join `ticket_sla_states`，list response 回傳 `sla` 物件，包含 applies_sla、status、next_due_at、response_breached 與 resolve_breached
  - _Requirements: 8.4, 8.6_

- [x] 7.2 Ticket detail response 加入完整 SLA state
  - response due / responded / breached
  - resolve due / resolved / breached
  - config source
  - excluded reason
  - 完成：Ticket detail 與 list 共用 `sla` response 契約，額外帶出 response / resolve due、completed、breached、config_source 與 excluded_reason
  - _Requirements: 8.5-8.6_

- [x] 7.3 新增專案 SLA 設定頁
  - P1-P4 設定
  - response minutes
  - resolve minutes
  - 啟用狀態
  - 完成：新增 `/projects/:id/sla` 前端頁面，接 `GET /PUT /projects/:id/sla`，可維護 P1-P4 回應分鐘、解決分鐘與啟用狀態
  - 完成：補專案工作區頁籤入口與 `0049_ticket_sla_menu_entry.sql` 的 `tickets/sla` 側邊欄 menu seed，避免 SLA 頁面只有路由但沒有 UI 入口
  - _Requirements: 9.1-9.2_

- [x] 7.4 Ticket 列表顯示 SLA chip
  - 無 SLA
  - 正常
  - 即將逾時
  - 已逾時
  - 已完成
  - 排除
  - 完成：新增共用 `SLAStatusChip`，Ticket 列表新增 SLA 欄位呈現未套用、追蹤中、即將逾時、已逾時、已達成與排除狀態
  - _Requirements: 9.3, 9.5-9.7_

- [x] 7.5 Ticket 詳情顯示 SLA panel
  - response countdown
  - resolve countdown
  - breach time
  - config source
  - 完成：Ticket 詳情右側新增 SLA panel，顯示設定來源、回應 SLA、解決 SLA、期限、完成時間與逾時時間
  - _Requirements: 9.4-9.7_

- [x] 7.6 補前端 i18n
  - SLA 設定頁
  - SLA chip
  - SLA panel
  - API error mapping
  - 完成：補齊繁中、簡中、英文 `ticket` namespace 的 SLA 設定頁、chip、panel、驗證與錯誤文案
  - _Requirements: 8.7, 9.7_

## 8. 測試與驗收

- [x] 8.1 補 SLA config 後端測試
  - validation
  - fallback
  - permission
  - 完成：新增 `internal/sla` service / delivery 測試，覆蓋 default fallback、整包更新、權限、invalid minutes、unknown JSON field 與 audit
  - _Requirements: 1.1-1.8, 8.1-8.3, 10.1-10.2_

- [x] 8.2 補 Ticket lifecycle 測試
  - 建立 SLA state
  - responded_at
  - resolved_at
  - cancelled excluded
  - priority 變更
  - 完成：新增 SLA lifecycle service 測試與 Ticket service hook 測試，覆蓋建立快照、非 SLA 類型跳過、狀態流轉接線、priority / type 變更重算與刪除排除
  - _Requirements: 2.1-5.6, 10.3, 10.6_

- [x] 8.3 補 SLA checker 測試
  - response breach
  - resolve breach
  - idempotency
  - lock busy
  - scheduler log
  - 完成：`internal/sla` service / repository 測試覆蓋 response breach、resolve breach、idempotent no-op 與只在成功標記後 publish notification
  - 完成：`internal/scheduler` service 測試覆蓋 `sla_checker` batch limit、scheduler log detail 與 lock busy 時不執行 checker
  - _Requirements: 6.1-6.8, 10.4-10.7_

- [x] 8.4 補 notification 整合測試
  - breach 產生 notification event
  - event idempotency
  - system actor
  - Webhook event type filter
  - 完成：`internal/sla` notification publisher 測試覆蓋 response / resolve breach event type、idempotency key、payload due_at 與空 actor system 契約
  - 完成：`internal/notification` repository 測試覆蓋 SLA event 允許 system actor、outbox idempotency 與 Webhook delivery 依 `project_webhooks.event_types` 篩選 SLA event type
  - _Requirements: 7.1-7.9_

- [x] 8.5 補前端驗證
  - `npm run typecheck`
  - SLA 設定表單基本測試
  - Ticket list SLA chip 顯示
  - Ticket detail SLA panel 顯示
  - 完成：`npm run typecheck` 通過，驗證 SLA 設定頁、Ticket list SLA chip 與 Ticket detail SLA panel 的前端型別接線
  - 註記：前端目前沒有 test runner script，本次未新增測試框架；表單與顯示驗證先以 typecheck 與既有路由 / component 接線為準
  - _Requirements: 9.1-9.7, 10.8_

- [x] 8.6 補手動驗收 SQL
  - 查詢 project SLA config
  - 查詢 Ticket SLA state
  - 查詢 SLA breach activity
  - 查詢 notification event
  - 查詢 scheduler log
  - 完成：手動驗收 SQL 如下，執行前替換 `:project_id` 與 `:ticket_id`

```sql
-- 1. 專案 SLA 設定
SELECT project_id, priority, response_minutes, resolve_minutes, is_active, updated_at
FROM sla_configs
WHERE project_id = :'project_id'
  AND deleted_at IS NULL
ORDER BY priority;

-- 2. Ticket SLA state
SELECT
  t.id AS ticket_id,
  t.title,
  t.priority,
  t.status AS ticket_status,
  s.applies_sla,
  s.status AS sla_status,
  s.config_source,
  s.response_due_at,
  s.responded_at,
  s.response_breached_at,
  s.resolve_due_at,
  s.resolved_at,
  s.resolve_breached_at,
  s.excluded_reason,
  s.last_checked_at
FROM tickets t
LEFT JOIN ticket_sla_states s ON s.ticket_id = t.id
WHERE t.id = :'ticket_id';

-- 3. SLA breach activity
SELECT id, ticket_id, actor_id, action_type, field_changes, created_at
FROM ticket_activities
WHERE ticket_id = :'ticket_id'
  AND action_type IN ('sla_response_breached', 'sla_resolve_breached', 'sla_recalculated')
ORDER BY created_at DESC;

-- 4. SLA notification event
SELECT id, event_type, project_id, ticket_id, actor_user_id, idempotency_key, payload, processed_at, created_at
FROM notification_events
WHERE ticket_id = :'ticket_id'
  AND event_type IN ('ticket.sla_response_breached', 'ticket.sla_resolve_breached')
ORDER BY created_at DESC;

-- 5. SLA checker scheduler log
SELECT task_key, status, detail, started_at, finished_at
FROM scheduler_logs
WHERE task_key = 'sla_checker'
ORDER BY started_at DESC
LIMIT 20;
```
  - _Requirements: 10.9_

- [ ] 8.7 執行整體驗證
  - 後端相關 package tests
  - `go test ./...`
  - 前端 typecheck
  - 手動觸發 `sla_checker`
  - 完成：`go test ./internal/sla ./internal/scheduler ./internal/notification`
  - 完成：`go test ./internal/...`
  - 完成：`go test ./...`
  - 完成：`go test -coverprofile=/private/tmp/opscenter-sla-coverage.out ./internal/sla ./internal/scheduler ./internal/notification`，總覆蓋率 42.9%
  - 完成：`npm run typecheck`
  - 完成：`git diff --check`
  - 待驗收：live 手動觸發 `sla_checker` 會寫入資料庫，本次未擅自執行；需使用者授權後搭配 8.6 SQL 驗收
  - _Requirements: 10.1-10.9_
