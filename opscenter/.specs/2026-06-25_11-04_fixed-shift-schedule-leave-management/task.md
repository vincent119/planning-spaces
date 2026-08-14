# 三班固定制排班與假勤管理任務

## 1. 文件與資料模型

- [x] 1.1 確認三班固定制範圍
  - 不包含三班輪班制
  - 不包含四班二輪
  - 不包含 On-call
  - _Requirements: 範圍_

- [x] 1.2 建立 migration
  - `shift_groups`
  - `duty_shifts`
  - `user_shift_assignments`
  - `shift_periods`
  - `user_schedules`
  - `user_schedule_history`
  - `shift_period_confirmations`
  - _Design: 資料模型_

- [x] 1.3 建立 seed
  - `3shift_fixed`
  - 早班
  - 中班
  - 晚班
  - _Requirements: 1.1_

- [x] 1.4 建立變形工時設定
  - `schedule.labor_law_mode`
  - 支援 `4week`
  - 支援 `8week`
  - 建立週期時寫入 mode 快照
  - _Requirements: 1.7_

- [x] 1.5 建立早中晚班最小值班人數設定
  - Global Setting key：`schedule.min_staff_per_core_shift`
  - 預設值 `1`
  - 最小值 `1`
  - 僅 `admin` / `op_admin` 可調整
  - 完成：新增 `0039_seed_schedule_min_staff_setting.sql` seed 設定與 DB 約束；Global Setting service 對此 key 強制 `system` 類別、`number` 型別與正整數值
  - 驗證：`go test ./internal/setting` 通過
  - _Requirements: 1.8; Design: 系統設定_

## 2. 班別與人員歸屬後端

- [x] 2.1 實作班別查詢 API
  - `GET /api/v1/schedule/shift-groups`
  - `GET /api/v1/schedule/duty-shifts`
  - _Requirements: 1.1_

- [x] 2.2 實作班別設定 API
  - `POST /api/v1/schedule/duty-shifts`
  - `PUT /api/v1/schedule/duty-shifts/:id`
  - `PATCH /api/v1/schedule/duty-shifts/:id/enabled`
  - _Requirements: 1.1_

- [x] 2.3 實作人員班別歸屬查詢
  - 依 user
  - 依日期
  - 顯示歷史區間
  - _Requirements: 1.2, 1.4_

- [x] 2.4 實作人員轉班流程
  - 不允許覆蓋既有 `duty_shift_id`
  - 截止舊歸屬區間
  - 新增新歸屬區間
  - 驗證歸屬區間不重疊
  - _Requirements: 1.3, 1.5, 1.6_

- [x] 2.5 實作 report 用班別解析查詢
  - `ResolveUserDutyShift(user_id, event_date)`
  - 依事件日期查當時有效歸屬
  - 禁止使用目前班別回填歷史
  - _Requirements: 9.3, 9.4_

## 3. 排班週期後端

- [x] 3.1 實作建立排班週期
  - 建立 `shift_periods`
  - 預設週期名稱
  - 日期範圍驗證
  - 四週模式預設 28 天
  - 八週模式預設 56 天
  - _Requirements: 1.7, 2.1, 2.2, 2.3_

- [x] 3.2 實作週期自動展開初稿
  - 依有效人員班別歸屬展開
  - `day_type = workday`
  - `source = auto`
  - 回傳 skipped users
  - draft 週期可補缺重建，不覆蓋既有手動編輯
  - _Requirements: 2.4, 2.5, 2.6_

- [x] 3.3 實作週期列表與詳情
  - `GET /api/v1/schedule/periods`
  - `GET /api/v1/schedule/periods/:id`
  - `POST /api/v1/schedule/periods/:id/refresh-draft`
  - _Requirements: 3.1_

- [x] 3.4 實作排班矩陣查詢
  - 日期欄
  - 早 / 中 / 晚分組
  - 人員列
  - cell
  - 統計與檢查結果
  - _Requirements: 3.1, 5.1, 5.2, 5.3_

- [x] 3.5 實作單格更新
  - 更新 `day_type`
  - 更新備註
  - 工程師只能改自己的列
  - 值班組長可改全部
  - _Requirements: 3.2, 3.3, 3.4_

- [x] 3.6 實作週期狀態限制
  - draft 可編輯
  - confirmed / locked 不可編輯
  - _Requirements: 3.6, 6.2, 6.3_

