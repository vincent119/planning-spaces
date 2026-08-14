# Ticket Activity Log 保留清理需求

## 背景

`system_settings` 已存在 `system.log.ticket_activity_keep_days`，預設值為 `0`，描述為 Ticket Activity Log 保留天數。目前後端未讀取此設定，也沒有對 `ticket_activities` 執行過期清理。

`ticket_activities` 是 Ticket 操作歷程、留言、欄位異動與附件操作紀錄，屬於 log / audit 類資料；`tickets` 是主業務資料，不在本需求清理範圍內。

## 範圍

### 包含

- 以 `system.log.ticket_activity_keep_days` 控制 `ticket_activities` 保留天數。
- 新增 scheduler task 清理過期 `ticket_activities`。
- 沿用既有 scheduler 分散式鎖與 scheduler log。
- 補 SQL seed，讓排程管理可看到並手動觸發此任務。

### 不包含

- 不刪除 `tickets` 主表資料。
- 不刪除 `attachments` 資料與實體檔案。
- 不改變 Ticket 列表、報表與附件查詢規則。
- 不修改既有已執行 SQL 檔案，只新增後續 SQL 檔案。

## 需求

- [ ] 1.1 後端需讀取 `system.log.ticket_activity_keep_days` 作為 `ticket_activities` 清理保留天數。
- [ ] 1.2 設定值缺值、停用、非數字或小於等於 `0` 時，不執行清理；`0` 代表長期保留。
- [ ] 1.3 設定值大於 `0` 時，清理 `created_at < now - keep_days` 的 `ticket_activities`。
- [ ] 1.4 清理動作不得刪除 `tickets`、`attachments`、`ticket_collaborators` 或其他 Ticket 主資料。
- [ ] 1.5 清理任務需寫入 `scheduler_logs`，記錄刪除筆數、保留天數、是否跳過與 lock 資訊。
- [ ] 1.6 多 Pod 環境下，同一時間只能有一個 instance 執行清理；需沿用現有 scheduler lock。
- [ ] 1.7 新增 scheduler task `clean_expired_ticket_activities`，預設每日執行一次。
- [ ] 1.8 排程管理頁需透過既有 scheduler API 顯示、啟停與手動觸發此任務，不新增選單。
- [ ] 1.9 清理後 Ticket 詳情頁、附件列表與報表查詢不得因舊 activity 不存在而白畫面或 API 500。
- [ ] 1.10 需補單元測試覆蓋啟用清理、`0` 跳過、lock busy 跳過與 SQL table whitelist。

## 驗收條件

- [ ] `system.log.ticket_activity_keep_days = 0` 時，觸發任務不刪資料，scheduler log 顯示 `skipped=true`。
- [ ] `system.log.ticket_activity_keep_days = 30` 時，只刪除 `created_at` 早於 30 天前的 `ticket_activities`。
- [ ] 手動觸發 `clean_expired_ticket_activities` 時，回傳成功並寫入 scheduler log。
- [ ] 同時觸發兩次相同任務時，第二個任務因 lock busy 寫入 skipped log。
- [ ] `tickets` 資料筆數不因執行此任務而改變。
- [ ] `attachments` 資料筆數不因執行此任務而改變。
