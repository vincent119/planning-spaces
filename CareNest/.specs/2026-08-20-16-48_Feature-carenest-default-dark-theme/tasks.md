# 任務文件：CareNest 預設深色主題

Status: Complete

## Execution Context

- 意圖：將 CareNest 由系統主題改為預設深色主題，建立一致、安心且可讀的深墨藍照護介面。
- 非目標：主題切換、資料流程、後端、帳號、同步、醫療判讀與第三方套件。
- 已定決策：深色為預設；青綠為主要操作色；暖珊瑚僅用於血壓第二資料線與必要錯誤提示。
- 邊界：全域主題入口、App 主題模式、量測圖視覺對齊與必要 Widget 測試。
- 關鍵檔案：`care_nest_theme.dart`、`app.dart`、`care_measurements_page.dart`、`widget_test.dart`。
- 完成條件：所有既有頁面套用一致深色表面層級，回歸測試與 Android 實機驗證通過。

## Protected Behavior

- 本機資料、被照顧者切換、照護記錄、量測新增修改刪除與照片處理不變。
- 不新增網路、帳號、同步、通知或資料庫 schema。
- 不以顏色顯示醫療健康判讀。

## 實作任務

- [x] T1：定義全域深色主題 Token
  - Status: Complete
  - Boundary:
    - Allowed Changes: `lib/shared/theme/care_nest_theme.dart`
    - Forbidden: 資料庫、Controller、第三方依賴
  - Depends: 設計文件色彩 Token
  - Context: 建立深墨藍背景、表面階層、主要／輔助文字、青綠操作與輸入／卡片／對話框的 Material Theme。
  - Verify: Theme 建構無 analyzer 問題；一般文字與操作元件使用語意色。

- [x] T2：設定預設深色模式並對齊量測圖
  - Status: Complete
  - Boundary:
    - Allowed Changes: `lib/app.dart`、`lib/features/measurements/care_measurements_page.dart`、必要 Widget 測試
    - Forbidden: 頁面流程、資料模型、資料寫入規則
  - Depends: T1
  - Context: 將 `ThemeMode.system` 改為 `ThemeMode.dark`；確認量測圖自訂 Token 與整體深色主題一致。
  - Verify: Widget 測試可讀取深色主題；量測圖仍含軸、資料線、定位線與文字提示。

- [x] T3：全頁回歸與 Android 驗證
  - Status: Complete
  - Boundary:
    - Allowed Changes: 任務文件與本功能範圍內修正
    - Forbidden: 範圍外功能、資料刪除、產品行為擴張
  - Depends: T1、T2
  - Context: 檢查首頁、設定、管理、量測、回顧、明細、對話框與選取狀態的深色可讀性。
  - Verify: `dart format`、`flutter analyze`、`flutter test`、`flutter build apk --profile`、`git diff --check`、Android 實機檢查。

## 品質檢查清單

- 主要、輔助文字與可操作元件在深色背景有足夠對比。
- 被選取、完成、錯誤與停用狀態不只用色彩區分。
- 卡片、輸入欄、對話框有可辨識的表面層級。
- 既有照護功能的文字與觸控目標維持可讀與可操作。

## Implementation Notes

- 2026-08-20：使用者確認參考色調為 CareNest 整體的預設深色主題。
- 2026-08-20：完成全域深墨藍主題、預設深色模式與主題 Widget 測試；完整測試通過，profile APK 已安裝 Android 實機，並確認首頁深色表面、文字與主要操作色顯示正確。
