# Group Based Operation Permission Verification

## 目的

確認一般操作授權已改為群組表單節點權限主導，`project_members` 不再參與 Ticket、Jira、Report、Schedule 等一般功能操作授權。`project_id` 只作為資料歸屬與資料查詢條件，管理端 `/api/v1/admin/*` 仍維持 `admin` 專用。

## SQL 檢查

### 1. 查詢使用者 SSO 身分與群組歸屬

```sql
SET search_path TO opscenter, public;

SELECT
  u.id,
  u.username,
  u.email,
  u.global_role,
  u.is_active,
  s.provider_key,
  s.protocol,
  s.external_subject,
  s.last_login_at
FROM sso_external_identities s
JOIN users u ON u.id = s.user_id
WHERE lower(u.email) = lower(:email)
   OR u.username = :username;

SELECT
  u.username,
  g.code AS group_code,
  g.name AS group_name,
  g.is_active AS group_active,
  gm.source,
  gm.source_provider
FROM users u
JOIN group_members gm ON gm.user_id = u.id
JOIN groups g ON g.id = gm.group_id
WHERE lower(u.email) = lower(:email)
   OR u.username = :username
ORDER BY g.code;
```

### 2. 查詢群組表單節點權限與 Casbin 規則

```sql
SET search_path TO opscenter, public;

SELECT
  u.username,
  g.code AS group_code,
  fn.full_path,
  gfp.can_read,
  gfp.can_create,
  gfp.can_update,
  gfp.can_delete,
  gfp.inherit_children,
  gfp.override_parent
FROM users u
JOIN group_members gm ON gm.user_id = u.id
JOIN groups g ON g.id = gm.group_id
JOIN group_form_permissions gfp ON gfp.group_id = g.id
JOIN form_nodes fn ON fn.id = gfp.form_node_id
WHERE lower(u.email) = lower(:email)
   OR u.username = :username
ORDER BY g.code, fn.full_path;

SELECT
  v0 AS subject,
  v1 AS full_path,
  string_agg(v2, ', ' ORDER BY v2) AS actions
FROM casbin_rule
WHERE ptype = 'p'
  AND v0 IN (
    SELECT 'group::' || g.code
    FROM users u
    JOIN group_members gm ON gm.user_id = u.id
    JOIN groups g ON g.id = gm.group_id
    WHERE lower(u.email) = lower(:email)
       OR u.username = :username
  )
GROUP BY v0, v1
ORDER BY v0, v1;
```

### 3. 確認使用者不在專案成員仍可依群組權限操作

```sql
SET search_path TO opscenter, public;

SELECT
  u.id AS user_id,
  u.username,
  u.global_role,
  p.id AS project_id,
  p.name AS project_name,
  pm.role AS project_role
FROM users u
CROSS JOIN projects p
LEFT JOIN project_members pm
  ON pm.user_id = u.id
 AND pm.project_id = p.id
WHERE (lower(u.email) = lower(:email) OR u.username = :username)
  AND p.id = :project_id;
```

驗收重點：

- `project_role` 為 `NULL` 時，只要群組對應節點有 action 權限，一般操作仍可執行。
- 若群組節點沒有 `read`，前端不得顯示該選單或允許直接進入頁面。
- 若群組節點沒有 `create` / `update` / `delete`，前端需隱藏或停用對應操作，後端也需回 `403`。

## API 操作驗收

1. 使用 `op_admin` SSO 使用者登入。
2. 確認使用者不在測試專案的 `project_members`。
3. 驗證 Jira 匯入可依 `jira/import create` 執行。
4. 驗證報表範本與報表查詢可依 `reports` 權限執行。
5. 驗證 Ticket 建立、修改、狀態變更、留言與附件依 `tickets/list` action 權限控制。
6. 驗證排班週期、人員班別與班別設定依 `schedule/periods`、`schedule/assignments`、`schedule/shifts` 權限控制。
7. 使用沒有 `read` 權限的群組登入，確認側邊選單不顯示該節點，直接輸入 URL 時顯示無權限畫面。
8. 使用非 `admin` 使用者呼叫 `/api/v1/admin/users`，確認仍回 `403`。

## 自動化驗證

```bash
cd opscenter-server
go test ./internal/auth ./internal/form ./internal/jira ./internal/report ./internal/ticket

cd ../opscenter-frontend
npm run typecheck
```
