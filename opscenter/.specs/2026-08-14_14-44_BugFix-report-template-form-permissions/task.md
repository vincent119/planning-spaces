# 報表範本表單權限修正工作項目

## 狀態

Status: Complete

## Execution Context

- 意圖：將報表範本從通用 `reports` 權限拆出，加入可管理的 `reports/templates` 表單節點，套用已確認的五群組矩陣。
- 非目標：不改資料集、範本資料模型、其他 Report API、專案可見性或既有群組。
- 已定決策：`ops_admin` 與 `ops_op_admin` 具完整 CRUD；`engineers`、`ops_member`、`ops_op_member` 唯讀；執行與匯出使用 read。
- 關鍵檔案：`opscenter-server/sql/`、`opscenter-server/internal/report/delivery.go`、`opscenter-server/internal/report/*_test.go`、`opscenter-frontend/src/features/report/`。
- 完成條件：需求文件所有驗收條件均有 migration、測試與前後端驗證證據，且沒有改變非範本 Report API 的權限行為。

## Protected Behavior

- `reports` 節點仍保護資料集、固定月報、通用預覽與通用匯出 API。
- 範本 API 的 URL、HTTP 方法、request／response 契約維持不變。
- 所有範本操作仍須通過既有專案可見性與資料範圍驗證。
- 既有五個群組名稱與代碼不變，未列入的群組不會被 migration 變更。

## 1. 固化表單節點與權限來源

- [x] 1.1 建立 `reports/templates` 節點與初始群組矩陣 migration
  - Boundary:
    - Allowed Changes: 新增單一版本化 SQL migration，僅處理 `reports/templates`、五個既有群組的 `group_form_permissions` 與對應 Casbin policy；同步既有資料庫初始化文件的 migration 清單。
    - Forbidden: 修改既有 `reports` 權限、建立新群組、刪除表單或覆寫其他節點權限。
  - Depends: `form_nodes`、`groups`、`group_form_permissions`、`casbin_rule` 既有 schema 與 seed 已存在。
  - Context: 權限矩陣以 requirements「已確認權限矩陣」為準。
  - Verify: migration 可重跑；查詢節點、五組來源權限與有效 policy 均符合矩陣。
  - 完成：新增 `opscenter-server/sql/0055_report_template_form_permissions.sql`，包含 parent／seed user／五群組前置檢查、節點 upsert、已確認矩陣 upsert 及指定 policy 的刪除後重建；初始化文件已加入執行順序。

- [x] 1.2 建立或調整 migration 驗證
  - Boundary:
    - Allowed Changes: 現有 migration 測試機制、必要的資料庫驗證 script 或測試 fixture。
    - Forbidden: 導入新的 migration framework 或修改 production seed 流程。
  - Depends: Task 1.1。
  - Context: 必須涵蓋 parent／群組缺漏時的可診斷行為。
  - Verify: 在既有測試或受控測試資料庫執行 migration 兩次，確認結果相同且無殘留 write policy。
  - 完成：使用者確認已完成 migration 套用與重跑驗證；此前唯讀查詢亦已確認五組來源權限及 11 筆有效 Casbin policy 符合矩陣。

## 2. 後端範本 API 授權

- [x] 2.1 將範本路由改用 `reports/templates`
  - Boundary:
    - Allowed Changes: `internal/report/delivery.go` 及直接相關 route 註冊程式。
    - Forbidden: 改動非範本 Report API 的 `reports` 授權、移除專案可見性檢查、變更 API URL 或 payload。
  - Depends: Task 1.1。
  - Context: list/get/execute/execute-layout/block export 對應 read；create/update/delete 對應同名操作。
  - Verify: 路由測試斷言每種 API 使用正確 full path 與 action。
  - 完成：範本 list、create、get、update、delete、execute、execute-layout、block export 已改用 `reports/templates`；其他 Report API 維持 `reports`。

- [x] 2.2 補齊後端授權與相容性測試
  - Boundary:
    - Allowed Changes: `internal/report` 的 handler、service、middleware 測試與必要 fixture。
    - Forbidden: 以硬編群組名稱繞過 authorization service，或削弱 Project Role／可見性驗證。
  - Depends: Task 2.1。
  - Context: 覆蓋完整管理者、三種唯讀群組、僅有 `reports` read、完全無權限四類情境。
  - Verify: `go test ./internal/report ./internal/auth` 通過；各 HTTP 狀態碼與既有錯誤語意符合需求。
  - 完成：新增 secured route 測試，覆蓋通用 `reports` read 不可繞過範本 read、範本 CRUD／執行／版面／區塊匯出的 action 對應，以及通用 preview 仍使用 `reports`；測試通過。

