# 任務文件：CareNest 照護細節

Status: Complete

## Execution Context

- 意圖: 在核心 MVP 後新增可選餐食照片與純量測紀錄
- 非目標: 醫療判讀、通知、備份、同步與 Backend Server
- 已定決策: 照片私有、量測不進入完成數
- 邊界: meals、files、measurements、history 與相關 migration／測試
- 關鍵檔案: `lib/features/meals/**`、`lib/core/files/**`、`lib/features/measurements/**`、`lib/features/history/**`
- 完成條件: requirements 三個情境與核心回歸通過

### Protected Behavior

- 不修改核心完成數算法
- 不新增健康閾值、建議或遠端傳輸
- 核心 MVP 未完成前不得開始

### 邊界

#### Allowed Changes

- 餐食照片、私有檔案、量測、history、Drift migration 與測試

#### Forbidden

- Backend Server、同步、通知、備份、影像辨識、醫療判讀

## 任務依賴

| 任務 | Depends | 狀態 | 備註 |
|------|---------|------|------|
| T1 餐食照片 | 核心 T6 | Complete | 私有檔案生命週期與 Android 實機操作已驗證 |
| T2 體重血壓 | 核心 T6 | Complete | 本機 CRUD、七天原始數值回顧與實機空資料介面已驗證 |

## 實作任務

- [x] T1：餐食照片與私有檔案
  - Status: Complete
  - Boundary:
    - Allowed Changes: meals、files、照片權限、migration、測試
    - Forbidden: 影像辨識、公開相簿、雲端上傳
  - Depends: 核心 MVP T6
  - Context: 照片可選，失敗不得破壞餐食紀錄
  - Verify:
    - requirements 照片生命週期情境
    - 核心餐食回歸

- [x] T2：體重、血壓與原始數值回顧
  - Status: Complete
  - Boundary:
    - Allowed Changes: measurements、history、資料表與測試
    - Forbidden: 健康區間、建議、完成數修改
  - Depends: 核心 MVP T6
  - Context: 只保存與呈現原始數值
  - Verify:
    - requirements 量測情境
    - CRUD、排序、空資料與核心完成數回歸

## 驗證任務

- [ ] 驗收情境覆蓋
  - Verify: 三個 requirements 情境皆有測試
- [ ] 回歸驗證
  - Verify: 核心 MVP 全部測試通過
- [ ] 品質檢查清單
  - 格式檢查通過
  - 測試通過
  - 文件一致性已確認
  - Protected Behavior 回歸驗證通過
  - `git diff --stat` 已檢查
  - `git diff --check` 已通過

## 實作中斷恢復

恢復時依序讀取 Execution Context、目前任務、Protected Behavior、Implementation Notes；不得掃描整個 `.specs`。

## Implementation Notes

- 2026-08-17：由過大 MVP 規格拆分，等待核心 MVP。
- 2026-08-17：依 SDD skill 樣本重整文件，尚未實作。
- 2026-08-18：開始 T1。採用 flutter.dev 官方 image_picker 1.2.3，只使用使用者主動選取相片的系統 picker；照片複製到 App 私有目錄。新增 MealRecord.photoPath／note schema version 2 migration、私有檔案服務、首頁縮圖與資料層測試。靜態分析、11 項測試與 Android Debug APK 通過。實體 Android 13 裝置已完成選圖、私有檔案與首頁縮圖驗證；等待刪除餐食後的檔案清理驗證。
- 2026-08-18：已授權刪除早餐測試照片。確認私有照片檔清除後首頁回到 1 / 5，沒有遺留孤兒檔案。但診斷發現：照片餐食在 App 重啟後未出現在持久化 `meal_records`，而照片檔曾存在，形成檔案與資料庫不同步風險。T1 維持 InProgress，必須先修正並新增實機重啟回歸測試，不能開始 T2。
- 2026-08-18：新增檔案 SQLite 重啟持久化測試，確認 MealRecord 與 photoPath 關閉／重新開啟資料庫後仍可讀取；12 項自動測試通過。實機不一致暫判定與 image picker 返回或 App／Debug 安裝生命週期有關，需在乾淨實機流程重現後再修正。
- 2026-08-18：修正 Android 系統選圖期間可能重建 MainActivity 的流程。現在會先保存餐食及待回填的餐食識別，再開啟選圖；重啟時以 `ImagePicker.retrieveLostData()` 只回填到該筆餐食。若使用者取消或無法取回照片，會清除待回填識別並保留餐食紀錄。新增待回填識別與 App 重啟回填測試；`flutter analyze`、15 項測試與 Android Debug APK 通過。實體手機已安裝新版，但因簽章不相容而解除安裝舊版，裝置上的舊測試資料可能已清除；尚未完成實機中斷選圖回歸，T1 維持 InProgress。
- 2026-08-18：完成實體 Android 13 中斷選圖回歸。先保存早餐與待回填識別，系統相簿開啟時終止 CareNest 程序；選取並確認照片後，重建 App 成功將照片回填到原早餐。確認私有檔案存在且待回填識別已清除。測試後刪除測試早餐並以原本的 `vbhj`、75% 重建無照片早餐；確認本次測試照片已清除。T1 的實機中斷風險已驗證，但照片替換與獨立移除介面仍未提供，T1 維持 InProgress。
- 2026-08-18：補齊照片替換與獨立移除介面。今日明細的每筆餐食現在會顯示縮圖與「附加照片」或「替換照片」；已有照片時可經確認對話框選擇「移除照片」，只清除本機照片並保留餐食。新增替換時舊檔清理、移除時保留餐食與清除新檔的回歸測試。`flutter analyze`、16 項測試、Android Debug APK 與 `git diff --check` 通過；新版 APK 尚未安裝到實體裝置，以避免重複安裝可能清除現有測試資料。T1 維持 InProgress，等待新介面的實機確認。
- 2026-08-18：以 `adb install -r` 成功更新實體 Android 13，保留既有本機資料。今天明細確認有照片的餐食顯示「替換照片」「移除照片」，無照片餐食顯示「附加照片」。開啟「移除照片」確認對話框後取消，確認明確說明只移除本機照片且取消後照片操作仍可用。T1 完成。
- 2026-08-18：開始 T2。Drift schema version 3 新增 WeightRecord 與 BloodPressureRecord；只保存量測日、量測時間及原始體重／收縮壓／舒張壓。新增量測頁，支援新增與最近七天原始數值瀏覽，且不進入首頁完成數。資料層 CRUD 與 17 項完整測試、靜態分析、Android Debug APK 已通過；量測頁的修改與刪除操作尚待實作。
- 2026-08-18：完成 T2。量測頁支援點選原始數值修改，以及經確認後刪除；刪除只影響該筆量測。以 `adb install -r` 保留本機資料更新實體 Android 13，確認量測入口、無醫療判讀說明、新增體重、新增血壓與七天空資料回顧正常。T2 完成。

## 驗證結果摘要

- 新行為驗證: schema migration、照片路徑讀寫、SQLite 重啟持久化、遺失選圖結果回填、替換舊檔清理與獨立移除、`flutter analyze`、16 項測試與 Android Debug APK 通過；實體 Android 13 中斷選圖後回填、檔案清理與新照片操作介面均通過
- 回歸驗證: 核心自動測試、Android APK、實機早餐還原與本機資料保留更新均通過
- 文件一致性: 已依樣本重整
- 剩餘風險: 照護細節功能無已知待處理風險

## 後續改善

- [ ] 核心 MVP 使用者測試後重新確認本 spec 範圍
