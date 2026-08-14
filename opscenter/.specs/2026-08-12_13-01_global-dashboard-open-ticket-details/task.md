# 全域儀表板未結案 Ticket 明細 Tasks

Status: Complete

## 文件定位

本文件追蹤 `requirements.md` 與 `design.md` 的實作工作。這是獨立新功能，上方 Dashboard 卡片不在本次執行範圍。

## Execution Context

### 意圖

在全域儀表板最近 10 筆 Ticket 右側新增最近 10 筆未結案 Ticket 明細，讓使用者同時掌握最新開單與尚待處理項目。

### 非目標

- 不新增上方統計卡片。
- 不修改專案 Dashboard。
- 不新增分頁、搜尋或 Ticket 狀態。
- 不進行資料庫 migration。

### 已定決策

- 未結案口徑為 `status NOT IN ('resolved', 'closed', 'cancelled')`。
- 後端獨立查詢最近 10 筆未結案 Ticket，不以前端過濾最近 Ticket 取代。
- 桌面左右各 30 欄，平板與手機上下排列。
- 新欄位使用 `open_ticket_details`。
- 上方卡片維持原狀。

### 邊界

Allowed Changes：Dashboard backend domain／repository／service／delivery／OpenAPI／測試、全域 Dashboard API type／頁面／共用元件／三語系／測試、本規格文件。

Forbidden：上方統計卡片、專案 Dashboard UI、Ticket API、Ticket 列表、資料庫 schema、權限模型及其他模組。

### 關鍵檔案

- `opscenter-server/internal/dashboard/domain.go`
- `opscenter-server/internal/dashboard/repository.go`
- `opscenter-server/internal/dashboard/service.go`
- `opscenter-server/internal/dashboard/delivery.go`
- `opscenter-frontend/src/features/dashboard/api/dashboard.ts`
- `opscenter-frontend/src/features/dashboard/pages/DashboardPage.tsx`
- `opscenter-frontend/src/features/dashboard/components/`
- `opscenter-frontend/src/locales/{zh-TW,zh-CN,en}/common.json`

### 完成條件

- Requirements 1.1 至 1.11 完成並具驗證證據。
- Repository、service、delivery 與前端 contract 一致。
- 桌面左右排列、窄螢幕上下排列。
- Go 與前端測試、型別、build、OpenAPI 及 `git diff --check` 通過。

## Protected Behavior

- `recent_tickets` 仍是不分狀態的最近 10 筆。
- `open_tickets` 統計卡片與其他上方卡片不變。
- 全域 Dashboard 權限 scope 與 30 秒刷新不變。
- 專案 Dashboard snapshot 必須維持可用；新增 contract 欄位不得造成既有頁面錯誤。
- 既有使用者自訂 Dashboard layout 不得遺失。

## 1. 後端資料契約

- [x] 1.1 新增未結案 Ticket domain 與 repository contract
  - Boundary:
    - Allowed Changes：Dashboard domain、repository interface 與測試替身。
    - Forbidden：Ticket domain、資料庫 schema。
  - Depends：無。
  - Context：snapshot 新增獨立 `OpenTicketDetails`，item 可沿用 `RecentTicket`。
  - Verify：Go 編譯與 domain／service 測試替身完成。

- [x] 1.2 實作未結案 Ticket repository 查詢
  - Boundary:
    - Allowed Changes：Dashboard repository 與 repository tests。
    - Forbidden：權限模型及其他 repository。
  - Depends：1.1。
  - Context：排除 resolved／closed／cancelled，依 created_at、id 倒序，limit 10，共用 applyScope。
  - Verify：條件、排序、limit、deleted_at、global member 與 project scope 測試。

- [x] 1.3 串接 service snapshot
  - Boundary:
    - Allowed Changes：Dashboard service 與 service tests。
    - Forbidden：summary 與既有 metric 口徑。
  - Depends：1.2。
  - Context：新查詢結果放入 snapshot；錯誤處理沿用既有策略。
  - Verify：成功與 repository error 測試。

- [x] 1.4 更新 delivery DTO 與 OpenAPI
  - Boundary:
    - Allowed Changes：Dashboard delivery、delivery tests、OpenAPI 產出。
    - Forbidden：路由與授權 middleware。
  - Depends：1.3。
  - Context：輸出 `open_ticket_details`，item contract 與 recent tickets 一致。
  - Verify：HTTP response JSON 與 OpenAPI schema 測試。

