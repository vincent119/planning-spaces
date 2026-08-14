# Group Based Operation Permission Tasks

## 1. 文件與授權模型確認

- [x] 1.1 確認全系統操作授權模型
  - 操作權限以群組管理表單節點為準
  - `read` 控制頁面、表單與選單可見性
  - `create` / `update` / `delete` 控制對應操作
  - `project_members` 不參與一般功能操作授權
  - `project_id` 僅作為資料歸屬與存在性檢查
  - _Requirements: 1.1-5.3_

## 2. 後端授權層調整

- [x] 2.1 盤點專案型 API 授權 middleware
  - 找出一般功能操作中使用 `RequireProjectFormPermission` 的 route
  - 標記哪些只是需要 `project_id` 資料歸屬
  - 標記哪些是真正專案成員管理功能
  - _Requirements: 1.1-3.5_

- [x] 2.2 拆分 project role 與 project 存在性檢查
  - 保留 project 存在性與啟用狀態檢查
  - 一般功能操作移除 `project_members` 授權檢查
  - 專案成員管理功能若需 project role，需明確列為例外
  - _Requirements: 3.1-3.5_

- [x] 2.3 補後端授權測試
  - 有 action 權限但無 `project_members` 可操作
  - 無 action 權限回 `403`
  - 專案不存在或停用回資料錯誤
  - `/api/v1/admin/*` 仍維持 `admin` 專用
  - _Requirements: 1.1-5.3_

## 3. 前端權限可見性調整

- [x] 3.1 確認有效權限 API 契約
  - 確認後端回傳 `full_path` 與 `can_read` / `can_create` / `can_update` / `can_delete`
  - 前端不得以 `global_role` 硬編選單與按鈕
  - _Requirements: 4.1-4.4_

- [x] 3.2 調整側邊選單可見性
  - leaf 節點沒有 `read` 不顯示
  - parent 節點沒有任何可讀子節點不顯示
  - parent 節點只有子節點可讀時只作為展開容器
  - _Requirements: 2.1-2.4_

- [x] 3.3 調整 route guard
  - 無 `read` 權限時不可進入頁面
  - 直接輸入 URL 需顯示無權限頁或導回第一個可用頁面
  - _Requirements: 2.2, 4.2-4.4_

- [x] 3.4 調整操作按鈕權限
  - 新增與匯入看 `create`
  - 編輯、狀態變更、確認、重新整理看 `update`
  - 刪除看 `delete`
  - _Requirements: 1.2-1.5, 2.5, 4.2-4.4_

## 4. 代表性功能驗證

- [x] 4.1 驗證 Ticket 操作
  - 建立、修改、狀態變更、指派、留言與附件依群組權限控制
  - 無 `project_members` 不應造成拒絕
  - _Requirements: 1.1-5.3_

- [x] 4.2 驗證 Jira 匯入與報表
  - Jira 匯入依 `jira/import create`
  - Jira 報表依對應 `read`
  - 無 `project_members` 不應造成拒絕
  - _Requirements: 1.1-5.3_

- [x] 4.3 驗證排班與系統功能入口
  - 無 `read` 不顯示選單或頁面入口
  - 有 action 才顯示或啟用對應操作
  - _Requirements: 2.1-2.5, 4.1-4.4_

## 5. 回歸驗證

- [x] 5.1 執行後端授權相關測試
  - `go test ./internal/auth ./internal/jira ./internal/ticket`
  - 依盤點結果補跑 report / schedule / admin 相關 package
  - _Requirements: 1.1-5.3_

- [x] 5.2 執行前端型別檢查與權限 UI 測試
  - `npm run typecheck`
  - 權限選單、route guard、action button 覆蓋測試
  - _Requirements: 2.1-4.4_

- [x] 5.3 驗證既有管理端權限不被放寬
  - `/api/v1/admin/*` 仍只允許 `admin`
  - 群組未授權的操作仍回 `403`
  - _Requirements: 1.6-1.7, 5.1-5.3_

- [x] 5.4 補人工驗收 SQL 與操作步驟
  - 查詢使用者群組權限
  - 查詢使用者不在 `project_members` 時仍可依群組權限操作
  - 驗證無 `read` 權限時前端不可見
  - _Requirements: 1.1-5.3_

- [x] 5.5 修正 `system/logs` 讀取權限入口
  - 具備 `system/logs read` 的非 admin 使用者可看到日誌查詢選單與頁面
  - 新增 `/api/v1/system/logs/*` 讀取 API，使用 `system/logs read` 授權
  - 保留 `/api/v1/admin/logs/*` 與其他 `/api/v1/admin/*` admin-only
  - _Requirements: 1.8, 4.5_

- [x] 5.6 修正 `/api/v1/projects` 專案清單可見性
  - 具備專案工作區相關 `read` 權限的使用者，即使不在 `project_members`，仍可取得 active project 清單
  - `project_members` 不再作為一般專案入口可見性的必要條件
  - `projects.visibility = private` 不在本次設計中隱性等同於 `project_members` 限制
  - 專案成員管理 API 本身仍維持既有管理規則
  - 補測 SSO `ops_op_member` 無 `project_members` 仍可取得專案清單
  - _Requirements: 3.1, 3.4, 3.6-3.7, 4.1-4.4_

- [x] 5.7 修正專案工作區支援資料讀取 API 的 `project_members` 殘留檢查
  - `GET /api/v1/projects/{id}` 不得因使用者未存在於 `project_members` 而回 `403`
  - `GET /api/v1/projects/{id}/sub-projects` 不得因使用者未存在於 `project_members` 而回 `403`
  - `GET /api/v1/projects/{id}/ticket-resources` 不得因使用者未存在於 `project_members` 而回 `403`
  - `GET /api/v1/projects/{id}/members` 作為 Ticket 畫面支援資料讀取時，不得因使用者未存在於 `project_members` 而回 `403`
  - 建立、修改、刪除專案設定與專案成員管理 API 本身仍維持既有管理規則
  - 補測 SSO `ops_op_member` 無 `project_members` 仍可載入專案工作區支援資料
  - _Requirements: 3.1, 3.4, 4.1-4.4, 5.3_
