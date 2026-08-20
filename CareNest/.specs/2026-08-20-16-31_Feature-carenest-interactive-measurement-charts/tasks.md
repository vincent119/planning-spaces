# 任務文件：CareNest 可互動量測趨勢圖

Status: Complete

## Execution Context

- 意圖：讓照護者可讀取最近七天體重與血壓的座標趨勢，並以手勢查看原始數值。
- 非目標：醫療判讀、後端、資料庫修改、第三方圖表套件。
- 邊界：量測頁趨勢元件與既有量測 Widget 測試。
- 完成條件：座標軸、選取提示、拖曳互動、資料不足與既有量測操作均通過驗證。

### Protected Behavior

- 僅呈現原始體重、收縮壓與舒張壓；不判讀健康狀態。
- 不變更量測資料的新增、修改、刪除與被照顧者資料隔離。
- 不新增網路、帳號、同步或雲端資料。

## 實作任務

- [x] T1：定義座標與選取資料模型
  - Status: Complete
  - Boundary:
    - Allowed Changes: `care_measurements_page.dart`、量測趨勢測試
    - Forbidden: 資料庫 schema、Controller 資料寫入規則
  - Depends: 既有 `CareDayMeasurements`
  - Context: 依日期建立可選取索引，計算含留白的動態 Y 軸範圍與刻度。
  - Verify: 多日體重與雙血壓資料可產生軸刻度與原始提示文字。

- [x] T2：繪製座標軸與可互動折線圖
  - Status: Complete
  - Boundary:
    - Allowed Changes: `care_measurements_page.dart`
    - Forbidden: 外部圖表套件、醫療判讀
  - Depends: T1
  - Context: CustomPainter 顯示 X／Y 軸、格線、線條、點與選取垂線；GestureDetector 將手勢對應日期索引。
  - Verify: 點擊與水平拖曳後顯示最接近日期及對應數值。

- [x] T3：回歸測試與 Android 驗證
  - Status: Complete
  - Boundary:
    - Allowed Changes: `measurement_trend_test.dart`、範圍內修正與任務文件
    - Forbidden: 範圍外功能與資料刪除
  - Depends: T1、T2
  - Context: 確保資料不足不繪圖，原始值與既有新增、修改、刪除呈現不變。
  - Verify: `dart format`、`flutter analyze`、`flutter test`、`flutter build apk --profile`、`git diff --check`、Android 手動點擊與拖曳。

## 品質檢查清單

- 軸與提示具日期、單位與文字意義，不僅依賴顏色。
- 體重與血壓資料點均能選取；血壓提示同時呈現兩個原始數值。
- 少於兩筆資料時不顯示不可靠的趨勢。
- 沒有醫療判讀詞彙。

## Implementation Notes

- 2026-08-20：使用者要求體重與血壓圖補上座標軸，並可動態選取座標點顯示數值。
- 2026-08-20：完成深墨藍圖表底、青綠與暖珊瑚資料線、淡薄荷選取定位線；`flutter analyze`、完整測試、profile APK 建置與 Android 安裝均已通過。
