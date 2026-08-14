# 全域儀表板未結案 Ticket 明細設計

## 文件定位

本設計實作 `requirements.md` 定義的全域 Dashboard 雙明細面板。保留既有 summary、trend、priority distribution、recent tickets、專案 Dashboard 與上方統計卡片行為。

## 已知契約狀態

- 全域 API：`GET /api/v1/dashboard`。
- 現有 snapshot 已包含 `recent_tickets`，資料依建立時間倒序且最多 10 筆。
- 現有 `open_tickets` 指標使用 `status NOT IN ('resolved', 'closed', 'cancelled')`。
- 全域 scope 由既有 `applyScope` 控制：Admin／OP Admin 可看全部，一般使用者只看可存取專案。
- 前端全域頁面為 `DashboardPage.tsx`，使用 `DashboardLayout` 的 60 欄桌面格線與個人化版面儲存。
- Dashboard 使用 TanStack Query 每 30 秒重新取得同一份 snapshot。

## Bounded Context

### 包含

- Dashboard snapshot 新增未結案 Ticket 明細欄位。
- Repository 新增未結案 Ticket 查詢。
- Service、delivery DTO、OpenAPI 與測試同步。
- 全域 Dashboard 新增右側未結案明細面板。
- 最近 Ticket 面板預設寬度由 60 調整為 30，與新面板並排。
- 三語系文案及必要前端驗證。

### 不包含

- 上方 Dashboard 統計卡片。
- 專案 Dashboard UI。
- Ticket 列表與 Ticket API。
- 新增分頁 endpoint、資料庫 migration 或索引。
- 自訂 Dashboard layout schema 升版；既有 hook 應以 panel 設定變更重新補齊新 panel。

## API 與資料契約

在既有 snapshot 新增：

```json
{
  "open_ticket_details": [
    {
      "id": "01...",
      "title": "Ticket title",
      "priority": "P2",
      "status": "in_progress",
      "project_id": "01...",
      "project_name": "MS",
      "sub_project_id": "01...",
      "sub_project_name": "運維",
      "created_at": "2026-08-12T03:00:00Z"
    }
  ]
}
```

明細欄位與 `RecentTicket` 相同，可共用 domain item、delivery item 與前端 row type，但 snapshot 必須使用獨立陣列，避免前端以 `recent_tickets` 自行篩選而漏掉較早但仍未結案的 Ticket。

## 後端設計

### Repository

在 `Repository` 新增：

```go
OpenTicketDetails(context.Context, QueryInput) ([]RecentTicket, error)
```

查詢基於既有 recent tickets select 與 join，增加：

```sql
WHERE t.status NOT IN ('resolved', 'closed', 'cancelled')
ORDER BY t.created_at DESC, t.id DESC
LIMIT 10
```

查詢最後套用既有 `applyScope`，確保 `deleted_at` 與專案權限口徑一致。為避免兩份查詢日後欄位漂移，可抽取內部 ticket detail query builder，但不得改變現有查詢結果。

### Service

`snapshot` 在取得 `recentTickets` 後取得 `openTicketDetails`，任一必要查詢失敗維持 snapshot 回傳錯誤的既有策略。`Snapshot` 新增 `OpenTicketDetails []RecentTicket`。

### Delivery 與 OpenAPI

`SnapshotResponse` 新增：

```go
OpenTicketDetails []RecentTicketResponse `json:"open_ticket_details"`
```

沿用 `RecentTicketResponse` 與既有轉換函式。更新 delivery 測試及產出的 OpenAPI contract。

## 前端設計

### API 型別

`DashboardSnapshot` 新增：

```ts
open_ticket_details: RecentTicket[];
```

不得由 `recent_tickets.filter(...)` 產生，因為最近 10 筆全部 Ticket 不保證包含最近 10 筆未結案 Ticket。

### 畫面結構

全域 Dashboard panels 改為：

```text
桌面 lg 以上
┌──────────────────────────────┬──────────────────────────────┐
│ 最近 10 筆 Ticket，span 30   │ 未結案 Ticket 明細，span 30  │
└──────────────────────────────┴──────────────────────────────┘

平板與手機
┌─────────────────────────────────────────────────────────────┐
│ 最近 10 筆 Ticket                                           │
├─────────────────────────────────────────────────────────────┤
│ 未結案 Ticket 明細                                          │
└─────────────────────────────────────────────────────────────┘
```

兩個面板都設定 `tabletFullWidth: true`，預設 `span: 30`，允許使用者在既有版面編輯模式調整支援寬度。表格高度一致，欄位至少顯示標題、優先級、狀態與建立時間；專案／子專案欄位依可用寬度及既有欄位規格納入，必要時由表格容器自身水平捲動，不讓整頁溢出。

建議抽出共用 Dashboard Ticket grid 元件，避免兩份欄位與格式化邏輯重複；抽取只限 Dashboard feature。

### 載入與空資料

- 初次載入沿用 `DashboardLoadingSkeleton`，需呈現兩個明細面板占位。
- 背景刷新時兩個 Data Grid 同時使用 `dashboardQuery.isFetching`。
- 未結案陣列為空時顯示「目前沒有未結案 Ticket」。
- API 失敗沿用頁面既有錯誤提示與重試按鈕。

## 受影響檔案計畫

- `opscenter-server/internal/dashboard/domain.go`
- `opscenter-server/internal/dashboard/repository.go`
- `opscenter-server/internal/dashboard/service.go`
- `opscenter-server/internal/dashboard/delivery.go`
- Dashboard 對應 Go 測試
- `opscenter-server/Docs/openapi.json` 或專案既有 OpenAPI 產出檔
- `opscenter-frontend/src/features/dashboard/api/dashboard.ts`
- `opscenter-frontend/src/features/dashboard/pages/DashboardPage.tsx`
- 必要的 Dashboard 共用表格元件與測試
- `opscenter-frontend/src/locales/{zh-TW,zh-CN,en}/common.json`

## 風險與處理方式

### 將最近 Ticket 在前端篩選

前端篩選只能得到最近 10 筆全部 Ticket 中的未結案項目，會漏掉第 11 筆以後仍未結案的資料，因此必須由後端獨立查詢。

### 權限範圍漂移

新查詢不得另寫權限條件；必須共用 `applyScope` 並補一般使用者 scope 測試。

### 既有自訂版面

新增 panel 後，`useDashboardLayout` 必須能將舊 storage 中不存在的新 panel 補入畫面。實作前先以既有 hook 測試確認；若不支援，只能在 Dashboard layout 邊界內修正並補 migration 測試。

### 兩個表格窄寬度

桌面各占 30／60 欄時欄位可能壓縮。以合理 `minWidth` 與面板內 overflow 處理，禁止用整頁 CSS scale 或縮小全站字級。

## 驗證方式

- Repository 測試驗證未結案條件、穩定排序、10 筆上限及 scope。
- Service／delivery 測試驗證 `open_ticket_details` contract。
- 前端驗證兩面板資料來源不同且 RWD 排列正確。
- OpenAPI 產出差異檢查。
- Go、前端測試、型別檢查、正式 build 與 `git diff --check`。
