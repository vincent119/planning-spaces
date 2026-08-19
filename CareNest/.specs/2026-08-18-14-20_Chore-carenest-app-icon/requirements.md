# 需求文件：CareNest App Icon 替換

## 來源

- Type：Chore
- Status：Planned
- 使用者已確認採用 CareNest 家屋與包覆巢穴圖示草案。

## 文件定位

本任務只替換 Flutter 專案模板的預設圖示，接續既有 Android 優先、保留 iOS 支援的專案設定；不調整 App 功能、資料、本機儲存或畫面設計。

## 目標

1. 使用使用者確認的 CareNest 圖示草案取代 Flutter 預設標誌。
2. 為 Android 與 iOS 產生各自既有資產目錄所需尺寸。
3. 保持圖示無文字、無第三方品牌與醫療判讀意象。

## 非目標

1. 不變更 App 名稱、Android adaptive icon 架構、啟動畫面或 App 內容。
2. 不引入 `flutter_launcher_icons` 或其他新套件。
3. 不建立後端、網路服務或資料傳輸。

## 驗收情境

### 情境：Android 顯示 CareNest 圖示

- 測試：`flutter build apk --debug` 與 Android 實機啟動器人工檢查。
- 假設：Android 專案使用既有 `@mipmap/ic_launcher`。
- 當：安裝 Debug APK。
- 那麼：啟動器顯示 CareNest 圖示，不再是 Flutter 預設標誌。

### 情境：iOS 保留可建置的完整圖示資產

- 測試：資產檔案尺寸檢查。
- 假設：iOS 專案保留既有 `AppIcon.appiconset` 清單。
- 當：檢查 `Contents.json` 列出的圖檔。
- 那麼：每個圖檔存在且像素尺寸與名稱要求一致。

## 驗收條件

1. Android mipmap 與 iOS AppIcon 資產均替換完成。
2. 圖示主稿保留在專案可追蹤位置，來源為使用者確認草案。
3. Android Debug APK 可成功建置，且未引入新相依套件。
