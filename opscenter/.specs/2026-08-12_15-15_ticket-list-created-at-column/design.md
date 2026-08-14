# Ticket 列表建立時間欄位設計

## 文件定位

本設計只修改 `ProjectTicketsPage` 的 Data Grid column definition。既有 `Ticket.created_at: string` 與 `field.created_at` 三語系文案可直接使用。

## Bounded Context

### 包含

- Ticket 列表新增 `created_at` column。
- 沿用頁面現有 `formatDateTime(value, locale)`。
- 本規格文件與前端驗證。

### 不包含

- API、資料庫、後端排序及其他頁面。
- 新增或修改語系 key。
- Data Grid 全域樣式。

## 欄位設計

在 `external_ref` 與 `updated_at` 之間加入：

```tsx
{
  field: 'created_at',
  headerName: t('field.created_at'),
  minWidth: 180,
  flex: 0.9,
  renderCell: (params) => (
    <Typography noWrap variant="body2">
      {formatDateTime(params.row.created_at, locale)}
    </Typography>
  ),
}
```

尺寸與更新時間一致，使兩個時間欄位在 RWD 表格中保持一致。

## 受影響檔案

- `opscenter-frontend/src/features/ticket/pages/ProjectTicketsPage.tsx`
- 本規格文件

## 風險與處理

- 欄位增加會擴大表格總寬度：沿用 Data Grid 現有欄寬及容器行為，不縮小其他欄位。
- 日期格式不一致：直接共用現有 formatter，不另建第二套格式。

## 驗證

- 靜態檢查欄位位於更新時間左側。
- 前端 typecheck、build 與 diff check。
