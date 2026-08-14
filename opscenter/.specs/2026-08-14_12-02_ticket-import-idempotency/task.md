# Ticket Excel 匯入冪等去重 Task

Status: InProgress

## Execution Context

- 意圖：讓固定歷史 Excel 資料可安全重覆匯入，尤其是 Jira 單號空白的列。
- 已定決策：有 Jira 單號使用正規化 Jira key；無 Jira 單號使用 `project_id + sheet_name + excel_row_id + opened_at + title` 的 SHA-256 fingerprint；來源識別保存於跨分區唯一的非分區表。
- 非目標：不實作 upsert、模糊比對、前端 UI、完整匯入批次、來源資料異動後的更新。
- 完成條件：符合 `requirements.md` 的全部驗收情境，且重覆匯入與並發提交不會建立重複 Ticket。

## Protected Behavior

- Jira單號與填寫Jira項目選擇持續為可選欄位。
- 無 Jira 單號的 Ticket 的 `external_ref` 保持空值，且不可被 OP Jira 指標計入。
- 既有 Excel 解析、欄位映射、填寫人對應、子專案／資訊來源自動建立、原始檔保存與 Ticket 列表 NULL 子專案行為不可回歸。
- dry-run 不可寫入 Ticket、來源識別、主資料、活動或來源檔。

## 任務

- [x] 1. 建立來源識別 Migration
  - Boundary:
    - Allowed Changes：新增下一個連號 SQL migration；必要的 migration 測試與部署說明。
    - Forbidden：修改已套用 migration、修改 `tickets.external_ref` 語意、啟用 ORM AutoMigrate。
  - Depends：無。
  - Context：建立非分區 `ticket_import_identities`，以 `(project_id, source_type, source_key)` 唯一約束跨月份分區去重。
  - Verify：2026-08-14 已套用 `0054_create_ticket_import_identities.sql`；已於本機 PostgreSQL `opscenter` schema 確認資料表、欄位、唯一約束與 CHECK 約束存在。rollback 尚未執行，避免移除已投入使用的來源識別資料。

- [x] 2. 建立來源鍵與預覽去重
  - Boundary:
    - Allowed Changes：`cmd/import_ticket_data` 的 import row、正規化、驗證、預覽輸出與單元測試。
    - Forbidden：變更 Excel 欄位映射、將 fingerprint 寫入 `external_ref`、修改報表公式。
  - Depends：任務 1 的資料模型。
  - Context：實作版本化 SHA-256 fingerprint、Jira key 正規化、同一批次記憶體去重與已存 identity 查詢；重複列在主資料自動建立前標記 `skip_duplicate`。
  - Verify：`go test ./cmd/import_ticket_data` 通過；已覆蓋 Excel fingerprint、不同分頁相同列號與 Jira key 正規化。實際重覆匯入 134 列時，CLI 結果為 `created=0 skipped=134 failed=0 invalid=0`。

- [ ] 3. 實作原子提交與 legacy Jira identity 採納
  - Boundary:
    - Allowed Changes：`cmd/import_ticket_data` 的提交 transaction、來源識別 repository helper 與測試。
    - Forbidden：變更一般 Ticket HTTP API、實作 upsert、建立完整匯入批次表。
  - Depends：任務 1、任務 2。
  - Context：transaction 先預留 identity；唯一衝突回傳 `skipped_duplicate`；成功後建立 Ticket、活動並回填 identity。對尚未有 identity 的既有 Jira Ticket，採納為 Jira identity 而非建立新 Ticket。
  - Verify：實際第二次 commit 已驗證 134 列全部略過，未新增 Ticket。尚待建立並發相同來源鍵的資料庫整合測試；legacy Jira Ticket identity 採納尚待補齊與驗證。

- [x] 4. 驗證報表與完整回歸
  - Boundary:
    - Allowed Changes：必要的 report regression test、匯入測試與文件驗證紀錄。
    - Forbidden：調整 OP Jira 指標定義或把 fingerprint 當作 Jira 單號。
  - Depends：任務 2、任務 3。
  - Context：確認無 Jira 單號的來源 fingerprint 不會影響既有 OP Jira 指標；確認 CLI 統計正確區分 created、skipped、failed、invalid。
  - Verify：`go test ./cmd/import_ticket_data`、`go test ./internal/report`、`go test ./...`、`git diff --check` 均通過。無 Jira 單號的來源 fingerprint 未寫入 `external_ref`，既有 OP Jira 判定邏輯未變更。

## 品質檢查清單

- [ ] 無 Jira 單號重覆匯入不增加 Ticket 或 ticket activity。
- [ ] 有 Jira 單號重覆匯入不增加 Ticket。
- [ ] 不同分頁相同列號不會誤略過。
- [ ] 兩個並發提交最多建立一張 Ticket。
- [ ] dry-run 沒有任何資料庫或 storage 寫入。
- [ ] `external_ref` 空白 Ticket 不會被 OP Jira 指標計入。
- [ ] migration 可套用、可驗證且具回滾程序。

## Implementation Notes

- 已新增 `0054_create_ticket_import_identities.sql`，建立跨 Ticket 月份分區的來源識別表與唯一約束。
- 已在匯入器加入 Jira／Excel 列來源鍵、預覽 identity 查詢、同批次去重與 transaction 內 identity 預留及回填。
- 2026-08-14 實際匯入已建立無 Jira 單號 Ticket；同一份 Excel 第二次以 `-commit` 重跑結果為 `rows=134 valid=0 invalid=0 skipped=134 created=0 failed=0`，已驗證固定資料的重覆匯入不會新增 Ticket。
- 尚待：補並發相同來源鍵的資料庫整合測試，以及既有 Jira Ticket 的 identity 採納與驗證；完成後才可將本 spec 標記為 Complete。
- 實作前必讀本檔、`requirements.md` 與 `design.md`；如需擴大範圍，先更新 spec 並取得使用者安排。
