# Dashboard Design

## 目前狀態

- 後端已有 `internal/dashboard` MVP summary endpoint：
  - `GET /api/v1/dashboard`
  - `GET /api/v1/projects/:id/dashboard`
- 現有後端 response 只包含 `open_tickets`、`created_today`、`resolved_today`、`pending_p1_p2`。
- 前端 `features/dashboard/api/dashboard.ts` 目前直接回傳 fallback snapshot，`is_api_connected=false`，所以畫面顯示「Dashboard API 尚未提供，目前顯示空資料狀態」。
- 舊設計包含卡片、趨勢圖、優先級分佈與最近 Ticket，但缺少完整 API contract。

## 目標

Dashboard 需收斂為單一 snapshot contract，前端直接串真實 API，不再使用 fake empty snapshot 作為正常資料。

## Scope

### Global Dashboard

路由：`/dashboard`

API：`GET /api/v1/dashboard`

資料範圍：

- `admin` / `op_admin`：全部未刪除專案。
- 其他使用者：使用者為專案成員的專案。

### Project Dashboard

路由：`/projects/:id/dashboard`

API：`GET /api/v1/projects/:id/dashboard`

資料範圍：

- 指定 project。
- 需通過 viewer 權限檢查。

## Snapshot Contract

```json
{
  "scope": {
    "type": "global",
    "project_id": "",
    "project_name": "",
    "timezone": "Asia/Taipei"
  },
  "metrics": {
    "open_tickets": { "value": 12, "trend": -8.3 },
    "created_today": { "value": 3, "trend": 50 },
    "resolved_today": { "value": 2, "trend": 0 },
    "pending_p1_p2": { "value": 1, "trend": -50 },
    "sla_breaches": { "value": 0, "trend": 0, "source": "not_implemented" }
  },
  "trend": [
    { "date": "2026-06-11", "label": "06/11", "created": 2, "resolved": 1 }
  ],
  "priority_distribution": [
    { "priority": "P1", "value": 1 },
    { "priority": "P2", "value": 3 },
    { "priority": "P3", "value": 8 },
    { "priority": "P4", "value": 0 }
  ],
  "recent_tickets": [
    {
      "id": "01...",
      "title": "Ticket title",
      "priority": "P2",
      "status": "open",
      "project_id": "01...",
      "project_name": "Main",
      "sub_project_id": "01...",
      "sub_project_name": "API",
      "created_at": "2026-06-17T08:00:00Z"
    }
  ],
  "generated_at": "2026-06-17T08:00:30Z"
}
```

## Metric Definitions

- `open_tickets`：`status NOT IN ('resolved', 'closed', 'cancelled')`。
- `created_today`：使用指定 timezone 的今天 00:00 到明天 00:00。
- `resolved_today`：`resolved_at` 落在今天區間。
- `pending_p1_p2`：未結束且 priority in `P1/P2`。
- `sla_breaches`：Phase 2；MVP 回傳 0 與 `source=not_implemented`。
- `trend`：最近 7 天每日新增與解決數。
- `priority_distribution`：目前未結束 Ticket 的 P1/P2/P3/P4 分佈。
- `recent_tickets`：依 `created_at DESC` 取 10 筆。

## Refresh / Cache

- 前端：`refetchInterval = 30_000`。
- 後端可加 Redis cache，key 應包含 actor scope、project_id、timezone，TTL 30 秒。
- 若暫不加 Redis，SQL 查詢需維持單次或少量聚合查詢，避免逐專案 N+1。

