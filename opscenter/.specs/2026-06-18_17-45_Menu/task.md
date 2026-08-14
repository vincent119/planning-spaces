# Menu Tasks

## 文件定位

本文件追蹤側邊欄依選單樹排序與權限顯示的後續工作。不得回頭修改其他 spec 中已完成 task 的狀態；若需修正舊文件敘述，需在執行對應 task 時明確記錄。

## 0. 文件與契約對齊

- [x] 0.1 對齊舊 Ticket 側欄固定排序文件
  - 修正 Ticket spec 中「側欄子選單順序固定」的敘述
  - 改為「預設 seed 排序為子專案、專案成員、資訊來源、事件類型、Ticket 列表；最終顯示順序由 `form_nodes.sort_order` 決定」
  - 不修改已完成 task 狀態；新增註記指向本 Menu spec
  - 完成：已更新 Ticket `design-frontend.md` 與 `task-frontend.md`，保留既有已完成 task 狀態並新增 Menu spec 對齊註記
  - _Requirements: 1.1, 2.1-2.4_

- [x] 0.2 對齊舊 Admin 選單管理文件
  - 補充一般側邊欄也使用 `GET /api/v1/forms/tree`
  - 明確區分管理端 `/api/v1/admin/forms` 與一般有效選單 `/api/v1/forms/tree`
  - 完成：已更新舊系統 `design.md` 與 `design-frontend.md`，明確區分管理端選單節點 API、一般有效選單 API 與 `/admin/groups` 群組管理資訊架構
  - _Requirements: 1.2-1.4_

## 1. Server seed 與權限

- [x] 1.1 補齊一般側邊欄預設 `form_nodes` seed
  - 新增或更新 `dashboard`、`tickets`、`reports`、`jira`
  - 使用 idempotent SQL
  - 不破壞既有 `system` seed
  - 完成：新增 `opscenter-server/sql/0028_seed_menu_tree_permissions.sql`，以 `ON CONFLICT (full_path)` 補齊一般側邊欄預設節點並維持可重跑
  - _Requirements: 4.1, 4.4_

- [x] 1.2 補齊 Tickets 子選單 `form_nodes` seed
  - 新增或更新 `tickets/sub-projects`
  - 新增或更新 `tickets/members`
  - 新增或更新 `tickets/sources`
  - 新增或更新 `tickets/ticket-types`
  - 新增或更新 `tickets/list`
  - 預設排序為 10、20、30、40、50
  - 完成：`0028_seed_menu_tree_permissions.sql` 已補齊 Tickets 子選單並固定預設排序為子專案、專案成員、資訊來源、事件類型、Ticket 列表
  - _Requirements: 4.2, 4.4_

- [x] 1.3 補齊一般側邊欄預設權限 seed
  - 權限先寫入 `group_form_permissions`
  - 同步或 seed 對應 Casbin policy
  - `ops_admin` 可讀全部預設節點
  - `ops_op_admin` 可讀一般工作入口與 `system/logs`
  - 已存在的 `ops_member` / `ops_op_member` 可讀一般工作入口
  - 完成：`0028_seed_menu_tree_permissions.sql` 已建立 `ops_admin`、`ops_op_admin`、`ops_member`、`ops_op_member` 的預設讀取權限，並從 `group_form_permissions` 同步限定範圍的 Casbin policy
  - _Requirements: 3.1-3.5, 4.5_

- [x] 1.4 補 Server 測試
  - 驗證 `/api/v1/forms/tree` 只回有 read 權限節點
  - 驗證同層排序使用 `sort_order` 與 `form_key`
  - 驗證缺少某子節點權限時不回傳該節點
  - 完成：已補 `effective_service_test.go` 的同層排序與缺少子節點 read 權限測試；驗證 `go test ./internal/form`、`go test ./...` 通過
  - _Requirements: 2.1-2.4, 3.1-3.3_

- [x] 1.5 清理舊版 `ticket` 選單節點 seed
  - 舊版 `0004_create_form_permission_tables.sql` 曾建立 `ticket` 與 `ticket/create`
  - 新版正確入口是 `tickets` 與 `tickets/*`
  - 需新增 idempotent SQL，停用或移除舊版 `ticket` / `ticket/create` 節點與對應權限，避免選單管理畫面同時顯示 `ticket` 與 `tickets`
  - 清理後 `/api/v1/forms/tree` 與管理端選單樹不得再顯示舊版 `ticket`
  - 完成：新增 `opscenter-server/sql/0029_remove_legacy_ticket_menu_nodes.sql`，先移除 legacy `ticket` / `ticket/*` 的 Casbin policy 與 `group_form_permissions`，再刪除 child/root 節點
  - 驗證：`go test ./internal/form`、`go test ./...` 通過
  - _Requirements: 4.1-4.5, 5.1-5.6_

- [x] 1.6 補齊權限子節點繼承展開
  - `inherit_children = true` 時需將父節點 action 展開到所有 descendant `form_nodes`
  - `override_parent = true` 的子節點需使用自己的直接設定覆寫父層繼承
  - 同步 `casbin_rule` 時不得殘留已取消的 inherited policy
  - 權限矩陣列表需顯示 `inherited` 來源，避免 UI 與 runtime policy 不一致
  - 補測試覆蓋父節點繼承、子節點覆寫、刪除父節點權限後移除 inherited policy
  - 完成：`group_form_permissions.inherit_children` 已展開至 descendant 節點，`override_parent` 可覆寫父層繼承；同步 Casbin policy 時會重建該群組有效權限，避免 inherited policy 殘留；權限矩陣列表會回傳 `inherited` 來源
  - 驗證：`go test ./...`、`npm run typecheck`、`npm run build` 通過
  - _Requirements: 3.9, 3.10_

