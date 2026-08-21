# 任務文件：CareNest 延伸回顧與量測範圍

Status: Completed

## Execution Context

- 意圖: 讓使用者取回既已保存在本機的七天以前資料，提供 7／30 天回顧、單日歷史與 7／30／90 天量測趨勢。
- 非目標: 不做自動刪除、保存期限設定、任意日期區間、醫療判讀、雲端、登入、訂閱或 Backend Server。
- 已定決策: 本機資料不因超過七天自動刪除；預設仍為七天；長範圍使用日期區間查詢；不新增套件與 schema migration。
- 待確認: 無；使用者已於 2026-08-21 以「run task」確認五項提案。
- 邊界: 僅修改資料庫讀取方法、controller 回顧與量測查詢、兩個頁面及直接相關測試；不得碰現有未提交的設定／關於頁成果。
- 關鍵檔案: `app_database.dart`、`care_nest_controller.dart`、`care_history_page.dart`、`care_measurements_page.dart`
- 完成條件: requirements 六個驗收情境均有測試；格式、analyze、完整測試、Android 實機與 diff 檢查通過。

### Protected Behavior

- 七天仍為回顧與量測預設值。
- 既有量測 X／Y 軸、深淺主題、血壓雙線、垂直定位線、原始值與編修刪除維持可用。
- 歷史查詢只讀，不建立每日快照。
- 多位被照顧者資料不混用。
- 本機資料沒有七天自動刪除機制。
- 現有設定與關於頁未提交變更不可覆寫、還原或格式化到無關檔案。

### 邊界

#### Allowed Changes

- `lib/core/database/app_database.dart`
- `lib/features/dashboard/care_nest_controller.dart`
- `lib/features/history/care_history_page.dart`
- `lib/features/measurements/care_measurements_page.dart`
- `test/core/database/` 內直接相關測試
- `test/features/history/`
- `test/features/measurements/`
- `lib/core/database/app_database.g.dart`，僅在 Drift 產生內容確有變更時

#### Forbidden

- `lib/features/settings/`、`test/features/settings/` 與設定／關於頁相關未提交變更
- 登入、雲端、同步、家庭空間、訂閱及 Backend Server
- 第三方圖表套件、醫療判讀、正常值或警示
- 自動刪除、背景清理及保存天數設定
- 資料庫 schema version、table 欄位或 migration，除非先更新本 spec 並獲使用者確認
- 已完成趨勢圖的色彩 token 與基本互動契約

## 任務依賴

| 任務 | Depends | 狀態 | 備註 |
|------|---------|------|------|
| Task 1：建立日期範圍型別與區間查詢 | 使用者確認五項提案 | Completed | 各表一次 query |
| Task 2：一般化 controller 回顧與量測載入 | Task 1 | Completed | 依日分組並保護只讀 |
| Task 3：擴充回顧頁 | Task 2 | Completed | 7／30 天及單日日期 |
| Task 4：擴充量測頁與趨勢密度 | Task 2 | Completed | 7／30／90 天 |
| Task 5：回歸、效能與實機驗證 | Task 3、Task 4 | Completed | Android 實機操作確認完成 |

## 實作任務

- [x] Task 1：建立日期範圍型別與資料庫區間查詢
  - Status: Completed
  - Boundary: `app_database.dart`、必要的型別檔案與 database tests；不得改 schema
  - Depends: 使用者確認 requirements 五項提案
  - Context: 照護四張表與量測兩張表各以 `recipientId`、`careDay >= startDay`、`careDay < endExclusive` 查詢一次。回傳依日期及 occurredAt 穩定排序的既有 row，不進行聚合或寫入。
  - Verify:
    - `flutter test test/core/database --plain-name '日期區間'`
    - 跨月、跨年、端點排除與多位被照顧者隔離測試通過

- [x] Task 2：一般化 controller 回顧與量測載入
  - Status: Completed
  - Boundary: `care_nest_controller.dart` 與 controller tests
  - Depends: Task 1
  - Context: 提供只接受允許天數的 range 方法；將 range rows 依 `careDay` 分組。每一天趨勢沿用當日最新一筆，單日歷史維持只讀。可保留舊七天方法作薄包裝，或同步遷移呼叫端後移除，避免重複邏輯。
  - Verify:
    - `flutter test test/features/history/care_history_test.dart`
    - `flutter test test/features/measurements/measurement_trend_test.dart`
    - 確認查看空白舊日期前後 snapshot 筆數不變

- [x] Task 3：擴充回顧頁
  - Status: Completed
  - Boundary: `care_history_page.dart` 與 `test/features/history/`
  - Depends: Task 2
  - Context: 預設 7 天；SegmentedButton 切換 7／30 天；「選擇日期」開啟 Material 日期選擇器，範圍為被照顧者建立日至今天。切換需處理 loading、success、empty、failure 及快速切換；特大文字下可捲動。
  - Verify:
    - `flutter test test/features/history`
    - `flutter test --plain-name '選擇無紀錄日期顯示空狀態且不建立快照'`

