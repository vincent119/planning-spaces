# Dashboard Frontend Design

## 目前問題

`src/features/dashboard/api/dashboard.ts` 目前沒有呼叫 API，而是回傳：

```ts
is_api_connected: false
metrics: { value: '—' }
trend: []
priority_distribution: []
recent_tickets: []
```

因此 Dashboard 畫面會顯示「Dashboard API 尚未提供，目前顯示空資料狀態」。這不是正常完成狀態。

## Route

### Global Dashboard

路由：`/dashboard`

資料來源：`GET /api/v1/dashboard`

### Project Dashboard

路由：`/projects/:id/dashboard`

資料來源：`GET /api/v1/projects/:id/dashboard`

`/projects/:id` 可導向 `/projects/:id/dashboard`，但不得用 placeholder page。

## API Client

```ts
export async function getDashboardSnapshot(params?: DashboardParams): Promise<DashboardSnapshot> {
  return apiClient.get<DashboardSnapshot>(withQuery('/dashboard', params));
}

export async function getProjectDashboardSnapshot(
  projectId: string,
  params?: DashboardParams,
): Promise<DashboardSnapshot> {
  return apiClient.get<DashboardSnapshot>(withQuery(`/projects/${encoded(projectId)}/dashboard`, params));
}
```

需移除 `is_api_connected` fallback 欄位。API 失敗時由 React Query `isError` 處理。

## UI Layout

沿用舊設計：

- 四張 StatCard：
  - 進行中 Ticket
  - 今日新增
  - 今日解決
  - 待處理 P1/P2
- SLA 違反卡片於 Phase 2 顯示；MVP 若保留卡片，需標示未啟用或顯示 `0`。
- 趨勢折線圖：
  - created
  - resolved
- 優先級分佈 donut：
  - P1/P2/P3/P4
- 最近 Ticket Data Grid：
  - title
  - priority
  - status
  - project
  - sub_project
  - created_at

## State

- 使用 TanStack Query。
- query key：
  - Global: `['dashboard', 'global', timezone]`
  - Project: `['dashboard', 'project', projectId, timezone]`
- `refetchInterval: 30_000`
- `staleTime: 30_000`

## Loading / Empty / Error

- loading：顯示 LinearProgress 或 skeleton。
- empty chart：只有當 API 真實回傳空陣列時顯示空圖表文字。
- error：顯示錯誤 Alert 與重新整理按鈕。
- 不得再顯示「Dashboard API 尚未提供」作為正常狀態。

## i18n

Dashboard 文案位於 `common.json` 的 `dashboard` namespace。新增欄位需同步 zh-TW / zh-CN / en。

## Visual Rules

- 與現有深色後台一致。
- 不使用毛玻璃作為 Dashboard 卡片。
- 卡片保留漸層背景，但頁面不能被單一色系支配。
- Data Grid 使用共用 `getDataGridLocaleText`。