## 2. 全域 Dashboard 前端

- [x] 2.1 更新 Dashboard TypeScript contract
  - Boundary:
    - Allowed Changes：Dashboard API types 與測試。
    - Forbidden：Ticket API types。
  - Depends：1.4。
  - Context：`DashboardSnapshot.open_ticket_details: RecentTicket[]`。
  - Verify：typecheck 與 contract fixture。

- [x] 2.2 建立可共用的 Dashboard Ticket 明細表格
  - Boundary:
    - Allowed Changes：Dashboard feature components、全域頁面及測試。
    - Forbidden：全域共用 DataGrid 行為與其他 feature。
  - Depends：2.1。
  - Context：共用欄位、locale 時間、loading 與空資料；避免兩份表格重複。
  - Verify：recent 與 open details 可傳入不同 rows／empty label。

- [x] 2.3 新增未結案明細面板與 RWD 版面
  - Boundary:
    - Allowed Changes：`DashboardPage.tsx` 與必要 Dashboard layout 相容測試。
    - Forbidden：上方卡片、專案 Dashboard UI。
  - Depends：2.2。
  - Context：recent 與 open details 預設 span 30；tabletFullWidth；舊 layout 能補入新 panel。
  - Verify：桌面左右排列、窄螢幕上下排列、舊 storage 不遺失且新 panel 可見。

- [x] 2.4 補齊三語系與狀態畫面
  - Boundary:
    - Allowed Changes：Dashboard 三語系與 loading skeleton。
    - Forbidden：其他 namespace 文案。
  - Depends：2.3。
  - Context：新增未結案標題及空資料文案，初始 skeleton 顯示兩個列表面板。
  - Verify：zh-TW／zh-CN／en key 完整且 typecheck 通過。

## 3. 驗證與文件

- [x] 3.1 執行後端驗證
  - Boundary：僅修正本需求造成的 Dashboard backend 問題。
  - Depends：1.1 至 1.4。
  - Verify：`go test ./internal/dashboard/...` 與 OpenAPI 產出檢查。

- [x] 3.2 執行前端自動化驗證
  - Boundary：僅修正本需求造成的 Dashboard frontend 問題。
  - Depends：2.1 至 2.4。
  - Verify：`npm test`、`npm run typecheck`、`npm run build`。

- [x] 3.3 完成人工驗收與文件回填
  - Boundary：本規格文件與人工驗收。
  - Depends：3.1、3.2。
  - Verify：
    - 最近 10 筆 Ticket 左側資料不變
    - 未結案明細右側只顯示允許狀態
    - 桌面左右、平板／手機上下排列
    - 30 秒刷新、空資料與錯誤狀態
    - `git diff --check`

## 品質檢查清單

- [x] Requirements、design、task 與實作可追溯。
- [x] 上方卡片未修改。
- [x] recent 與 open details 使用不同後端資料陣列。
- [x] 未結案條件與 open tickets 指標一致。
- [x] 權限 scope 與 deleted_at 條件一致。
- [x] RWD 與舊自訂版面相容。
- [x] 後端、前端與人工驗收全部通過。

## Implementation Notes

- 2026-08-12：使用者確認新增卡片先跳過，本規格只處理全域儀表板下方「最近 10 筆 Ticket」右側的未結案 Ticket 明細；目前狀態為 Planned，尚未修改程式。
- 2026-08-12：使用者指示執行，狀態更新為 InProgress；依後端 contract、OpenAPI、全域前端面板與驗證順序實作。
- 2026-08-12：完成 `open_ticket_details` 後端獨立查詢、snapshot／delivery／OpenAPI contract、全域 Dashboard 雙面板、共用 Ticket grid、三語系與舊 layout 相容處理。`go test ./internal/dashboard/...`、前端 16 項測試、`npm run typecheck`、`npm run build` 與 `git diff --check` 通過；人工瀏覽器驗收尚待確認，狀態維持 InProgress。
- 2026-08-12：使用者確認 Completed，人工瀏覽器驗收與文件回填完成，規格狀態更新為 Complete。
