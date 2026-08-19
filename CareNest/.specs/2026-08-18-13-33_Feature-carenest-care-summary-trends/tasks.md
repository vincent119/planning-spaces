# 任務文件：CareNest 照護摘要與量測趨勢

Status: InProgress

## Execution Context

- 意圖：讓照顧者快速回顧今天與最近七天的照護事實及量測原始數值。
- 非目標：醫療判讀、通知、同步、備份、帳號、Backend Server 與首頁大型圖表。
- 已定決策：首頁文字摘要、七天固定範圍、量測原始折線、少於兩筆資料不畫曲線。
- 邊界：dashboard、history、measurements、必要共用 UI 與測試。
- 關鍵檔案：`lib/features/dashboard/**`、`lib/features/history/**`、`lib/features/measurements/**`、`test/features/**`。
- 完成條件：requirements 三個驗收情境、核心回歸、Android Debug APK 與實機可讀性檢查通過。

### Protected Behavior

- 不修改每日完成數算法、待確認排序或量測 CRUD。
- 不建立 Backend Server、API、同步、帳號、分析追蹤或資料上傳。
- 不產生健康閾值、異常警示、醫療建議或處置文字。

## 任務依賴

| 任務 | Depends | 狀態 | 備註 |
|------|---------|------|------|
| T1 今日與七天照護摘要 | 照護細節完成 | Complete | 首頁與七天回顧摘要已通過實機確認 |
| T2 原始量測趨勢圖 | T1 | InProgress | 本機原始趨勢圖已完成，待實機可讀性確認 |
| T3 整合驗證 | T1、T2 | Planned | 自動與 Android 實機檢查 |

## 實作任務

- [x] T1：今日與七天照護摘要
  - Status: Complete
  - Boundary:
    - Allowed Changes: dashboard、history、摘要組合器與相關測試
    - Forbidden: 完成數規則、資料寫入、圖表、通知與後端
  - Depends: 照護細節完成
  - Context: 摘要只能整理既有資料，首頁保持待確認事項優先
  - Verify: 今日摘要、七天摘要、空資料與核心排序回歸

- [ ] T2：原始量測趨勢圖
  - Status: InProgress
  - Boundary:
    - Allowed Changes: measurements、必要本機繪圖元件與相關測試
    - Forbidden: 醫療閾值、判讀、第三方服務與資料傳輸
  - Depends: T1
  - Context: 最近七天；至少兩筆才畫線；數值與日期不只以顏色傳達
  - Verify: 空資料、單筆、多筆排序、血壓雙線與無醫療詞彙

- [ ] T3：整合驗證
  - Status: Planned
  - Boundary:
    - Allowed Changes: 測試、文件與範圍內的修正
    - Forbidden: 新產品功能與範圍外重構
  - Depends: T1、T2
  - Context: 確認首頁仍可快速操作，趨勢在 Android 小螢幕可讀
  - Verify: `dart format`、`flutter analyze`、`flutter test`、`flutter build apk --debug`、`git diff --check`、Android 實機檢查

## 品質檢查清單

- 摘要與圖表不寫入資料。
- 無健康判讀與外部連線。
- 少於兩筆量測不畫曲線。
- 主要操作的觸控目標至少 48px。
- T1 至 T3 驗證結果與文件一致。

## Implementation Notes

- 2026-08-18：由已完成的照護細節功能延伸建立；尚未開始實作。
- 2026-08-18：開始 T1、T2。今日首頁新增文字摘要卡，保留待確認事項與快速操作位置。量測頁新增最近七天體重與血壓原始數值趨勢區塊，使用 Flutter `CustomPainter`，不新增圖表套件；少於兩個資料點時顯示資料不足文字。`flutter analyze`、17 項測試與 `git diff --check` 通過。七天照護摘要文字與 Android 實機可讀性確認待完成。
- 2026-08-18：完成 T1。以 `adb install -r` 保留資料更新 Android，確認今日首頁顯示摘要卡，七天回顧顯示完成數與最常缺口。T2 已有實作與空資料檢查，但需至少兩天同類量測資料進行曲線實機可讀性確認。
- 2026-08-18：補齊 T2 Widget 驗證。新增 `test/features/measurements/measurement_trend_test.dart`，驗證零筆、單筆、多天體重資料與多天血壓雙線；少於兩筆時不建立趨勢畫布。趨勢卡補上每個資料點的日期與原始數值文字，避免只以線條顏色區分。`flutter analyze`、20 項 `flutter test`、`flutter build apk --debug` 與 `git diff --check` 通過。未在實機建立測試量測資料，故保留 T2 InProgress，待使用者以既有或自行輸入的至少兩天同類資料確認小螢幕可讀性。
