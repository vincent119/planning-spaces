# Op Admin Cross Project Read Access Requirements

## Introduction

`op_admin` 是運維管理角色，需能跨專案查看營運資料，例如 Jira 報表、Report 報表中心與報表範本列表。現行後端部分專案 API 同時檢查表單權限與 `project_members` 專案角色，導致 `op_admin` 即使具備運維群組權限，仍因不是專案成員而收到 `403`。

本規格只處理跨專案讀取營運資料，不處理全域使用者管理、角色管理、安全設定、SSO / MFA 設定，也不放寬跨專案寫入操作。

## Requirements

### 需求 1：`op_admin` 可跨專案讀取營運資料

**使用者故事**：身為運維管理員，我希望不需要逐一加入每個專案成員，也能查看各專案的 Jira 報表與 Report 報表資料，方便跨專案營運管理。

#### 驗收條件

- [ ] 1.1 `global_role = op_admin` 呼叫專案 API 的 `read` 類操作時，可在通過表單權限後略過 `project_members` 專案角色檢查。
- [ ] 1.2 `global_role = admin` 維持既有全域放行行為。
- [ ] 1.3 `global_role = member` 維持既有規則，仍需同時通過表單權限與專案角色檢查。
- [ ] 1.4 `op_admin` 的 `create`、`update`、`delete` 類專案操作不得因本需求被跨專案放行，除非該 API 另有明確規格。
- [ ] 1.5 受影響 API 至少包含 `GET /api/v1/projects/:id/jira/report` 與 `GET /api/v1/projects/:id/report-templates`。
- [ ] 1.6 `/api/v1/admin/*` 仍維持 `admin` 專用，不因本需求放寬。

### 需求 2：權限失敗可診斷

**使用者故事**：身為系統維運者，我希望權限拒絕時可以快速分辨是表單權限不足、專案角色不足，或 token 角色不正確。

#### 驗收條件

- [ ] 2.1 後端授權測試需覆蓋 `op_admin` 跨專案 `read` 放行。
- [ ] 2.2 後端授權測試需覆蓋 `op_admin` 跨專案 `update` 仍被專案角色拒絕。
- [ ] 2.3 後端授權測試需覆蓋 `member` 即使有表單權限，仍需符合專案角色。

### 需求 3：排班人員選項不得依賴管理端使用者 API

**使用者故事**：身為運維管理員，我希望在人員班別頁可以選擇啟用人員建立班別歸屬，但不能因此取得使用者管理能力。

#### 驗收條件

- [ ] 3.1 後端需提供非 `/admin/*` 的使用者選項 API，供排班人員選擇使用。
- [ ] 3.2 使用者選項 API 允許 `admin` 與 `op_admin` 讀取；`member` 不得讀取完整人員選項。
- [ ] 3.3 使用者選項 API 僅回傳選人所需欄位：`id`、`username`、`full_name`、`email`、`is_active`。
- [ ] 3.4 使用者選項 API 不得回傳 `global_role`、MFA 狀態、備註、建立時間、更新時間等管理欄位。
- [ ] 3.5 使用者選項 API 需排除 `global_role = admin` 的使用者，避免系統管理員被納入排班人員選項。
- [ ] 3.6 人員班別頁不得再呼叫 `GET /api/v1/admin/users` 作為人員下拉選項來源。
- [ ] 3.7 `/api/v1/admin/users` 維持 `admin` 專用，不因本需求放寬。
