# Group Based Operation Permission Design

## Scope

本次設計處理全系統操作授權模型收斂，不只限於 Ticket 與 Jira 匯入。

核心規則：

1. 功能操作是否允許，由群組管理中的表單節點權限與 Casbin policy 決定。
2. `read` 權限同時控制後端讀取 API 與前端可見性。
3. `create`、`update`、`delete` 權限控制對應操作。
4. `project_members` 不參與一般功能操作授權。
5. 值班人員與指派人員不是專案成員概念，不需要被加入特定專案才可操作。
6. `project_id` 仍可作為資料歸屬、查詢條件與報表維度，但不作為一般功能操作授權條件。

## Current Behavior

目前後端部分專案 API 使用 `RequireProjectFormPermission`：

1. 先檢查表單節點權限。
2. 再呼叫 `ProjectRoleService.Check` 檢查 `project_members`。
3. `admin` 可略過專案角色。
4. `op_admin` 目前只有部分 `read` 可略過專案角色，`create`、`update`、`delete` 仍會被專案角色擋住。

因此 Terry 這類 SSO 帳號即使具備 `ops_op_admin` 群組，且群組管理中已授權對應表單節點，仍可能因不是目標專案成員而無法操作。前端也可能顯示入口或按鈕，但後端再用不同條件拒絕，造成權限模型不一致。

## Target Behavior

### 全系統授權規則

所有功能操作分成兩層：

1. 表單節點可見性：由 `read` 決定。
2. 操作可執行性：由 action 決定。

action 對應：

- `read`：讀取、列表、詳情、報表預覽、匯出檢視型資料。
- `create`：新增、匯入、建立週期、建立範本。
- `update`：編輯、狀態變更、指派、重新整理、確認週期、補齊資料。
- `delete`：刪除。

後端不得使用 `project_members` 作為一般功能操作授權的第二道必要條件。若 API 需要 `project_id`，只做資料存在性、啟用狀態、資料歸屬或查詢條件檢查。

### 前端可見性規則

前端不得用 `global_role` 硬編選單與按鈕可見性。前端應使用後端提供的有效表單權限資料：

- 沒有 `read` 的表單節點，不顯示頁面入口。
- 沒有 `read` 的 leaf 節點，不允許進入該頁面。
- 父節點沒有 `read`，但有可讀子節點時，可作為展開容器顯示。
- 父節點與所有子節點都沒有 `read` 時，整個選單區塊不可見。
- 沒有 `create`、`update`、`delete` 時，對應操作按鈕應隱藏或 disabled。若使用 disabled，需提供清楚 tooltip 或無權限文案。

直接輸入 URL 時，前端 route guard 應再次檢查 `read` 權限；後端 API 仍需回 `403`，前端不能作為唯一防線。

### Project Members

`project_members` 保留作為專案成員管理資料，但不再參與一般功能操作授權。至少包含：

- Ticket 建立。
- Ticket 修改。
- Ticket 狀態變更。
- Ticket 指派。
- Ticket 留言與附件。
- Jira 匯入。
- Jira 報表讀取與匯出。
- Report 報表預覽、範本、匯出。
- Schedule 排班週期、人員班別、班別設定與國定假日管理。
- 系統中其他由群組管理節點覆蓋的功能。

專案成員管理 API 本身仍可保留管理權限限制。若後續需要「專案資料可見範圍」或「專案協作邊界」，應另開規格，不與本次操作授權混在一起。

### Project List Visibility

`GET /api/v1/projects` 是專案工作區入口資料來源。它不應再把 `project_members` 當成一般使用者能否看到專案入口的必要條件。

現行問題範例：

1. SSO 使用者 `steven_68bbd75d` 已同步到 `ops_op_member`。
2. `ops_op_member` 已具備 `dashboard`、`tickets`、`reports`、`schedule`、`jira` 等節點的 `read` 權限。
3. `/api/v1/forms/tree` 回傳 `visible_count = 17`，代表表單權限有效。
4. `/api/v1/projects` 因使用者不存在於 `project_members` 或舊 `ops_user` 對應而回傳空清單。
5. 前端顯示「目前沒有可存取的專案」，造成使用者雖有功能權限卻無法進入專案工作區。

目標規則：

- `admin` 與 `op_admin` 維持可列出全部未刪除專案。
- 一般使用者若具備任一專案工作區相關表單的 `read` 權限，`GET /api/v1/projects` 應可列出可進入的 active projects。
- `project_members` 不再作為 `GET /api/v1/projects` 的入口可見性必要條件。
- `projects.visibility = private` 可保留作為資料可見性欄位，但不得在本次設計中隱性等同於 `project_members` 限制。
- 若後續要支援「私有專案只允許特定群組或特定成員可見」，需另開專案資料隔離規格，不可與本次操作授權收斂混在一起。