- [x] 3.7 修正週期矩陣管理者預覽與重疊週期補缺
  - `admin` / `op_admin` 檢視週期矩陣時，不得因自己不在排班名單而顯示工程師排休警告
  - `user_schedules` 的唯一鍵需以 `period_id + user_id + schedule_date` 為範圍，避免不同週期的同日排班互相擋住
  - draft 週期重新整理補缺需具備冪等性，不得因同週期既有格子造成錯誤或空矩陣
  - 人員班別歸屬需統一使用 `3shift_fixed`，不得再寫入舊 `3shift`
  - 補測試或驗證覆蓋管理者預覽與排班補缺
  - 完成：新增 `0037_scope_user_schedule_unique_index.sql` 將週期排班唯一鍵改為 period-scoped；補缺 insert 加上同週期 conflict ignore；新增 `0038_align_assignments_to_fixed_shift_group.sql` 將舊 `3shift` 歸屬搬到 `3shift_fixed`；前端管理者不再顯示工程師不在排班名單警告；人員班別頁與後端轉班 API 限制使用三班固定制
  - 驗證：`go test ./internal/schedule`、`go test ./...`、`npm run typecheck`、`npm run build` 通過
  - _Requirements: 2.4, 3.1, 3.2, 3.3_

- [x] 3.8 實作草稿週期刪除 API
  - 新增 `DELETE /api/v1/schedule/periods/:id`
  - 僅允許 `admin` / `op_admin` 操作
  - 僅允許 `draft` 狀態刪除，`confirmed` / `locked` 必須拒絕
  - 刪除週期時同步清除該週期排班資料並寫入審計紀錄
  - 完成：後端新增刪除週期 service、repository transaction 與 HTTP route；刪除前寫入 `user_schedule_history`，刪除後寫入週期審計，非草稿回 409
  - 驗證：`go test ./internal/schedule`、`go test ./...` 通過
  - _Requirements: 3.1, 6.2, 7.2_

## 4. 驗證與審計

- [x] 4.1 實作同班同日休例假衝突檢查
  - `statutory_holiday`
  - `day_off`
  - 違反回傳 409
  - _Requirements: 4.1, 4.4_

- [x] 4.2 實作一週一例檢查
  - 依 `4week` / `8week` 模式輸出檢查結果
  - 第一階段保留每 7 天區間缺例假提示
  - 矩陣顯示結果
  - 確認週期前阻擋缺例假
  - _Requirements: 1.7, 4.2, 4.3, 5.3_

- [x] 4.3 實作排班異動審計
  - INSERT
  - UPDATE
  - DELETE
  - before / after snapshot
  - _Requirements: 7.1_

- [x] 4.4 實作週期操作審計
  - 建立
  - 確認
  - 鎖定
  - _Requirements: 7.2_

- [x] 4.5 以最小值班人數取代同班同日休例假互斥規則
  - 舊規則不再以「同班同日已有其他人排例假 / 休假」阻擋
  - 僅套用 `morning` / `afternoon` / `night`
  - 以更新後 `workday` 人數判斷是否低於 `schedule.min_staff_per_core_shift`
  - 非 `workday` 包含例假、休假、國定假、年假、公假、病假、事假、其他
  - 單格儲存與確認週期都需檢查
  - 違反時回傳 409 與繁中可診斷訊息
  - 完成：後端移除舊同班同日休例假互斥檢查，改由 `workday` 人數與 Global Setting 最小人數判斷；單格更新與確認週期皆套用早 / 中 / 晚核心班別檢查；409 回傳具體日期、班別、需求人數與更新後人數
  - 驗證：`go test ./internal/schedule`、`go test ./internal/setting ./internal/schedule`、`go test ./...` 通過
  - _Requirements: 4.1, 4.4_

- [x] 4.6 預留排班驗證規則分層
  - `staffing_rule`
  - `labor_rule`
  - `single_cell_update` / `refresh_draft` / `confirm_period`
  - `block` / `warning` / `info`
  - 第一階段不補完整勞基法規則引擎
  - 完成：新增排班驗證分層型別與 `ValidationIssue` / `ValidationResult`；單格更新走 `staffing_rule + single_cell_update + block`；確認週期走 `staffing_rule` 與既有 `labor_rule` 的 block 檢查；保留 `refresh_draft`、`warning`、`info` 作為後續擴充點
  - 驗證：`go test ./internal/schedule` 通過
  - _Requirements: 4.1, 4.2, 4.3, 4.4; Design: 規則引擎分層_

## 5. 確認、鎖定與匯出

- [x] 5.1 實作週期確認
  - `POST /api/v1/schedule/periods/:id/confirm`
  - 記錄確認人與時間
  - 空週期不得確認
  - 一週一例未通過時拒絕
  - _Requirements: 6.1, 6.2_

- [x] 5.2 實作週期鎖定
  - `POST /api/v1/schedule/periods/:id/lock`
  - Admin / SRE 經理可操作
  - _Requirements: 6.4_

- [x] 5.3 實作 CSV 匯出
  - 日期列
  - 星期列
  - 人員列
  - 統計欄
  - 彙總列
  - _Requirements: 8.2, 8.3, 8.4_

- [x] 5.4 實作 Excel 匯出
  - 格式對齊 Google Sheet
  - 組長簽名欄
  - _Requirements: 8.1, 8.3, 8.4_

