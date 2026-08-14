# Ticket Activity Log 保留清理驗收紀錄

## 自動驗證

| 項目 | 結果 | 說明 |
| --- | --- | --- |
| `go test ./internal/scheduler` | 通過 | 覆蓋 Ticket Activity cleanup、跳過清理、lock busy 與 repository whitelist |
| `go test ./...` | 通過 | 確認 scheduler 變更未破壞其他後端套件 |

## SQL 驗收查詢

### 1. 確認 scheduler task seed

```sql
SELECT name, task_key, category, cron_expr, description, is_enabled
FROM schedulers
WHERE task_key = 'clean_expired_ticket_activities';
```

預期：

- 有 1 筆資料。
- `cron_expr = '0 2 * * *'`
- `is_enabled = true`

### 2. 確認 retention setting

```sql
SELECT key, value, category, value_type, description, is_active
FROM system_settings
WHERE key = 'system.log.ticket_activity_keep_days';
```

預期：

- 有 1 筆資料。
- 預設 `value = '0'`。
- `is_active = true`。

### 3. 清理前統計

```sql
WITH params AS (
  SELECT
    COALESCE(NULLIF(value, '')::int, 0) AS keep_days
  FROM system_settings
  WHERE key = 'system.log.ticket_activity_keep_days'
    AND is_active = TRUE
)
SELECT
  (SELECT COUNT(*) FROM tickets) AS tickets_before,
  (SELECT COUNT(*) FROM attachments) AS attachments_before,
  (SELECT COUNT(*) FROM ticket_activities) AS ticket_activities_before,
  (
    SELECT COUNT(*)
    FROM ticket_activities, params
    WHERE params.keep_days > 0
      AND ticket_activities.created_at < NOW() - (params.keep_days || ' days')::interval
  ) AS expired_ticket_activities_before;
```

### 4. 手動觸發後統計

觸發 `clean_expired_ticket_activities` 後執行：

```sql
WITH params AS (
  SELECT
    COALESCE(NULLIF(value, '')::int, 0) AS keep_days
  FROM system_settings
  WHERE key = 'system.log.ticket_activity_keep_days'
    AND is_active = TRUE
)
SELECT
  (SELECT COUNT(*) FROM tickets) AS tickets_after,
  (SELECT COUNT(*) FROM attachments) AS attachments_after,
  (SELECT COUNT(*) FROM ticket_activities) AS ticket_activities_after,
  (
    SELECT COUNT(*)
    FROM ticket_activities, params
    WHERE params.keep_days > 0
      AND ticket_activities.created_at < NOW() - (params.keep_days || ' days')::interval
  ) AS expired_ticket_activities_after;
```

預期：

- `tickets_before = tickets_after`
- `attachments_before = attachments_after`
- `ticket_activities_after <= ticket_activities_before`
- 若 `keep_days > 0`，`expired_ticket_activities_after = 0`

### 5. Scheduler log 查詢

```sql
SELECT task_key, status, detail, started_at, finished_at
FROM scheduler_logs
WHERE task_key = 'clean_expired_ticket_activities'
ORDER BY started_at DESC
LIMIT 5;
```

預期：

- `keep_days = 0` 時，`detail` 包含 `deleted_rows=0 keep_days=0 skipped=true`。
- `keep_days > 0` 時，`detail` 包含 `deleted_rows={N} keep_days={D} skipped=false`。
- lock busy 時，`status = 'skipped'` 且 `detail` 包含 `lock_result=busy`。

## 手動驗收紀錄

本次未直接連線 DB，也未觸發刪資料排程。以下為實際環境執行時需填入的紀錄。

| 情境 | 執行結果 | 備註 |
| --- | --- | --- |
| `keep_days = 0` 跳過結果 | 待執行 | 預期 scheduler log 顯示 `skipped=true` |
| `keep_days > 0` 刪除筆數 | 待執行 | 預期只刪 `ticket_activities` 過期資料 |
| `tickets` 筆數不變 | 待執行 | 比對清理前後 `COUNT(*)` |
| `attachments` 筆數不變 | 待執行 | 比對清理前後 `COUNT(*)` |
