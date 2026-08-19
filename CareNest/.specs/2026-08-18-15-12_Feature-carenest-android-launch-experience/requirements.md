# 需求文件：CareNest Android 啟動體驗

## 來源

- Type：Feature
- Status：InProgress
- 使用者回饋：Android 系統啟動畫面呈現黑底與淺色圖示方塊，辨識與品牌一致性不足。

## 文件定位

本功能接續已完成的 CareNest App Icon 替換。它只調整 Android 從系統啟動到 Flutter 載入頁的短暫過場，不修改桌面圖示、照護功能、本機資料或 iOS。

## 目標

1. Android 系統啟動畫面使用深墨綠背景與小型、比例正確的 CareNest 過渡圖示。
2. Flutter 載入頁延續相同背景，提供簡短可理解的開啟狀態。
3. 系統啟動與 Flutter 首幀之間不出現白色或淺色方形圖示底板。
4. Android 12 以上顯示以透明留白縮小的原始圖示，並在 Flutter 首幀就緒時直接交給 Flutter 載入頁。
5. Flutter 載入頁在啟動時至少可見 500ms，讓使用者能看到正確比例的標誌與開啟文字。

## 非目標

1. 網路預載、超過 500ms 的人為延長、品牌文字動畫、啟動廣告與仿製其他 App 的多色樣式。
2. iOS Launch Screen、桌面 App Icon、App 名稱或一般頁面主題重設計。
3. Backend Server、同步、帳號、資料傳輸與照護資料變更。

## 驗收情境

### 情境：Android 系統啟動畫面一致

- 測試：Android 實機冷啟動人工檢查。
- 假設：裝置使用 Android 12 以上。
- 當：使用者從桌面開啟 CareNest。
- 那麼：系統啟動畫面顯示深墨綠背景與小型比例正確圖示，不出現大型圖示或淺色方形底板。

### 情境：Flutter 載入頁不突兀

- 測試：`test/widget_test.dart`。
- 假設：本機資料初始化尚未完成。
- 當：Flutter 首幀顯示。
- 那麼：至少 500ms 看到同色背景、正確比例標誌與「正在開啟 CareNest」，不只顯示通用轉圈圈；進入首次設定或今日首頁時使用 240ms 淡入。

### 情境：Flutter 載入頁顯示原始圖形

- 測試：Android 實機冷啟動人工檢查。
- 假設：裝置使用 Android 12 以上且系統動畫未停用。
- 當：Flutter 首幀準備顯示載入頁。
- 那麼：系統畫面直接移除，顯示未變形的原始 CareNest 標誌與開啟文字，背景不中斷。

## 驗收條件

1. Android API 31 以上與既有 Android 樣式皆有深墨綠啟動背景。
2. 載入頁使用品牌標誌與文字，並在第一次 Flutter 繪製後等待 500ms 才開始既有初始化；不修改初始化資料與結果。
3. `flutter analyze`、`flutter test`、`flutter build apk --debug`、Android 實機冷啟動與 `git diff --check` 通過。