## 3. 前端權限驅動操作

- [x] 3.1 將範本列表與設計器切換至新表單節點
  - Boundary:
    - Allowed Changes: `src/features/report/` 中範本列表、設計器與直接相關 hooks／測試。
    - Forbidden: 改變報表資料計算、直接以群組代碼判斷權限、修改非範本 Report 畫面。
  - Depends: Task 2.1、後端 permission metadata 可取得。
  - Context: UI 僅做按鈕與入口控制；API 為最終邊界。
  - Verify: 管理群組可見新增／編輯／刪除；唯讀群組僅可檢視與執行；無 read 時不載入範本資料。
  - 完成：`TemplateList` 的讀取、建立、編輯、刪除操作改查 `reports/templates`；無 read 時不送出範本列表請求並顯示既有權限不足訊息。設計器的儲存按鈕與提交 handler 需具 create 權限。

- [x] 3.2 執行前端既有品質檢查
  - Boundary:
    - Allowed Changes: 僅必要測試與測試設定修正。
    - Forbidden: 以跳過型別檢查或關閉測試方式取得通過。
  - Depends: Task 3.1。
  - Context: 使用專案既有 test、lint、typecheck 命令。
  - Verify: 記錄實際執行命令與結果；無既有元件測試時，記錄替代驗證證據。
  - 完成：已執行 `npm run typecheck`、`npm test -- --run`、`npm run build`，均通過；build 僅出現既有 bundle 大小警告。

## 4. 整合驗證與交付

- [x] 4.1 驗證權限矩陣與 API 對應
  - Boundary:
    - Allowed Changes: 測試、受控開發環境設定與本 spec 的 Implementation Notes。
    - Forbidden: 寫入 production 權限資料、手動直接修改 Casbin policy 作為替代 migration。
  - Depends: Tasks 1 至 3。
  - Context: 應以 migration 後的表單權限來源驗證 effective policy。
  - Verify: 五群組與無權限使用者的 list/create/update/delete/execute/export 結果均有證據。
  - 完成：使用者確認已完成部署後的實際驗證。

- [x] 4.2 執行回歸與 diff 檢查
  - Boundary:
    - Allowed Changes: 不修改產品程式碼；僅記錄驗證結果。
    - Forbidden: 忽略失敗測試或將未驗證項目標記完成。
  - Depends: Task 4.1。
  - Context: 特別確認固定月報、資料集 API 與一般 report preview/export 仍使用 `reports`。
  - Verify: 後端與前端相關檢查通過，並執行 `git diff --check`。
  - 完成：本機 `go test ./...`、前端型別檢查、既有前端測試、前端 build 與 `git diff --check` 已通過；使用者確認部署驗證完成。

## 品質檢查清單

- [x] SQL migration 可重跑且只影響指定節點、群組與 policy。
- [x] 不存在前端群組代碼硬編判斷。
- [x] 每條範本 API 均有後端權限測試。
- [x] 前端管理操作與後端操作權限一致。
- [x] `git diff --check` 通過。

## Implementation Notes

- 2026-08-14：依使用者確認矩陣建立本規格；尚未開始程式碼或資料庫 migration 實作。
- 2026-08-14：使用者已排程執行；開始處理 migration、後端授權與前端權限判斷。
- 2026-08-14：已完成 migration 檔、初始化文件、後端範本路由授權、後端 route 測試與前端管理操作權限切換。`go test ./...`、前端型別檢查、既有前端測試、前端 build 與 `git diff --check` 皆通過。
- 2026-08-14：Task 1.2 與整合 Task 4 尚待使用者於受控資料庫套用 `0055` 後，驗證五組 `group_form_permissions`、`casbin_rule`、migration 重跑，以及實際 API／UI 矩陣；依資料庫分工未自行執行 SQL。
- 2026-08-14：使用者已套用 `0055`。唯讀查詢確認 `reports/templates` 有五筆來源權限：`ops_admin`、`ops_op_admin` 完整 CRUD；`engineers`、`ops_member`、`ops_op_member` 僅 read。Casbin 有 11 筆有效 policy，與矩陣一致。尚未重跑 migration，也尚未在已部署 API／UI 以實際群組成員驗證。
- 2026-08-14：使用者確認已完成 migration 重跑與部署後實際驗證；所有工作項目具備本機測試、唯讀資料庫查核或使用者確認證據，task 結案。