## 6. 前端基礎

- [x] 6.1 新增 schedule i18n namespace
  - 繁體中文
  - 簡體中文
  - 英文
  - day type 文案
  - _Requirements: 3.5_

- [x] 6.2 新增側邊欄節點
  - 排班週期
  - 人員班別
  - 班別設定
  - _Design: 導覽與權限_

- [x] 6.3 建立 schedule API client
  - periods
  - matrix
  - assignments
  - duty shifts
  - export
  - _Design: API 對接_

## 7. 前端頁面

- [x] 7.1 建立排班週期列表
  - 篩選
  - 狀態
  - 匯出入口
  - _Design: SchedulePeriodList_

- [x] 7.2 建立排班週期新增頁
  - 日期區間
  - 四週 / 八週變形工時模式
  - 依模式自動帶入 28 / 56 天
  - 自動產生週期名稱
  - 展開結果摘要
  - skipped users
  - _Requirements: 1.7, 2.3; Design: SchedulePeriodCreateForm_

- [x] 7.3 建立排班矩陣頁
  - 早 / 中 / 晚分組
  - 日期欄
  - 人員列
  - 統計欄
  - 底部彙總
  - _Design: ScheduleMatrix_

- [x] 7.4 實作單格 day type 編輯
  - 工程師只能改自己的列
  - 值班組長可改全部
  - confirmed / locked 唯讀
  - _Requirements: 3.2, 3.3, 3.6_

- [x] 7.5 建立人員班別歸屬管理頁
  - 時間軸顯示歷史
  - 新增班別歸屬入口
  - 新增轉班流程
  - 顯示不影響歷史報表提示
  - _Requirements: 1.5, 9.4_

- [x] 7.6 建立班別設定頁
  - 早 / 中 / 晚
  - 起訖時間
  - 跨午夜
  - 啟用狀態
  - _Requirements: 1.1_

- [x] 7.7 補齊矩陣頁三態顯示與狀態操作
  - 矩陣頁標題區需顯示 `draft`、`confirmed`、`locked` 三種狀態
  - `draft` 顯示確認週期操作
  - `confirmed` 顯示鎖定週期操作
  - `locked` 不顯示可變更狀態操作
  - 完成：矩陣頁標題區已加入狀態 chip，`draft` 顯示確認週期，`confirmed` 顯示鎖定週期，`locked` 僅可檢視與匯出
  - 驗證：`npm run typecheck`、`npm run build` 通過
  - _Requirements: 6.2, 6.3, 6.4; Design: ScheduleMatrix_

- [x] 7.8 補齊確認週期檢查提示
  - 矩陣頁在 `draft` 且例假檢查未通過時，需顯示缺例假的人員
  - 確認週期按鈕需在缺例假或無排班 rows 時阻擋送出
  - 後端確認週期 409 不得只回 `schedule conflict`，需回傳可診斷的繁中訊息
  - 完成：矩陣頁會列出缺例假的人員並停用確認週期按鈕；無排班 rows 時也會提示先重新整理；後端確認週期 409 改回繁中診斷訊息
  - 驗證：`go test ./internal/schedule`、`npm run typecheck`、`npm run build` 通過
  - _Requirements: 4.2, 4.3, 6.2; Design: ScheduleMatrix_

- [x] 7.9 排班週期列表刪除操作
  - 操作欄在「查看」前方顯示「刪除」按鈕
  - `draft` 可刪除，其他狀態顯示停用狀態
  - 無 `admin` / `op_admin` 權限者顯示停用狀態
  - 破壞性刪除操作需二次確認
  - 完成：列表操作欄已加入刪除按鈕與停用原因提示，送出前會二次確認，刪除成功後重新載入週期列表
  - 驗證：`npm run typecheck`、`npm run build` 通過
  - _Requirements: 3.1, 6.2; Design: SchedulePeriodList_

- [x] 7.10 排班矩陣最小值班人數提示與選項停用
  - Matrix response 讀取 `staffing_policy`
  - 早班 / 中班 / 晚班群組列顯示最小值班人數
  - day type 選單預先 disabled 會造成最小值班人數不足的非 `workday` 選項
  - disabled 選項需顯示停用原因
  - 後端 409 需顯示日期、班別與需求人數
  - 完成：矩陣 API 回傳 `staffing_policy`；前端群組列顯示早 / 中 / 晚最小值班人數；假別選單依目前矩陣資料預先停用會低於最小值班人數的非 `workday` 選項，並顯示停用原因與停用儲存按鈕
  - 驗證：`go test ./internal/schedule`、`go test ./...`、`npm run typecheck`、`npm run build` 通過
  - _Requirements: 4.1, 4.4, 4.5; Design: ScheduleMatrix, DayTypeSelect_