- [x] Task 4：擴充量測頁與趨勢密度
  - Status: Completed
  - Boundary: `care_measurements_page.dart` 與 `test/features/measurements/`
  - Depends: Task 2
  - Context: 預設 7 天；SegmentedButton 切換 7／30／90 天；依 design 圖表密度規則繪圖。折線使用全部實際每日趨勢點，30／90 天隱藏一般點並降低 X 標籤；選取後仍顯示垂直線、日期與原始值。新增、修改、刪除後保留目前範圍。
  - Verify:
    - `flutter test test/features/measurements`
    - `flutter test --plain-name '九十天趨勢拖曳後顯示選取日期與原始數值'`
    - 淺色、深色及 130% 文字 Widget tests 通過

- [x] Task 5：完整回歸、效能與 Android 實機驗證
  - Status: Completed
  - Boundary: 所有 Allowed Changes；只能修正本功能引入的問題
  - Depends: Task 3、Task 4
  - Context: 以至少 90 天測試資料驗證切換與圖表互動；不得為通過測試而減少資料點或忽略 analyzer。實機確認 90 天操作無明顯卡頓，若超過約 300ms 才回報並討論索引，不直接改 schema。
  - Verify:
    - `/opt/homebrew/bin/dart format --output=none --set-exit-if-changed lib test`
    - `/opt/homebrew/bin/flutter analyze`
    - `/opt/homebrew/bin/flutter test`
    - `/opt/homebrew/bin/flutter build apk --debug`
    - Android 實機切換 7／30／90 天、拖曳圖表及返回頁面
    - `git diff --stat`
    - `git diff --check`

## 驗證任務

- [x] 驗收情境覆蓋
  - Verify: requirements 六個驗收情境皆有明確自動化測試 selector

- [x] 回歸驗證
  - Verify: 七天預設、趨勢互動、量測編修、回顧摘要、多位對象隔離及資料不自動刪除均通過

- [ ] 品質檢查清單
  - [x] 格式檢查通過
  - [x] `flutter analyze` 通過
  - [x] 相關與完整測試通過
  - [x] 文件一致性已確認
  - [x] 主要驗收情境已覆蓋
  - [x] Protected Behavior 回歸驗證通過
  - [x] 90 天圖表、特大文字及深淺主題已由 Widget tests 驗證
  - [x] 現有未提交設定頁成果未被覆寫
  - [x] `git diff --stat` 已檢查
  - [x] `git diff --check` 已通過
  - [x] Android 實機安裝與操作驗證

## 實作中斷恢復

恢復時優先讀取：

1. 本文件的 `Execution Context`
2. 目前未完成 task
3. `Protected Behavior`
4. `Implementation Notes`

不得掃描整個 `.specs` 目錄。可先執行：

```bash
rg -n "^#|^##|^###|Boundary:|Depends:|Implementation Notes|Status:" /Users/vincent/Documents/git_home/vin/planning-spaces/CareNest/.specs/2026-08-20-18-18_Feature-carenest-extended-history-ranges
```

## Implementation Notes

- 2026-08-21 開始實作；使用者已確認 requirements 五項提案。
- 2026-08-21 完成日期區間查詢、回顧 7／30 天與單日日期、量測 7／30／90 天、圖表密度及最近實際資料點選取。
- 2026-08-21 `flutter analyze`、47 項完整測試、`git diff --check` 與 debug APK 建置通過；執行 `adb devices -l` 時沒有已連線裝置，因此尚未安裝至 Android 實機。
- 2026-08-21 手機重新連線後以 `adb install -r` 安裝成功；CareNest 程序正常存活，啟動錯誤日誌為空，實機畫面可正常顯示量測趨勢與既有資料。
- 2026-08-21 使用者已完成 Android 實機 30／90 天切換與圖表互動確認，任務結束。
- 建立文件時，程式碼工作樹已有上一個設定與關於頁任務的未提交變更；這些變更屬於 Protected Behavior，不得覆寫或還原。
- 既有可互動趨勢規格每天取第一筆查詢結果；由於資料庫依 `occurredAt` 降冪排序，此契約等同「當日最新一筆」。

## 驗證結果摘要

- 新行為驗證: 日期區間、30 天回顧、單日空狀態、30／90 天量測、圖表選取、深淺主題與 130% 文字測試通過
- 回歸驗證: `flutter test` 共 47 項通過
- 文件一致性: requirements、design、tasks 與實作一致
- 剩餘風險: 無。本功能後續若需一年以上分析，依「後續改善」另案規劃。

## 後續改善

- [ ] 若實機 90 天查詢明顯超過 300ms，另案評估 `(recipientId, careDay)` 複合索引與 migration
- [ ] 未來若需要一年以上分析，再另案評估月視圖、資料聚合與匯出
