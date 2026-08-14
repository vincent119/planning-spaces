# Menu Design

## 目標

前端側邊欄改由後端有效選單樹驅動。選單管理頁維護 `form_nodes` 的父子結構、排序與啟用狀態；一般側邊欄只讀取目前使用者有 `read` 權限的 active 節點，並依樹狀資料排序顯示。

## 既有基礎

- 管理端選單節點 API：`GET /api/v1/admin/forms`
- 一般使用者有效選單樹 API：`GET /api/v1/forms/tree`
- 權限來源表：`groups`、`group_members`、`group_form_permissions`
- Runtime 授權：Casbin policy
- 節點排序欄位：`form_nodes.sort_order`

## API Contract

前端側邊欄只讀取：

```http
GET /api/v1/forms/tree
```

Response `data` 為樹狀陣列：

```ts
type EffectiveMenuNode = {
  id: string;
  parent_id?: string;
  node_type: 'root' | 'category' | 'form';
  form_key: string;
  full_path: string;
  name: string;
  description?: string;
  sort_order: number;
  children?: EffectiveMenuNode[];
};
```

後端責任：

- 只回傳 `is_active = true` 的節點。
- 只回傳目前使用者具備 `read` 權限的節點。
- 以 `parent_id` 組成樹。
- 同層依 `sort_order ASC, form_key ASC` 排序。
- 父節點不可見但子節點可見時，可將可見子節點提升為 root，避免授權結果被丟棄。

前端責任：

- 不重新推測權限。
- 不使用 `/api/v1/admin/forms` 作為一般側邊欄來源。
- 只針對已知 `full_path` 做 route/icon/i18n mapping。
- 未知 `full_path` 不顯示。

## 預設節點設計

預設選單樹需包含以下節點。`sort_order` 數字越小越前面。

| full_path | node_type | 顯示用途 | sort_order |
| --- | --- | --- | --- |
| `dashboard` | form | 儀表板 | 10 |
| `tickets` | root | Tickets 父選單 | 20 |
| `tickets/sub-projects` | form | 子專案 | 10 |
| `tickets/members` | form | 專案成員 | 20 |
| `tickets/sources` | form | 資訊來源 | 30 |
| `tickets/ticket-types` | form | 事件類型 | 40 |
| `tickets/list` | form | Ticket 列表 | 50 |
| `reports` | form | 報表 | 30 |
| `jira` | form | Jira | 40 |
| `system` | root | 系統管理父選單 | 90 |
| `system/users` | form | 用戶管理 | 10 |
| `system/roles` | form | 角色管理 | 20 |
| `system/menus` | form | 選單管理 | 30 |
| `system/groups` | form | 群組管理 | 40 |
| `system/projects` | form | 專案管理 | 50 |
| `system/logs` | form | 日誌查詢 | 60 |
| `system/schedulers` | form | 排程管理 | 70 |
| `system/settings` | form | 全域設定 | 80 |

## 系統管理資訊架構

系統管理下的角色、選單與權限群組需拆清楚，避免使用者誤解權限來源。

| 功能 | route | 管理資料 | 不負責 |
| --- | --- | --- | --- |
| 角色管理 | `/admin/roles` | `roles`、角色顯示名稱、角色狀態 | 不設定選單權限 |
| 選單管理 | `/admin/menus` | `form_nodes`、父子結構、顯示名稱、排序、啟用狀態 | 不管理群組成員與權限矩陣 |
| 群組管理 | `/admin/groups` | `groups`、`group_members`、`group_form_permissions` | 不修改角色定義與選單節點結構 |

`/admin/menus?tab=permissions` 不再作為目標資訊架構。既有選單權限功能需搬移為 `/admin/groups`，畫面名稱改為「群組管理」。群組管理內仍需提供：

- 權限群組列表
- 權限群組詳情
- 群組成員管理
- 選單權限矩陣
- 新增 / 編輯 / 停用 / 刪除權限群組

角色與選單權限只可間接關聯：

```text
users.global_role / roles
        │
        │ 不直接綁選單節點
        ▼
OIDC mapping 或 Admin 手動加入群組
        ▼
group_members
        ▼
group_form_permissions
        ▼
form_nodes
```

