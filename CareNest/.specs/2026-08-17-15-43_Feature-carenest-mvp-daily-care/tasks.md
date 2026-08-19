# CareNest MVP：每日照護確認實作任務

Status: Planned

Implementation: Do not execute

> 停止：本任務清單已被拆分規格取代，不得執行 T0 至 T9。請先閱讀同目錄 `roadmap.md`。

## Execution Context

意圖：建立一個 Local-first、Offline-first 的 Flutter MVP，讓一位主要照顧者確認一位長輩當日的餐食、高蛋白補充與飲水照護狀態，並可記錄體重、血壓、提醒、回顧與本機備份。

非目標：醫療判讀、藥品管理、帳號、家庭同步、多人照護、雲端 API、食物辨識與自動營養估算。

已定決策：以 Flutter、SQLite 與 Drift 為基礎；照護日與計畫版本不可回寫；完成度只代表照護確認；所有資料預設存於裝置；私有規格位於 `$PRIVATE_SPEC_WORKSPACE/CareNest/.specs/`。

關鍵文件：`requirements.md`、`design.md`、Flutter 專案根目錄的 `AGENTS.md`、`pubspec.yaml`、`README.md`。

完成條件：完成 S1 至 S10 的驗收情境、`flutter analyze` 無新增問題、相關 `flutter test` 通過、核心流程能離線完成、且無醫療或用藥管理功能。

## Protected Behavior

- 不新增遠端 API、登入、雲端同步、分析追蹤或敏感資料外傳。
- 不以數值提供醫療建議、異常判定或風險分級。
- 不將 `.specs/` 寫入程式 repository；規格更新只能位於私有規格工作區。
- 不為未確認需求建立多位被照顧者、家庭成員或權限模型。
- 不用固定醫療數值當作飲水、高蛋白、體重或血壓目標。
- 不修改使用者既有的無關工作樹變更。

## 任務

### T0：確認產品假設與發版範圍

Status: Planned

Boundary:

- Allowed Changes：私有規格中的本 spec 文件、研究訪談紀錄與可用性測試結果。
- Forbidden：Flutter 原始碼、相依套件、產品實作。

Depends: 無。

Context: A1 至 A5 尚需以目標使用者驗證，特別是餐食摘要的輸入成本與單一長輩限制。

Work:

1. 訪談 5 至 8 位主要照顧者，蒐集一天中照護確認、餐食交接與補充品紀錄的真實語句。
2. 以低保真流程測試首次設定、首頁、餐食、高蛋白與飲水紀錄。
3. 量測首頁判讀是否小於 5 秒、主要紀錄是否小於 10 秒。
4. 將結果回填需求假設，對未通過項目提出明確變更提案。

Verify: 研究紀錄完整；確認 A1 至 A5 為接受、調整或拒絕；使用者確認 MVP 範圍。

### T1：建立 Flutter 基礎專案與品質閘門

Status: Planned

Boundary:

- Allowed Changes：`pubspec.yaml`、`analysis_options.yaml`、`lib/main.dart`、`lib/app.dart`、`test/`、專案 README。
- Forbidden：功能頁面、外部 API、任何雲端認證或同步設定。

Depends: T0。

Context: 專案尚不存在。新增套件前必須依 Flutter `AGENTS.md` 說明用途、替代方案與維護影響。

Work:

1. 建立最小 Flutter iOS／Android 專案。
2. 設定 lint、格式化、測試與本機開發說明。
3. 建立主題、路由入口與通用錯誤／空狀態容器，但不預先建立多層架構。
4. 新增最小冒煙測試，確認應用程式能啟動。

Verify: `dart format`、`flutter analyze`、`flutter test` 與 `git diff --check` 通過。

### T2：建立本機資料庫、照護日與計畫版本

Status: Planned

Boundary:

- Allowed Changes：`lib/core/database/**`、`lib/core/time/**`、`lib/features/care_plan/**`、相關測試、`pubspec.yaml`。
- Forbidden：遠端資料來源、完成度介面、通知與備份實作。

Depends: T1。

Context: Drift 資料表與計畫版本規則依 `design.md` 的資料契約；`careDay` 一經寫入不可因後續設定重算。

Work:

