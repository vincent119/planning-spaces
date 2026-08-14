# Ticket 列表建立時間欄位 Tasks

Status: InProgress

## Execution Context

### 意圖

在 Ticket 列表更新時間左側顯示 `created_at` 建立時間。

### 非目標

- 不修改後端、資料庫、篩選、排序及其他頁面。

### 已定決策

- 欄位順序為外部單號、建立時間、更新時間、操作。
- 日期格式及尺寸與更新時間一致。
- 使用既有三語系 key。

### 邊界

Allowed Changes：`ProjectTicketsPage.tsx` 與本規格文件。

Forbidden：後端、API type、資料庫、語系內容、其他列表與共用元件。

## Protected Behavior

- Ticket URL 查詢狀態、篩選與分頁不變。
- 外部單號與更新時間欄位不變。
- 其他欄位順序與操作功能不變。

## 1. 前端實作

- [x] 1.1 新增 created_at Data Grid 欄位
  - Boundary：`ProjectTicketsPage.tsx` columns。
  - Depends：無。
  - Verify：欄位位於 external_ref 與 updated_at 之間，使用既有 formatter 與語系 key。

## 2. 驗證與文件

- [x] 2.1 執行前端驗證
  - Depends：1.1。
  - Verify：`npm run typecheck`、`npm run build`、`git diff --check`。

- [ ] 2.2 人工驗收並回填文件
  - Depends：2.1。
  - Verify：欄位順序、日期顯示與 RWD 表格。

## 品質檢查清單

- [x] 建立時間位於更新時間左側。
- [x] API 與資料庫未修改。
- [ ] 自動化與人工驗收通過。

## Implementation Notes

- 2026-08-12：建立獨立規格，目前為 Planned，尚未修改程式。
- 2026-08-12：使用者指示執行，狀態更新為 InProgress。
- 2026-08-12：完成 Ticket 列表 `created_at` 欄位，位置在外部單號與更新時間之間；前端 16 項測試、typecheck、build 與 `git diff --check` 通過，等待人工畫面驗收。
