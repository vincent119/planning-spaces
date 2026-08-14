# 專案成員跨專案關聯 Tasks

Status: InProgress

## 文件定位

本文件追蹤 `requirements.md` 與 `design.md` 的實作及驗證工作。既有 `.kiro/specs/2026-06-15_15-35_Ticket` 專案成員 task 已完成，本次不得將新需求追加到舊 task 並維持完成狀態。

## Execution Context

### 意圖

修正 Project Member 新增與移除流程，使同一個啟用中的 `ops_user` 能同時關聯多個專案，且移除單一專案關聯不影響其他專案。

### 非目標

- 不修改 Admin `users`、SSO、`linked_user_id` 或群組權限模型。
- 不新增專案角色 UI。
- 不建立成員歷史 schema 或資料清理工具。
- 不重寫 Ticket、mention、協作者與通知模組。

### 已定決策

- 新增時以不分大小寫的 `user_name` 重用啟用中的 `ops_user`。
- 找不到啟用人員時才建立新的 `ops_user`。
- 同專案重複關聯回 `409`，不得靜默更新 role。
- 移除時刪除指定 `(project_id, user_id)` 關聯。
- 仍有其他專案關聯時維持 `ops_user.is_active = true`。
- 移除最後一個關聯後設定 `ops_user.is_active = false`。
- API path、request 與成功 response 維持既有契約。

### 邊界

核心修改限於 Project Member domain、repository、錯誤映射、必要前端錯誤文案與相關測試。若實作需要 schema migration、API request 欄位變更或超出列出的模組，必須先更新本 task 邊界並取得使用者安排。

### 關鍵檔案

- `opscenter-server/internal/project/domain.go`
- `opscenter-server/internal/project/repository.go`
- `opscenter-server/internal/project/middleware.go`
- `opscenter-server/internal/project/delivery.go`
- `opscenter-server/Docs/openapi.json`
- `opscenter-server/internal/project/repository_test.go`
- `opscenter-server/internal/project/service_test.go`
- `opscenter-server/internal/project/delivery_test.go`
- `opscenter-frontend/src/features/ticket/pages/ProjectMembersPage.tsx`
- `opscenter-frontend/src/locales/*/ticket.json`

### 完成條件

- `requirements.md` 場景 A 至 G 全部有自動測試或明確人工驗證。
- Terry 可使用同一個 `ops_user.id` 同時加入 V16 與 IM。
- 移除 V16 關聯不影響 IM 關聯與 Terry 的啟用狀態。
- 移除最後一個關聯後 Terry 轉為停用。
- 指定測試、型別檢查與 `git diff --check` 全部通過。

## Protected Behavior

- 專案成員管理仍需通過 `tickets/members` 對應 action 權限。
- `project_members.user_id` 持續指向 `ops_user.id`，不得改回 Admin `users.id`。
- 專案成員列表只顯示指定專案的啟用人員。
- Ticket 指派、mention、協作者與通知仍依指定專案關聯判斷。
- 前端不得新增 `global_role`、MFA、phone、狀態或角色選擇欄位。
- 新人員建立與專案關聯必須維持 transaction 原子性。
- 不得修改使用者已存在的無關工作區變更。

## 1. 後端 domain 與新增流程

- [x] 1.1 定義跨專案新增的 domain 錯誤契約
  - Boundary:
    - Allowed Changes: `internal/project/domain.go`、`internal/project/middleware.go`、`internal/project/delivery.go`、`Docs/openapi.json` 與對應測試。
    - Forbidden: API path、request 欄位、權限模型與非 Project Member 錯誤契約。
  - Depends: 無。
  - Context: 區分「同專案成員已存在」與「無法解決的人員名稱衝突」，兩者皆回 `409`，但需提供可診斷語意。
  - Verify: 執行 `go test ./internal/project -run 'TestHandlerCreateProjectMember|Test.*Member.*Conflict'`。