1. 建立 CareRecipient、CarePlan、CarePlanItem 與所有紀錄資料表、遷移與 DAO。
2. 實作單一 `CareDay` 計算器，處理時區與使用者日界線。
3. 實作以 `effectiveFromCareDay` 建立新計畫版本的流程，包含可修改的三餐預計時間與 08:00、12:00、18:00 預設值。
4. 建立首次設定與修改未來計畫的資料操作，包含三餐、高蛋白、飲水預設啟用，以及體重、血壓預設關閉的項目選擇。

Verify: 計畫生效日、日界線、單一對象限制、首次設定預設項目、目標必填規則與資料遷移單元測試通過；S1c、S1d、S6 的資料層測試通過。

### T3：實作完成度讀模型與首頁資料來源

Status: Planned

Boundary:

- Allowed Changes：`lib/features/dashboard/**`、必要 DAO、完成度純 Dart 規則、相關測試。
- Forbidden：醫療閾值、風險分數、用藥項目、遠端資料快取。

Depends: T2。

Context: 完成度只含三餐、高蛋白與飲水；體重、血壓不得進入分子或分母。

Work:

1. 實作依照護日、有效計畫與紀錄產生的完成度讀模型與待處理事項優先順序。
2. 為新增、修改、刪除紀錄提供一致的快取失效或重算機制。
3. 建立首頁資料狀態，以待確認事項作為第一優先，再顯示分子、分母、百分比、各項狀態與累積數值。
4. 實作未設定、無紀錄、部分完成與全部完成的呈現狀態。

Verify: 完成度公式、未啟用項目、計畫版本、刪除後重算與待處理事項排序的單元測試通過；S1、S1a、S2a、S10 通過。

### T4：實作餐食紀錄與本機相片處理

Status: Planned

Boundary:

- Allowed Changes：`lib/features/meals/**`、相片私有檔案服務、相關資料表與測試、必要權限設定。
- Forbidden：影像辨識、營養估算、公開相簿寫入、雲端相片上傳。

Depends: T2、T3。

Context: 有吃時必須保留餐食摘要與五段食量；未吃也是完成照護確認，但不得被呈現成進食成功。

Work:

1. 建立餐食新增、檢視、修改、刪除流程，以及首頁的有吃、未吃、補充詳細內容快速確認入口。
2. 建立自由文字摘要輸入與最近使用摘要快速選取；最近清單依使用時間去重並最多顯示 8 筆，不建立固定食物分類或營養資料庫。
3. 實作照片選取、私有目錄複製、替換、刪除與遺失檔案處理。
4. 餐食資料變更後立即更新首頁完成度。

Verify: S2、S2b、S2c、S2d、S3；餐食輸入驗證、最近摘要去重、取消不產生紀錄、照片縮圖與明細可見性、照片清理、歷史日編輯與首頁即時更新測試通過。

### T5：實作高蛋白補充與飲水快速紀錄

Status: Planned

Boundary:

- Allowed Changes：`lib/features/supplements/**`、`lib/features/water/**`、必要 DAO 與測試。
- Forbidden：自動熱量／蛋白質推估、醫療目標建議、雲端產品目錄。

Depends: T2、T3。

Context: 補充品營養資料只能來自使用者建立的常用品項；飲水快速量固定含 100、200、300 ml。

Work:

1. 建立常用補充品、預設品項設定、首頁一鍵新增一份，以及新增、修改、刪除補充紀錄流程。
2. 建立首頁 `+100`、`+200`、`+300` ml 的立即累加，以及「其他水量」自訂 ml 與補登時間流程。
3. 於每個異動後重算當日累積、差額與完成度。
4. 處理取消、輸入錯誤與歷史日期補登。

Verify: S4、S4a、S5、S5a；數量邊界、預設品項缺漏、累積、補登、修改、刪除與離線測試通過。

### T6：實作體重、血壓與 7 天回顧

Status: Planned

Boundary:

- Allowed Changes：`lib/features/measurements/**`、`lib/features/history/**`、圖表或資料呈現套件設定、相關測試。
- Forbidden：異常判斷、醫療建議、風險通知、完成度規則修改。

Depends: T2、T3、T4、T5。

Context: 體重與血壓只做數值輸入驗證與描述性回顧；7 天回顧須顯示原始資料可追溯性。

Work:

