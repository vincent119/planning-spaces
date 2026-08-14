# Ticket 分區維護與排程需求

## 文件定位

此 spec 補齊 Ticket 月分區維護缺失。原始設計已要求分區由內建排程自動建立，但目前實作只有 2026-06 初始分區，沒有 Ticket 分區建立任務，也沒有背景 cron runner，導致 2026-07-01 建立 Ticket 時發生 `no partition of relation "tickets" found for row`。

本文件不修改初始 spec，將缺失收斂成可執行修復項目。

## 目前缺失

- `tickets` 只存在 `tickets_2026_06` 初始分區。
- `ticket_activities` 只存在 `ticket_activities_2026_06` 初始分區。
- `schedulers` seed 沒有 Ticket 分區維護任務。
- `scheduler.taskCatalog()` 沒有 Ticket 分區維護任務。
- 系統目前只提供排程管理 API，沒有背景 runner 依 `schedulers.cron_expr` 自動執行。
- 啟動流程沒有兜底建立當月與下月 Ticket 分區。
- 當月 `tickets` 分區即使手動建立，也可能缺少索引與對應的 `ticket_activities` 分區。
- 多 Pod 部署時，所有 Pod 都會各自執行啟動兜底與到期排程；目前沒有跨 Pod 分散式鎖，可能造成重複執行、重複投遞或重複寫入排程紀錄。
- runtime app role 只有 `opscenter` schema `USAGE`，沒有 `CREATE`；若 scheduler 直接執行 `CREATE TABLE ... PARTITION OF`，會在啟動兜底或排程中發生 `permission denied for schema opscenter`。

## 需求 1：Ticket 分區建立

系統需自動維護 Ticket 相關月分區，避免跨月後 Ticket 主流程中斷。

### 驗收條件

- [ ] 1.1 系統需支援建立指定月份的 `tickets_YYYY_MM` 分區。
- [ ] 1.2 系統需支援建立指定月份的 `ticket_activities_YYYY_MM` 分區。
- [ ] 1.3 建立分區時必須同步建立該月份需要的查詢索引。
- [ ] 1.4 建立分區需具備 idempotent 行為，重複執行不得失敗。
- [ ] 1.5 若 `tickets_YYYY_MM` 已被手動建立但索引缺漏，任務仍需補齊索引。
- [ ] 1.6 若 `tickets_YYYY_MM` 已存在，任務仍需檢查並建立對應的 `ticket_activities_YYYY_MM`。

## 需求 2：啟動兜底

服務啟動時需先確保近期分區存在，避免排程錯過執行時間後造成業務中斷。

### 驗收條件

- [ ] 2.1 服務啟動時需建立目前月份與下一月份的 Ticket 分區。
- [ ] 2.2 啟動兜底失敗需寫入結構化錯誤日誌。
- [ ] 2.3 啟動兜底成功或失敗需寫入 `scheduler_logs`。
- [ ] 2.4 啟動兜底不得因分區已存在而回報失敗。

## 需求 3：排程自動執行

系統需真的依排程表自動執行任務，而不是只提供手動 trigger API。

### 驗收條件

- [ ] 3.1 系統需啟動背景 scheduler runner。
- [ ] 3.2 runner 需定期讀取 `schedulers` 表中的啟用任務。
- [ ] 3.3 runner 需依 `cron_expr` 判斷是否該執行。
- [ ] 3.4 runner 同一分鐘內不得重複執行同一個 task。
- [ ] 3.5 runner 執行成功或失敗都需寫入 `scheduler_logs`。
- [ ] 3.6 runner 不得阻塞 HTTP server 啟動與關閉。
- [ ] 3.7 runner 需支援 graceful shutdown。

## 需求 4：分區維護排程

Ticket 分區維護需作為系統內建排程任務。

### 驗收條件

