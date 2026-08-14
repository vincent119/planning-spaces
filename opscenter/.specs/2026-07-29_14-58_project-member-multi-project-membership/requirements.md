# 專案成員跨專案關聯需求

## 文件定位

本規格接續 `.kiro/specs/2026-06-15_15-35_Ticket` 的專案成員功能，修正既有新增與移除流程將 `ops_user` 誤當成單一專案人員的行為。

本規格覆寫既有 Ticket spec 中下列行為，不回頭修改或重開已完成 task：

- 新增專案成員固定建立新 `ops_user`。
- `ops_user.user_name` 已存在時一律回傳名稱衝突。
- 移除任一專案成員時直接停用整筆 `ops_user`。

既有 Ticket、專案工作區、群組權限、Ticket 指派、mention 與協作者功能維持原模組，不在本規格重寫。

## 背景

`project_members` 以 `(project_id, user_id)` 作為複合主鍵，資料模型允許同一個 `ops_user` 同時加入多個專案。現行 `POST /api/v1/projects/{id}/members` 卻會先建立新的 `ops_user`，再建立 `project_members`，因此已在 V16 專案的 Terry 無法加入 IM 專案，系統會因啟用中的 `ops_user.user_name` 唯一索引而回傳 `409`。

現行移除流程會將 `ops_user.is_active` 設為 `false`。若以人工方式讓 Terry 同時加入 V16 與 IM，從其中一個專案移除 Terry 會使他在另一個專案也不可見，與跨專案成員關聯不相容。

## 目標

1. 同一位啟用中的運維人員可同時加入多個專案。
2. 新增專案成員時，既有啟用中的相同 `user_name` 必須重用原 `ops_user.id`。
3. 同一位人員不得重複加入同一專案。
4. 移除成員只解除指定專案的 `project_members` 關聯。
5. 人員仍有其他專案關聯時，不得停用 `ops_user`。
6. 新增與移除操作必須維持 transaction 一致性與既有權限檢查。

## 非目標

- 不調整 Admin `users`、SSO identity 或 `linked_user_id` 的資料模型。
- 不以 `project_members` 取代群組管理的一般功能授權。
- 不新增專案角色選擇介面，新增關聯仍使用內部預設角色 `engineer`。
- 不新增人員合併、同名停用人員清理或歷史資料修復工具。
- 不重新設計 Ticket 指派、mention、協作者與通知的業務規則。
- 不在本文件階段修改程式、資料庫或正式資料。

## 現有行為與新行為

| 情境 | 現有行為 | 新行為 |
|---|---|---|
| `Terry` 已在 V16，再加入 IM | 嘗試新增第二筆 `ops_user`，回 `409` 名稱衝突 | 重用 Terry 的 `ops_user.id`，新增 IM 的 `project_members` |
| `Terry` 已在 IM，再加入 IM | 回名稱衝突，無法區分原因 | 回 `409`，明確表示已是該專案成員 |
| 新名稱加入專案 | 建立 `ops_user` 與 `project_members` | 維持既有 transaction 建立流程 |
| 從 V16 移除同時屬於 IM 的 Terry | 停用 `ops_user`，IM 也看不到 Terry | 只移除 V16 關聯，Terry 維持啟用並保留 IM 關聯 |
| 移除 Terry 的最後一個專案關聯 | 停用 `ops_user`，保留原關聯 | 移除最後一筆關聯後停用 `ops_user` |

## 使用情境

### 情境 1：既有人員加入第二個專案

專案管理者在 IM 的專案成員頁輸入 `Terry`。系統找到啟用中的 Terry，重用其 `ops_user.id`，並建立 IM 的成員關聯。V16 與 IM 的成員列表都能顯示 Terry。

### 情境 2：避免同專案重複加入

專案管理者在 IM 再次新增 Terry。系統確認 `(IM, Terry)` 關聯已存在，回傳可辨識的 `409`，不得重複寫入資料。

### 情境 3：移除其中一個專案關聯

Terry 同時屬於 V16 與 IM。專案管理者從 V16 移除 Terry，系統只移除 V16 關聯；IM 成員列表、Ticket 指派與協作候選仍可使用 Terry。

### 情境 4：移除最後一個專案關聯

Terry 只剩 IM 關聯。專案管理者從 IM 移除 Terry，系統移除該關聯並將 `ops_user.is_active` 設為 `false`，避免無專案歸屬的人員繼續出現在啟用人員查詢中。

## 驗收情境

### 場景 A：重用啟用中的既有人員

測試：`TestRepositoryDBCreateProjectMemberReusesActiveOpsUser`

假設 Terry 為啟用中的 `ops_user`，已屬於 V16，但尚未加入 IM。