- [x] 7.11 強化排班格假別色彩與主題相容
  - 排班矩陣每格依 `day_type` 與核心班別顯示不同主題色
  - 深色 / 淺色主題需使用相容語意色票與透明度計算，不使用突兀固定色
  - 出勤格保持低對比，假別格需比出勤更高對比
  - 唯讀但已有排班的格子仍需保留假別辨識度
  - 完成：矩陣按鈕依出勤、例假、休假、國定假、年假、公假、病假、事假、其他套用主題相容色彩，假別使用較高對比色票，並補上 tooltip 與未排班 i18n
  - 驗證：`npm run typecheck`、`npm run build` 通過
  - _Requirements: 3.4, 3.5; Design: ScheduleMatrix_

- [x] 7.12 修正排班衝突 fallback 訊息繁中化
  - schedule 409 若沒有具體 `ConflictError` 訊息，不得回傳 `schedule conflict`
  - 共用 fallback 改為繁中，避免中文介面 toast 顯示英文
  - 完成：`handleError` 的 schedule conflict fallback 改為「排班衝突，請重新整理後再試」，並補 delivery 測試防回歸
  - 驗證：`go test ./internal/schedule` 通過
  - _Requirements: 3.5; Design: ErrorHandling_

- [x] 7.13 人員班別歸屬表格排序
  - 人員班別歸屬管理頁的「人員」與「班別」表頭需可點擊排序
  - 排序互動需與 Ticket 列表 DataGrid 一致
  - 預設依人員名稱升冪顯示
  - 班別排序需依三班固定制業務順序：早班、中班、晚班
  - 排序只作用在目前篩選後的列表資料
  - 完成：人員班別歸屬表格改用與 Ticket 列表一致的 DataGrid；預設人員升冪，班別排序使用 `morning`、`afternoon`、`night` 業務順序
  - 驗證：`npm run typecheck`、`npm run build` 通過
  - _Requirements: 1.2, 1.4; Design: 人員班別歸屬管理_

- [x] 7.14 排班週期列表 DataGrid 化
  - 排班週期列表需使用與 Ticket 列表一致的 DataGrid
  - 保留週期名稱、模式、日期範圍、狀態與操作欄
  - 操作欄需保留查看、匯出 CSV、匯出 Excel、鎖定與刪除
  - 刪除停用原因與草稿限制需維持既有行為
  - 完成：週期列表改為 DataGrid，保留既有篩選與操作欄行為，狀態排序使用草稿、已確認、已鎖定固定順序
  - 驗證：`npm run typecheck`、`npm run build` 通過
  - _Requirements: 3.1, 6.2, 6.4; Design: SchedulePeriodList_

- [x] 7.15 班別設定列表 DataGrid 化
  - 班別設定列表需使用與 Ticket 列表一致的 DataGrid
  - 保留班別代碼、名稱、開始時間、結束時間、跨午夜、排序、狀態與操作欄
  - 操作欄需保留編輯與啟用 / 停用
  - 預設依排序欄升冪顯示
  - 完成：班別設定列表改為 DataGrid，保留編輯與啟用 / 停用操作，預設依 `sort_order` 升冪顯示
  - 驗證：`npm run typecheck`、`npm run build` 通過
  - _Requirements: 1.1; Design: 班別設定_

## 8. 後續追蹤

- [ ] 8.1 三班輪班制
- [ ] 8.2 四班二輪
- [ ] 8.3 On-call 時間窗口排班
- [x] 8.4 自動國定假日 API
  - 建立 `public_holidays` 與 `public_holiday_sync_runs` migration，支援 manual / api source、provider、raw payload、soft delete 與同步摘要
  - 補 Global Setting：`schedule.public_holiday_source = manual | api`、`schedule.public_holiday_provider = data_gov_tw_123662 | taiwan_calendar_json`
  - 建立官方資料 adapter：主要 provider 為 `data_gov_tw_123662`，對應 `https://data.gov.tw/dataset/123662`
  - 建立 JSON 備援 adapter：`taiwan_calendar_json`，對應 `https://github.com/ruyut/TaiwanCalendar` / `https://cdn.jsdelivr.net/gh/ruyut/TaiwanCalendar/data/{year}.json`
  - 兩個 adapter 都需將外部日期格式轉為 Asia/Taipei 業務日期，並統一輸出內部 `PublicHolidayCandidate`
  - 實作 `GET/POST/PUT/DELETE /api/v1/schedule/public-holidays`
  - 實作 `POST /api/v1/schedule/public-holidays/sync`，支援 preview / apply，manual override 不得被 API 靜默覆蓋
  - 已完成版本曾規劃建立週期與重新整理草稿時自動套用 `public_holidays`；此行為已被 8.10 / 8.11 新規則取代，後續不再作為目標行為
  - Matrix response 增加 `calendar_marks`，日期欄需直接顯示國定假名稱並以高對比欄底色區分，但不代表該日所有格子已排定 `public_holiday`
  - 前端於系統管理新增 `/admin/public-holidays` 管理頁，包含年度篩選、同步 preview / apply、手動新增 / 編輯
  - 週期矩陣不提供手動「套用國定假」入口，國定假日欄位需以高對比欄底色標示
  - 同步與手動維護需寫入審計紀錄；自動改格與套用週期審計由 8.10 / 8.11 移除或調整
  - _Requirements: 10.1-10.8; Design: 國定假日同步與標記流程, 國定假日管理_
