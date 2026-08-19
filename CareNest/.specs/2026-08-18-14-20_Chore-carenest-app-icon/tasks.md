# 任務文件：CareNest App Icon 替換

Status: Complete

## Execution Context

- 意圖：將使用者確認的 CareNest 圖示納入 Android 與 iOS 平台資產。
- 非目標：Dart 功能、App 名稱、啟動畫面、新套件與後端。
- 已定決策：不使用第三方圖示產生套件；直接輸出既有資產檔名及尺寸。
- 邊界：`assets/branding/`、Android mipmap、iOS AppIcon，及必要驗證。
- 關鍵檔案：`AndroidManifest.xml`、`AppIcon.appiconset/Contents.json`。
- 完成條件：所有既有圖示檔案均有效、Android APK 建置成功並實機顯示。

### Protected Behavior

- 不修改 Android Manifest 的 launcher icon 參照。
- 不修改 App 功能、資料、依賴或使用者現有本機資料。
- 保留 iOS 專案，不要求目前環境進行 Xcode 建置。

## 實作任務

- [x] T1：保存主稿並輸出 Android 與 iOS 圖示
  - Status: Complete
  - Boundary:
    - Allowed Changes: 品牌主稿、現有 Android mipmap PNG、現有 iOS AppIcon PNG
    - Forbidden: Dart、Manifest 參照、App 名稱、新套件與其他平台設定
  - Depends: 使用者確認圖示草案
  - Context: 同一主稿縮放至現有檔案要求；不裁切核心符號
  - Verify: 檔案存在、PNG 尺寸正確、Android Debug APK 建置

- [x] T2：實機與資產完整性驗證
  - Status: Complete
  - Boundary:
    - Allowed Changes: 驗證、任務文件與範圍內修正
    - Forbidden: 產品功能與外部服務
  - Depends: T1
  - Context: Android 使用保留資料的安全更新方式
  - Verify: Android 啟動器人工檢查、`git diff --check`

## 品質檢查清單

- 不含 Flutter 預設標誌。
- 不新增 package。
- Android 與 iOS 資產檔名保留。
- 不影響使用者本機照護資料。

## Implementation Notes

- 2026-08-18：使用者確認採用圖示草案，建立任務並開始處理平台尺寸。
- 2026-08-18：主稿保存為 `assets/branding/carenest_app_icon.png`，並已輸出 Android 五組 mipmap 與 iOS AppIcon 全部既有尺寸。逐一檢查 Android 與 iOS 共 20 個檔案尺寸正確；`flutter build apk --debug`、`git diff --check` 通過。以 `adb install -r` 成功更新實體 Android，保留既有本機照護資料並重新啟動 App。
- 2026-08-18：實機啟動器回饋顯示前一版的白色外側留白降低辨識度。使用者確認改採滿版薄荷綠底、加粗並放大的深青綠符號；重新開啟 T1 與 T2 以輸出及驗證替換資產。
- 2026-08-18：已採用滿版加粗版主稿並重新輸出所有平台尺寸。實際檢視 Android 48px `mipmap-mdpi`，符號在小尺寸下可辨識；`flutter build apk --debug`、`git diff --check` 通過。以 `adb install -r` 成功更新實體 Android，保留既有本機照護資料並重啟 App。