## 2. Frontend API 與 Adapter

- [x] 2.1 建立 effective menu API client
  - 新增 `GET /api/v1/forms/tree` client
  - 型別對齊 `EffectiveMenuNode`
  - Query key 不與 admin forms 混用
  - 完成：新增 `features/menu/api/effectiveMenu.ts` 與 `features/menu/types`，使用 `effectiveMenuTreeQueryKey = ['menu', 'effective-tree']`
  - _Requirements: 1.2-1.4_

- [x] 2.2 建立 sidebar menu adapter
  - 以 `full_path` 對應 route、icon、i18n key
  - mapping 不負責排序與權限
  - 未知 `full_path` 不顯示
  - 完成：新增 `features/menu/sidebar/adapter.ts`，依 `full_path` 建立 route、`iconKey`、i18n key mapping；未知節點不建立自身入口，只保留可映射子節點
  - _Requirements: 5.1, 5.2, 5.6_

- [x] 2.3 建立樹狀 sidebar render model
  - 將有效選單樹轉成 sidebar render items
  - 支援群組與子選單
  - `Tickets` 與 `system` 使用同一 renderer
  - 完成：新增 `buildSidebarMenuItems`，輸出 `group` / `item` render model，`tickets` 與 `system` 共用同一轉換流程；已補 `nav.admin_groups` i18n key
  - _Requirements: 5.3-5.5_

## 3. Frontend Sidebar 整合

- [x] 3.1 將 `AppSidebar` 改讀有效選單樹
  - 移除最終顯示順序對硬編陣列的依賴
  - 保留 current project route fallback 行為
  - 保留 sidebar 展開 / 收合互動
  - 完成：`AppSidebar` 已使用 `listEffectiveMenuTree` 與 `buildSidebarMenuItems` 產生顯示項目，保留 current project fallback 與群組展開 / 收合
  - _Requirements: 1.1, 1.2, 2.1-2.4_

- [x] 3.2 套用全權限可見性
  - 不再用 `currentUser.global_role === 'admin'` 決定系統管理是否顯示
  - 是否顯示只取決於 `/forms/tree` 回傳結果與 frontend route mapping
  - 完成：Sidebar 不再依 `global_role` 顯示 Admin 群組；`/admin` layout 取消 frontend-only admin gate，實際入口仍由 `/forms/tree` 與後端 API 權限控制
  - _Requirements: 3.1-3.5_

- [x] 3.3 補 loading / error 狀態
  - 首次載入顯示安全狀態
  - API 失敗不得顯示假入口
  - 提供重新整理操作
  - 完成：選單載入中只顯示 loading，載入失敗顯示錯誤與刷新按鈕，不渲染未經授權確認的 fallback 選單
  - _Requirements: 1.5_

- [x] 3.4 補 Frontend 驗證
  - `npm run typecheck`
  - `npm run build`
  - 視現有測試架構補 sidebar adapter 單元測試
  - 完成：專案目前沒有測試 runner；已執行 `npm run typecheck` 與 `npm run build` 通過
  - _Requirements: 5.1-5.6_

- [x] 3.5 修正選單管理節點更新 400 診斷與送出規則
  - `PUT /api/v1/admin/forms/:id` 回 400 時需能辨識是 request body、節點規則或 parent/type 組合錯誤
  - 前端送出前需確保 `node_type = root` 時 `parent_id = null`
  - 前端送出前需確保 `node_type = category/form` 時帶入有效 `parent_id`
  - 後端 400 response 需回傳可診斷訊息，避免只顯示 `invalid form node`
  - 完成：前端 `payloadFromValues` 已依節點類型正規化 `parent_id`，提交前阻擋不合法 parent/type 組合；後端已回傳 body decode 與 form node 規則的具體 400 訊息並補測試
  - 驗證：`go test ./internal/form`、`go test ./...`、`npm run typecheck`、`npm run build` 通過
  - _Requirements: 5.1-5.6_

- [x] 3.6 固定 Header brand 副標題
  - Header 左側 `Opscenter` 下方不得顯示 current project name
  - 切換專案時 Header brand 區塊不得跟著變動
  - 預設固定顯示系統名稱或移除副標題
  - 完成：`AppHeader` 不再接收或顯示 `currentProject`，副標題固定為 `Opscenter System`
  - 驗證：`npm run typecheck`、`npm run build` 通過
  - _Requirements: 5.1-5.6_

- [x] 3.7 修正 root-level form 節點停用 400
  - `dashboard`、`reports`、`jira` 這類 root-level form 節點允許 `parent_id = null`
  - 前端提交檢查不得把所有 `form` 節點都視為必須有 parent
  - 後端 `PUT /api/v1/admin/forms/:id` 更新 root-level form 時不得回 400
  - 補測試覆蓋 root-level form 更新與 category 缺 parent 的錯誤訊息
  - 完成：後端允許 root-level form 更新時計算自身 `full_path`，前端只要求 `category` 必須選擇父節點；已補 root-level form 停用與 category 缺 parent 測試
  - 驗證：`go test ./internal/form`、`go test ./...`、`npm run typecheck`、`npm run build` 通過
  - _Requirements: 4.1-4.5, 5.1-5.6_