- [x] 1.2 新增時重用啟用中的 `ops_user`
  - Boundary:
    - Allowed Changes: `internal/project/repository.go`、`repository_test.go`，必要時調整同 package 的測試 helper。
    - Forbidden: SQL schema、Admin `users`、SSO 與 `linked_user_id` 寫入。
  - Depends: 1.1。
  - Context: transaction 內先以不分大小寫名稱查找 `is_active = true` 人員；找到只建立 `project_members`，找不到才建立新人員。
  - Verify: 執行 `go test ./internal/project -run 'TestRepositoryDBCreateProjectMember(ReusesActiveOpsUser|CreatesNewOpsUserWhenMissing|ReusesUsernameCaseInsensitively)'`。

- [x] 1.3 防止同專案重複關聯與併發殘缺資料
  - Boundary:
    - Allowed Changes: Project Member repository transaction 與 repository tests。
    - Forbidden: 使用 upsert 靜默覆寫 role、放寬複合主鍵或移除名稱唯一索引。
  - Depends: 1.2。
  - Context: 同 `(project_id, user_id)` 必須回專用 `409`；任一步驟失敗均回滾。
  - Verify: 執行 `go test ./internal/project -run 'TestRepositoryDBCreateProjectMemberRejectsExistingProjectRelation|TestRepositoryDBProjectMemberMutationRollsBack'`。

## 2. 後端移除流程

- [x] 2.1 移除指定專案關聯
  - Boundary:
    - Allowed Changes: `internal/project/repository.go`、`repository_test.go` 與必要測試 helper。
    - Forbidden: 刪除 `ops_user`、移除其他專案關聯或修改其他人員欄位。
  - Depends: 1.1。
  - Context: 在 transaction 中定位 path 指定的專案與人員，只刪除該筆 `project_members`。
  - Verify: 執行 `go test ./internal/project -run 'TestRepositoryDBDeleteMember(KeepsSharedOpsUserActive|NotFound)'`。

- [x] 2.2 最後關聯移除後停用人員
  - Boundary:
    - Allowed Changes: Project Member repository transaction、鎖定策略與 repository tests。
    - Forbidden: 尚有其他專案關聯時停用 `ops_user`。
  - Depends: 2.1。
  - Context: 刪除指定關聯後計算相同 `user_id` 的剩餘關聯；只有零筆時設定 `is_active = false`，並處理新增與移除併發風險。
  - Verify: 執行 `go test ./internal/project -run 'TestRepositoryDBDeleteMember(DeactivatesOpsUserWithoutMemberships|KeepsSharedOpsUserActive)'`。

- [x] 2.3 補齊 service 與 delivery 回歸測試
  - Boundary:
    - Allowed Changes: `internal/project/service_test.go`、`delivery_test.go`，必要時調整 Project Member fake repository。
    - Forbidden: 放寬 `tickets/members` 權限或改變非本需求 API。
  - Depends: 1.1、1.3、2.2。
  - Context: 驗證成功、`404`、`409` 與權限行為。
  - Verify: 執行 `go test ./internal/project`。

## 3. 前端錯誤呈現

- [x] 3.1 對齊同專案成員已存在的錯誤文案
  - Boundary:
    - Allowed Changes: `ProjectMembersPage.tsx`、Ticket locale 中與專案成員錯誤直接相關的鍵。
    - Forbidden: 新增角色 UI、候選人 UI、Admin 使用者欄位或改造表單流程。
  - Depends: 1.1。
  - Context: 若後端新增專用錯誤字串，前端需顯示「此人員已是目前專案成員」；既有人員成功加入第二專案不得顯示名稱衝突。
  - Verify: 執行 locale 鍵集合既有驗證與 `npm run typecheck`。

## 4. 跨模組回歸驗證

- [x] 4.1 驗證 Ticket 專案成員範圍
  - Boundary:
    - Allowed Changes: `internal/ticket/*_test.go`，只有測試發現本需求直接造成失敗時才可修改對應 Project Member 查詢。
    - Forbidden: 重新設計 Ticket 指派、mention、協作者或一般授權模型。
  - Depends: 1.3、2.2。
  - Context: 同一人加入 V16 與 IM 後，兩個專案都可作為該專案的指派與協作者候選；移除 V16 後只從 V16 排除。
  - Verify: 執行 `go test ./internal/ticket`。

