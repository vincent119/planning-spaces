# 任務文件：CareNest Android 啟動體驗

Status: Complete

## Execution Context

- 意圖：移除 Android 啟動時的黑底淺色方塊感，讓系統與 Flutter 載入過場一致。
- 非目標：iOS、桌面圖示、新套件、資料初始化內容與後端。
- 已定決策：系統階段顯示深墨綠背景與小型原始過渡標誌；Flutter 顯示原始透明標誌與「正在開啟 CareNest」，不使用旋轉環。
- 邊界：Android 啟動資源、`lib/app.dart`、資產宣告與測試。
- 關鍵檔案：`styles.xml`、`launch_background.xml`、`lib/app.dart`、`test/widget_test.dart`。
- 完成條件：系統與 Flutter 過場一致、驗收測試與 Android 實機冷啟動通過。

### Protected Behavior

- 不修改桌面 App Icon、Android Manifest launcher icon 參照、iOS 或任何照護資料。
- 不修改 `CareNestController.initialize()` 的資料讀取、錯誤與重試行為；只允許調整首次呼叫時機。
- 不新增 package、網路請求或追蹤。

## 實作任務

- [x] T1：建立 Android 系統啟動畫面資源與離場動畫
  - Status: Complete
  - Boundary:
    - Allowed Changes: Android values、drawable、透明啟動標誌、`MainActivity.kt`
    - Forbidden: Manifest、桌面圖示、iOS、Dart 初始化邏輯與新相依套件
  - Depends: 已確認啟動畫面視覺方向
  - Context: API 31 使用 SplashScreen；舊版使用既有 LaunchTheme 窗口背景
  - Verify: 資產存在、APK 建置、Android 冷啟動檢查與系統畫面直接移除

- [x] T2：建立 Flutter 品牌化載入頁與測試
  - Status: Complete
  - Boundary:
    - Allowed Changes: `lib/app.dart`、資產宣告、首次初始化呼叫時機、相關 Widget 測試
    - Forbidden: Controller、資料庫、一般功能頁與新套件
  - Depends: T1
  - Context: 使用相同背景色，不顯示通用全頁轉圈圈；第一個 Flutter 畫面可見 500ms 後才開始既有初始化，並以 240ms 淡入切換後續頁面
  - Verify: 載入頁至少維持 499ms 的 Widget 測試、核心 widget 回歸

- [x] T3：整合驗證
  - Status: Complete
  - Boundary:
    - Allowed Changes: 驗證、任務文件與範圍內修正
    - Forbidden: 新功能與範圍外重構
  - Depends: T1、T2
  - Context: 保留現有 Android 資料的安全更新方式
  - Verify: `dart format`、`flutter analyze`、`flutter test`、`flutter build apk --debug`、`git diff --check`、Android 冷啟動

## 品質檢查清單

- 系統與 Flutter 階段均為深墨綠背景。
- 系統啟動畫面不使用滿版桌面圖示。
- 載入頁文字清楚且不以顏色作為唯一訊息。
- 未修改照護資料與既有初始化行為。

## Implementation Notes

