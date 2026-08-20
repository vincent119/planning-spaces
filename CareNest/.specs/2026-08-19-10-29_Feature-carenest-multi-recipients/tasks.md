# 任務文件：CareNest 多位被照顧者

Status: PartiallyComplete

## Execution Context

- 意圖：讓同一裝置可安全管理多位被照顧者，並提供新增、切換、改名與完整刪除。
- 非目標：帳號、同步、後端、封存、還原、資料合併與照護設定編輯介面。
- 已定決策：完整刪除需輸入正確稱呼；目前對象須跨重啟保存；新增沿用既有三步驟設定。
- 邊界：Drift schema／DAO、Controller、首次設定重用、首頁管理入口、管理頁與相關測試。
- 關鍵檔案：`app_database.dart`、`care_nest_controller.dart`、`care_setup_page.dart`、`care_dashboard_page.dart`。
- 完成條件：多位 CRUD、資料隔離、完整刪除與 Android 實機流程均通過。

### Protected Behavior

- 不修改任何既有照護紀錄的 `recipientId`，不混用不同對象資料。
- 不新增帳號、網路、後端、同步或雲端資料傳輸。
- 不修改量測、餐食、高蛋白、飲水、回顧的既有記錄規則，只依目前對象重載。
- 刪除只在使用者完成稱呼確認後執行，不得以一般切換或返回動作觸發。

## 實作任務

- [x] T1：擴充資料庫為多位對象與目前選取狀態
  - Status: Complete
  - Boundary:
    - Allowed Changes: `app_database.dart`、產生檔、資料庫測試
    - Forbidden: 網路相依、既有紀錄資料遷移、刪除 UI
  - Depends: 已確認完整刪除規則
  - Context: schema version 4 新增 active recipient 表，解除單一對象檢查，提供 transaction 刪除與照片路徑清單
  - Verify: 多位建立、資料隔離、升級 migration、完整關聯列刪除測試

- [x] T2：擴充 Controller 管理對象生命週期
  - Status: Complete
  - Boundary:
    - Allowed Changes: `care_nest_controller.dart`、Controller 測試、必要照片清理呼叫
    - Forbidden: 修改既有照護紀錄輸入規則、跨對象資料複製
  - Depends: T1
  - Context: 初始化讀取 active recipient；切換、新增、改名、刪除後正確重載或進入首次設定
  - Verify: 切換、改名、刪除目前／非目前／最後一位對象的 Controller 測試

- [x] T3：建立被照顧者管理介面並重用新增設定
  - Status: Complete
  - Boundary:
    - Allowed Changes: recipients feature、dashboard 管理入口、onboarding 完成返回行為、Widget 測試
    - Forbidden: 一般首頁資訊架構重設、照護設定欄位編輯、外部導航套件
  - Depends: T2
  - Context: 清單明確標示目前對象；新增開啟既有設定；改名與永久刪除提供繁中錯誤與狀態回饋
  - Verify: Widget 測試覆蓋管理入口、名稱驗證、永久刪除稱呼確認與切換

- [ ] T4：整合驗證與 Android 實機確認
  - Status: InProgress
  - Boundary:
    - Allowed Changes: 驗證、範圍內修正與任務文件
    - Forbidden: 新功能與範圍外重構
  - Depends: T1、T2、T3
  - Context: 使用保留資料更新方式安裝，不建構或刪除任何使用者健康資料進行測試
  - Verify: `dart format`、`flutter analyze`、`flutter test`、`flutter build apk --debug`、`git diff --check`、Android 手動新增／切換／改名；刪除僅由使用者明確同意後以測試對象操作

## 品質檢查清單

- 多位對象的首頁、量測、回顧資料完全隔離。
- 永久刪除文字清楚列出資料與照片影響，不依顏色作唯一提示。
- 刪除後不保留 active recipient 指向不存在的 ID。
- 舊有單一對象資料可開啟。
- 不新增網路、帳號或後端依賴。

## Implementation Notes

- 2026-08-19：由草案升格；使用者確認完整刪除規則，包含所有本機資料與餐食照片。
- 2026-08-19：完成 schema version 4、目前對象保存、多位對象 CRUD、完整刪除與照片檔清理。全套 `flutter test`、`flutter analyze`、`flutter build apk --debug` 與 `git diff --check` 均通過，並已用保留資料方式更新安裝至 Android 裝置，且可正常啟動。尚待使用者以測試對象手動確認新增、切換、改名與完整刪除流程。