- [x] 4.2 驗證通知專案成員範圍
  - Boundary:
    - Allowed Changes: `internal/notification/*_test.go`，只有測試發現本需求直接造成失敗時才可修改成員候選查詢。
    - Forbidden: 通知事件、Webhook 與投遞策略重構。
  - Depends: 2.2。
  - Context: 移除 V16 關聯不得讓 Terry 在 IM 被判斷為停用專案成員。
  - Verify: 執行 `go test ./internal/notification`。

- [x] 4.3 執行完整相關驗證
  - Boundary:
    - Allowed Changes: 僅修正本需求直接造成的測試或型別錯誤；超出邊界需先更新 task。
    - Forbidden: 為通過測試刪除斷言、停用測試或放寬資料隔離。
  - Depends: 2.3、3.1、4.1、4.2。
  - Context: 完成自動測試、型別檢查與差異品質檢查。
  - Verify:
    - `go test ./internal/project ./internal/ticket ./internal/notification`
    - `npm run typecheck`
    - `git diff --check`
    - `git diff --stat`

## 5. 人工驗收

- [ ] 5.1 驗證同一人跨專案新增與移除
  - Boundary:
    - Allowed Changes: 測試環境的 V16、IM 與 Terry 關聯資料。
    - Forbidden: 未備份的正式環境資料修改、直接更改 `ops_user.id` 或繞過應用流程修補結果。
  - Depends: 4.3。
  - Context: 依 API 與 UI 驗證 Terry 跨專案共用相同 `user_id`，並確認移除其中一個關聯不影響另一專案。
  - Verify:
    - 新增前記錄 Terry 的 `ops_user.id`。
    - 從 IM 新增 Terry 後，V16 與 IM 的 `project_members.user_id` 相同。
    - 從 V16 移除 Terry 後，IM 關聯仍存在且 `ops_user.is_active = true`。
    - 從最後一個專案移除 Terry 後，關聯為零且 `ops_user.is_active = false`。

## 品質檢查清單

- [x] 所有新行為都有對應測試 selector。
- [x] 新增既有人員不更新 `description` 或 `linked_user_id`。
- [x] 同專案重複新增回可診斷的 `409`。
- [x] 新增與移除 transaction 失敗均完整回滾。
- [x] 尚有其他專案關聯時不會停用 `ops_user`。
- [x] 最後關聯移除後才停用 `ops_user`。
- [x] Project Member 權限沒有被放寬。
- [x] Ticket 與通知的專案隔離沒有退化。
- [x] 前端 locale 鍵集合一致且型別檢查通過。
- [x] `git diff --check` 通過且差異未超出 task Boundary。

## Implementation Notes

- 2026-07-29：建立本規格，確認既有 `project_members` 複合主鍵已支援同一人跨專案，不預期新增 migration。
- 2026-07-29：確認既有完成 task 將新增定義為固定建立 `ops_user`，並將移除定義為直接停用人員；本規格以新 `Planned` task 覆寫該行為，不修改舊 task 完成狀態。
- 2026-07-29：使用者已安排執行，task 狀態切換為 `InProgress`。
- 2026-07-29：新增流程改為在 transaction 中鎖定並重用啟用中的同名 `ops_user`；只有查無人員時才建立新資料。
- 2026-07-29：移除流程改為刪除指定專案關聯；僅在剩餘關聯為零時停用 `ops_user`。
- 2026-07-29：新增同專案重複成員專用 `409` 錯誤、三語前端文案及 OpenAPI 契約描述。
- 2026-07-29：`go test ./...`、前端 `npm run typecheck`、locale 鍵集合與 `git diff --check` 均通過。
- 待辦：尚未對實際測試環境的 V16、IM 與 Terry 執行人工資料驗收，因此 task 維持 `InProgress`。