- [ ] 4.1 `schedulers` seed 需新增 Ticket 分區維護任務。
- [ ] 4.2 `scheduler.taskCatalog()` 需包含 Ticket 分區維護任務。
- [ ] 4.3 分區維護任務需每次建立目前月份與下一月份分區。
- [ ] 4.4 分區維護任務預設啟用。
- [ ] 4.5 管理者可在排程管理頁看到此任務並可手動 trigger。

## 需求 5：可觀測與錯誤處理

維運人員需要能追蹤分區維護是否正常執行。

### 驗收條件

- [ ] 5.1 分區維護成功時，`scheduler_logs.detail` 需包含建立或確認過的月份與資料表。
- [ ] 5.2 分區維護失敗時，`scheduler_logs.detail` 需包含錯誤摘要。
- [ ] 5.3 當 Ticket 建立遇到分區不存在時，後端應回傳可判讀錯誤，不得只回泛用 500。
- [ ] 5.4 分區維護 SQL 不得接受外部輸入直接拼接表名；月份與分區名稱只能由 server 依時間產生。

## 需求 6：驗證

修復需覆蓋跨月建立 Ticket 的核心風險。

### 驗收條件

- [ ] 6.1 單元測試需覆蓋分區任務建立目前月份與下一月份。
- [ ] 6.2 單元測試需覆蓋已存在分區時仍補索引。
- [ ] 6.3 單元測試需覆蓋 runner 的同一分鐘去重。
- [ ] 6.4 手動驗收需確認 2026-07 可建立 Ticket 與 Ticket Activity。
- [ ] 6.5 手動驗收需確認排程管理頁可看到 Ticket 分區維護任務。

## 需求 7：多 Pod 排程分散式鎖

系統可能以多 Pod 部署，所有排程任務在同一個到期時間只能由一個 Pod 執行，避免重複清理、重複匯入、重複 Webhook 投遞或重複產生排程副作用。

### 驗收條件

- [ ] 7.1 所有 scheduler task 在執行前都必須先取得跨 Pod 分散式鎖。
- [ ] 7.2 未取得鎖的 Pod 不得執行該 task。
- [ ] 7.3 鎖需有 lease timeout，避免執行中的 Pod 異常退出後永久卡住。
- [ ] 7.4 鎖需記錄持有者識別、到期時間與最後更新時間。
- [ ] 7.5 取得鎖、釋放鎖與因鎖被占用而跳過執行需有可觀測紀錄。
- [ ] 7.6 手動 trigger 與背景 runner 都需使用同一套鎖機制。
- [ ] 7.7 啟動兜底任務也需使用鎖，避免多 Pod 啟動時同時執行分區 DDL。
- [ ] 7.8 鎖不得只存在單一 Pod 記憶體內；必須使用共享狀態，例如 PostgreSQL `scheduler_locks` 表。
- [ ] 7.9 鎖取得需具備原子性，多 Pod 同時競爭時只能有一個成功。
- [ ] 7.10 鎖機制需覆蓋所有現有與未來 scheduler task，不得只針對 Ticket 分區任務。

## 需求 8：Runtime 分區 DDL 權限收斂

Ticket 分區維護需要在 runtime 自動建立分區，但服務使用的 DB role 不應直接取得 schema `CREATE` 或父表 owner 權限。

### 驗收條件

- [ ] 8.1 runtime app role 不需 schema `CREATE` 權限也能執行 Ticket 分區維護。
- [ ] 8.2 分區建立 DDL 需收斂在受控資料庫函式中，並以 migration owner 權限執行。
- [ ] 8.3 app role 只需取得受控函式的 `EXECUTE` 權限。
- [ ] 8.4 受控函式需自行產生分區表名與索引名稱，不接受外部傳入任意表名。
- [ ] 8.5 新建立的月份分區需補齊 app / admin role 對該分區的資料操作權限。
- [ ] 8.6 repository 測試需確認 runtime 不再直接送出 `CREATE TABLE ... PARTITION OF`。
