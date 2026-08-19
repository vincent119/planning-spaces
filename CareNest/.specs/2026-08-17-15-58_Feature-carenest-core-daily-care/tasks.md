# 任務文件：CareNest 核心 MVP 每日照護確認

Status: InProgress

## Execution Context

- 意圖: 以最小 Flutter 版本驗證一位照顧者能否快速確認一位長輩當日的三餐、高蛋白與飲水狀態
- 非目標: 照片、體重、血壓、通知、備份、還原、同步、醫療判讀、藥品管理與 Backend Server
- 已定決策: Android 優先並保留 iOS；Local-first、Offline-first、Drift、單一被照顧者、每日設定快照、待確認事項優先
- 邊界: 只修改核心 Flutter 功能、相關本機資料與測試；禁止後續功能與遠端服務
- 關鍵檔案: `pubspec.yaml`、`lib/app.dart`、`lib/core/database/**`、`lib/features/**`、`test/**`、`integration_test/**`
- 完成條件: requirements S1 至 S6 與骨架回歸通過，Android Debug APK 可建置，無非目標功能

### Protected Behavior

- 不新增 Backend Server、API、帳號、雲端服務或資料上傳
- 不請求照片、通知、檔案或健康資料權限
- 不提供固定健康目標、異常判讀或用藥功能
- `android/` 與 `ios/` 平台目錄保留
- 不在程式 repository 建立 `.specs/`

### 邊界

#### Allowed Changes

- `pubspec.yaml`、`pubspec.lock`、`analysis_options.yaml`、README
- `lib/main.dart`、`lib/app.dart`、`lib/core/database/**`、`lib/features/**`
- `test/**`、`integration_test/**`
- Android／iOS 因 Flutter 套件產生的必要平台設定
- 本 spec 的 requirements、design、tasks 狀態與實作紀錄

#### Forbidden

- Backend Server、遠端 API、帳號、同步、分析追蹤與雲端設定
- 餐食照片、體重、血壓、通知、備份與 AI 功能
- 使用者無關變更與其他 spec 文件

## 任務依賴

| 任務 | Depends | 狀態 | 備註 |
|------|---------|------|------|
| T1 最小 Flutter 專案 | 無 | Complete | Android Debug APK 已通過 |
| T2 本機資料與每日快照 | T1 | Complete | Drift schema version 1 與資料層測試已通過 |
| T3 首次設定與今日首頁 | T2 | Complete | 三步設定、今日首頁、排序與 Widget 測試已通過 |
| T4 核心快速紀錄 | T2、T3 | Complete | 本機 CRUD、首頁快速操作與明細修改／刪除已通過 |
| T5 七天文字回顧 | T2、T4 | Complete | 七天摘要、常見缺口與單日文字明細已通過 |
| T6 整合與使用者測試 | T3、T4、T5 | InProgress | 自動與 Android 裝置驗證完成；等待真實使用者測試 |

## 實作任務

- [x] T1：建立最小 Flutter 專案
  - Status: Complete
  - Boundary:
    - Allowed Changes: Flutter 骨架、`pubspec.yaml`、lint、README、冒煙測試
    - Forbidden: 產品功能、遠端服務與後續套件
  - Depends: 無
  - Context: Flutter 3.47.0、Android SDK 36、JDK 17；Android 優先並保留 iOS
  - Verify:
    - `dart format lib test`
    - `flutter analyze`
    - `flutter test`
    - `flutter build apk --debug`
    - `git diff --check`

- [x] T2：建立本機資料與每日設定快照
  - Status: Complete
  - Boundary:
    - Allowed Changes: `pubspec.yaml`、`lib/core/database/**`、資料模型、DAO、migration、相關測試
    - Forbidden: UI 功能、照片、量測、通知、備份與遠端資料來源
  - Depends: T1
  - Context: 建立 schema version 1 與六個核心實體；設定修改只影響下一個照護日
  - Verify:
    - 資料約束、CRUD、每日快照不可變與設定隔日生效測試
    - `flutter analyze`
    - `flutter test`
    - `git diff --check`

- [x] T3：建立首次設定與今日首頁
  - Status: Complete
  - Boundary:
    - Allowed Changes: `lib/app.dart`、`lib/shared/theme/**`、`lib/features/onboarding/**`、`dashboard/**`、`settings/**` 與相關測試
    - Forbidden: 量測、圖表、提醒、備份與同步入口
  - Depends: T2
  - Context: 首頁以待確認事項為主；每頁最多一個主要操作；必填錯誤可聚焦
  - Verify:
    - requirements 的首次設定與待確認排序情境
    - 44×44px 觸控目標、可見焦點與錯誤狀態 Widget 測試
    - T1 冒煙測試回歸

