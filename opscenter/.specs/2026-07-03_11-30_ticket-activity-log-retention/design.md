# Ticket Activity Log 保留清理設計

## 現況

已存在設定：

```text
system.log.ticket_activity_keep_days
```

目前已實作的 log cleanup 包含：

- `security_audit_logs`
- `system_audit_logs`
- `login_logs`
- `scheduler_logs`
- `user_notifications`
- `webhook_deliveries`

缺口是 `ticket_activities` 沒有接到 scheduler cleanup。`ensure_ticket_monthly_partitions` 只負責建立 `tickets` 與 `ticket_activities` 月分區，不負責刪除資料。

## 設計決策

### 任務拆分

新增獨立 scheduler task：

```text
clean_expired_ticket_activities
```

不把清理塞進 `ensure_ticket_monthly_partitions`，原因：

- 建立分區與刪除資料是不同維運行為。
- 清理任務需要可單獨啟停與手動觸發。
- scheduler log 能清楚反映清理結果。
- 失敗時不影響每月分區建立。

### 清理方式

第一版使用 parent table 條件刪除：

```sql
DELETE FROM ticket_activities
WHERE created_at < $1;
```

PostgreSQL 會依 `created_at` 條件做分區裁剪。此方式與既有 log cleanup 一致，實作風險低。

後續若資料量變大，再新增分區級最佳化，例如針對完整過期月份執行 detach / drop partition；該最佳化需另開需求，避免第一版同時處理大量 DDL 風險。

## 後端設計

### Scheduler domain

新增 task key：

```go
TaskCleanTicketActivities = "clean_expired_ticket_activities"
```

### Scheduler catalog

在 `taskCatalog()` 加入預設任務：

```go
{
  Name: "清理過期 Ticket Activity Log",
  TaskKey: TaskCleanTicketActivities,
  Cron: "0 2 * * *",
  Description: "依 system.log.ticket_activity_keep_days 清理 ticket_activities",
  Enabled: true,
}
```

### Cleanup target

擴充 `cleanupTarget()`：

```go
case TaskCleanTicketActivities:
  return "ticket_activities",
         "system.log.ticket_activity_keep_days",
         "SYSTEM_LOG_TICKET_ACTIVITY_KEEP_DAYS",
         0
```

預設值為 `0`，代表不清理。這是保守策略，避免 setting 缺值或解析失敗時誤刪歷史紀錄。

### Repository table whitelist

擴充 `DeleteBefore()` table whitelist，允許：

```text
ticket_activities
```

仍需維持 whitelist，不接受任意 table name，避免 SQL injection。

### Scheduler log

沿用既有 detail 格式：

```text
deleted_rows={N} keep_days={D} skipped={true|false} lock_task_key=... locked_by=...
```

若 `keep_days <= 0`，不呼叫 repository，回傳：

```text
deleted_rows=0 keep_days=0 skipped=true
```

## SQL 設計

新增後續 SQL 檔案，不修改既有已執行檔案。

建議檔名：

```text
opscenter-server/sql/0046_ticket_activity_retention_cleanup.sql
```

內容：

```sql
INSERT INTO schedulers (name, task_key, category, cron_expr, description, is_enabled)
VALUES
  (
    '清理過期 Ticket Activity Log',
    'clean_expired_ticket_activities',
    'system',
    '0 2 * * *',
    '依 system.log.ticket_activity_keep_days 清理 ticket_activities',
    TRUE
  )
ON CONFLICT (task_key) DO UPDATE SET
  name = EXCLUDED.name,
  category = EXCLUDED.category,
  cron_expr = EXCLUDED.cron_expr,
  description = EXCLUDED.description,
  is_enabled = TRUE,
  updated_at = NOW();
```

不新增 `system_settings` seed，因為 `system.log.ticket_activity_keep_days` 已由 `0007_create_setting_scheduler_tables.sql` 建立。

## 前端設計

不新增前端頁面。

排程管理頁透過既有 scheduler API 取得任務清單後，會顯示 `clean_expired_ticket_activities`。Global Setting 頁仍透過既有設定管理 `system.log.ticket_activity_keep_days`。

## 相容性

- `tickets` 不受影響。
- `attachments` 不受影響。
- `attachments.activity_id` 沒有資料庫外鍵約束；清理 activity 後，附件仍可透過 `ticket_id` 查詢。
- 詳情頁若依 activity 顯示附件操作歷程，清理後只是不顯示已過期 activity，不應造成 API 500。

## 測試規劃

### 單元測試

- `scheduler.Service.Trigger(TaskCleanTicketActivities)` 在 `keep_days = 30` 時呼叫 `DeleteBefore("ticket_activities", cutoff)`。
- `keep_days = 0` 時跳過清理，不呼叫 repository。
- lock busy 時寫入 skipped log。
- `repository.DeleteBefore()` 允許 `ticket_activities`，仍拒絕未知 table。

### 整合驗證

```sql
SELECT task_key, cron_expr, is_enabled
FROM schedulers
WHERE task_key = 'clean_expired_ticket_activities';
```

```sql
SELECT key, value, is_active
FROM system_settings
WHERE key = 'system.log.ticket_activity_keep_days';
```

```sql
SELECT COUNT(*) AS old_activity_rows
FROM ticket_activities
WHERE created_at < NOW() - INTERVAL '30 days';
```

## 風險與控制

- `ticket_activities` 可能包含留言與處理歷程，啟用清理前需由管理者確認保留天數。
- 預設 `0` 不清理，避免新版本部署後自動刪歷史資料。
- 若未來改成 drop partition，必須另行設計資料量評估、partition bound 解析與 rollback 流程。
