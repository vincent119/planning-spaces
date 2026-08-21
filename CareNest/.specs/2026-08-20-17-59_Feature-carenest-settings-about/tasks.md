# 任務文件：CareNest 設定與關於頁

Status: Complete

## Execution Context

- 意圖: 建立一般設定入口、設定首頁與關於頁，動態顯示實際安裝版本及建置編號。
- 非目標: 不實作登入、雲端、家庭空間、訂閱、政策網站、版本遞增或 Backend Server。
- 已定決策: 設定首頁直接包含顯示模式與文字大小，不保留顯示設定子頁；關於 CareNest 使用獨立子頁；版本顯示為 `版本 <version>（<buildNumber>）`；使用 `package_info_plus`；全產品無廣告。
- 邊界: 只修改 Dashboard 設定入口、settings 新模組、依賴宣告與直接相關測試；既有資料庫及照護功能不得修改。
- 關鍵檔案: `lib/features/dashboard/care_dashboard_page.dart`、`lib/features/appearance/appearance_settings_page.dart`、`lib/features/settings/`、`pubspec.yaml`
- 完成條件: requirements 的四個驗收情境有自動化測試；格式、analyze、完整測試與 diff 檢查通過。

### Protected Behavior

- 顯示模式與文字大小仍可立即更新並保存。
- 今日首頁的量測、回顧、明細與被照顧者管理入口不受影響。
- 本機功能維持不需登入、不需網路且沒有廣告。
- Android 與保留的 iOS 平台目錄不得移除。

### 邊界

#### Allowed Changes

- `pubspec.yaml`
- `pubspec.lock`
- `lib/features/dashboard/care_dashboard_page.dart`
- `lib/features/appearance/appearance_settings_page.dart`，限遷移後移除
- `lib/features/settings/`
- `test/features/settings/`
- 與 Dashboard 設定入口直接相關的既有 Widget test
- `test/features/appearance/appearance_settings_page_test.dart`，限將既有覆蓋遷入設定頁測試後移除

#### Forbidden

- `lib/core/database/` 與資料庫 schema
- 每日照護、量測、回顧、明細及被照顧者的業務邏輯
- Google 登入、Google Drive、家庭空間、訂閱與任何 Backend Server
- 新增無關套件、政策網址、公司名稱、聯絡資訊或分析追蹤
- 硬編碼 App 版本至 Widget

## 任務依賴

| 任務 | Depends | 狀態 | 備註 |
|------|---------|------|------|
| Task 1：加入版本資訊依賴與介面 | 無 | Complete | 使用 `package_info_plus 10.2.1` |
| Task 2：建立設定首頁並遷入顯示控制 | 無 | Complete | 未建立顯示設定子頁 |
| Task 3：建立關於 CareNest 頁 | Task 1 | Complete | 已涵蓋成功、錯誤與特大字 |
| Task 4：更新今日首頁入口 | Task 2、Task 3 | Complete | 其他導航未變更 |
| Task 5：補測試與完整驗證 | Task 1 至 Task 4 | Complete | 36 項完整測試通過 |

## 實作任務

- [x] Task 1：加入版本資訊依賴與最小介面
  - Status: Complete
  - Boundary: Allowed Changes 中的 `pubspec.yaml`、`pubspec.lock`、`lib/features/settings/app_version_info.dart`
  - Depends: 無
  - Context: 以 `package_info_plus` 取得 `version` 與 `buildNumber`；先確認套件版本支援 Flutter 3.47.0、Dart 3.13.0、Android 及 iOS。介面只服務關於頁，不建立通用服務架構。
  - Verify:
    - `flutter pub get`
    - `flutter analyze`

- [x] Task 2：建立設定首頁並遷入顯示控制
  - Status: Complete
  - Boundary: Allowed Changes 中的 `lib/features/settings/settings_page.dart`、`lib/features/appearance/appearance_settings_page.dart`、兩者直接相關測試
  - Depends: 無
  - Context: 設定首頁使用可捲動列表，直接呈現既有「顯示模式」與「文字大小」RadioGroup，繼續接收 mode、textSize 與 onChanged callback；遷移完成後移除獨立 `AppearanceSettingsPage` 及舊測試，但所有既有斷言必須移入設定頁測試。設定首頁另保留「關於 CareNest」入口。
  - Verify:
    - `flutter test test/features/settings/settings_page_test.dart`
    - `rg -n "AppearanceSettingsPage" lib test` 應無殘留引用