若 OIDC 群組需要同時給角色與選單權限，需透過 mapping 同步兩件事：

- 設定 `users.global_role`
- 加入對應內部權限群組

不得以角色代碼硬編碼選單可見性或操作權限。

## 預設權限設計

全部側邊欄入口都走權限，沒有 `read` 權限就不顯示。

- `ops_admin`
  - 對全部預設節點具備 `read`。
  - 對 `system` 子樹維持既有 `read/create/update/delete`。
- `ops_op_admin`
  - 對一般工作入口 `dashboard`、`tickets` 子樹、`reports`、`jira` 具備 `read`。
  - 對 `system/logs` 具備 `read`。
  - 不因本設計取得其他系統管理子節點操作權限。
- `ops_member` / `ops_op_member`
  - 若群組存在，可對一般工作入口 `dashboard`、`tickets` 子樹、`reports`、`jira` 具備 `read`。
  - 不預設取得 `system` 子樹權限。

若某環境尚未建立 `ops_member` 或 `ops_op_member` 權限群組，seed 不得失敗；可只對已存在群組建立權限，或由後續 OIDC mapping / 管理端補齊。

## 自定義權限設計

預設權限只負責初始化，不代表系統只能使用預設群組。正式授權模型必須允許 Admin 透過群組管理建立自定義權限群組，並將該群組套用到任意選單節點。

### 自定義群組

- Admin 可在群組管理建立自定義 `groups`，例如 `ticket_readonly`、`ticket_operator`、`audit_viewer`。
- 自定義群組與 `ops_admin`、`ops_op_admin` 等預設群組使用同一張 `groups` 表。
- 自定義群組不得以 `roles.code`、OIDC 原始群組名稱或前端常數取代。
- OIDC 群組若要對應自定義權限，應透過 OIDC group mapping 將外部群組同步為內部 `group_members`，不直接改變權限判斷來源。

### 自定義權限設定

- Admin 可對自定義群組設定任意 `form_nodes` 節點權限。
- 可設定 action：
  - `read`
  - `create`
  - `update`
  - `delete`
- 可設定繼承行為：
  - `inherit_children`
  - `override_parent`
- 自定義權限必須寫入 `group_form_permissions`。
- 後端需依 `group_form_permissions` 同步 Casbin policy；前端不得直接讀寫 `casbin_rule`。

### 多群組合併

同一使用者可同時屬於多個預設或自定義群組。有效權限採聯集：

- 任一群組具備 `tickets/list:read`，使用者即可看到 Ticket 列表入口。
- 任一群組具備 `tickets/list:create`，使用者即可看到建立 Ticket 的操作入口。
- 若某群組沒有權限，不會抵銷另一群組已授予的權限。
- 若未來需要 deny 規則，需新增明確資料模型與設計；目前不以停用或空權限表示 deny。

### 自定義權限範例

| 群組 code | 用途 | 節點權限 |
| --- | --- | --- |
| `ticket_readonly` | 只讀 Ticket 工作區 | `tickets:read` 並繼承 children |
| `ticket_operator` | Ticket 維運操作 | `tickets/list:read/create/update`、`tickets/members:read`、`tickets/sources:read` |
| `audit_viewer` | 稽核查詢 | `system/logs:read` |
| `menu_admin` | 選單節點管理 | `system/menus:read/create/update/delete` |
| `group_admin` | 權限群組管理 | `system/groups:read/create/update/delete` |

### 自定義權限邊界

- 自定義權限控制功能入口與操作能力，不取代專案資料範圍權限。
- 使用者具備 `tickets/list:read` 後，仍只能看到後端 project permission 允許的專案資料。
- 使用者具備 `system/menus:update` 後，仍需通過後端 API 的管理端授權檢查。
- 自定義群組停用後，不得再產生有效 Casbin policy。

## 權限控制設計

Menu 權限控制分成三層：選單可見性、頁面操作能力、後端 API enforcement。前端只能改善使用體驗，不能作為唯一安全邊界。

### 權限來源

- `form_nodes` 定義可授權資源，`full_path` 是穩定授權物件。
- `groups` 定義權限群組，與 `roles`、`users.global_role` 分離。
- `group_members` 定義使用者與權限群組關係；同一使用者可同時屬於多個群組。
- `group_form_permissions` 是業務授權來源表，保存 `read/create/update/delete` 與繼承設定。
- Casbin policy 是 runtime enforcement 資料，需由後端依 `group_form_permissions` 同步；前端不得直接讀寫 `casbin_rule`。