- [x] T4：建立餐食、高蛋白與飲水快速紀錄
  - Status: Complete
  - Boundary:
    - Allowed Changes: `lib/features/meals/**`、`supplements/**`、`water/**`、必要 DAO 與測試
    - Forbidden: 餐食照片、營養資料庫、自動估算、量測與通知
  - Depends: T2、T3
  - Context: 高頻操作採最短路徑；取消不得產生不完整紀錄
  - Verify:
    - requirements 的餐食與一鍵新增情境
    - 最近摘要去重、修改、刪除、輸入邊界與首頁更新測試
    - T1、T2、T3 回歸

- [x] T5：建立七天文字回顧
  - Status: Complete
  - Boundary:
    - Allowed Changes: `lib/features/history/**`、必要查詢與測試
    - Forbidden: 圖表、照片、體重、血壓、健康評語與趨勢推論
  - Depends: T2、T4
  - Context: 第一層顯示每日完成數，第二層只顯示核心原始文字資料
  - Verify:
    - 最近 7 日範圍、日期排序、空資料、單日明細與歷史快照測試
    - 所有核心回歸測試

- [ ] T6：整合驗證與目標使用者測試
  - Status: InProgress
  - Boundary:
    - Allowed Changes: 測試、文件、核心流程小型修正與本 spec 驗證結果
    - Forbidden: 新增拆分後的後續功能
  - Depends: T3、T4、T5
  - Context: 驗證核心假設，不以功能數量作為完成標準
  - Verify:
    - requirements S1 至 S6 與骨架回歸
    - 飛航模式、`flutter analyze`、`flutter test`、Android Debug APK、`git diff --check`
    - 至少 5 位使用者的判讀與操作時間紀錄

## 驗證任務

- [ ] 驗收情境覆蓋
  - Verify: requirements.md 的所有情境均對應自動化測試或明確人工檢查

- [ ] 回歸驗證
  - Verify: Flutter 骨架、Android build、iOS 目錄與無後端行為不被破壞

- [ ] 品質檢查清單
  - 格式檢查通過
  - 測試或 dry-run 通過
  - 文件一致性已確認
  - 主要驗收情境已覆蓋
  - Protected Behavior 回歸驗證通過
  - 風險項目已處理
  - `git diff --stat` 已檢查
  - `git diff --check` 已通過

## 實作中斷恢復

恢復時優先讀取：

1. 本文件的 `Execution Context`
2. 目前未完成 task
3. `Protected Behavior`
4. `Implementation Notes`

不得預設掃描整個 `.specs` 目錄。若文件很大，先使用：

```bash
rg -n "^#|^##|^###|Boundary:|Depends:|Implementation Notes|Status:" "$PRIVATE_SPEC_WORKSPACE/CareNest/.specs/2026-08-17-15-58_Feature-carenest-core-daily-care"
```

## Implementation Notes

