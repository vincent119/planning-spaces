# Ticket 分區維護與排程設計

## 目標

修復 Ticket 月分區跨月失效問題，並補齊排程自動執行能力。此修復要確保：

- 服務啟動時先兜底建立目前月份與下一月份分區。
- 背景 runner 會依 `schedulers` 表自動執行任務。
- Ticket 分區維護任務可由管理頁手動 trigger。
- 手動已建立的分區仍會被補齊索引與對應 activity 分區。

## 資料表範圍

需要維護的分區表：

```text
tickets
ticket_activities
```

分區命名：

```text
tickets_YYYY_MM
ticket_activities_YYYY_MM
```

分區邊界：

```text
FROM YYYY-MM-01
TO   next_month_YYYY-MM-01
```

## 分區索引

每個 `tickets_YYYY_MM` 需具備：

```sql
CREATE INDEX IF NOT EXISTS idx_tickets_YYYY_MM_project_status_created
  ON tickets_YYYY_MM (project_id, status, created_at);
CREATE INDEX IF NOT EXISTS idx_tickets_YYYY_MM_type
  ON tickets_YYYY_MM (ticket_type_id);
CREATE INDEX IF NOT EXISTS idx_tickets_YYYY_MM_assignee
  ON tickets_YYYY_MM (assignee_id);
CREATE INDEX IF NOT EXISTS idx_tickets_YYYY_MM_services_level
  ON tickets_YYYY_MM (services_level_id);
CREATE INDEX IF NOT EXISTS idx_tickets_YYYY_MM_ticket_resource
  ON tickets_YYYY_MM (ticket_resource_id);
CREATE INDEX IF NOT EXISTS idx_tickets_YYYY_MM_deleted_at
  ON tickets_YYYY_MM (deleted_at);
```

每個 `ticket_activities_YYYY_MM` 需具備：

```sql
CREATE INDEX IF NOT EXISTS idx_ticket_activities_YYYY_MM_ticket_created
  ON ticket_activities_YYYY_MM (ticket_id, created_at);
CREATE INDEX IF NOT EXISTS idx_ticket_activities_YYYY_MM_actor_created
  ON ticket_activities_YYYY_MM (actor_id, created_at);
```

## 後端設計

### Repository

在 scheduler repository 補上：

```go
EnsureTicketMonthlyPartitions(ctx context.Context, months []time.Time) (PartitionResult, error)
```

規則：

- `months` 由 service 產生，不接受 API request 直接指定任意表名。
- 每個 month 先正規化到該月 1 日。
- runtime repository 不直接執行 `CREATE TABLE ... PARTITION OF`，避免 app role 需要 schema `CREATE` 或父表 owner 權限。
- repository 逐月呼叫 DB function `ensure_ticket_monthly_partition(month_start DATE)`。
- SQL table name 只能由 DB function 依 `YYYY_MM` 格式組合。
- `CREATE TABLE IF NOT EXISTS ... PARTITION OF ...` 與 `CREATE INDEX IF NOT EXISTS ...` 都在 DB function 內執行。
- 若 `tickets_YYYY_MM` 已存在，仍繼續建立索引與 `ticket_activities_YYYY_MM`。

### Runtime DDL 權限

runtime app role 不應直接取得 `opscenter` schema `CREATE`。分區 DDL 由 migration 建立的 `SECURITY DEFINER` 函式承接：

```sql
ensure_ticket_monthly_partition(p_month DATE)
```

規則：

- 函式 owner 必須是 migration owner 或具備 parent table DDL 權限的 DB role。
- 函式使用固定 `search_path = opscenter, pg_temp`。
- 函式只接受月份日期，不接受任意表名。
- 函式內部用 `date_trunc('month', p_month)` 正規化月份。
- 函式內部用 `format('%I', identifier)` 組合分區表名與索引名稱。
- app role 只授權 `EXECUTE`，不授權 schema `CREATE`。
- 分區建立後，若 `app_group` / `admin_group` 存在，函式需補齊該分區資料操作權限。

### Service

新增 task key：

```text
ensure_ticket_monthly_partitions
```

`Service.Trigger()` 支援此 task：

- 建立目前月份與下一月份。
- 成功 detail 範例：

```text
ensured_months=2026-07,2026-08 tables=tickets,ticket_activities
```

### 啟動兜底

`RegisterSystemSchedulers` 建立 scheduler service 後，需呼叫：

```go
service.Trigger(context.Background(), TaskEnsureTicketPartitions)
```

此動作與 MFA startup cleanup 一樣寫入 `scheduler_logs`。失敗時寫結構化錯誤日誌，但不阻止 server 啟動。

## 背景 runner

目前系統沒有 `robfig/cron` 依賴。第一版不新增外部套件，先實作最小 runner：

- 每分鐘 tick 一次。
- 每次從 repository 讀取 tasks，確保管理頁更新 cron / enabled 後可生效。
- 支援常見五欄 cron：
  - `*`
  - `*/N`
  - 單一數字
  - 逗號分隔數字
- 以目前時間分鐘為執行粒度。
- 使用記憶體 `lastRun[taskKey]minuteKey` 防止同一分鐘重複執行。
- 每個 due task 以 goroutine 執行，避免慢任務阻塞下一個任務。
- runner context cancel 後停止 tick。

若後續需要完整 cron 語法，再導入成熟套件；第一版避免引入新 dependency。

## 分散式鎖設計

多 Pod 部署時，單一 Pod 內的 `lastRun` 只能避免本 Pod 同分鐘重複執行，不能阻止其他 Pod 同時執行同一個 task。所有 scheduler task 需在共用執行入口取得分散式鎖。

第一版採用 PostgreSQL lease lock table，不採用 advisory lock。原因：

