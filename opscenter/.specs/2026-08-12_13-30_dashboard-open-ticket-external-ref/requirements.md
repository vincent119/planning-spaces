# 儀表板未結案 Ticket 外部單號需求

## 文件定位

本規格接續已完成的 `2026-08-12_13-01_global-dashboard-open-ticket-details`，只調整全域儀表板「未結案 Ticket 明細」的顯示欄位。不得回開或改寫既有已完成 task。

## 背景

未結案 Ticket 明細目前與最近 Ticket 共用欄位，包含「優先級」。使用者需要在該位置查看 Ticket 的 `external_ref` 外部單號，以便直接核對外部系統單號。

## 目標

1. 未結案 Ticket 明細將「優先級」欄改為「外部單號」。
2. Dashboard API 明細提供 `external_ref`。
3. 左側「最近 10 筆 Ticket」維持原本優先級欄。

## 非目標

- 不把外部單號轉成超連結。
- 不修改未結案條件、排序、筆數或權限。
- 不修改上方卡片、專案 Dashboard 或 Ticket 列表。
- 不進行資料庫 migration。

## 驗收條件

- [x] 1.1 Dashboard Ticket 明細 DTO 新增可為空的 `external_ref`。
- [x] 1.2 後端 recent 與 open ticket detail 查詢回傳 `tickets.external_ref`，OpenAPI 同步更新。
- [x] 1.3 右側未結案明細不顯示優先級欄，原位置顯示外部單號。
- [x] 1.4 外部單號為空時顯示 `-`。
- [x] 1.5 左側最近 10 筆 Ticket 仍顯示優先級，不受右側欄位設定影響。
- [x] 1.6 未結案口徑、排序、10 筆上限、RWD 與 30 秒刷新維持不變。
- [x] 1.7 繁體中文、簡體中文與英文欄位名稱完整。

## 驗收情境

### 場景 A：未結案外部單號

假設未結案 Ticket 的 `external_ref` 為 `JIRA-123`。

當使用者開啟全域儀表板。

那麼右側表格原「優先級」位置顯示「外部單號」及 `JIRA-123`。

### 場景 B：左右欄位不同

假設同一 Ticket 同時出現在左右兩份明細。

當兩個表格顯示資料。

那麼左側顯示優先級，右側顯示外部單號。

## 驗證需求

- Dashboard repository、service 與 delivery 測試。
- 共用 Dashboard Ticket grid 欄位變體測試或型別驗證。
- OpenAPI、三語系、前端測試、typecheck、build 與 `git diff --check`。