- 2026-08-17：由過大 MVP 規格拆分。
- 2026-08-17：完成 T1。安裝 Flutter 3.47.0，建立 Android 與 iOS 骨架、最小 CareNestApp、繁體中文 README 與冒煙測試。
- 2026-08-17：Android Debug APK 建置通過；iOS 建置等待完整 Xcode 與 CocoaPods。
- 2026-08-17：依 SDD skill 樣本重整 requirements、design、tasks；T2 尚未開始。
- 2026-08-17：開始 T2。採用 drift、path_provider、drift_dev、build_runner；使用 NativeDatabase.createInBackground。移除 drift_flutter，因其目前帶入停止維護的 SQLite 相容套件；不加入 sqlite3_flutter_libs、網路或後端套件。
- 2026-08-17：完成 T2。建立 Drift schema version 1，包含 CareRecipient、CareSettings、DailyCareSnapshot、MealRecord、SupplementRecord、WaterRecord；新增單一對象、目標驗證、每日快照不可變與隔日生效測試。產碼、靜態分析、全部測試與 Android Debug APK 均通過。
- 2026-08-17：開始 T3。首次設定採三步流程；首頁只呈現待確認狀態與排序，快速紀錄保留給 T4。新增 shared theme 以落實 UI token、觸控尺寸與 Light／Dark 狀態，已更新 T3 Boundary。
- 2026-08-17：完成 T3。新增三步首次設定、ChangeNotifier 控制器、今日首頁與餐次優先排序；建立 Primitive、Semantic、Component 對應的 ThemeData／ThemeExtension。Widget 測試確認首次開啟與完成設定後首頁；單元測試確認已過時間午餐先於今天稍後事項。修正按鈕 token 的無限寬度問題，維持 48px 最小高度。
- 2026-08-18：開始 T4。範圍限於本機餐食、高蛋白與飲水 CRUD、首頁快速操作、今日明細與測試；不新增照片、量測、提醒、備份、同步或 Backend Server。
- 2026-08-18：完成 T4。餐食可選擇有吃或未吃；有吃時需填餐食摘要與食用比例。首頁支援預設高蛋白 `+ 1 瓶`、飲水 `+100/+200/+300 ml` 與其他水量。今日明細支援高蛋白與飲水修改、所有核心紀錄刪除；餐食以相同餐別重新儲存完成更新。新增資料層 CRUD 與累積測試，9 項測試、靜態分析與 Android Debug APK 均通過。
- 2026-08-18：開始 T5。回顧只讀取既有每日快照，不為歷史空白日期建立資料；不加入圖表、量測、照片或醫療解讀。
- 2026-08-18：完成 T5。新增七天回顧、每日完成數、常見缺口與單日核心文字明細。回顧只讀取既有快照，空白日期顯示尚無照護紀錄，不會寫入資料。新增歷史資料測試；10 項測試、靜態分析與 Android Debug APK 均通過。
- 2026-08-18：開始 T6。先執行自動整合驗證與 Android 建置；目標使用者可用性測試需要真實照顧者，待使用者安排後執行。
- 2026-08-18：T6 自動驗證完成。Drift 產碼、靜態分析、10 項測試與 Android Debug APK 均通過。找到既有 Android Emulator `test_api36`，但啟動後未連線至 Flutter，尚未完成裝置手動驗證。已建立 `usability-test.md`，等待至少 5 位真實主要照顧者參與。
- 2026-08-18：恢復 `test_api36`。System UI 卡死原因為可用主機記憶體不足導致軟體 GL 渲染，以及殘留的 AVD 鎖定檔；移除已確認的 stale lock 後以冷啟動完成開機。CareNest Debug APK 已安裝，`com.carenest.care_nest/.MainActivity` 已確認為 Android 前景 Activity。T6 只剩真人可用性測試。
- 2026-08-18：新增 `integration_test/core_flow_test.dart`，覆蓋首次設定、餐食與回顧的裝置流程。該測試在本機 `test_api36` 因 Emulator 再次離線而無法完成；高蛋白與飲水維持由資料層與 Widget 測試覆蓋。需在記憶體較充足或實體 Android 裝置上重跑 `flutter test integration_test/core_flow_test.dart -d <device-id>`。
- 2026-08-18：連接實體 Android 13 裝置 `CPH2237`。integration_test runner 在測試 suite 載入前中斷，但 `flutter run --debug` 已成功連接 Dart VM Service、同步 App 並完成 Flutter 首幀；實機截圖確認首次設定第 3 步正常呈現。實機 App 啟動驗證通過，裝置整合測試 runner 穩定性仍待處理。
- 2026-08-18：實體裝置手動完成一次核心流程：首次設定、三餐、高蛋白、飲水均保存成功，今日首頁顯示 5 / 5 已確認，語意樹與截圖均確認餐食摘要、食用比例、高蛋白與飲水累積正確。此為一輪實機手動驗證，不等同至少 5 位主要照顧者可用性測試。

## 驗證結果摘要

- 新行為驗證: T1、T2、T3、T4、T5 與 T6 自動驗證通過；`dart format`、`dart run build_runner build`、`flutter analyze`、`flutter test`、`flutter build apk --debug`
- 回歸驗證: `android/`、`ios/` 保留；無 Backend Server、API、帳號、同步、drift_flutter 或 SQLite 相容套件相依；Android Emulator 已完成安裝與前景 Activity 驗證，但裝置整合測試需在穩定裝置重跑，真人可用性測試待完成
- 文件一致性: 核心三份文件已依 skill 樣本重整
- 剩餘風險: 正式 applicationId 尚未確認；iOS 工具鏈尚未完成

## 後續改善

- [ ] 核心 MVP 驗證後再確認正式 applicationId 與 iOS bundle identifier
- [ ] 完整 Xcode 與 CocoaPods 就緒後補 iOS build 驗證
