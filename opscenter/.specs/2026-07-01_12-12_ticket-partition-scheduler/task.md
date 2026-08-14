# Ticket 分區維護與排程任務

## 1. 文件與缺失追蹤

- [x] 1.1 建立 Ticket 分區維護需求文件
  - 明確記錄 2026-07-01 建立 Ticket 因分區不存在失敗
  - _Requirements: 1.1-6.5_

- [x] 1.2 建立 Ticket 分區維護設計文件
  - 包含啟動兜底、背景 runner、分區 task、索引補齊
  - _Requirements: 1.1-6.5_

## 2. Migration 與 seed

- [x] 2.1 新增 `0042_ticket_partition_scheduler.sql`
  - 建立 / 補齊 `tickets_2026_07`
  - 建立 / 補齊 `ticket_activities_2026_07`
  - 預建 `tickets_2026_08` 與 `ticket_activities_2026_08`
  - 補齊所有月份分區索引
  - 完成：新增 idempotent migration，已涵蓋你手動建立 `tickets_2026_07` 後仍需補索引與 activity 分區的情境
  - _Requirements: 1.1-1.6_

- [x] 2.2 seed Ticket 分區維護排程
  - `task_key=ensure_ticket_monthly_partitions`
  - `cron_expr=5 0 * * *`
  - 預設啟用
  - 完成：`0042_ticket_partition_scheduler.sql` 已 seed 內建任務
  - _Requirements: 4.1-4.5_

## 3. 後端分區維護

- [x] 3.1 新增 scheduler repository 分區維護方法
  - 僅由 server 產生分區名稱
  - 建立目前月份與下一月份
  - 重複執行需成功
  - 完成：新增 `EnsureTicketMonthlyPartitions`，以 `YYYY_MM` server-side 產生表名並補齊兩張分區表與索引
  - _Requirements: 1.1-1.6, 5.4_

- [x] 3.2 新增 `ensure_ticket_monthly_partitions` task
  - 加入 `taskCatalog`
  - `Trigger` 可手動執行
  - detail 顯示 ensured months
  - 完成：scheduler service 已支援手動 trigger 與 scheduler log detail
  - _Requirements: 4.1-5.2_

- [x] 3.3 啟動時兜底建立分區
  - 啟動時執行一次分區維護 task
  - 成功 / 失敗寫入 scheduler logs
  - 失敗寫結構化日誌但不阻止服務啟動
  - 完成：`RegisterSystemSchedulers` 啟動時先執行 Ticket 分區維護，再執行 MFA 清理
  - _Requirements: 2.1-2.4_

## 4. 背景 scheduler runner

- [x] 4.1 新增最小 cron parser
  - 支援 `*`、`*/N`、單一數字、逗號分隔數字
  - 支援五欄 cron
  - 完成：新增 `runner.go` cron parser 並補測試
  - _Requirements: 3.3_

- [x] 4.2 新增背景 runner
  - 每分鐘讀取啟用任務
  - due task 自動執行
  - 同一分鐘同一 task 不重複執行
  - 支援 context cancel
  - 完成：新增 `Runner`，每分鐘讀取 tasks 並用 `lastRun` 防止同分鐘重複執行
  - _Requirements: 3.1-3.7_

- [x] 4.3 將 runner 接入 server 啟動流程
  - 不阻塞 HTTP server
  - graceful shutdown 時停止
  - 完成：server main 傳入 scheduler context，shutdown cleanup 時 cancel
  - _Requirements: 3.6-3.7_

## 5. 錯誤處理

- [x] 5.1 Ticket create 分區不存在錯誤收斂
  - 偵測 `no partition of relation`
  - 回傳可判讀錯誤
  - 完成：Ticket service 轉為 `ErrPartitionNotFound`，delivery 回 `503 ticket partition not found`
  - _Requirements: 5.3_

## 6. 驗證

- [x] 6.1 補 scheduler service 測試
  - 分區 task detail
  - cron parser
  - runner 同分鐘去重
  - 完成：已補 service / runner 測試
  - _Requirements: 6.1-6.3_

- [x] 6.2 補 repository SQL mock 測試
  - 建立 tickets / ticket_activities 分區與索引
  - 完成：已補 `EnsureTicketMonthlyPartitions` SQL mock 測試
  - _Requirements: 1.1-1.6_

- [x] 6.3 跑後端測試
  - `go test ./internal/scheduler ./internal/ticket`
  - 完成：已通過 `go test ./internal/scheduler ./internal/ticket` 與 `go test ./...`
  - _Requirements: 6.1-6.5_

