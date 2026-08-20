# 任務文件：CareNest 三種顯示模式

Status: Complete

## Execution Context

- 意圖：提供淺色、深色與跟隨系統三種可保存的 App 顯示模式，以及三段式文字大小；預設跟隨系統與標準文字。
- 非目標：雲端、帳號、主題同步、依被照顧者設定、自訂色盤與醫療判讀。
- 已定決策：淺色採冷灰白與淡藍；深色採深墨藍與青綠；文字大小為 100%、115%、130%；設定只保存於本機。
- 邊界：本機 App 偏好、全域主題、顯示設定入口、必要資料庫 migration 與測試。
- 完成條件：三種模式可切換並跨重啟保存，既有照護資料與功能回歸驗證通過。

## Protected Behavior

- 不變更照護設定、照護紀錄、照片、量測與被照顧者資料。
- 不新增網路、遠端服務、帳號或第三方套件。
- 不以主題或顏色做健康判讀。

## 實作任務

- [x] T1：建立 App 層級顯示偏好與 migration
  - Status: Complete
  - Boundary:
    - Allowed Changes: `app_database.dart`、產生的 `app_database.g.dart`、資料庫測試
    - Forbidden: 既有照護資料表欄位、遠端儲存、第三方偏好套件
  - Depends: 模式資料模型
  - Context: 新增獨立單列表保存主題模式與文字大小；缺值與無效值分別退回 `system`、`standard`。
  - Verify: 新裝資料庫、舊版 migration、主題與文字大小保存／讀取測試。

- [x] T2：建立主題模式、文字縮放狀態與淺／深 Token
  - Status: Complete
  - Boundary:
    - Allowed Changes: `appearance` feature、`care_nest_theme.dart`、`app.dart`
    - Forbidden: 照護業務流程、資料寫入規則
  - Depends: T1
  - Context: App 啟動讀取偏好；三種模式均映射至正確 `ThemeMode`，三個字級均映射至正確 `TextScaler`。
  - Verify: Widget 測試驗證預設、固定模式、系統模式與三段文字縮放。

- [x] T3：建立顯示設定入口與兩組單選頁
  - Status: Complete
  - Boundary:
    - Allowed Changes: 顯示設定頁、首頁入口、必要 Widget 測試
    - Forbidden: 無關設定功能、路由重構、資料刪除
  - Depends: T2
  - Context: 使用 RadioListTile 或同等可存取單選元件，分別顯示主題模式與文字大小的目前選取和簡短說明。
  - Verify: 選擇後立即更新畫面，切換被照顧者不影響偏好，特大文字不裁切設定頁內容。

- [x] T4：回歸與 Android 實機驗證
  - Status: Complete
  - Boundary:
    - Allowed Changes: 本規格範圍內修正與任務文件
    - Forbidden: 範圍外功能、資料刪除、未確認的套件
  - Depends: T1、T2、T3
  - Context: 驗證三種主題與三段文字在首頁、量測、對話框與被照顧者管理頁的可讀性。
  - Verify: `dart format`、`flutter analyze`、`flutter test`、`flutter build apk --profile`、`git diff --check`、Android 實機檢查。

## 品質檢查清單

- 三種模式有明確文字與單選狀態，不只靠色彩。
- 三段文字大小有明確文字、比例與單選狀態，不只靠預覽大小。
- 預設跟隨系統，無效保存值安全退回跟隨系統。
- 深色與淺色都有背景、卡片、提高表面、主要文字、輔助文字與邊框 Token。
- 新增的 migration 不改寫或刪除任何照護資料。
- 量測圖維持座標、文字、選取定位線與原始數值提示。
- 特大文字下，頂端導覽、卡片、表單、對話框與量測圖不可裁切且保持可操作。

## Implementation Notes

- 2026-08-20：使用者澄清先前兩張參考圖分別是深色與淺色主題，並要求新增跟隨系統模式。
- 2026-08-20：本規格取代前一份規格中的強制 `ThemeMode.dark` 決策；程式尚未開始修改。
- 2026-08-20：使用者要求加入文字大小修改；採用標準 100%、較大 115%、特大 130% 的三段式本機偏好。
- 2026-08-20：完成本機偏好 migration、全域 `ThemeMode` 與 `TextScaler`、首頁顯示設定入口；靜態分析與完整 27 項測試通過，profile APK 已安裝至 Android，待實機逐一確認三種模式與特大文字。
- 2026-08-20：已完成淺色、深色、跟隨系統、三段文字大小與重開 App 保存的 Android 實機確認；顯示設定單選狀態與既有自動化驗證均確認完成。
