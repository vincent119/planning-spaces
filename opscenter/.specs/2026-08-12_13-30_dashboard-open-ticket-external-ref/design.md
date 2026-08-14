# 儀表板未結案 Ticket 外部單號設計

## 文件定位

本設計只修改 Dashboard Ticket item contract 與共用表格的欄位變體，保護已完成的雙面板資料口徑與版面。

## 已知契約

- `tickets.external_ref` 已存在，無需 migration。
- Dashboard `RecentTicket` item 目前含 ID、標題、優先級、狀態、專案、子專案與建立時間。
- 左右面板共用 `DashboardTicketGrid`，目前欄位完全相同。

## Bounded Context

### 包含

- Dashboard repository select／record／domain／delivery DTO 新增 `external_ref`。
- OpenAPI 與 TypeScript `RecentTicket` type 同步。
- `DashboardTicketGrid` 支援優先級或外部單號欄位變體。
- 全域頁面右側指定外部單號變體，左側維持優先級。
- 三語系與測試。

### 不包含

- 外部網址跳轉。
- 未結案查詢條件及 Dashboard layout。
- 專案 Dashboard 欄位調整。

## 後端設計

`RecentTicket`、`recentTicketRecord` 與 `RecentTicketResponse` 新增 `ExternalRef string`，JSON 名稱為 `external_ref`。兩份 Dashboard Ticket 查詢均 select `t.external_ref`，確保共用 item contract 一致；空值透過 nullable scan 或 `COALESCE(t.external_ref, '')` 安全輸出空字串。

## 前端設計

`RecentTicket` 新增：

```ts
external_ref?: string;
```

`DashboardTicketGrid` 新增明確 prop：

```ts
secondaryColumn: 'priority' | 'external_ref';
```

- `priority`：欄名與值維持既有行為。
- `external_ref`：欄名使用「外部單號」，空值顯示 `-`。

左側傳 `priority`，右側傳 `external_ref`。不得透過 title 判斷欄位，避免文案與行為耦合。

## 受影響檔案

- `opscenter-server/internal/dashboard/{domain,repository,delivery}.go` 與測試
- `opscenter-server/Docs/openapi.json`
- `opscenter-frontend/src/features/dashboard/api/dashboard.ts`
- `opscenter-frontend/src/features/dashboard/components/DashboardTicketGrid.tsx`
- `opscenter-frontend/src/features/dashboard/pages/DashboardPage.tsx`
- 三語系 `common.json`

## 驗證

- 驗證 API 同時輸出 priority 與 external_ref。
- 驗證左右面板分別選用正確欄位。
- Go 測試、前端測試、typecheck、build、OpenAPI 與 diff check。
