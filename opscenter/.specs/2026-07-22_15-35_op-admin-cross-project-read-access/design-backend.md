# Op Admin Cross Project Read Access Backend Design

## Scope

本次修改集中在後端授權層，不調整資料表，不新增 migration，不調整 SSO mapping。核心目標是讓 `op_admin` 對專案 API 的 `read` 操作具備跨專案營運檢視能力，同時保留寫入操作的專案角色邊界。

第二個目標是補齊排班人員選項 API，避免人員班別頁為了取得人員清單而呼叫 `/api/v1/admin/users`。此 API 僅提供選人所需欄位，不提供使用者管理資料。

## Current Behavior

`RequireProjectFormPermission` 目前會呼叫 `AuthorizationService.EnforceProject`：

1. 先呼叫 `Enforce` 驗證表單 `full_path` 與 action。
2. `admin` 直接略過專案角色。
3. 其他角色一律呼叫 `ProjectRoleService.Check` 檢查 `project_members`。

因此 `op_admin` 即使擁有 `reports/read` 或 `jira/report/read`，仍會因未加入專案成員而收到 `403`。

## Target Behavior

`EnforceProject` 調整為：

1. 仍先執行表單權限檢查。
2. `admin` 維持直接放行。
3. `op_admin` 僅在 action 正規化後為 `read` 時略過專案角色。
4. `op_admin` 對 `create`、`update`、`delete` 仍需通過專案角色檢查。
5. `member` 維持既有專案角色檢查。

## Affected APIs

第一批受益 API：

- `GET /api/v1/projects/:id/jira/report`
- `GET /api/v1/projects/:id/jira/export`
- `GET /api/v1/projects/:id/report-templates`
- `GET /api/v1/projects/:id/report-templates/:template_id`
- `GET /api/v1/projects/:id/reports/datasets`
- `GET /api/v1/projects/:id/reports/datasets/:dataset/metadata`
- `GET /api/v1/projects/:id/reports/export`

`POST /api/v1/projects/:id/reports/preview` 目前雖使用 HTTP POST，但 middleware action 是 `read`，因此屬於讀取型預覽，會受本規則放行。

## Non Goals

- 不放寬 `/api/v1/admin/*`。
- 不改 `group_members` 或 `sso_group_mappings`。
- 不讓 `op_admin` 跨專案建立、修改或刪除報表範本。
- 不更動前端選單權限計算。

## User Option API

### Route

新增已登入 API：

```text
GET /api/v1/users/options?keyword=&active_only=true&limit=1000
```

### Authorization

使用 `RequireOpsAdmin()`：

- `admin` 可讀。
- `op_admin` 可讀。
- `member` 不可讀。

`/api/v1/admin/users` 維持 `RequireAdmin()`，不因本 API 放寬。

### Response Contract

```json
{
  "items": [
    {
      "id": "01KY...",
      "username": "terry_d798115f",
      "full_name": "terry",
      "email": "terry@twroa.com",
      "is_active": true
    }
  ],
  "total": 1,
  "limit": 1000
}
```

不得回傳：

- `global_role`
- `has_mfa`
- `remark`
- `created_at`
- `updated_at`

### Query Rules

- `active_only` 預設 `true`。
- 永遠排除 `global_role = admin` 的使用者，避免系統管理員被選入排班班別。
- `keyword` 可比對 `username`、`full_name`、`email`。
- `limit` 上限固定為 `1000`，避免一次取出過大資料量。
- 排序以 `full_name`、`username` 穩定排序。

## Frontend Wiring

人員班別頁需改用新的使用者選項 API：

- 移除 `ScheduleAssignmentsPage` 對 `listAdminUsers()` 的依賴。
- 新增 `listUserOptions()` API client。
- 人員下拉選單只使用 `UserOption` contract。
- 若 API 回 `403`，顯示「目前帳號無法讀取排班人員選項」，不得顯示使用者管理權限錯誤文案。

## Tests

需在 `internal/auth/authorization_test.go` 補測：

- `op_admin` + action `read` + 無專案角色仍通過。
- `op_admin` + action `update` + 專案角色不足仍回 `ErrProjectRoleDenied`。
- `member` + action `read` + 專案角色不足仍回 `ErrProjectRoleDenied`。

使用者選項 API 需補測：

- `op_admin` 可讀取使用者選項。
- `member` 讀取使用者選項回 `403`。
- response 不含 `global_role`、`has_mfa`、`remark`、`created_at`、`updated_at`。
- response 不包含 `global_role = admin` 的使用者。