建議實作方向：

1. 在 service 層判斷呼叫者是否具備專案工作區入口權限。
2. 可重用表單授權服務，檢查下列任一節點 `read` 權限：
   - `tickets`
   - `tickets/list`
   - `reports`
   - `jira`
   - `schedule`
3. 若具備上述任一 `read` 權限，`ListProjects` 不套用 `project_members` scope。
4. 若完全沒有專案工作區相關 `read` 權限，回傳空清單或 `403` 需保持前後端一致；本階段建議回傳空清單，讓首頁顯示沒有可進入專案。
5. 專案成員管理 API 本身仍可保留成員管理規則，不因專案清單放寬而自動放寬。

## Backend Design

### System Logs

`system/logs` 是群組管理可授權的讀取節點，不應因頁面路徑位於 `/admin/logs` 而被前端或後端硬性限制為 `admin`。

為了保留 `/api/v1/admin/*` 仍只允許 `admin` 的安全邊界，日誌查詢需提供等價的非 admin API：

```text
GET /api/v1/system/logs/login
GET /api/v1/system/logs/activity
```

授權規則：

- `/api/v1/admin/logs/*` 維持 `RequireAdmin()`。
- `/api/v1/system/logs/*` 使用 `RequireFormPermission(authz, "system/logs", "read")`。
- 前端日誌查詢頁改使用 `/api/v1/system/logs/*`。
- `/admin/logs` 頁面 route guard 使用 `system/logs read`，不得再被整個 `/admin` layout 的 `adminOnly` 擋住。

### Middleware

保留一般表單權限 middleware：

```text
RequireFormPermission(authz, fullPath, action)
```

專案型 API 若只是需要 `project_id` 作為資料歸屬，應避免使用會檢查 `project_members` 的 middleware。可改為：

```text
RequireFormPermission(authz, fullPath, action)
ValidateProject(projectID)
```

`ValidateProject` 只驗證專案存在、啟用狀態與必要的資料條件，不判斷呼叫者是否為專案成員。

### Project Role Check

`RequireProjectFormPermission` 與 `ProjectRoleService.Check` 不應再作為一般功能操作授權路徑。若保留，必須只用於明確規格指定的專案成員管理場景。

### Permission Contract API

前端需要一份登入者有效權限資料。若既有 API 已能取得表單樹與權限，需確認契約包含：

- `full_path`
- `can_read`
- `can_create`
- `can_update`
- `can_delete`
- 父子節點關係
- 來源群組可不用回傳給一般前端

若既有 API 不完整，需補齊有效權限 API，避免前端自行拼湊角色規則。

## Frontend Design

### Sidebar

側邊選單渲染流程：

1. 取得有效表單權限樹。
2. 過濾不可見節點。
3. leaf 節點需 `can_read = true` 才顯示。
4. parent 節點若本身不可讀但仍有可讀子節點，可顯示為展開容器。
5. parent 節點沒有可讀子節點時不顯示。

### Route Guard

每個受保護頁面需對應一個 `full_path`。route guard 依 `can_read` 判斷：

- 有 `read`：允許進入。
- 無 `read`：顯示無權限頁或導回第一個可用頁面。

### Action Buttons

按鈕顯示依 action 權限：

- `can_create`：新增、匯入、建立週期。
- `can_update`：編輯、確認、重新整理、狀態變更。
- `can_delete`：刪除。

若保留 disabled 狀態，需提供無權限提示，且樣式與既有按鈕規範一致。

## Non Goals

- 不調整 SSO group mapping。
- 不調整 `groups`、`group_members`、`group_form_permissions` schema。
- 不調整群組管理 UI。
- 不放寬 `/api/v1/admin/*`。
- 不重新設計專案成員管理。
- 不將 `project_members` 刪除或 migration 到其他表。

## Tests

後端需補測：

- 有表單節點 action 權限但無 `project_members`，操作可通過授權。
- 無表單節點 action 權限，即使是啟用帳號也回 `403`。
- 專案不存在或停用時，仍回對應資料錯誤，不誤判為專案成員不足。
- `RequireProjectFormPermission` 不再被一般功能操作路由使用，或只保留於明確例外。

前端需補測：

- 沒有 `read` 權限時，側邊選單不顯示該 leaf 節點。
- 沒有任何可讀子節點時，父節點不顯示。
- 有可讀子節點時，父節點可顯示為容器。
- 直接輸入無 `read` 權限 route 時，顯示無權限狀態或導回可用頁面。
- 沒有 action 權限時，對應按鈕隱藏或 disabled。

回歸測試需確認：

- `/api/v1/admin/*` 仍只允許 `admin`。
- 群組沒有對應權限時仍被拒絕。
- 既有 `op_admin` 與 `op_member` 的群組權限行為不被角色硬編覆蓋。