當管理者透過 IM 的新增專案成員 API 提交 `user_name = Terry`。

那麼系統不得新增第二筆 `ops_user`，必須以 Terry 既有的 `user_id` 建立 IM 的 `project_members`，並回傳成功結果。

### 場景 B：同專案重複關聯

測試：`TestRepositoryDBCreateProjectMemberRejectsExistingProjectRelation`

假設 Terry 已是 IM 的專案成員。

當管理者再次將 Terry 加入 IM。

那麼系統回傳 `409` 專案成員已存在，且 `ops_user` 與 `project_members` 均不得新增資料。

### 場景 C：建立全新人員

測試：`TestRepositoryDBCreateProjectMemberCreatesNewOpsUserWhenMissing`

假設系統不存在啟用中的 `NewMember`。

當管理者將 `NewMember` 加入 IM。

那麼系統在同一 transaction 建立 `ops_user` 與 IM 的 `project_members`，其中 `role = engineer`。

### 場景 D：名稱比對不分大小寫

測試：`TestRepositoryDBCreateProjectMemberReusesUsernameCaseInsensitively`

假設啟用中的人員名稱為 `Terry`。

當管理者輸入 `terry` 加入另一專案。

那麼系統必須重用 `Terry`，不得建立大小寫不同的第二筆啟用人員。

### 場景 E：移除其中一個專案關聯

測試：`TestRepositoryDBDeleteMemberKeepsSharedOpsUserActive`

假設 Terry 同時屬於 V16 與 IM。

當管理者從 V16 移除 Terry。

那麼只移除 V16 的 `project_members`，IM 關聯必須保留，且 `ops_user.is_active` 仍為 `true`。

### 場景 F：移除最後一個專案關聯

測試：`TestRepositoryDBDeleteMemberDeactivatesOpsUserWithoutMemberships`

假設 Terry 只屬於 IM。

當管理者從 IM 移除 Terry。

那麼系統移除 IM 的 `project_members`，確認沒有其他專案關聯後，將 `ops_user.is_active` 設為 `false`。

### 場景 G：關聯異動失敗時回滾

測試：`TestRepositoryDBProjectMemberMutationRollsBack`

假設新增或移除流程中的任一資料庫操作失敗。

當 transaction 結束。

那麼 `ops_user` 狀態與 `project_members` 關聯必須回到操作前狀態，不得留下部分完成資料。

## 驗收條件

- [x] 1.1 同一個 `ops_user.id` 可同時存在於不同 `project_id` 的 `project_members`。
- [x] 1.2 新增成員以不分大小寫的 `user_name` 尋找啟用中的 `ops_user`。
- [x] 1.3 找到既有人員時不得更新其 `description`、`linked_user_id` 或其他人員欄位，只新增專案關聯。
- [x] 1.4 找不到啟用中的人員時，才建立新的 `ops_user`。
- [x] 1.5 同專案相同 `user_id` 的重複關聯回 `409`，錯誤需與全新人員建立失敗區分。
- [x] 1.6 新增流程在同一 transaction 完成查找、必要的人員建立與專案關聯建立。
- [x] 2.1 移除成員只處理 path 指定的 `project_id` 與 `user_id` 關聯。
- [x] 2.2 尚有其他專案關聯時，`ops_user.is_active` 必須維持 `true`。
- [x] 2.3 沒有任何專案關聯時，`ops_user.is_active` 應設為 `false`。
- [x] 2.4 移除流程需在同一 transaction 完成關聯刪除、剩餘關聯計數與必要的人員停用。
- [x] 3.1 `GET /api/v1/projects/{id}/members` 只回傳指定專案的啟用成員，既有人員加入第二個專案後兩邊都可查到。
- [x] 3.2 Ticket 指派、mention、協作者與通知仍依各自專案的 `project_members` 範圍運作。
- [x] 3.3 專案成員管理仍使用 `tickets/members` 對應 action 權限，不放寬未授權操作。
- [x] 3.4 既有全新人員新增、成員列表與 API response contract 不得退化。

## 影響範圍

- 後端 Project Member service、repository、錯誤映射與測試。
- 前端新增成員錯誤文案，僅在後端新增專用錯誤 contract 時調整。
- Ticket 指派、mention、協作者與通知的專案成員回歸測試。
- 不預期修改資料表 schema；`project_members` 複合主鍵已支援跨專案關聯。

## 驗證需求

- 執行 `go test ./internal/project ./internal/ticket ./internal/notification`。
- 執行前端 `npm run typecheck`。
- 執行 `git diff --check`。
- 使用兩個專案與同一個 `ops_user` 進行人工驗收，確認新增、重複新增、單一關聯移除與最後關聯移除結果。