- [ ] 8.5 人事系統請假資料匯入
- [ ] 8.6 完整勞基法規則引擎
- [ ] 8.7 請假審批流程
- [x] 8.8 Ticket 列表與報表依班別查詢整合
  - Ticket 列表新增 `duty_shift_id` 查詢參數，依 Ticket 建立時間解析建立者當時有效班別
  - Ticket 列表回傳建立者當時班別，前端列表提供班別篩選與欄位顯示
  - Report `ticket_events` 新增 `shift` 維度與篩選，依 `person_basis` 使用 Ticket 建立者或活動操作者在事件時間的有效班別
  - 每日值班執行統計改用排班歸屬解析出的班別，不再以事件時間時段推估
  - 完成：後端 Ticket 列表 scope 與 response 已整合班別；Report V2 metadata / planner / facts 查詢已支援 `shift`；前端 Ticket 列表與 Report 設計器已加入班別篩選與顯示
  - 修正：每日值班統計矩陣改為依「統計群組 → 班別 → 人員」階層輸出，並合併同一班別同姓名的人員列，避免畫面出現同名重複列或跨群組錯位
  - 驗證：`go test ./internal/ticket ./internal/report ./internal/schedule`、`go test ./...`、`npm run typecheck`、`npm run build` 通過
  - _Requirements: 9.3, 9.4_

- [x] 8.9 現行值班班制 Global Setting 與班別選項收斂
  - 新增 Global Setting key：`schedule.current_shift_group_code`
  - 預設值為 `3shift_fixed`
  - 新增 seed / migration，並驗證 value 必須對應啟用中的 `shift_groups.code`
  - 新增後端 resolver：由 `schedule.current_shift_group_code` 解析目前現行值班班制與啟用班別
  - 新增 `GET /api/v1/schedule/current-shift-group`，回傳現行班制與其 `duty_shifts`
  - Ticket 列表班別下拉改用 current shift group endpoint，不得再抓全部啟用班別
  - Report 設計器與班別篩選選項改用 current shift group endpoint
  - Ticket 收到 `duty_shift_id` 篩選時需驗證該班別屬於現行值班班制；Report V2 現行 contract 為 `shift` 名稱 filter，本次不改 request schema，改由 current shift group endpoint 收斂可選值
  - 設定缺失或無效時前端顯示可診斷錯誤並停用班別下拉，不得顯示全部班別
  - 完成：新增 setting seed、setting 更新參照驗證、schedule current shift group resolver/API、Ticket 後端 `duty_shift_id` 現行班制驗證、Ticket / Report / Schedule 前端班別選項收斂
  - 驗證：`go test ./internal/setting ./internal/schedule ./internal/ticket`、`npm run typecheck`、`git diff --check` 通過
  - _Requirements: 1.9, 9.5; Design: 系統設定, Ticket / Report 班別查詢整合_

- [x] 8.10 後端：排班初稿全工作日與國例休餘額規則調整
  - 建立週期與重新整理草稿時，週期內每一天都維持 `day_type = workday`，包含週六、週日與特殊國定假日
  - 移除建立 / 重新整理草稿時自動把 `public_holidays` 套用成 `public_holiday` 格子的行為
  - 官方行事曆 adapter 需區分普通週六 / 週日與特殊國定假日；普通週末由 Matrix `columns.is_weekend` 標示，特殊國定假日進入 `calendar_marks`，是否納入 `國` 應排餘額以後端餘額規則判斷
  - 週檢查區間以週期起日每 7 天切分；每人每個區間必須剛好 1 天 `statutory_holiday`，同一區間第二個例假需被前後端阻擋
  - `day_off` 可在週期內移動，確認週期時檢查每人已排休假數與週期應排休假數一致
  - Matrix response 需回傳每人 `public_holiday`、`statutory_holiday`、`day_off` 的 `target_count`、`used_count`、`remaining_count`
  - 單格更新需回傳可診斷的 409 / warning，包含例假超排、缺例假、國 / 例 / 休餘額與最小值班人數衝突
  - 確認週期需重新檢查國 / 例 / 休餘額，任一人餘額未補齊或超排時阻擋確認
  - 2026-09-21 至 2026-10-04 範例需驗證每人應排 `例 = 2`、`休 = 2`，且特殊國定假日為 2 天時 `國 = 2`
  - 完成：建立 / 重整初稿不再自動套用國定假日；Matrix response 回傳 `columns.is_weekend`、`weekly_check.over_quota_weeks` 與個人 `leave_balances`；單格更新阻擋同週第二個例假與國 / 例 / 休超排；確認週期重新檢查最小值班人數、每週例假與國 / 例 / 休餘額
  - 驗證：`go test ./internal/schedule` 通過
  - _Requirements: 2.7, 4.2, 4.3, 4.6, 5.4, 5.5, 10.6, 10.7, 10.8; Design: 週期初稿展開, 四週 / 八週變形工時例假與休假檢查, 國例休餘額, 國定假日同步與標記流程_

