# Dashboard Frontend Tasks

## 0. Contract

- [x] 0.1 讀取 Dashboard OpenAPI contract
  - 確認 `/api/v1/dashboard`
  - 確認 `/api/v1/projects/{id}/dashboard`
  - 若 response schema 不完整，標記 blocked，不得串假資料
  - 完成：`Docs/openapi.json` 已包含 `GET /api/v1/dashboard`，支援 `timezone` query
  - 完成：`Docs/openapi.json` 已包含 `GET /api/v1/projects/{id}/dashboard`，支援 `id` path 與 `timezone` query
  - 完成：200 response `data` schema 已包含 `scope`、`metrics`、`trend`、`priority_distribution`、`recent_tickets`、`generated_at`
  - 完成：`metrics` 包含 `open_tickets`、`created_today`、`resolved_today`、`pending_p1_p2`、`sla_breaches`
  - 完成：`trend` item 包含 `date`、`label`、`created`、`resolved`
  - 完成：`priority_distribution` item 包含 `priority`、`value`
  - 完成：`recent_tickets` item 包含 `id`、`title`、`priority`、`status`、`project_id`、`project_name`、`sub_project_id`、`sub_project_name`、`created_at`
  - 結論：contract 足夠支援前端 `1.1` API client 與 `1.2` TypeScript types，不需要 blocked

## 1. API Client

- [x] 1.1 改造 Dashboard API client
  - 移除 fallback `is_api_connected=false`
  - `getDashboardSnapshot`
  - `getProjectDashboardSnapshot`
  - 支援 timezone query
  - 完成：`getDashboardSnapshot(params?)` 串接 `/dashboard`
  - 完成：`getProjectDashboardSnapshot(projectId, params?)` 串接 `/projects/${encodeURIComponent(projectId)}/dashboard`
  - 完成：`timezone` 空值不輸出 query

- [x] 1.2 對齊 Dashboard TypeScript types
  - Metric
  - TrendPoint
  - PriorityPoint
  - RecentTicket
  - Scope
  - 完成：型別已對齊 OpenAPI snapshot `data` contract
  - 驗證：`npm run typecheck` 通過

## 2. Global Dashboard

- [ ] 2.1 串接 `/dashboard`
  - query key: `['dashboard', 'global', timezone]`
  - refetchInterval 30 秒
  - API error 顯示錯誤，不顯示 API 尚未提供

- [ ] 2.2 更新 StatCard 欄位
  - 進行中 Ticket
  - 今日新增
  - 今日解決
  - 待處理 P1/P2
  - SLA 違反若 source=not_implemented，顯示未啟用狀態

- [ ] 2.3 更新圖表資料 mapping
  - trend created / resolved
  - priority distribution P1/P2/P3/P4

- [ ] 2.4 更新最近 Ticket Data Grid
  - project / sub_project 欄位
  - created_at 依 locale 格式化
  - Data Grid i18n

## 3. Project Dashboard

- [x] 3.1 建立 `/projects/:id/dashboard`
  - 使用 ProjectWorkspaceLayout 或專案 header
  - 串 `GET /api/v1/projects/{id}/dashboard`
  - 不使用 placeholder
  - 完成：新增 `ProjectDashboardPage`，使用 `ProjectWorkspaceLayout`
  - 完成：query key 為 `['dashboard', 'project', projectId, timezone]`，並以 30 秒 refetch
  - 完成：顯示專案 scope、metrics、trend、priority distribution、recent tickets

- [x] 3.2 `/projects/:id` 導向 project dashboard
  - 若現有路由使用 placeholder，改為 redirect 或真實頁面
  - 完成：`/projects/:id` 已改為 redirect 到 `dashboard`
  - 驗證：`npm run typecheck` 通過
  - 驗證：`npm run build` 通過，僅有 Vite chunk size warning

## 4. UX

- [x] 4.1 Loading / Empty / Error state
  - 首次載入 skeleton 或 progress
  - 空圖表顯示真實空狀態
  - error 顯示重試按鈕
  - 完成：全域與專案 Dashboard 首次載入顯示 skeleton
  - 完成：trend 與 priority distribution 在 API 回傳空陣列時顯示空圖表文案
  - 完成：API error 使用 React Query error state 並提供重試按鈕

- [x] 4.2 補三語 i18n
  - zh-TW
  - zh-CN
  - en
  - 完成：補齊 dashboard retry、empty_trend、empty_priority_distribution 三語文案

- [x] 4.3 驗證
  - `npm run typecheck`
  - `npm run build`
  - 驗證：`npm run typecheck` 通過
  - 驗證：`npm run build` 通過，僅有 Vite chunk size warning