- [x] 6.4 本機 DB 補救與驗收
  - 補齊 2026-07 / 2026-08 分區與索引
  - 查詢 `pg_inherits`
  - 建立 Ticket 驗證
  - 完成：本機 DB 已套用 `0042`，`tickets` 與 `ticket_activities` 均有 2026-07 / 2026-08 分區；`ensure_ticket_monthly_partitions` 已 seed 且啟用；2026-07 建立 Ticket 與 `created` activity 已成功
  - _Requirements: 6.4-6.5_

## 7. 多 Pod 排程分散式鎖

- [x] 7.1 新增 `0043_scheduler_distributed_locks.sql`
  - 建立 `scheduler_locks`
  - `scheduler_logs.status` 新增 `skipped`
  - seed `scheduler.lock_lease_seconds`
  - 完成：新增 idempotent migration，建立 lock table、擴充 scheduler log status，並 seed lease 與 instance id 設定
  - _Requirements: 7.1-7.10_

- [x] 7.2 擴充 scheduler repository lock 介面
  - `TryAcquireTaskLock`
  - `ReleaseTaskLock`
  - 原子取得 lock，lease 過期才能搶鎖
  - 完成：repository 以 PostgreSQL `INSERT ... ON CONFLICT ... WHERE locked_until < NOW() RETURNING` 原子搶鎖，並限制只有持有者可釋放
  - _Requirements: 7.1-7.4, 7.8-7.9_

- [x] 7.3 將 lock 套用到 scheduler 共用執行入口
  - 背景 runner
  - 手動 trigger
  - 啟動兜底
  - 不得只保護單一 task
  - 完成：`Service.Trigger`、`TriggerScheduled`、`TriggerStartup` 均走同一套 lock 流程，覆蓋全部 scheduler task
  - _Requirements: 7.1, 7.6-7.10_

- [x] 7.4 實作 instance id
  - 支援設定或環境變數
  - fallback hostname / process id
  - `scheduler_logs.detail` 或結構化日誌需可追蹤 locked_by
  - 完成：支援 `app.instance_id`、`APP_INSTANCE_ID`，fallback hostname + process id，log detail 帶 `locked_by` 與 `locked_until`
  - _Requirements: 7.4-7.5_

- [x] 7.5 實作 lock busy 行為
  - 背景 runner 未取得鎖時不得執行 task
  - 手動 trigger 未取得鎖時回 `409 scheduler task already running`
  - 啟動兜底未取得鎖時只寫 info log
  - 完成：lock busy 會寫 `skipped` log；runner 寫 info、不執行 task；手動 trigger 回 409；startup 寫 info
  - _Requirements: 7.2, 7.5-7.7_

- [x] 7.6 補 scheduler lock 測試
  - 多 runner 競爭同一 task 只能一個成功
  - lease 過期後可重新取得
  - 非持有者不得釋放鎖
  - 手動 trigger lock busy 回 409
  - 完成：補 repository lock SQL mock、runner 多實例競爭、service lock busy、delivery 409 測試
  - _Requirements: 7.1-7.10_

- [x] 7.7 補管理頁與 API 狀態顯示
  - scheduler log 可顯示 `skipped`
  - i18n 補 `skipped` 與 task running 訊息
  - 完成：前端 scheduler log 型別支援 `skipped`，狀態 chip 與 zh-TW / en / zh-CN i18n 已補；手動 trigger 409 會顯示在地化訊息
  - _Requirements: 7.5_

## 8. Runtime 分區 DDL 權限收斂

- [x] 8.1 新增 `0044_ticket_partition_security_definer.sql`
  - 建立 `ensure_ticket_monthly_partition(DATE)` SECURITY DEFINER 函式
  - 授權 app role 只執行函式，不授權 schema `CREATE`
  - 完成：新增 migration，由受控函式承接分區 DDL，並補 app/admin group 分區權限
  - _Requirements: 8.1-8.5_

- [x] 8.2 改 repository 分區維護呼叫受控函式
  - `EnsureTicketMonthlyPartitions` 不直接執行 `CREATE TABLE ... PARTITION OF`
  - 每個月份只呼叫 `ensure_ticket_monthly_partition`
  - 完成：repository 每月呼叫 `SELECT ensure_ticket_monthly_partition(?)`
  - _Requirements: 8.1-8.4_

- [x] 8.3 補 repository 測試
  - 確認 repository 呼叫受控函式
  - 確認不再直接產生 partition DDL
  - 完成：repository SQL mock 已改為驗證 function call
  - _Requirements: 8.6_

- [x] 8.4 驗證
  - 跑 scheduler 測試
  - 跑完整 Go 測試
  - 完成：`go test ./internal/scheduler` 與 `go test ./...` 已通過
  - _Requirements: 8.1-8.6_
