# 儀表板未結案 Ticket 外部單號 Tasks

Status: Complete

## Execution Context

### 意圖

將全域儀表板右側未結案 Ticket 明細的優先級欄改為 `external_ref` 外部單號，同時保護左側最近 Ticket 欄位。

### 非目標

- 不建立超連結。
- 不修改查詢口徑、版面、上方卡片與專案 Dashboard。

### 已定決策

- API item 新增 `external_ref`。
- 共用表格以明確 prop 選擇優先級或外部單號。
- 空外部單號顯示 `-`。

### 邊界

Allowed Changes：Dashboard Ticket item 後端 contract／查詢／OpenAPI／測試、前端 Dashboard API type／共用 grid／全域頁面／三語系／測試、本規格。

Forbidden：資料庫、Ticket API、查詢口徑、Dashboard layout、上方卡片、專案 Dashboard UI。

## Protected Behavior

- 左側最近 10 筆 Ticket 維持優先級欄。
- 右側未結案條件、排序與筆數不變。
- RWD、30 秒刷新、權限與錯誤狀態不變。

## 1. 後端 contract

- [x] 1.1 Dashboard Ticket domain／record／DTO 新增 external_ref
  - Boundary：Dashboard domain、repository record、delivery DTO 與測試。
  - Depends：無。
  - Verify：空值與有值轉換測試。

- [x] 1.2 兩份 Dashboard Ticket 查詢選取 external_ref
  - Boundary：Dashboard repository 與測試。
  - Depends：1.1。
  - Verify：recent 與 open details 都取得 external_ref，既有條件不變。

- [x] 1.3 更新 OpenAPI
  - Boundary：Dashboard delivery 註記與 OpenAPI 產出。
  - Depends：1.2。
  - Verify：schema 包含 external_ref。

## 2. 前端欄位變體

- [x] 2.1 更新 Dashboard TypeScript item contract
  - Boundary：Dashboard API type。
  - Depends：1.3。
  - Verify：typecheck。

- [x] 2.2 共用 Ticket grid 支援 secondaryColumn
  - Boundary：DashboardTicketGrid 與測試。
  - Depends：2.1。
  - Verify：priority／external_ref 欄位與空值行為。

- [x] 2.3 左右面板套用不同欄位並補三語系
  - Boundary：全域 Dashboard 頁面與三語系。
  - Depends：2.2。
  - Verify：左側優先級、右側外部單號。

## 3. 驗證與文件

- [x] 3.1 執行 Go、前端測試、typecheck、build、OpenAPI 與 diff check
  - Depends：1.1 至 2.3。

- [x] 3.2 人工驗收並回填文件
  - Depends：3.1。
  - Verify：左右欄位、空值、RWD 與既有資料口徑。

## 品質檢查清單

- [x] 左側優先級未被替換。
- [x] 右側顯示 external_ref 且不產生超連結。
- [x] 後端與 OpenAPI contract 一致。
- [x] 自動化與人工驗收通過。

## Implementation Notes

- 2026-08-12：建立獨立變更規格；目前為 Planned，尚未修改程式。
- 2026-08-12：使用者指示執行，狀態更新為 InProgress。
- 2026-08-12：完成 Dashboard Ticket `external_ref` contract、兩份 repository 查詢、OpenAPI、前端欄位變體與三語系。Dashboard Go 測試、前端 16 項測試、typecheck、build 與 `git diff --check` 通過；等待人工畫面驗收，狀態維持 InProgress。
- 2026-08-12：使用者確認完成，人工驗收與文件回填完成，狀態更新為 Complete。
