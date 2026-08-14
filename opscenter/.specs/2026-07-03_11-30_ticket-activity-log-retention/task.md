# Ticket Activity Log 保留清理 Task

## 1. Scheduler 任務模型

- [x] 1.1 新增 Ticket Activity cleanup task key
  - 在 scheduler domain 新增 `TaskCleanTicketActivities`
  - task key 固定為 `clean_expired_ticket_activities`
  - 完成：已新增 scheduler domain 常數
  - 對應需求：1.7

- [x] 1.2 擴充 scheduler catalog
  - 在 `taskCatalog()` 加入「清理過期 Ticket Activity Log」
  - 預設 cron：`0 2 * * *`
  - description 明確標示依 `system.log.ticket_activity_keep_days` 清理 `ticket_activities`
  - 完成：已加入預設任務 catalog；清理執行邏輯於 2.1 完成
  - 對應需求：1.5、1.7、1.8

## 2. 後端清理邏輯

- [x] 2.1 接入 `system.log.ticket_activity_keep_days`
  - 在 `cleanupTarget()` 加入 `TaskCleanTicketActivities`
  - table：`ticket_activities`
  - setting key：`system.log.ticket_activity_keep_days`
  - env fallback：`SYSTEM_LOG_TICKET_ACTIVITY_KEEP_DAYS`
  - fallback keep days：`0`
  - 完成：已接入 generic log cleanup 流程，`0` 維持跳過清理
  - 對應需求：1.1、1.2、1.3

- [x] 2.2 擴充 repository table whitelist
  - `DeleteBefore()` 允許 `ticket_activities`
  - 保留 whitelist，不接受任意 table name
  - 未知 table 仍回 `ErrInvalidInput`
  - 完成：已允許 `ticket_activities`，未知 table 防護不變
  - 對應需求：1.3、1.4

- [x] 2.3 確認清理範圍不包含主資料
  - 不新增任何刪除 `tickets` 的邏輯
  - 不新增任何刪除 `attachments` 的邏輯
  - 不新增任何刪除 `ticket_collaborators` 的邏輯
  - 完成：本次僅新增 `ticket_activities` whitelist，未新增其他 Ticket 主資料刪除邏輯
  - 對應需求：1.4、1.9

## 3. SQL 變更

- [x] 3.1 新增 scheduler seed SQL
  - 新增 `opscenter-server/sql/0046_ticket_activity_retention_cleanup.sql`
  - `INSERT INTO schedulers ... ON CONFLICT (task_key) DO UPDATE`
  - 不修改既有 `0007`、`0013`、`0042` SQL
  - 完成：已新增 `clean_expired_ticket_activities` scheduler seed
  - 對應需求：1.7、1.8

- [x] 3.2 確認不新增重複 setting
  - 不新增 `system.log.ticket_activity_keep_days`
  - 沿用既有 `0007_create_setting_scheduler_tables.sql` seed
  - 完成：SQL 搜尋確認只有 `0007` 建立此 setting
  - 對應需求：1.1

## 4. 後端測試

- [x] 4.1 補 service 啟用清理測試
  - `system.log.ticket_activity_keep_days = 30`
  - 觸發 `TaskCleanTicketActivities`
  - 驗證呼叫 `DeleteBefore("ticket_activities", now - 30 days)`
  - 驗證 scheduler log detail 包含 `deleted_rows`、`keep_days=30`、`skipped=false`
  - 完成：已新增 `TestServiceTriggerCleansTicketActivities`
  - 對應需求：1.3、1.5、1.10

- [x] 4.2 補 service 跳過清理測試
  - `system.log.ticket_activity_keep_days = 0`
  - 觸發 `TaskCleanTicketActivities`
  - 驗證不呼叫 repository delete
  - 驗證 scheduler log detail 包含 `keep_days=0`、`skipped=true`
  - 完成：已新增 `TestServiceTriggerSkipsTicketActivityCleanupWhenKeepDaysIsZero`
  - 對應需求：1.2、1.10

- [x] 4.3 補 lock busy 測試
  - 模擬同任務 lock busy
  - 驗證不執行 delete
  - 驗證寫入 skipped scheduler log
  - 完成：已新增 `TestServiceTriggerSkipsTicketActivityCleanupWhenTaskLockBusy`
  - 對應需求：1.6、1.10

- [x] 4.4 補 repository whitelist 測試
  - `DeleteBefore("ticket_activities", cutoff)` 可執行
  - `DeleteBefore("unknown_table", cutoff)` 回 `ErrInvalidInput`
  - 完成：已新增 `TestRepositoryDBDeleteBeforeAllowsTicketActivitiesOnlyThroughWhitelist`
  - 對應需求：1.4、1.10

## 5. 驗證

- [x] 5.1 執行後端測試
  - 執行 `go test ./internal/scheduler`
  - 若影響共用 interface，執行 `go test ./...`
  - 完成：兩項測試皆已通過
  - 對應需求：1.10

- [x] 5.2 補 SQL 驗收查詢
  - 查詢 `schedulers` 是否存在 `clean_expired_ticket_activities`
  - 查詢 `system_settings` 是否存在並啟用 `system.log.ticket_activity_keep_days`
  - 查詢清理前後 `ticket_activities` 過期筆數
  - 完成：已補於 `verification.md`
  - 對應需求：驗收條件

- [x] 5.3 手動驗收記錄
  - 記錄 `keep_days = 0` 跳過結果
  - 記錄 `keep_days > 0` 刪除筆數
  - 記錄 `tickets` 筆數不變
  - 記錄 `attachments` 筆數不變
  - 完成：已補於 `verification.md`；本次未直接連線 DB，也未觸發刪資料排程
  - 對應需求：驗收條件