- 2026-08-18：依 Android 實機畫面建立本規格並開始 T1。
- 2026-08-18：建立 432 × 432、透明背景的啟動標誌；Android API 31 使用 `windowSplashScreenBackground` 與 `windowSplashScreenAnimatedIcon`，舊版 LaunchTheme 背景亦固定為深墨綠。冷啟動截圖確認黑底淺色方塊已消失，改為深墨綠背景與透明標誌。
- 2026-08-18：Flutter `_LoadingPage` 改為同色背景、標誌、開啟文字與小型進度指示；新增初始載入 Widget 驗收。`flutter analyze`、21 項 `flutter test`、`flutter build apk --debug`、`git diff --check` 通過。以 `adb install -r` 成功更新實體 Android，保留既有本機照護資料。曾測試 288px 系統標誌，但冷啟動擷取落在 Android 離場縮放階段，比例不穩；已恢復 432px 的穩定版本。
- 2026-08-18：使用者要求為第一次載入的大型系統標誌加入過場動畫，重新開啟 T1 與 T3。範圍限定 Android 12 系統 SplashScreen 離場，不加入循環動畫或延長資料初始化。
- 2026-08-18：`MainActivity` 在 Android 12 以上設定系統 SplashScreen 離場監聽：標誌於 280ms 內縮小至 88% 並淡出，系統停用 Animator 時直接移除。經兩次 Kotlin 型別修正後，`flutter build apk --debug` 與 `git diff --check` 通過；新版以 `adb install -r` 更新 Android 並保留本機資料。完整 `flutter analyze` 與 21 項 `flutter test` 已於同次變更前通過。T3 等待實機動態手感確認。
- 2026-08-18：使用者確認保留 Android 系統啟動階段的小型標誌；完成實機方向決策並結束 T3。
- 2026-08-18：使用者要求大型載入標誌改為循環呼吸。系統 SplashScreen 保留小型標誌，Flutter 載入頁負責循環動畫，避免原生系統 SplashScreen 因無限動畫而阻擋離場。
- 2026-08-18：Flutter 載入標誌改為 1400ms 循環呼吸，縮放範圍 96% 至 104%，透明度範圍 82% 至 100%；系統減少動態設定時停止動畫。載入頁測試加入呼吸縮放驗收。`flutter analyze`、21 項 `flutter test`、`flutter build apk --debug`、`git diff --check` 通過，並以 `adb install -r` 更新實體 Android 且保留本機資料。
- 2026-08-18：新錄影確認 Flutter 載入頁在系統 SplashScreen 離場後才可見，循環呼吸無法在使用者看到的等待階段呈現。改用 Android API 31 `AnimatedVectorDrawable` 讓系統標誌以 700ms、92% 至 108% 範圍循環呼吸，重新開啟 T1 與 T3。
- 2026-08-18：使用者提供參考影片並指定中央固定圖示與外圍旋轉環的載入語言。Android 與 Flutter 載入頁改為固定 CareNest 標誌、薄荷綠圓弧 900ms 線性旋轉；不使用參考 App 的多色環，也不增加啟動等待時間。
- 2026-08-18：`dart format`、`flutter analyze`、21 項 `flutter test`、`flutter build apk --debug` 與 `git diff --check` 通過。新版已使用 `adb install -r` 安裝到 Android 實機，保留既有本機照護資料；等待使用者進行冷啟動動態確認。
- 2026-08-18：實機錄影確認系統 SplashScreen 的自動遮罩裁掉原本位於外緣的圓弧，只留下中央標誌。圓弧半徑已由 45 至 47 縮小至 32、筆畫加粗為 5；中央標誌縮小至 72% 並將筆畫加粗為 9，使所有元素留在系統安全區內。
- 2026-08-18：使用者回報中央圖示變形。根本原因是系統旋轉環版本以手寫向量重畫原始標誌，無法保證圖形一致。已移除該向量與動畫資源，Android 系統 SplashScreen 改回既有透明 PNG 原始標誌；Flutter 載入頁的旋轉環不受影響。後續若需原生動畫，必須採用不重繪標誌的素材策略。
- 2026-08-18：使用者確認原生啟動階段需要可見旋轉環。改採 12 格 PNG 動畫：每格均保留相同原始透明標誌，只改變外圍兩段薄荷綠圓弧的位置，總週期為 900ms。這避免 Android 向量資源重繪品牌圖形而變形。
- 2026-08-18：使用者確認旋轉環視覺不符合期待，要求還原為深墨綠背景、原始圖示與「正在開啟 CareNest」的乾淨版本。已移除原生與 Flutter 的旋轉環參照，回到靜態載入畫面。
- 2026-08-18：使用者回報實機只看見 Android 系統 SplashScreen，未看見 Flutter 的開啟文字，且系統遮罩令圖示比例不同。將既有初始化改為 Flutter 首幀後 700ms 才啟動，確保靜態載入頁可見；不更動初始化資料、錯誤與重試行為。重新開啟 T2 與 T3 驗證。
- 2026-08-18：載入頁 Widget 測試明確驗證 699ms 後仍顯示原始圖示與開啟文字；`dart format`、`flutter analyze`、21 項 `flutter test`、`flutter build apk --debug` 與 `git diff --check` 通過。新版已用 `adb install -r` 安裝至 Android 實機並保留既有照護資料；T3 等待冷啟動人工確認。
- 2026-08-18：使用者要求不展示 Android 系統大圖示，直接展示第二階段 Flutter 載入頁。Android API 31 改用透明系統 icon，系統 SplashScreen 首幀後直接移除；不再讓系統遮罩縮放 CareNest 圖示。
- 2026-08-18：`dart format`、`flutter analyze`、21 項 `flutter test`、`flutter build apk --debug` 與 `git diff --check` 通過。新版已以 `adb install -r` 更新 Android 實機並保留照護資料；等待冷啟動人工確認系統大圖示不再出現。
- 2026-08-18：使用者確認空白系統階段等待感過長，改採小型原生過渡標誌。素材以原始 432 × 432 PNG 置中於 720 × 720 透明畫布，僅增加留白、不重繪圖示；Flutter 畫面仍維持原始比例與開啟文字。
- 2026-08-18：使用者確認 1.8 秒精簡視覺預覽方向。Flutter 載入頁等待時間由 700ms 縮短為 500ms；載入頁切入後續頁面採 240ms 淡入。開啟文字維持與圖示同一水平中心線，重新開啟 T2 與 T3 驗證。
- 2026-08-18：載入頁 Widget 測試已調整為 499ms 可見驗收。`dart format`、`flutter analyze`、21 項 `flutter test`、`flutter build apk --debug` 與 `git diff --check` 通過；新版已用 `adb install -r` 安裝至 Android 實機並保留照護資料。T3 等待冷啟動人工確認。
- 2026-08-18：使用者已在 Android 實機確認小型原生過渡圖示、居中開啟文字與淡入首頁的節奏正常；T1 至 T3 完成。