- [x] Task 3：建立關於 CareNest 頁
  - Status: Complete
  - Boundary: Allowed Changes 中的 `lib/features/settings/about_care_nest_page.dart`、品牌資產引用與直接相關測試
  - Depends: Task 1
  - Context: 顯示 CareNest 品牌圖、產品名稱、「讓家人知道，今天是否照顧到了。」、「本機功能不需要登入，沒有廣告。」及動態版本。需處理載入、成功、失敗及 130% 文字狀態。
  - Verify:
    - `flutter test test/features/settings/about_care_nest_page_test.dart`
    - 人工檢查品牌圖保持長寬比且淺色、深色皆可辨識

- [x] Task 4：更新今日首頁設定入口
  - Status: Complete
  - Boundary: Allowed Changes 中的 `lib/features/dashboard/care_dashboard_page.dart` 與直接相關 Dashboard Widget test
  - Depends: Task 2、Task 3
  - Context: 將原顯示設定圖示改為齒輪設定圖示，tooltip 改為「設定」，push 新設定首頁；其他 AppBar 操作保持原順序與行為。
  - Verify:
    - `flutter test --plain-name '今日首頁的設定按鈕可開啟設定首頁'`
    - `flutter test test/widget_test.dart`

- [x] Task 5：補齊驗收測試與完整驗證
  - Status: Complete
  - Boundary: 所有 Allowed Changes；僅修正本功能造成的格式、analyze 或測試問題
  - Depends: Task 1、Task 2、Task 3、Task 4
  - Context: 覆蓋正常版本、讀取失敗、設定導航、既有顯示偏好及特大文字；不得為通過測試而忽略 analyzer。
  - Verify:
    - `/opt/homebrew/bin/dart format --output=none --set-exit-if-changed lib test`
    - `/opt/homebrew/bin/flutter analyze`
    - `/opt/homebrew/bin/flutter test`
    - `git diff --stat`
    - `git diff --check`

## 驗證任務

- [x] 驗收情境覆蓋
  - Verify: requirements 的「設定入口」「版本顯示」「讀取失敗」「既有顯示設定」均有 Widget test

- [x] 回歸驗證
  - Verify: `flutter test` 全數通過，且 Dashboard 其他入口與外觀偏好測試沒有回歸

- [x] 品質檢查清單
  - [x] 格式檢查通過
  - [x] `flutter analyze` 通過
  - [x] 相關與完整測試通過
  - [x] 文件一致性已確認
  - [x] 主要驗收情境已覆蓋
  - [x] Protected Behavior 回歸驗證通過
  - [x] 特大字、深淺主題及版本錯誤狀態已處理
  - [x] `git diff --stat` 已檢查
  - [x] `git diff --check` 已通過

## 實作中斷恢復

恢復時優先讀取：

1. 本文件的 `Execution Context`
2. 目前未完成 task
3. `Protected Behavior`
4. `Implementation Notes`

不得預設掃描整個 `.specs` 目錄。可先執行：

```bash
rg -n "^#|^##|^###|Boundary:|Depends:|Implementation Notes|Status:" /Users/vincent/Documents/git_home/vin/planning-spaces/CareNest/.specs/2026-08-20-17-59_Feature-carenest-settings-about
```

## Implementation Notes

- 2026-08-20 開始實作。
- `package_info_plus 10.2.1` 官方需求為 Flutter 3.38.1 以上、Dart 3.10 以上、Java 17、Android Gradle Plugin 8.12.1 以上、Gradle 8.13 以上及 iOS 13 以上；專案為 Flutter 3.47、Dart 3.13、Java 17、Android Gradle Plugin 9.1.0、Gradle 9.3.1 及 iOS 15，符合需求。
- 原 `AppearanceSettingsPage` 的顯示模式與文字大小控制已遷入 `SettingsPage`，舊頁面及舊測試已移除，測試覆蓋已移至 `test/features/settings/settings_page_test.dart`。
- 第一輪測試發現「關於 CareNest」位於可捲動區底部時尚未建立 Widget；修正測試為先捲動至項目後驗證，實際 UI 行為不需改動。
- Android Debug APK 已成功建置並覆蓋安裝至裝置 `4HMNVSYXIF9TEURW`；安裝版本為 `1.0.0`、建置編號為 `1`，App 程序可正常啟動。
- 雲端與商業化草稿維持暫停，不是本任務依賴。

## 驗證結果摘要

- 新行為驗證: 通過；`flutter test test/features/settings test/widget_test.dart` 共 12 項通過
- 回歸驗證: 通過；`flutter analyze` 無問題，`flutter test` 共 36 項通過，Android Debug APK 建置成功
- 文件一致性: 已確認，requirements 狀態與 tasks 均已更新為 Complete
- 剩餘風險: 尚未在本機執行 iOS 建置；iOS 目錄及最低版本設定未變更

## 後續改善

- [ ] 正式隱私權政策、使用條款與聯絡方式確定後，另案加入設定頁