### 有效權限計算

- 使用者有效權限為所有啟用權限群組的聯集。
- 任一群組對同一 `form_node` 具備某 action，即視為使用者具備該 action。
- `inherit_children = true` 時，該權限可套用到目前節點所有 descendant。
- `override_parent = true` 時，該節點的直接設定覆寫父節點繼承結果。
- `is_active = false` 的 group、group member 或 form node 不得產生有效權限。
- `read` 是選單可見性的最低條件；沒有 `read` 時，即使具備其他 action，也不得顯示該節點。

### 前端使用規則

- Sidebar 只使用 `GET /api/v1/forms/tree` 判斷可見節點。
- Sidebar 不呼叫 `/api/v1/forms/permissions` 逐項推算可見性，避免前端重建權限邏輯。
- 頁面內操作按鈕若需要依權限控制，應使用：

```http
GET /api/v1/forms/permissions?path={full_path}
```

- `read` 控制頁面入口與查看能力。
- `create` 控制新增按鈕與建立流程入口。
- `update` 控制編輯、啟用 / 停用、狀態切換等修改操作。
- `delete` 控制刪除按鈕與刪除流程入口。
- 前端隱藏或 disabled 按鈕只作為 UX；實際 API 仍需由後端回傳 401 / 403 / 409 等錯誤。

### API Enforcement 對應

後端需要將業務 API 對應到穩定 `full_path` 與 action。第一版對應如下：

| UI / API 範圍 | full_path | read | create | update | delete |
| --- | --- | --- | --- | --- | --- |
| 儀表板 | `dashboard` | 查看 Dashboard | - | - | - |
| Ticket 列表 | `tickets/list` | 列表 / 詳情入口 | 建立 Ticket | 更新 Ticket / 狀態 / 指派 / 留言 | 刪除 Ticket |
| 子專案管理 | `tickets/sub-projects` | 列表 / 詳情 | 新增子專案 | 編輯 / 啟停 | 刪除 |
| 專案成員管理 | `tickets/members` | 列表 | 新增成員 | 更新成員 | 移除成員 |
| 資訊來源管理 | `tickets/sources` | 列表 | 新增資訊來源 | 編輯 / 啟停 | soft delete |
| 事件類型管理 | `tickets/ticket-types` | 列表 | 新增事件類型 | 編輯 / 啟停 | soft delete |
| 報表 | `reports` | 查看報表 | - | - | - |
| Jira | `jira` | 查看 Jira 頁 | 匯入 | 更新設定或狀態 | 刪除匯入資料 |
| 用戶管理 | `system/users` | 列表 / 詳情 | 新增用戶 | 編輯 / 啟停 | 刪除 |
| 角色管理 | `system/roles` | 列表 / 詳情 | 新增角色 | 編輯 / 啟停 | 刪除 |
| 選單管理 | `system/menus` | 查看節點 | 新增節點 | 編輯節點 / 啟停 | 刪除節點 |
| 群組管理 | `system/groups` | 查看群組 / 成員 / 權限矩陣 | 新增群組 / 成員 / 權限 | 編輯群組 / 成員 / 權限 | 刪除群組 / 成員 / 權限 |
| 專案管理 | `system/projects` | 列表 / 詳情 | 新增專案 | 編輯 / 啟停 | 刪除 |
| 日誌查詢 | `system/logs` | 查詢日誌 | - | - | - |
| 排程管理 | `system/schedulers` | 列表 / 記錄 | 新增任務 | 編輯 / 啟停 | 刪除 |
| 全域設定 | `system/settings` | 查看設定 | 新增設定 | 編輯設定 | 刪除設定 |

若某功能尚未實作對應 API enforcement，前端仍需依本設計顯示 / 隱藏入口；後端 enforcement task 需另行補齊，不得用前端隱藏宣稱安全完成。

### 專案範圍權限邊界

`form_nodes` 控制功能入口與操作類型，不取代 project scope 權限。