- 排程定義與執行紀錄都已在 PostgreSQL。
- lease table 容易測試與查詢。
- 不依賴 connection pool 必須持有同一條連線才能 unlock。
- Pod 異常退出後可依 `locked_until` 自動釋放。

### `scheduler_locks`

```sql
CREATE TABLE scheduler_locks (
  task_key VARCHAR(64) PRIMARY KEY,
  locked_by VARCHAR(128) NOT NULL,
  locked_until TIMESTAMPTZ NOT NULL,
  acquired_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_scheduler_locks_locked_until ON scheduler_locks (locked_until);
```

欄位規則：

- `task_key`：排程任務 key。
- `locked_by`：Pod / instance 識別，建議來源為 hostname 加 process id 或設定值。
- `locked_until`：lease 到期時間。
- `acquired_at`：本次取得鎖時間。
- `updated_at`：最後更新時間。

### 取得鎖

鎖取得必須是單一 SQL 原子操作。語意：

```sql
INSERT INTO scheduler_locks (task_key, locked_by, locked_until)
VALUES ($1, $2, NOW() + $3::interval)
ON CONFLICT (task_key) DO UPDATE
SET locked_by = EXCLUDED.locked_by,
    locked_until = EXCLUDED.locked_until,
    acquired_at = NOW(),
    updated_at = NOW()
WHERE scheduler_locks.locked_until < NOW()
RETURNING task_key;
```

有回傳 row 代表取得鎖；沒有 row 代表其他 Pod 仍持有有效 lease。

### 釋放鎖

任務結束後只允許目前持有者釋放：

```sql
DELETE FROM scheduler_locks
WHERE task_key = $1
  AND locked_by = $2;
```

若釋放失敗，需記錄 warning，但不得將已完成 task 視為失敗。

### lease 時間

第一版預設：

```text
scheduler.lock_lease_seconds = 600
```

此設定代表單一 task 預設最多持鎖 10 分鐘。未來若有長任務，需要補 heartbeat 延長 lease 或 task-level lease 設定。

### 執行入口

鎖必須放在 `Service.Trigger()` 或其下層共用入口，而不是只放在 runner。原因：

- 背景 runner 需要鎖。
- 管理頁手動 trigger 需要鎖。
- 啟動兜底需要鎖。
- 未來任何呼叫 scheduler service 的入口都需要鎖。

建議流程：

```text
Trigger(task)
  -> find task
  -> validate enabled
  -> try acquire lock
  -> lock busy: 回傳 skipped log 或明確 ErrTaskLocked
  -> execute task
  -> record scheduler log
  -> release lock
```

### 鎖忙碌行為

背景 runner：

- 未取得鎖時不執行 task。
- 可選擇寫入 `scheduler_logs` 狀態 `skipped`，或只寫結構化 debug log。
- 第一版建議新增 `scheduler_logs.status='skipped'`，讓管理頁可看到多 Pod 競爭結果。

手動 trigger：

- 未取得鎖時回 `409 scheduler task already running`。
- 不得等待鎖釋放，避免管理 API 長時間占用 request。

啟動兜底：

- 未取得鎖時寫結構化 info log，不視為啟動失敗。

### 狀態值調整

`scheduler_logs.status` 需新增：

```text
skipped
```

用途：

- lock busy。
- task disabled 時由 runner 跳過通常不寫 log；手動 trigger disabled 仍回錯。

### instance id

新增 scheduler service option：

```go
WithInstanceID(instanceID string)
```

來源優先序：

1. 設定 `app.instance_id` 或環境變數。
2. hostname。
3. hostname + process id。

日誌需包含：

```text
task_key
locked_by
locked_until
lock_result
```

## Scheduler seed

新增 migration：

```text
0042_ticket_partition_scheduler.sql
```

內容：

- 補 2026-07 與 2026-08 分區與索引，修復目前環境。
- seed `ensure_ticket_monthly_partitions` 到 `schedulers`。

預設 cron：

```text
5 0 * * *
```

每天 00:05 執行，因為任務 idempotent，日執行可降低月初排程錯過造成的風險。服務啟動仍會兜底建立目前與下月。

鎖機制需新增後續 migration：

```text
0043_scheduler_distributed_locks.sql
```

內容：

- 建立 `scheduler_locks`。
- 將 `scheduler_logs.status` check constraint 擴充為 `running`、`success`、`failed`、`skipped`。
- seed `scheduler.lock_lease_seconds` global setting。

## 錯誤處理

Ticket create 若遇到 PostgreSQL partition missing：

- service log 需保留原始錯誤。
- delivery 可回傳明確錯誤碼與訊息，例如 `ticket partition not found`。
- 這不取代分區維護，只是避免使用者看到不明確 500。

## 驗證方式

SQL 驗證：

```sql
SELECT inhrelid::regclass AS partition_name, pg_get_expr(c.relpartbound, c.oid) AS partition_bound
FROM pg_inherits
JOIN pg_class c ON c.oid = inhrelid
WHERE inhparent = 'tickets'::regclass
ORDER BY partition_name;

SELECT inhrelid::regclass AS partition_name, pg_get_expr(c.relpartbound, c.oid) AS partition_bound
FROM pg_inherits
JOIN pg_class c ON c.oid = inhrelid
WHERE inhparent = 'ticket_activities'::regclass
ORDER BY partition_name;
```

API 驗收：

- 建立 2026-07 Ticket 應成功。
- Ticket 建立後需寫入 `ticket_activities`。
- 排程管理頁需看到 `ensure_ticket_monthly_partitions`。
- 手動 trigger 此任務應成功，且 detail 顯示 ensured months。
- 多 Pod 或雙 runner 模擬時，同一 task 同一時間只能一個取得鎖並執行。
