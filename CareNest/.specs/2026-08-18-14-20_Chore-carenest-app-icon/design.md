# 設計文件：CareNest App Icon 替換

## 文件定位

對應同目錄 requirements。本任務只改平台圖示二進位資產，不修改 Flutter Dart 程式、資料庫或功能模組。

## 已知契約狀態

- Android：`AndroidManifest.xml` 使用 `@mipmap/ic_launcher`，並已有五組密度 PNG。
- iOS：`AppIcon.appiconset/Contents.json` 列出 iPhone、iPad 與 App Store 所需 PNG。
- 主稿：使用者確認的 1254 × 1254 PNG，視覺為淡薄荷綠圓角底板與深青綠家屋包覆符號。

## 設計原則

- 使用主稿縮放，不裁切、不補畫新圖形。
- 不增加套件，直接使用系統影像工具輸出既有檔名與尺寸。
- iOS 與 Android 共用同一構圖，確保品牌一致。

## 受影響檔案

| 類型 | 位置 | 變更 |
|---|---|---|
| 主稿 | `assets/branding/carenest_app_icon.png` | 保存已確認圖示主稿 |
| Android | `android/app/src/main/res/mipmap-*/ic_launcher.png` | 依密度輸出 PNG |
| iOS | `ios/Runner/Assets.xcassets/AppIcon.appiconset/*.png` | 依 Contents.json 輸出 PNG |

## 風險與處理

| 風險 | 處理 |
|---|---|
| 圖示在小尺寸失真 | 輸出後檢查全部像素尺寸，Android 以實機檢查 |
| iOS 欠缺完整資產 | 逐一依 Contents.json 的檔名及實際像素尺寸寫入 |
| 原始草稿有微小明暗差異 | 保留使用者已確認的圖像，不在此任務另行改繪 |