- [x] 8.11 前端：排班矩陣週末 / 國假標示、國例休餘額顯示與選項停用
  - 排班矩陣右側或底部統計需顯示個人尚未排定的 `國`、`例`、`休` 餘額
  - 日期欄需對週六 / 週日與特殊國定假日顯示高對比欄底色；特殊國定假日需顯示節日名稱，例如「中秋節」
  - 若週六 / 週日同時是特殊國定假日，日期標題需優先顯示節日名稱；是否納入 `國` 應排餘額以後端 `calendar_marks.is_balance_date` 判斷；普通週末只顯示週末欄底色，不納入 `國` 餘額
  - 初稿格子仍顯示早 / 中 / 晚工作日，不因週六、週日或國定假日標記自動顯示 `休`、`例`、`國`
  - day type 選單需停用會造成同一人同一週檢查區間第二個例假的 `statutory_holiday` 選項，並顯示停用原因
  - day type 選單仍需停用會造成核心班別低於最小值班人數的非 `workday` 選項
  - `day_off` UI 不預設綁定週六或週日，工程師可在週期內移動日期
  - 409 / warning 需顯示繁中可診斷訊息，包含例假超排、缺例假、餘額未補齊與最小值班人數不足
  - 2026-09-21 至 2026-10-04 範例需驗證畫面顯示每人 `例 = 2`、`休 = 2`、`國 = 2` 的未排餘額，排定後餘額遞減
  - 完成：前端 Matrix 型別接上 `columns.is_weekend`、`weekly_check.over_quota_weeks` 與 `summary.leave_balances`；日期欄依週末 / 特殊國定假日顯示不同底色與節日名稱；右側與底部統計顯示國 / 例 / 休未排餘額；假別選單停用同週第二個例假、餘額超排與最小值班人數不足選項；確認週期按鈕會在例假檢查或餘額未符合時停用
  - 驗證：`npm run typecheck` 通過
  - _Requirements: 2.7, 4.2, 4.3, 4.5, 4.6, 5.4, 5.5, 10.7; Design: ScheduleMatrix, DayTypeSelect, 排班矩陣視覺規則, 前端錯誤處理_

- [x] 8.12 國定假日與例假 / 休假重疊餘額抵扣修正
  - 週日特殊國定假日只作日期欄節日標示，不納入 `public_holiday.target_count`
  - 週日特殊國定假日日期若排為 `statutory_holiday`，只抵扣 `statutory_holiday.used_count = 1`，不抵扣 `public_holiday.used_count`
  - `calendar_marks.is_balance_date = true` 的特殊國定假日日期若排為 `statutory_holiday` 或 `day_off`，需同時抵扣 `public_holiday.used_count = 1` 與原本假別 used count
  - 特殊國定假日日期若排為 `public_holiday`，`public_holiday.used_count` 同日只計 1 天，不得重覆計算
  - 前端餘額提示與假別停用規則需依上述重疊抵扣預估，不要求使用者把特殊國定假日同一天同時改成兩種 day type
  - 驗證案例：2026-10-15 至 2026-11-11 期間若有 2026-10-25 週日特殊國定假日與 2026-10-26 補假，初始未排餘額需顯示 `國 1`、`例 4`、`休 4`
  - 完成：後端 `leave_balances.public_holiday.target_count` 會排除週日特殊國定假日；計入國假餘額的特殊國定假日當天若排為 `statutory_holiday` / `day_off`，同日抵扣 1 天 `public_holiday`；前端 day type 選單用相同規則預估餘額，避免錯誤停用或要求同日重覆選 `public_holiday`
  - 驗證：`go test ./internal/schedule`、`npm run typecheck` 通過
  - _Requirements: 5.4, 5.6, 10.7; Design: 國例休餘額, ScheduleMatrix, DayTypeSelect_