1. 建立體重與血壓的新增、修改、刪除與歷史查閱。
2. 建立最近 7 天每日完成度與各類紀錄摘要。
3. 實作描述性摘要，只描述是否已記錄與使用者設定目標的達成狀況。
4. 確保空資料、資料不足與遺失照片不會造成頁面失敗。

Verify: S1b；7 天日期範圍、摘要與日期明細分層、空資料、數據排序與無醫療判讀的 Widget 測試通過。

### T7：實作本機提醒

Status: Planned

Boundary:

- Allowed Changes：`lib/core/notifications/**`、`lib/features/reminders/**`、平台通知設定、相關測試與文件。
- Forbidden：遠端推播、背景資料上傳、用藥提醒、未經設定的權限請求。

Depends: T2、T3。

Context: 通知權限僅在使用者設定第一個提醒時請求；提醒被拒絕不得阻止核心功能。

Work:

1. 建立提醒設定、權限狀態與本機排程服務。
2. 依照護日、時區與計畫變更重新排程有效提醒。
3. 實作通知權限拒絕、系統關閉與排程失敗的可恢復狀態。
4. 確保提醒文案是中性照護確認，無醫療或責備語意。

Verify: S8；提醒新增、停用、時區變更、計畫變更與權限拒絕測試通過。

### T8：實作加密備份、還原與資料刪除

Status: Planned

Boundary:

- Allowed Changes：`lib/core/backup/**`、`lib/features/settings/**`、檔案選取與加密相關設定、相關測試與文件。
- Forbidden：雲端備份、自動外傳、未加密明文備份、未確認的資料取代。

Depends: T2、T4、T5、T6。

Context: 還原必須先驗證；取代前必須建立安全備份；加密與金鑰策略需在實作前記錄並經使用者確認。

Work:

1. 先提出加密格式、金鑰保存、檔案選取與平台限制的決策紀錄，獲確認後再實作。
2. 建立含資料與相片的版本化加密備份。
3. 實作備份檢查、摘要、合併還原與取代還原。
4. 實作個別紀錄與被照顧者資料的刪除、二次確認與相片清理。

Verify: S9、S10；格式錯誤、密碼或解密失敗、取消還原、合併衝突、取代失敗回復與刪除清理測試通過。

### T9：整合驗證、可用性測試與 MVP 發布準備

Status: Planned

Boundary:

- Allowed Changes：測試、文件、必要的小型修正、私有規格的驗證結果。
- Forbidden：新增未規劃功能、架構重寫、同步或 AI 功能。

Depends: T3、T4、T5、T6、T7、T8。

Context: 只在核心流程可靠、離線可用且無醫療越界時才進入 MVP 發布準備。

Work:

1. 執行 S1 至 S10 的整合測試與手動驗證腳本。
2. 在飛航模式下驗證首次設定後的所有核心紀錄、回顧與提醒設定。
3. 以至少 5 位目標使用者進行可用性測試，記錄 5 秒與 10 秒指標。
4. 檢查產品用語、錯誤訊息、空狀態與說明沒有醫療建議或用藥管理內容。
5. 驗證備份還原、資料刪除、相片清理與敏感資料日誌。

Verify: `dart format`、`flutter analyze`、`flutter test`、整合測試、`git diff --check` 全數通過；使用者驗收結果回填私有規格。

## 品質檢查清單

- [ ] 所有新增 Dart 程式碼通過格式化與靜態分析。
- [ ] 完成度演算法、計畫版本與日界線有單元測試。
- [ ] 高頻紀錄操作與首頁更新有 Widget 或整合測試。
- [ ] 離線、空資料、失敗與取消流程均經驗證。
- [ ] 不存在 API key、帳密、健康資料或照片路徑的敏感日誌。
- [ ] 不存在醫療判讀、固定醫療目標、用藥管理、雲端同步或外傳。
- [ ] 實際修改檔案均在各任務 Boundary 的 Allowed Changes 內。
- [ ] 每個完成任務後執行 `git diff --stat` 與 `git diff --check`。

## Implementation Notes

- 2026-08-17：正式規格建立；尚未開始任何 Flutter 實作。
- 餐食摘要的具體輸入方式受 A1 可用性驗證約束；不可在未驗證下大量預設餐點或建立營養分類系統。
- 備份加密的實作決策必須先獨立記錄與確認，避免臨時選用不適當的金鑰或格式。
