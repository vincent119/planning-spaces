# Ticket Excel 匯入冪等去重需求

## 文件定位

本需求接續既有 spec：

```text
2026-06-22_15-24_Import_ticket_tools
```

既有匯入工具已將「Jira單號」改為可選欄位。其現況只在 Jira 單號有值時，以 `project_id + external_ref` 進行應用程式層查詢去重；Jira 單號留白時，每次 `-commit` 都會新增 Ticket。

本需求只處理 Excel CLI 匯入的冪等與重複防護。不重寫 Excel 解析、欄位映射、填寫人對應、自動建立子專案／資訊來源、原始檔保存或 Ticket 列表讀取等已完成行為。

## 背景

部分歷史 Excel 開單紀錄沒有 Jira 單號。使用者重覆執行相同資料的匯入時，現行工具無法辨識無外部單號的既有 Ticket，因而建立重複資料。

此類匯入資料為歷史固定資料。匯入後不會修改資料內容、分頁名稱、列順序或原始 Excel 列位置。

## 目標

- 讓同一主專案的相同 Excel 來源列可以安全重覆執行 `-commit`，且只建立一張 Ticket。
- 保留 `Jira單號` 為可選欄位，不把人工產生的去重值寫入 `tickets.external_ref`。
- 維持 OP Jira 指標以真實非空 `external_ref` 判定，不讓無 Jira 單號的 Excel Ticket 被錯誤計入。
- 由資料庫提供原子性的去重保障，避免同時執行多個匯入程序時發生競態重複。

## 非目標

- 不建立完整的匯入批次、匯入歷史 UI 或前端匯入功能。
- 不實作 `upsert`，本需求仍採 `insert_only`：重複來源列略過，不更新既有 Ticket。
- 不使用標題、日期、資訊來源或填寫人做模糊比對，不自動合併「疑似相同」但沒有相同來源識別的 Ticket。
- 不支援已修改內容、重新排序、插入列、刪除列或更名分頁後的來源檔重新匯入。
- 不變更既有 `external_ref` 的報表、API 與 Ticket 語意。

## 來源列識別規則

### Excel Row ID 定義

本需求中的 `excel_row_id` 指 Excel 工作表 XML 提供的原始列位置，也就是目前 CLI 顯示的 Excel 列號。它不是使用者可編輯的 Ticket 欄位。

來源資料必須保持下列不變，才可視為相同來源列：

- 主專案。
- 分頁名稱。
- Excel 列位置。
- 日期欄位正規化後的值。
- 已知問題欄位正規化後的值。

### 無 Jira 單號的來源鍵

無 Jira 單號的列必須產生固定的 `source_row_fingerprint`。其輸入必須包含下列已正規化值：

```text
schema_version = excel-ticket-import-v1
project_id
sheet_name
excel_row_id
opened_at
title
```

系統使用 SHA-256 將上述資料產生為固定長度字串。相同主專案、相同來源列與相同固定資料再次匯入時，必須得到相同 fingerprint。

### 有 Jira 單號的來源鍵

Jira 單號有值時，系統必須維持以正規化後的 `project_id + external_ref` 作為主要來源識別。無 Jira 單號時才使用 `source_row_fingerprint`。

## 冪等與去重需求

- 系統必須持久保存每個已成功建立 Ticket 的匯入來源識別，且該識別必須在所有 Ticket 月份分區間維持唯一。
- 此持久資料僅用於匯入冪等與來源列對應，不取代完整匯入批次或稽核系統。
- 持久來源識別必須至少保存主專案、來源種類、來源鍵、對應 Ticket 識別、Ticket 建立時間、分頁名稱與 Excel 列號。
- 無 Jira 單號的列不可將 fingerprint 寫入 `tickets.external_ref`；`external_ref` 必須維持空值。
- 同一主專案下已存在相同來源識別時，`insert_only` 匯入必須略過該列，不建立 Ticket 或新的建立活動紀錄。
- 同一份 Excel 中重覆出現相同 Jira 單號或相同來源列時，預覽必須標示後續列為重複，不得在提交階段建立多張 Ticket。
- 去重寫入與 Ticket、初始 `ticket_activities` 建立必須具備原子性；任一步驟失敗時不得留下沒有對應 Ticket 的來源識別。
- 兩個匯入程序同時提交相同來源列時，最多只能成功建立一張 Ticket；另一個程序必須被識別為略過或可理解的重複結果，不得回報虛假的建立成功。

## CLI 與預覽需求

- dry-run 預覽必須顯示每列預定動作：`create`、`skip_duplicate` 或 `error`。
- 對已存在的來源識別，預覽與 commit 輸出必須包含可辨識的略過原因：`已匯入相同來源列` 或 `external_ref 已存在`。
- commit 統計必須正確區分 `created`、`skipped`、`failed` 與 `invalid`。
- 不得因去重略過而建立子專案、資訊來源、Ticket 或活動紀錄。

## 使用情境與驗收情境

### 場景：無 Jira 單號的相同來源列重覆匯入

測試：`go test ./cmd/import_ticket_data`

假設同一主專案的 Excel 列沒有 Jira 單號，且分頁名稱、列位置、日期與已知問題均未變更，

當使用者第二次對相同資料執行 `-commit`，

那麼系統必須略過該列，且資料庫中只存在第一次建立的一張 Ticket 與一筆初始建立活動。

### 場景：有 Jira 單號的列重覆匯入

測試：`go test ./cmd/import_ticket_data`

假設同一主專案已有相同正規化 Jira 單號的 Ticket，

當使用者再次匯入該 Jira 單號，

那麼系統必須略過該列，且不得新增 Ticket。

### 場景：不同分頁的相同列號

測試：`go test ./cmd/import_ticket_data`

假設同一主專案的兩個分頁都有相同 Excel 列號，

當兩列的分頁名稱不同時，

那麼系統必須視為不同來源列，不得因列號相同而誤略過。

### 場景：無 Jira 單號不影響 OP Jira 指標

測試：`go test ./internal/report`

假設無 Jira 單號的 Excel 列已透過 fingerprint 成功匯入，

當報表計算 OP Jira 開單數量，

那麼該 Ticket 不得因 fingerprint 而被計入 OP Jira 指標。

### 場景：同時匯入相同來源列

測試：待建立資料庫整合測試

假設兩個匯入程序同時提交相同主專案與來源鍵，

當兩者嘗試建立 Ticket 時，

那麼最多只能有一個程序建立 Ticket，另一個程序必須取得重複略過結果。

## 驗收條件

- Jira 單號為空的固定 Excel 資料重覆匯入後，Ticket 總數不增加。
- Jira 單號為空時，`tickets.external_ref` 維持空值。
- 同一來源列在並發提交下不會建立多張 Ticket。
- 不同分頁的相同 Excel 列號不會互相去重。
- CLI 預覽與 commit 統計可清楚區分新增、略過、失敗與無效資料。
- 所有既有 `cmd/import_ticket_data` 與 `internal/report` 相關回歸測試持續通過。

## 風險與限制

- 本需求以「歷史匯入資料不異動」為前提。資料內容、分頁名稱或列位置改變時，會產生不同 fingerprint，系統會將其視為不同來源列。
- 使用資料庫唯一鍵可消除匯入競態，但需要新增版本化 migration 與可回滾部署程序。
- `tickets` 為按月份分區的表；來源識別的唯一性不可只依賴單一月份分區上的索引，必須跨分區有效。