- [x] 8.13 補假來源日不重覆計入國定假餘額
  - 若官方行事曆 API 的補假日 raw payload / description 註明其補自另一個週六國定假日，補假日納入 `public_holiday.target_count`，原週六國定假日只作日期欄標記
  - 例：2026-10-09 為 2026-10-10 國慶日補假時，2026-10-09 計入 `國`，2026-10-10 不計入 `國`
  - 原週六國定假日若被排為 `day_off` 或 `statutory_holiday`，不得抵扣 `public_holiday.used_count`
  - Matrix response 需回傳每個 `calendar_marks` 是否計入 `public_holiday` 餘額，前端不得只用「非週日」自行推算
  - 後端單格更新需阻擋在非 `public_holiday` 餘額日設定 `public_holiday`
  - 前端假別選單停用規則需使用後端回傳的國假餘額日期判斷
  - 完成：後端 Matrix 會回傳 `calendar_marks.is_balance_date`；若補假日描述 / raw payload 指向週六國定假日，補假日計入 `國`，來源週六只作日期標記；後端單格更新阻擋非餘額日設定 `public_holiday`；前端假別選單改用 `is_balance_date` 判斷餘額與停用「國」
  - 驗證：`go test ./internal/schedule`、`npm run typecheck`、`git diff --check` 通過
  - _Requirements: 5.6, 10.7; Design: 國例休餘額, ScheduleMatrix, DayTypeSelect_

- [x] 8.14 補假來源日國例休餘額驗收案例修正
  - 針對 2026-09-17 至 2026-10-14 四週週期建立回歸驗收
  - 若期間 `calendar_marks` 有 4 筆特殊國定假日標記，其中 1 筆為已有補假日的原始國定假日，該原始日不得計入 `public_holiday.target_count`
  - 每人初始未排餘額需顯示 `國 3`、`例 4`、`休 4`
  - 補假來源日不得增加 `day_off.target_count`；四週仍為 `休 4`，八週仍為 `休 8`
  - 補假來源日若被排為 `day_off` 或 `statutory_holiday`，只抵扣 `休` 或 `例` 已排數，不抵扣 `國` 已排數
  - 後端 `calendar_marks.is_balance_date`、`leave_balances.public_holiday.target_count`、`leave_balances.public_holiday.used_count` 需一致
  - 前端餘額顯示與 day type 停用規則需完全依後端 `is_balance_date`，不得自行用週末或節日名稱推算
  - 完成：後端補上「補假描述不完整時，鄰近週六國定假日視為補假來源日」判斷；新增 2026-09-17 至 2026-10-14 回歸測試，確認初始餘額為 `國 3`、`例 4`、`休 4`；Matrix response 測試確認 2026-10-10 `is_balance_date = false`
  - 驗證：`go test ./internal/schedule`、`npm run typecheck`、`git diff --check` 通過
  - _Requirements: 5.6, 5.7, 10.7; Design: 國例休餘額, ScheduleMatrix, DayTypeSelect_

- [x] 8.15 Excel 公式相容的國例休與一週一例檢核顯示
  - 文件來源：`2026-四周變形工時班表_OP組.xlsx` 既有公式與使用者回報截圖
  - 一週一例檢查需以週期起日每 7 天切分，逐段等同 Excel `COUNTIF(區間, "*例*") = 1`
  - 既有系統已能統計 `國`、`例`、`休` 已排數；本項重點是修正右側欄位顯示公式
  - 國 / 例 / 休顯示值需對齊 Excel：若已排數大於應排數顯示 `錯`，否則顯示 `已排數 - 應排數`
  - 前端右側統計欄、底部合計與警示訊息需顯示 Excel 相容檢核值，不得直接顯示 API `remaining_count` 正負號
  - 後端確認週期 409 診斷訊息需使用相同 `國 / 例 / 休` 檢核值，避免與畫面公式不一致
  - 回歸案例：2026-09-17 至 2026-10-14 週期若某人已排 `國 3`、`例 4`、`休 3` 且目標為 `國 3`、`例 4`、`休 4`，畫面與訊息需顯示 `0 / 0 / -1`
  - 回歸案例：同週期若某人已排 `休 5` 且休目標為 4，畫面與訊息需顯示 `休 錯`
  - 完成：前端右側個人欄、底部合計與 warning 改顯示 Excel 相容檢核值；後端確認週期 409 訊息改用相同口徑；API `remaining_count` 維持 `target_count - used_count`
  - 驗證：`go test ./internal/schedule`、`npm run typecheck` 通過
  - _Requirements: 4.2, 4.6, 5.3, 5.6; Design: 四週 / 八週變形工時例假與休假檢查, 國例休餘額, ScheduleMatrix_

