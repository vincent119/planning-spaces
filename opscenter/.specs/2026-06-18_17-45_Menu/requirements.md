# Menu Requirements

## 文件定位

本文件定義前端側邊欄與選單管理選單樹的整合需求。既有 Auth、Admin、Ticket、Dashboard 與後端 API 細節仍以原 spec 為準；本 spec 只補齊側邊欄可見性、排序與選單樹依賴規則。

## 1. 側邊欄來源

- [ ] 1.1 前端側邊欄不得再以硬編陣列作為最終顯示與排序來源。
- [ ] 1.2 側邊欄入口與子選單需依 `GET /api/v1/forms/tree` 回傳的有效選單樹顯示。
- [ ] 1.3 `GET /api/v1/forms/tree` 只可回傳目前使用者具備有效 `read` 權限且 `is_active = true` 的 `form_nodes` 節點。
- [ ] 1.4 前端不得使用 `/api/v1/admin/forms` 作為一般使用者側邊欄來源；管理端 API 只用於選單管理頁。
- [ ] 1.5 API 載入失敗時，不得顯示未經授權確認的假入口；需顯示錯誤或最小安全狀態。

## 2. 排序

- [ ] 2.1 側邊欄同層節點需依 `sort_order ASC` 排序。
- [ ] 2.2 `sort_order` 相同時，需依 `form_key ASC` 做穩定排序。
- [ ] 2.3 父節點與子節點皆需套用同一排序規則。
- [ ] 2.4 後端調整 `form_nodes.sort_order` 後，前端重新讀取選單樹即需反映新順序。

## 3. 權限與可見性

- [ ] 3.1 所有側邊欄入口都必須走選單權限，不得只對系統管理套用權限。
- [ ] 3.2 沒有有效 `read` 權限的節點不得顯示於側邊欄、子選單或可操作入口。
- [ ] 3.3 直接開啟 route 時，後端 API 仍需執行授權檢查；前端隱藏選單不得取代後端權限。
- [ ] 3.4 `global_role`、OIDC 群組名稱或 `roles.code` 不得被前端硬編碼為側邊欄顯示依據。
- [ ] 3.5 權限來源固定為內部模型：`groups`、`group_members`、`group_form_permissions` 與同步後的 Casbin policy。
- [ ] 3.6 選單權限必須支援 Admin 自定義權限群組；預設群組只是初始化 seed，不是唯一可用權限模型。
- [ ] 3.7 自定義權限群組與預設權限群組在有效權限計算中同等有效。
- [ ] 3.8 Admin 可將使用者加入多個自定義或預設權限群組；最終有效權限採群組聯集。
- [ ] 3.9 Admin 可針對任一自定義權限群組設定 `read/create/update/delete`、`inherit_children` 與 `override_parent`。
- [ ] 3.10 自定義權限儲存後，必須先寫入 `group_form_permissions`，再由後端同步 Casbin policy；前端不得直接讀寫 `casbin_rule`。
- [ ] 3.11 自定義權限不得繞過 project scope 權限；具備功能節點權限不代表可存取所有專案資料。
- [ ] 3.12 角色管理不得承載選單權限設定；角色只管理 `roles` 與 `users.global_role` 的系統層級身分。
- [ ] 3.13 群組管理需成為系統管理下的獨立功能入口，用於管理 `groups`、`group_members` 與 `group_form_permissions`。
- [ ] 3.14 選單管理只管理 `form_nodes` 節點本身，不再承載權限群組、群組成員或權限矩陣。

## 4. 預設選單樹

- [ ] 4.1 後端需提供一般側邊欄預設選單節點 seed：
  - `dashboard`
  - `tickets`
  - `reports`
  - `jira`
- [ ] 4.2 後端需提供 Tickets 子選單預設節點 seed：
  - `tickets/sub-projects`
  - `tickets/members`
  - `tickets/sources`
  - `tickets/ticket-types`
  - `tickets/list`
- [ ] 4.3 後端需保留既有系統管理預設節點 seed：
  - `system`
  - `system/users`
  - `system/roles`
  - `system/menus`
  - `system/groups`
  - `system/projects`
  - `system/logs`
  - `system/schedulers`
  - `system/settings`
- [ ] 4.4 預設節點必須寫入 `form_nodes`，不得只存在於前端常數。
- [ ] 4.5 預設權限必須先寫入 `group_form_permissions`，再由後端同步或 seed 對應 Casbin policy；不得只寫入 `casbin_rule`。

## 5. Frontend 行為

- [ ] 5.1 前端需以 `form_nodes.full_path` 作為 route、icon 與 i18n label mapping 的穩定 key。
- [ ] 5.2 無 route mapping 的節點不得顯示，避免產生無法導航或 route not found 的入口。
- [ ] 5.3 `node_type = root` 或 `category` 可渲染為可展開群組。
- [ ] 5.4 `node_type = form` 可渲染為可導航項目。
- [ ] 5.5 `Tickets` 與 `系統管理` 都需走同一套樹狀 renderer，不得各自維護獨立排序邏輯。
- [ ] 5.6 前端仍可保留 route/icon mapping 常數，但 mapping 只能負責呈現與導向，不得決定最終可見性與排序。