- 使用者具備 `tickets/list:read` 只代表可進入 Ticket 列表功能。
- 使用者是否能讀取某個 project 的 Ticket，仍需由 project membership / project role 判斷。
- 私有專案、專案成員、專案管理員等資料邊界仍由既有 Project / Ticket 後端權限控制。
- API enforcement 需同時檢查 form permission 與 project permission；任一不通過即拒絕。

### 權限資料異動

- Admin 在選單管理或群組管理修改節點、權限群組或群組成員後，後端需同步 Casbin policy。
- 選單管理只會異動 `form_nodes`；群組管理才會異動 `groups`、`group_members` 與 `group_form_permissions`。
- 前端成功修改權限後，需 invalidate：
  - admin forms query
  - permission groups query
  - effective menu tree query
  - 目前頁面使用的 permissions query
- 使用者自身權限被移除後，下一次選單樹刷新需移除已失效入口。
- 若使用者停留在已失效頁面，後端 API 應回 403，前端顯示無權限狀態並提供返回可見入口的操作。

## Frontend Adapter

前端建立 `full_path` mapping。此 mapping 只負責呈現與導向，不負責權限或排序。

| full_path | route | i18n key | icon |
| --- | --- | --- | --- |
| `dashboard` | `/dashboard` | `nav.dashboard` | `DashboardIcon` |
| `tickets` | current project tickets fallback | `nav.tickets` | `ConfirmationNumberIcon` |
| `tickets/sub-projects` | `/projects/:id/sub-projects` | `nav.ticket_sub_projects` | `SegmentIcon` |
| `tickets/members` | `/projects/:id/members` | `nav.ticket_members` | `GroupsIcon` |
| `tickets/sources` | `/projects/:id/sources` | `nav.ticket_sources` | `SourceIcon` |
| `tickets/ticket-types` | `/projects/:id/ticket-types` | `nav.ticket_types` | `CategoryIcon` |
| `tickets/list` | `/projects/:id/tickets` | `nav.ticket_list` | `ConfirmationNumberIcon` |
| `reports` | `/projects/:id/reports` 或 `/projects` fallback | `nav.reports` | `BarChartIcon` |
| `jira` | `/projects/:id/jira` 或 `/projects` fallback | `nav.jira` | `ImportExportIcon` |
| `system` | 展開群組 | `nav.admin` | `SettingsIcon` |
| `system/users` | `/admin/users` | `nav.admin_users` | `PersonIcon` |
| `system/roles` | `/admin/roles` | `nav.admin_roles` | `GroupsIcon` |
| `system/menus` | `/admin/menus` | `nav.admin_menus` | `TableChartIcon` |
| `system/groups` | `/admin/groups` | `nav.admin_groups` | `GroupsIcon` |
| `system/projects` | `/admin/projects` | `nav.admin_projects` | `AccountTreeIcon` |
| `system/logs` | `/admin/logs` | `nav.admin_logs` | `ArticleIcon` |
| `system/schedulers` | `/admin/schedulers` | `nav.admin_schedulers` | `ScheduleIcon` |
| `system/settings` | `/admin/system` | `nav.admin_global` | `TuneIcon` |

## Rendering Rules

- `node_type = root` 或 `category`
  - 有可顯示 children 時渲染為可展開群組。
  - 無可顯示 children 且自身沒有 route mapping 時不顯示。
- `node_type = form`
  - 有 route mapping 時渲染為可導航項目。
  - 無 route mapping 時不顯示。
- `Tickets` 父節點點擊行為：
  - 展開狀態下切換展開 / 收合。
  - 收合 sidebar 狀態下導向目前專案的 Ticket 列表；若沒有 current project，導向 `/projects` fallback。
- `system` 父節點點擊行為：
  - 展開狀態下切換展開 / 收合。
  - 不用 `global_role` 決定是否顯示；是否出現完全取決於 `/forms/tree` 是否回傳該節點或其可顯示子節點。

## Error / Loading

- 首次載入選單樹時，可顯示 sidebar skeleton 或保留空白安全狀態。
- 載入失敗時，不顯示假入口；可顯示簡短錯誤文字與重新整理按鈕。
- 重新登入、權限異動、語系切換不應破壞目前展開狀態；權限資料重新載入後，已不存在的節點需自動消失。