- [x] 8.16 國定假與休假週期內挪移及精準計數
  - Status: Complete
  - Depends: 8.15
  - Context: 行事曆國假資料只決定每人國假應排數；實際已排數依格子短名精準計算，例假維持一週一例，休假與國定假可在週期內挪移
  - Boundary: Allowed Changes 為排班需求與設計文件、後端 schedule 驗證 / 餘額計算 / 測試、前端 ScheduleMatrix 選項判斷 / 顯示 / 測試；Forbidden 為資料庫 schema、國定假日同步與目標數排除規則、排班以外模組
  - 規則來源：使用者回報 2026-09-17 至 2026-10-14 排班週期中，2026-10-08 應可依額度選擇國定假或例假
  - 本項覆蓋 8.12、8.13、8.14 中「國只能排在 `calendar_marks.is_balance_date = true` 日期」及「例 / 休可自動抵扣國」的既有行為；已完成項目保留為歷史紀錄，不回改完成狀態
  - `calendar_marks.is_balance_date` 只計算 `public_holiday.target_count` 與提供日期欄標記，不限制 `public_holiday` 可排日期
  - `public_holiday` 與 `day_off` 均可在整個週期內移動；`public_holiday.used_count` 只計算短名為「國」的格子，`day_off.used_count` 只計算短名為「休」的格子
  - `statutory_holiday` 維持以週期起日每 7 天切分，每個區間必須剛好 1 天；同一區間第二個例假仍需前後端阻擋
  - `day_off` 不限制每週 1 天，只在週期確認時檢查總已排數等於目標數
  - `public_holiday` 排在一般工作日仍須套用核心班別最小值班人數檢查；前後端不得因選項為「國」而略過
  - 前端需移除非國假餘額日停用「國」的條件，並以個人國假總額度、最小值班人數決定選項是否可用
  - 後端需移除非國假餘額日設定 `public_holiday` 的阻擋，改以週期總額度與最小值班人數驗證
  - 驗收案例：2026-10-08 不是國假餘額日，個人仍有國假額度且班別人數足夠時，可設定為 `public_holiday`
  - 驗收案例：2026-10-08 所在 7 天區間尚未有例假時可設定為 `statutory_holiday`；若該區間已有 1 天例假則維持阻擋
  - 驗收案例：4 週週期的 `day_off.target_count = 4`，4 天可分布於任意週；少於、超過 4 天均不得確認週期
  - Verify: `go test ./internal/schedule`、`npm run typecheck`、`git diff --check`
  - Implementation Notes: 後端移除非國假餘額日設定 `public_holiday` 的限制，`public_holiday.used_count` 改為精準計算所有短名為「國」的格子；前端移除非餘額日停用「國」並同步使用精準額度預估。真正國假餘額日維持既有人力豁免，挪到一般日期的「國」則套用最小值班人數。新增 2026-10-08 可挪移與人力不足阻擋回歸案例；`go test ./internal/project ./internal/server ./internal/schedule`、`npm run typecheck`、`git diff --check` 通過
  - _Requirements: 4.1-4.7, 5.4-5.7; Design: 四週 / 八週變形工時例假與休假檢查, 國例休餘額, ScheduleMatrix, DayTypeSelect_

- [x] 8.17 前端國例休欠額改為非負數顯示
  - Status: Complete
  - Depends: 8.15
  - Context: 8.15 已完成 Excel 公式相容顯示，但負數不利於使用者理解；前端改以 `缺 N` 表示尚未排足，Excel 匯出與 API 數值 contract 不變
  - Boundary: Allowed Changes 為排班需求與設計文件、前端 ScheduleMatrix 國例休顯示與 locale、會呈現在前端的後端確認週期診斷訊息、相關測試；Forbidden 為 API 數值欄位 schema、Excel 匯出公式、國例休目標數與已排數計算規則
  - 本項覆蓋 8.15 中「前端顯示已排數減應排數」的行為；8.15 保留為歷史紀錄，不回改完成狀態
  - 前端右側個人統計、底部合計與 warning 在欠額時顯示 `缺 N`，不得顯示 `-N`
  - 已排數等於應排數時顯示 `0`；已排數超過應排數時維持顯示 `錯`
  - 後端確認週期的診斷訊息若會直接呈現在前端，也需顯示 `缺 N`，避免 toast 再出現負數
  - API `remaining_count` 維持 `target_count - used_count`；Excel 匯出維持既有 `COUNTIF - 目標數` 公式與負數結果
  - 驗收案例：目標 `國 3、例 4、休 4`，已排 `國 1、例 4、休 3` 時，前端顯示 `國 缺 2、例 0、休 缺 1`
  - 驗收案例：休假已排 5 天且目標為 4 時，前端顯示 `休 錯`
  - Verify: `go test ./internal/schedule`、`npm run typecheck`、`git diff --check`
  - Implementation Notes: 前端共用顯示 helper 已改為欠額顯示 `缺 N`、相等顯示 `0`、超排顯示語系化的 `錯`，右側個人統計、底部合計、tooltip 與 warning 全部套用；繁中、簡中、英文語系已補齊。後端確認週期診斷訊息同步改為人類可讀口徑，API `remaining_count` 與 Excel 匯出未修改；`go test ./internal/schedule`、`go test ./internal/server`、`npm run typecheck`、`git diff --check` 通過
  - _Requirements: 5.4, 5.8; Design: 國例休餘額, ScheduleMatrix, 前端錯誤處理_
