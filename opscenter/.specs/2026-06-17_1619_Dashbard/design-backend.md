# Dashboard Backend Design

## Package

沿用既有 package：

```text
opscenter-server/internal/dashboard
```

## API

### GET /api/v1/dashboard

回傳目前使用者可存取專案的全域 Dashboard snapshot。

### GET /api/v1/projects/{id}/dashboard

回傳指定 project Dashboard snapshot。

## Request Query

MVP 支援：

| Query | 必填 | 說明 |
| --- | --- | --- |
| `timezone` | 否 | 預設 `Asia/Taipei` |

後續可擴充：

| Query | 說明 |
| --- | --- |
| `sub_project_id` | 單一子專案視圖 |
| `range_days` | 趨勢天數，預設 7 |

## Response DTO

```go
type SnapshotResponse struct {
    Scope                ScopeResponse              `json:"scope"`
    Metrics              MetricsResponse            `json:"metrics"`
    Trend                []TrendPointResponse       `json:"trend"`
    PriorityDistribution []PriorityPointResponse    `json:"priority_distribution"`
    RecentTickets        []RecentTicketResponse     `json:"recent_tickets"`
    GeneratedAt          string                     `json:"generated_at"`
}

type ScopeResponse struct {
    Type        string `json:"type"` // global | project
    ProjectID   string `json:"project_id,omitempty"`
    ProjectName string `json:"project_name,omitempty"`
    Timezone    string `json:"timezone"`
}

type MetricResponse struct {
    Value  int64    `json:"value"`
    Trend  *float64 `json:"trend,omitempty"`
    Source string   `json:"source,omitempty"`
}
```

## Repository

Repository 需提供：

```go
type Repository interface {
    Summary(ctx context.Context, in QueryInput) (Summary, error)
    Trend(ctx context.Context, in QueryInput) ([]TrendPoint, error)
    PriorityDistribution(ctx context.Context, in QueryInput) ([]PriorityPoint, error)
    RecentTickets(ctx context.Context, in QueryInput) ([]RecentTicket, error)
}
```

保留 `Summary` 相容既有 summary API 與測試；Dashboard snapshot 由 service 統一組裝 metrics、trend、priority distribution、recent tickets、scope 與 generated_at。

`QueryInput`：

- `ProjectID`
- `ActorID`
- `ActorGlobalRole`
- `Timezone`
- `TodayStart`
- `TomorrowStart`
- `TrendStart`
- `TrendEnd`

## SQL Strategy

MVP 可使用 4 類查詢：

1. 指標聚合：
   - open tickets
   - created_today
   - resolved_today
   - pending_p1_p2
2. 趨勢：
   - `date_trunc('day', created_at AT TIME ZONE timezone)`
   - `date_trunc('day', resolved_at AT TIME ZONE timezone)`
3. 優先級分佈：
   - group by priority where status 未結束
4. 最近 Tickets：
   - order by created_at desc limit 10

全域 Dashboard 非 admin 使用者必須加入：

```sql
EXISTS (
  SELECT 1
  FROM project_members pm
  WHERE pm.project_id = t.project_id
    AND pm.user_id = ?
)
```

## Permission

- Global:
  - `admin` / `op_admin` 可看全部。
  - 其他使用者只看自己專案成員範圍。
- Project:
  - 呼叫 project access checker，最低角色 viewer。

## Error Mapping

| 錯誤 | HTTP |
| --- | --- |
| current user missing | 401 |
| invalid timezone / invalid request | 400 |
| project not found | 404 |
| permission denied | 403 |
| query failed | 500 |

## OpenAPI

需補 Swagger / OpenAPI 註解，並讓 `make openapi` 產出：

- `/api/v1/dashboard`
- `/api/v1/projects/{id}/dashboard`
- `DashboardSnapshotResponse`
- nested DTO schemas
