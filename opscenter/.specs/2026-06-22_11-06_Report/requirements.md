# Report Requirements

## Introduction

Report 模組提供運維資料的查詢、圖表化、範本化與匯出能力，用於取代人工整理週報、月報與專案運維統計。Report 只負責讀取與彙總既有業務資料，不反向修改 Ticket、Project、Schedule 或 Jira 資料。

本文件從舊規格 `.kiro/specs/2026-06-01_10-22_oncall-ticket-system` 的需求 14、Ticket 報表相關段落與前後端設計抽出，作為後續 Report 實作 task 的唯一需求來源。

## Scope

### MVP

建立 Report 中心頁、內建月報 A / B / C 查詢、值班統計矩陣、報表預覽、CSV 匯出與專案範圍權限檢查。MVP 不實作完整拖拉式報表設計器，但需保留模式 D 的 API 與資料結構擴充空間。

### Phase 2

提供有限維度的自訂查詢、範本 CRUD、範本執行與更多統計口徑。

### Phase 3

提供完整報表設計器、範本版本管理、排程產報、非同步大型匯出與更多圖表版型。

## Requirements

### 需求 1：報表入口與專案範圍

**使用者故事**：身為專案成員，我希望能在所屬專案中查看報表，快速了解運維處理量與趨勢。

#### 驗收條件

- [ ] 1.1 Report 入口位於 `/projects/:id/reports`，並由選單樹 `reports` 或專案內報表入口導向。
- [ ] 1.2 報表查詢必須限定專案範圍；私有專案僅專案成員可查詢，公開專案依 Project 可見性規則允許讀取。
- [ ] 1.3 使用者需具備對 Report 表單節點的 `read` 權限，且符合專案可見規則，才可查看報表中心與執行查詢。
- [ ] 1.4 報表不得顯示使用者無權存取的專案、子專案、Ticket 或人員資料。
- [ ] 1.5 API 查無資料時回傳空資料 payload，不得用假資料或前端硬編資料顯示服務正常。
- [ ] 1.6 Report 中心需提供主專案下拉選單，使用者只能切換到自己可見的主專案；切換後導向該主專案的 `/projects/:id/reports`。
- [ ] 1.7 Report 中心需提供子專案下拉選單，資料來源限定目前主專案下的可見子專案；子專案預設為空值。
- [ ] 1.8 Report 查詢若未選擇子專案，需以目前主專案範圍查詢；若選擇子專案，查詢範圍需限定該子專案。
- [ ] 1.9 Report 主專案與子專案下拉選單需沿用 Ticket 工作區專案資料來源與顯示格式，不得另外硬編專案或子專案清單。

### 需求 2：報表資料來源與時間規則

**使用者故事**：身為值班組長，我希望報表統計來源清楚且時間口徑一致，避免月報數字與 Ticket 資料不一致。

#### 驗收條件

- [ ] 2.1 Ticket 統計資料來源為 `tickets`、`ticket_activities`、`ticket_types`、`ticket_resources`、`projects`、`sub_projects` 與專案成員資料。
- [ ] 2.2 事件類型需引用 `ticket_types`，不得直接使用顯示文字統計。
- [ ] 2.3 資訊來源需引用 `ticket_resources`，不得使用已廢棄的 `ticket_sources`。
- [ ] 2.4 子專案統計需引用 `sub_projects`，子專案已停用或已刪除時仍需能依歷史 Ticket 顯示快照或保留名稱。
- [ ] 2.5 人員統計口徑至少支援 `created_by` 與 `actor`；`created_by` 依 Ticket 建立人統計，`actor` 依 `ticket_activities.actor_id` 統計。
- [ ] 2.6 班別統計若啟用，需依操作人員與時間查詢排班資料推導；Ticket 不新增班別快照欄位。
- [ ] 2.7 報表日期區間以 `Asia/Taipei` 計算日、週、月邊界，再轉 UTC 查詢資料庫。
- [ ] 2.8 日期區間採閉區間日期語意：`date_from` 與 `date_to` 皆為 `YYYY-MM-DD`，後端轉換為 `[date_from 00:00:00, date_to + 1 day 00:00:00)`。

### 需求 3：內建月報 A / B / C 與值班統計

**使用者故事**：身為值班組長，我希望能快速產出固定格式月報，不需要每次重新設計維度。

#### 驗收條件

- [ ] 3.1 月報標題格式預設為 `YYYY年MMDD-MMDD運維處理事件數量`，以 `Asia/Taipei` 顯示。
- [ ] 3.2 模式 A 為指標導向月報：橫軸為週區間與總計欄，每個指標獨立一張圖，縱軸為人員，並提供明細數字表。
- [ ] 3.3 模式 A 指標至少包含 Ticket 事件類型、資訊來源、子專案三類統計拆分；後續可擴充 Jira 指標但不得混用 Ticket 資料表。
- [ ] 3.3a 模式 A 必須支援三個固定 OP 月報指標：`op_jira_issue_count`（OP Jira 開單數量統計）、`op_alert_notification_count`（OP 告警通知數量統計）、`op_payment_domain_change_count`（OP 支付or業務項目域名更換統計）。
- [ ] 3.3b 模式 A 固定 OP 月報指標呈現格式為上方柱狀圖、下方人員 × 週區間明細表；橫軸人員需包含班別前綴，例如 `早-Chenmin`、`中-Eric`、`晚-Turry`。
- [ ] 3.3c 模式 A 週區間欄位需依月內週段產生，例如 `6/1-6/4`、`6/5-6/11`、`6/12-6/18`、`6/19-6/25`、`6/26-6/30`，並提供 `總計` 欄。
- [ ] 3.4 模式 B 為任務導向月報：縱軸為 Ticket 標題或任務內容，橫軸為人員，呈現堆疊長條圖與交叉明細表。
- [ ] 3.5 模式 C 為人員 × 子專案堆疊月報：整月彙總為單一圖表，橫軸為人員，堆疊區段為子專案或專案內業務項目。
- [ ] 3.6 模式 C 需支援 `person_basis = created_by` 與 `person_basis = actor`。
- [ ] 3.7 內建月報 API 需回傳統一 `ReportChartPayload`，前端不得自行補算核心統計數字。
- [x] 3.8 值班統計為 Excel 矩陣導向報表，橫軸為每日日期，縱軸為指標群組與人員；內部模式維持 `E`，API 維持 `/reports/daily-shift-execution`。
- [x] 3.9 報表模式僅顯示「值班統計」，選取後由「數值」欄位選擇告警通知、域名更換或支付域名更換；每次預覽與匯出只送出單一 `metric_group`，不得由前端填假資料。
- [x] 3.10 「值班統計－告警通知」、「值班統計－域名更換」與「值班統計－支付域名更換」畫面不顯示早班、中班、晚班的獨立班別彙總列，但保留含班別前綴的人員明細列；後端既有 shift row 契約維持相容。
- [x] 3.11 值班統計需支援 CSV / Excel-friendly 匯出，保留既有日期、群組、班別、人員與數值矩陣契約；本次畫面隱藏班別彙總列不改變匯出契約。
- [x] 3.12 上述三張值班統計矩陣需在第一個日期欄前顯示「總計」，其值為該列在目前查詢日期區間內所有日期數值的合計；指標與個別人員列皆適用。
- [x] 3.13 顯示名稱統一為「值班統計－告警通知」、「值班統計－域名更換」與「值班統計－支付域名更換」，不得顯示舊名稱「每日值班統計」；此調整不需資料庫 migration。

### 需求 4：自訂報表與範本

**使用者故事**：身為值班組長，我希望能把常用報表設定存成範本，之後選日期即可重複執行。

#### 驗收條件

- [ ] 4.1 報表範本為專案層級資料，儲存在 `report_templates`。
- [ ] 4.2 同專案成員可查看與執行範本；建立、修改、刪除範本需 Project Manager 以上或具備對應 Report `create/update/delete` 權限。
- [ ] 4.3 範本需包含名稱、描述、報表模式、維度、指標、圖表類型、統計口徑與標題模板。
- [ ] 4.4 模式 D 為自訂維度報表，需支援橫軸、縱軸、指標、圖表類型與時間區間設定。
- [ ] 4.5 報表預覽不需先儲存範本；使用者可直接送出 config 預覽。
- [ ] 4.6 範本執行時使用者需選擇日期區間，支援本週、上週、本月、上月與自訂區間。
- [ ] 4.7 範本新增、修改、刪除與執行失敗需顯示 Toast；成功不得在後端確認前先顯示。

### 需求 5：圖表、表格與匯出

**使用者故事**：身為專案成員，我希望報表能同時以圖表與表格呈現，並可匯出 CSV 交付或留存。

#### 驗收條件

- [ ] 5.1 報表支援長條圖與堆疊長條圖。
- [ ] 5.2 圖表下方需提供與圖表一致的明細表，欄位與數字以後端 payload 為準。
- [ ] 5.3 CSV 匯出所有具 Report `read` 權限且可見專案的成員皆可操作。
- [ ] 5.4 CSV 匯出需使用目前查詢條件，不得匯出超出畫面權限或專案範圍的資料。
- [ ] 5.5 大型匯出後續可改成非同步 Job；同步匯出超過限制時需回傳明確錯誤，不得逾時後顯示成功。
- [ ] 5.6 圖表與表格文字需完整接 i18n，包含空資料、載入、錯誤、欄位、圖例、匯出按鈕與 Data Grid 內建文字。

### 需求 6：審計、效能與錯誤處理

**使用者故事**：身為系統管理員，我希望報表操作可追蹤且查詢效能可控，避免報表拖慢核心 Ticket 流程。

#### 驗收條件

- [ ] 6.1 範本新增、修改、刪除需寫入系統審計日誌。
- [ ] 6.2 報表查詢錯誤需記錄 request id、project id、report mode、date range 與操作者，不得記錄敏感 token。
- [ ] 6.3 一般報表查詢 P95 目標小於 3 秒；超過限制需提供可診斷日誌。
- [ ] 6.4 後端需驗證所有維度、指標與排序欄位白名單，禁止將前端字串直接拼入 SQL。
- [ ] 6.5 報表查詢需使用參數化 SQL 或安全 query builder。
- [ ] 6.6 報表 API 失敗時前端顯示錯誤狀態與重試操作，不得顯示假圖表或假資料。

### 需求 7：模式 D 自定義報表品質強化

#### 文件定位

本需求接續需求 4、5、6 與既有 `ReportDesignerPage`、V2 config、preview、template、CSV export 實作，來源為 2026-08-03 前端審查結果。此需求修正既有模式 D 的功能落差與品質風險，不重寫模式 A / B / C、值班統計、既有 Report 權限模型或核心聚合邏輯。

#### 背景

目前模式 D 已能選擇資料集、日期、維度、指標、篩選、排序與圖表類型，但預覽仍被範本名稱驗證綁定；`show_table`、純表格圖型及圖例換行沒有完整反映到畫面；V2 config 缺少標題模板；metadata 與錯誤顯示未完整支援 i18n；同步匯出以 GET query 傳送完整 config，且 CSV 文字欄位缺少公式注入防護。

#### 目標

- 讓查詢預覽、儲存範本與匯出具有各自正確的驗證狀態。
- 讓使用者選擇的表格、圖例、標題與圖表設定在預覽和範本執行時一致生效。
- 改善 i18n、鍵盤焦點、圖表色彩與大量資料保護。
- 建立可驗證的 POST 匯出與 CSV 安全契約。

#### 非目標

- 不實作任意拖拉式 BI 畫布、公式編輯器或跨資料集 join。
- 不新增新的核心統計指標，也不由前端重算後端統計結果。
- 不在本需求實作非同步大型匯出 Job；僅保留後續擴充介面。
- 不改變 `reports` read / create / update / delete 與專案可見性規則。

#### 驗收條件

- [x] 7.1 預覽只驗證查詢必要欄位，不要求範本名稱；儲存範本才驗證名稱，匯出則驗證目前查詢條件與有效預覽狀態。
- [x] 7.2 使用者修改任何會影響 payload 的欄位後，舊預覽需明確標示為過期，且舊 response 不得被誤標為目前設定的成功預覽。
- [x] 7.3 V2 config 支援選填 `title_template`；只允許既定文字變數，不接受 HTML、JavaScript 或任意模板執行語法。第一階段允許 `{{project_name}}`、`{{date_from}}`、`{{date_to}}`。
- [x] 7.4 `visualization.show_table = true` 時預覽與範本執行需顯示後端 `table.columns` / `table.rows`；為 false 時可隱藏明細表；`chart_type = table` 時必須顯示實際表格，不得只顯示筆數。
- [x] 7.5 `visualization.legend.wrap` 必須實際控制圖例行為；系列超過 6 個時預設使用可捲動圖例或明確限制，不得讓圖表高度無上限增加。
- [x] 7.6 metadata 以穩定 code 作為前端 i18n key；前端先查 `report.json` 翻譯，未知 code 才退回 API label。API label 不得成為唯一語系來源。
- [x] 7.7 API validation error 必須以白名單 type guard 解析；未知錯誤使用通用 i18n 文案，不得把任意 `Error.message` 拼成翻譯 key。
- [x] 7.8 Data Grid 與所有互動控制需保留可見的 `:focus-visible` 樣式，並支援鍵盤完成預覽、儲存、匯出與篩選操作。
- [x] 7.9 固定業務時區日期快捷需在不同瀏覽器時區得到相同 `Asia/Taipei` 日期區間，不得以瀏覽器本地午夜推導月份邊界。
- [x] 7.10 自定義預覽最多回傳 50 個 category；畫面不提供「不限」預覽。完整資料只能透過受限制的匯出流程取得。
- [x] 7.11 日期維度優先建議折線圖；人員、子專案等排名維度優先建議長條圖；建議不強制覆寫使用者仍有效的選擇。
- [x] 7.12 類別圖表使用穩定、色弱友善的 categorical palette；同一 series key 在重新排序或篩選後應維持同色，不使用 success / error 紅綠語意色區分類別。
- [x] 7.13 自定義 config 匯出改用 `POST /api/v1/projects/{id}/reports/export` JSON request body；GET export 僅在相容期保留舊版簡單查詢，不再承載完整 V2 config。
- [x] 7.14 CSV 匯出需在後端序列化邊界防止 Formula Injection；文字欄位首個非空白字元為 `=`、`+`、`-`、`@` 時需安全轉義，並保留原始可讀文字。
- [x] 7.15 Report API response 在前端邊界需進行 runtime schema 驗證；單筆異常 template、metadata、chart payload 或 table row 不得使整個 Report 畫面崩潰。
- [x] 7.16 自定義篩選的列舉值需由 metadata 或獨立 options API 提供；使用者不應被要求猜測 status、priority、ticket type 等內部 code。

#### 驗收情境

場景：未命名直接預覽
測試：待建立，`ReportDesignerPage` component test
假設：查詢欄位有效且範本名稱為空
當：使用者按下「預覽」
那麼：系統送出 preview request 並呈現結果，但「儲存範本」維持停用且顯示名稱必填原因

場景：表格與圖例設定生效
測試：待建立，`ReportDesignerPage`、`ReportChartPanel` component test
假設：preview payload 同時包含 chart 與 table
當：使用者切換顯示表格、純表格圖型與圖例換行
那麼：畫面依 config 顯示實際明細表及對應圖例行為

場景：跨時區日期一致
測試：待建立，`reportDateRange` unit test
假設：系統日期相同
當：測試分別以 UTC-08:00、UTC、UTC+14:00 執行本月與上月快捷
那麼：送出的 `date_from`、`date_to` 皆符合 `Asia/Taipei` 同一日期區間

場景：安全匯出
測試：待建立，Report handler 與 CSV serializer test
假設：config 包含多個篩選值，資料欄位包含 `=SUM(1,1)`
當：使用者匯出 CSV
那麼：config 只出現在 POST body，CSV 儲存格安全轉義，且權限與專案範圍仍由後端驗證

#### 驗證需求

- 前端需新增日期、驗證狀態、i18n fallback、runtime schema、圖表選項與表格 row id 測試。
- 後端需新增 POST export、V2 title template、metadata options 與 CSV 公式字元測試。
- 必須執行 `npm run typecheck`、`npm run build`、相關前端測試、`go test ./internal/report`、`go test ./...`、`make openapi` 與 `git diff --check`。

### 需求 8：V3 多區塊彈性報表範本

#### 文件定位

本需求接續需求 4 的自訂報表範本與需求 7 的模式 D 品質強化，將既有單一查詢、單一視覺化的 `ReportTemplateConfigV2` 擴充為多區塊 `ReportTemplateConfigV3`。既有 V1、V2 範本、模式 A / B / C / E、Report 權限與專案範圍均為受保護行為，不原地改寫既有範本。

#### 背景

目前自定義範本一次只能呈現一張圖表與一份明細表，無法在同一份報表組合總覽指標、趨勢、分布與明細。使用者需要可拖拉、縮放、複製及自動重排的區塊式設計，同時必須維持手機可讀性、鍵盤操作、查詢上限與後端權限驗證。

#### 目標

- 一份範本可包含指標卡、圖表、明細表與純文字說明等多個區塊。
- 桌面使用 12 欄格線拖拉與縮放，碰撞時自動向下挪移，不允許區塊互相覆蓋。
- 平板與手機由桌面配置自動重排，第一階段不要求使用者維護三套版面。
- 日期區間、子專案等共用參數可一次套用所有有綁定的資料區塊。
- 每個資料區塊可繼承共用參數，並保留自己的維度、指標、篩選、排序及視覺化設定。
- V3 與 V1、V2 並存，建立新版本不影響既有範本執行結果。

#### 非目標

- 第一階段不提供像素級自由畫布、任意圖層重疊、旋轉、連接線或無限畫布。
- 不提供任意 JavaScript、SQL、HTML、公式編輯器或跨資料集 join。
- 不實作多人即時共同編輯、完整發佈審批流程或非同步大型匯出 Job。
- 不將整份多區塊報表直接匯出 PDF；第一階段維持資料區塊 CSV 匯出。
- 不改寫既有 A / B / C / E 統計口徑。

#### 使用情境

- 專案經理在同一份月報中放置事件總數、每日趨勢、優先級分布與明細表。
- 使用者拖動或縮放區塊後，其他區塊自動挪移到不重疊的位置。
- 值班人員在手機查看桌面版範本時，區塊依閱讀順序自動改為單欄。
- 使用者切換日期與子專案後，所有綁定共用參數的區塊一起重新查詢。
- 使用者複製既有圖表區塊，再修改維度或圖表類型建立新的分析角度。

#### 驗收條件

- [x] 8.1 V3 config 使用 `version: 3`，包含 `parameters`、`layout` 與 `blocks`，V1、V2 config 仍能正常讀取與執行。
- [x] 8.2 第一階段支援 `metric_card`、`chart`、`table`、`text` 四種 block type；未知類型由前後端拒絕，不得靜默呈現空白。
- [x] 8.3 每個 block 具有範本內唯一且穩定的 `id`；複製區塊必須產生新 id，不得共用 React key 或查詢快取 key。
- [x] 8.4 桌面 layout 使用 12 欄格線；位置以 `x`、`y`、`w`、`h` 儲存，所有數值需為整數並符合各 block 的 min / max size。
- [x] 8.5 拖拉或縮放發生碰撞時，系統依固定演算法將受影響區塊向下挪移並進行垂直 compact；相同輸入必須產生相同結果。
- [x] 8.6 區塊不得超出格線、互相覆蓋或產生負座標；不合法 layout 不得儲存，後端需再次驗證。
- [x] 8.7 手機寬度以單欄依桌面 `y`、`x`、block id 穩定排序；平板使用衍生欄數，第一階段不另外儲存手機座標。
- [x] 8.8 使用者可新增、複製、刪除、拖拉、縮放及復原尚未儲存的版面變更；刪除操作需可取消或經確認。
- [x] 8.9 拖拉與縮放具有鍵盤替代操作、可見焦點、可感知的移動結果及螢幕閱讀器標籤，不以滑鼠作為唯一操作方式。
- [x] 8.10 V3 支援 `date_range` 與 `sub_project` 共用參數；block 可選擇繼承或不綁定，但不得繞過專案與子專案權限。
- [x] 8.11 `metric_card`、`chart`、`table` 的 query 使用既有 V2 dataset / dimension / metric / filter 白名單；`text` 只接受純文字，不接受 HTML。
- [x] 8.12 預覽只查詢目前可見或使用者主動要求的資料區塊；同一份範本最多 20 個 block、最多 12 個資料 block，並限制同時查詢數量。
- [x] 8.13 多區塊預覽需保留 block id 對應、獨立 loading / empty / error 狀態；單一 block 失敗不得使其他成功 block 消失。
- [x] 8.14 使用者修改 block query 後只讓該 block 預覽過期；修改共用參數後讓所有已綁定 block 過期。
- [x] 8.15 範本儲存需使用 revision 或等效樂觀鎖；伺服器版本較新時回傳衝突，不得覆蓋他人的更新。
- [x] 8.16 資料 block CSV 匯出使用 POST JSON body 或已儲存 template / block id，不把完整 V3 config 放入 URL。
- [x] 8.17 新增、更新、刪除及複製 block、layout 變更與 V3 範本儲存需保留可追蹤的 audit 摘要，但不得記錄完整 filter values。
- [x] 8.18 範本編輯器需提供離開未儲存變更提示；儲存成功前不得清除 dirty 狀態或顯示成功通知。

#### 驗收情境

場景：建立多區塊報表
測試：待建立，`ReportLayoutDesignerPage` component test
假設：使用者具 Report create 權限
當：新增指標卡、趨勢圖與明細表並完成預覽
那麼：三個 block 各自呈現結果，儲存後重新開啟仍保持相同設定與桌面位置

場景：碰撞後自動挪移
測試：待建立，`compactReportLayout` unit test
假設：兩個 block 位於相鄰格線
當：使用者將第一個 block 移至第二個 block 的位置
那麼：第二個 block 依固定規則向下挪移，結果無重疊且不超出 12 欄

場景：手機自動重排
測試：待建立，`deriveResponsiveLayout` unit test
假設：桌面版具有多欄與不同高度 block
當：viewport 進入手機 breakpoint
那麼：所有 block 依穩定閱讀順序顯示為單欄，不遺失內容與操作

場景：共用參數更新
測試：待建立，`useReportLayoutPreview` hook test
假設：三個資料 block 中兩個綁定 date range
當：使用者修改日期區間
那麼：只有兩個綁定 block 變為過期並重新查詢，未綁定 block 保留目前結果

場景：單一區塊查詢失敗
測試：待建立，Report layout API integration test
假設：範本包含三個資料 block
當：其中一個 block config 不合法或查詢失敗
那麼：response 以 block id 回傳該區塊錯誤，其餘區塊仍回傳成功 payload

場景：版本衝突
測試：待建立，Report V3 template service test
假設：兩個瀏覽器載入相同 revision
當：第一個瀏覽器先儲存，第二個瀏覽器再以舊 revision 儲存
那麼：第二次更新回傳 409，使用者可重新載入或另存，不覆蓋新版

#### 驗證需求

- layout collision、compact、responsive ordering 與 boundary property tests。
- V3 config parse / normalize / validation 與 V1、V2 regression tests。
- 多 block partial success、權限、revision conflict 與 audit integration tests。
- block editor、鍵盤移動縮放、dirty guard、局部 stale preview component / hook tests。
- `go test ./internal/report`、`go test ./...`。
- `npm test`、`npm run typecheck`、`npm run build`。
- `make openapi`、三語系 key parity、桌面／平板／手機瀏覽器檢查。
- `git diff --check`。

### 需求 9：固定 OP 月報對齊 2026 年 7 月 Excel 基準

#### 文件定位

本需求接續需求 3.3a 至 3.3c 與已完成 Task 12。Task 12 以文字搜尋完成初版聚合，但 2026-08-03 使用 `202607-Jira開單紀錄表.xlsx` 與 `MS` 專案實資料驗證後，確認 Jira 判斷、人員班別、月底週段與標題尚未符合既有 Excel 契約。本需求新增可回歸的修正契約，不重寫模式 B、C、D、E 或既有專案權限。

#### 背景與現有行為

- `MS` 專案 2026 年 7 月 Excel 匯入資料共有 663 筆，支付或業務項目域名更換共有 492 筆。
- 現有 `op_jira_issue_count` 以標題、事件類型、資訊來源與子專案是否包含 `jira` 判斷，畫面只回傳 1 筆，未使用已儲存在 `tickets.external_ref` 的 Jira 單號。
- 現有固定 OP 月報只顯示人員名稱，未組合歷史班別前綴。
- 2026 年 7 月目前被切成 `7/24-7/30` 與 `7/31-7/31`，Excel 基準要求最後一日併入前一段為 `7/24-7/31`。
- 現有標題為日期區間加 `Report A`，未使用固定 OP 指標標題。

#### 目標

- 讓 `op_jira_issue_count` 與 `op_payment_domain_change_count` 的圖表、明細表、標題及 CSV 使用相同後端 payload，並對齊 Excel 統計口徑。
- 使用穩定資料欄位判斷指標，不依顯示文字或標題關鍵字猜測。
- 建立 2026 年 7 月 `MS` 專案的可重複驗證基準，但不得在應用程式碼硬編專案 ID、人名或驗收數字。

#### 非目標

- 本次不修正 `op_alert_notification_count`；Excel 告警 642 筆的獨立彙總資料來源另案處理。
- 不搬移、複製或改寫 `projects` 與既有 Ticket 資料。
- 不新增班別快照欄位，不解析匯入 description 作為正式統計契約。
- 不修改模式 B、C、D、E 的查詢與 payload。

#### 驗收條件

- [ ] 9.1 `op_jira_issue_count` 必須以 `tickets.external_ref` 非空白作為 Jira 開單判斷；不得再以標題或顯示名稱是否包含 `jira` 作為主要規則。
- [ ] 9.2 `op_payment_domain_change_count` 必須以 `ticket_resources.code = business_domain_change` 作為主要判斷；不得依顯示名稱包含「支付」與「域名」作為主要規則。
- [ ] 9.3 固定 OP 月報的 Ticket fact contract 必須提供 `external_ref`、`ticket_resource_code`、建立人與事件時間有效的班別資料，且 project / sub-project / date range 過濾維持不變。
- [ ] 9.4 人員標籤必須組合短班別與人員名稱，例如 `早-Chenmin`、`中-Eric`、`晚-ARK`；同一人員同一班別在整月只產生一列。
- [ ] 9.5 固定 OP 指標的列集合以日期區間內的有效 Ticket facts 人員為基礎；指標值為 0 的既有人員仍需出現在明細表與圖表中。
- [ ] 9.6 月內週段先以第一個星期四結束首段，再以星期五至星期四分段；若月底只剩單一日期，需併入前一週。2026 年 7 月必須輸出 `7/1-7/2`、`7/3-7/9`、`7/10-7/16`、`7/17-7/23`、`7/24-7/31`。
- [ ] 9.7 Jira 標題必須為 `YYYY年MM月 OP Jira開單數量統計`；支付域名標題必須為 `YYYY年MM月 OP 支付or業務項目域名更換統計`。
- [ ] 9.8 報表中心維持模式 A 與指標選擇流程，需清楚區隔固定 OP 月報與模式 E 每日值班矩陣；前端不得重算人員、週段或總計。
- [ ] 9.9 CSV 必須使用與畫面相同的標題、列順序、總計與週段欄位，不得另行套用不同聚合規則。
- [ ] 9.10 以 `MS` 專案 2026 年 7 月資料進行整合驗證時，Jira 總計必須為 663，支付域名總計必須為 492；應用程式與單元測試不得硬編該專案 ID。
- [ ] 9.11 `op_alert_notification_count` 在本次修正中保持原行為，不得為了對齊 642 而建立假 Ticket 或偽造報表資料。
- [ ] 9.12 Jira 與支付域名固定 OP 月報必須限定由 Excel「開單記錄-*」分頁匯入的 Ticket；匯入來源以 `ticket_activities.action_type = created` 且 `content` 符合 importer 寫入的 `imported from Excel 開單記錄-*` 稽核契約判斷，不得解析 `tickets.description`。告警通知不套用此來源限制。

#### 驗收情境

場景：Jira 固定 OP 月報對齊 Excel
測試：`TestQueryServiceModeAFixedOPJiraExcelBaseline`
假設：日期區間內存在具有 `external_ref` 的不同班別人員 Ticket
當：以 `op_jira_issue_count` 執行模式 A 預覽
那麼：每筆外部單號 Ticket 計入一次，標題、班別人員、總計及五個週段符合 Excel 契約

場景：支付域名使用穩定資源代碼
測試：`TestQueryServiceModeAFixedOPPaymentDomainResourceCode`
假設：資訊來源顯示名稱已修改，但 code 仍為 `business_domain_change`
當：以 `op_payment_domain_change_count` 執行模式 A 預覽
那麼：Ticket 仍被正確計入，其他資源不因文字相似而誤算

場景：月底單日併入前一週
測試：`TestMonthWeekRangesMergesSingleTrailingDay`
假設：日期區間為 2026-07-01 至 2026-07-31
當：後端產生月內週段
那麼：最後週段為 `7/24-7/31`，不得額外產生 `7/31-7/31`

場景：零值人員仍保留
測試：`TestQueryServiceModeAFixedOPKeepsZeroValuePeople`
假設：日期區間內某人具有 Jira Ticket，但沒有支付域名 Ticket
當：執行支付域名固定 OP 月報
那麼：該人員仍顯示在圖表與明細表，總計及週段皆為 0

#### 驗證需求

- `TicketFact` 與 query repository 欄位映射測試。
- Excel 開單記錄匯入來源、一般 Ticket 排除與告警不受來源限制的測試。
- Jira、支付域名、班別標籤、零值人員、標題及月底週段單元測試。
- Report handler / service payload 與 CSV 一致性測試。
- `go test ./internal/report`、`go test ./...`。
- `npm test`、`npm run typecheck`、`npm run build`。
- 以不寫入資料的 SQL 比對 `MS` 專案 2026 年 7 月 663／492 基準。
- 瀏覽器確認桌面圖表、明細表、標題與五個週段；`git diff --check`。

### 需求 10：固定 OP 月報明細表緊湊全展開

#### 文件定位

本需求接續需求 3.3b、需求 9 與 Task 16，只調整模式 A 固定 OP 月報的明細呈現。一般自定義報表、模式 E 每日值班矩陣、後端聚合、圖表與 CSV 不在本次範圍。

#### 背景與現有行為

- 固定 OP 月報目前共用 `ReportDetailTable` 的 Data Grid，容器固定高度為 420px，並啟用分頁與表格內捲動。
- 人員、總計與五個週區間只有七欄、九筆左右資料，現有欄寬與列高造成大量留白，無法快速比較同一人員的各週數字。
- 使用者需要在同一畫面區塊看到後端回傳的完整明細，不應透過表格內捲軸或分頁才能找到其餘資料。

#### 目標

- 桌面版以緊湊表格一次展開固定 OP 月報的全部列與全部欄位。
- 移除固定 OP 明細表本身的水平／垂直捲軸與分頁；資料增加時由頁面自然延伸與捲動。
- 在窄螢幕維持所有人員與週區間可讀，且不建立表格內水平捲軸。

#### 非目標

- 不變更後端 payload、聚合規則、人員排序、週區間或總計。
- 不修改柱狀圖、CSV 匯出、模式 E 矩陣或一般報表 Data Grid 的捲動與分頁策略。
- 不以前端重新計算、合併或省略任何明細列與欄位。

#### 驗收條件

- [ ] 10.1 當 `report_mode = A` 且明細欄位為 `person`、`total` 與 `week_*` 時，前端必須使用固定 OP 專用明細呈現，不得套用一般 Data Grid 的固定 420px 高度。
- [ ] 10.2 桌面寬度 1024px 以上必須在單一表格中一次呈現後端回傳的全部列與全部欄位，不顯示分頁控制器，也不產生表格內水平或垂直捲軸。
- [ ] 10.3 「一次全部顯示」是指所有資料均存在同一頁面流程中；當資料高度超過視窗時允許頁面自然捲動，不得建立獨立的表格捲動區。
- [ ] 10.4 桌面表頭與資料列高度應控制在 36 至 40px，儲存格採緊湊間距；文字不得小於 13px，並維持清楚的列分隔與可讀性。
- [ ] 10.5 人員欄靠左並配置約 128 至 144px；總計與週區間為數字欄、靠右顯示，總計約 72 至 88px，週區間平均使用剩餘寬度，避免現有過度鬆散的欄距。
- [ ] 10.6 小於 1024px 時改以每位人員一張緊湊卡片呈現，卡片內完整列出總計與所有週區間；所有卡片留在頁面自然流程，不使用表格內水平捲軸或分頁。
- [ ] 10.7 前端必須忠實保留後端 `table.columns` 順序、`table.rows` 順序及 0 值，不得截斷資料、重新排序或重算。
- [ ] 10.8 桌面表格需使用語意化 table 結構及欄標題關聯；窄螢幕卡片需保留標籤和值的可理解閱讀順序。
- [ ] 10.9 loading、empty、error 狀態與 CSV 匯出行為維持既有契約。
- [ ] 10.10 模式 B／C／D／V3 的一般明細表維持既有 Data Grid，模式 E 維持水平捲動與 sticky 左欄，不得因本需求產生回歸。

#### 驗收情境

場景：桌面一次查看完整固定 OP 明細
假設：後端回傳九位人員與 `person`、`total`、五個 `week_*` 欄位
當：使用者以 1024px 以上桌面寬度預覽固定 OP 月報
那麼：九列七欄全部顯示於單一緊湊表格，沒有分頁及表格內水平／垂直捲軸

場景：窄螢幕完整查看固定 OP 明細
假設：後端回傳相同九位人員與五個週區間
當：畫面寬度小於 1024px
那麼：每位人員以緊湊卡片完整顯示總計與各週數字，使用者只需捲動頁面，不需操作卡片或表格內捲軸

場景：一般報表維持既有大量資料能力
假設：模式 B、C、D 或 V3 回傳大量列欄
當：前端顯示一般明細表
那麼：仍使用既有 Data Grid、分頁與必要捲動，不套用固定 OP 的全展開版面

#### 驗證需求

- 固定 OP payload 辨識與專用 renderer 測試。
- 桌面完整列欄、無分頁、無內部捲動與緊湊尺寸測試。
- 平板／手機卡片呈現、欄位順序、0 值及可存取名稱測試。
- 一般報表 Data Grid、模式 E、loading、empty、error 與 CSV 回歸測試。
- `npm test`、`npm run typecheck`、`npm run build`、桌面／平板／手機瀏覽器檢查與 `git diff --check`。

### 需求 11：三個固定 OP 統計圖人員顯示一致

#### 文件定位

本需求接續需求 3.3a 至 3.3c、需求 9 與 Task 12，修正三個固定 OP 月報的人員顯示契約。告警數量統計圖目前只顯示純人名，與 Jira 開單及支付域名更換所使用的班別前綴和排序不同；本需求統一顯示方式，不改動指標計數口徑。

#### 目標

- `op_alert_notification_count`、`op_jira_issue_count`、`op_payment_domain_change_count` 均使用「班別短名-人員」標籤，例如 `早-Chenmin`、`中-Eric`、`晚-ARK`。
- 每張圖的 x 軸人員名稱及順序，必須與同一 payload 下方明細表的 `person` 欄完全一致。
- 三個指標均依早班、中班、晚班、未歸屬班別，再依人員名稱穩定排序。

#### 非目標

- 不修改三個指標的 Ticket predicate、名冊來源、總計或週區間數字。
- 不將告警通知改為 Excel 開單記錄來源，也不處理 Excel 告警 642 筆的資料來源差異。
- 不在前端重組班別、人名或排序，不修改一般模式 A 指標與模式 B／C／D／E。

#### 驗收條件

- [ ] 11.1 三個固定 OP 指標的每個人員標籤都必須使用相同的 `固定班別短名-人員名稱` 格式；不得只有告警圖使用純人名。
- [ ] 11.2 班別代碼 `morning`、`afternoon`、`night` 或既有班別名稱，必須正規化為 `早`、`中`、`晚`；缺少有效班別時使用既有明確未歸屬標籤。
- [ ] 11.3 三個指標都必須使用相同穩定順序：早班、中班、晚班、未歸屬班別；同一班別內依人員名稱排序。
- [ ] 11.4 `x_axis.labels[index]`、`series[0].data[index]` 與 `table.rows[index].person` 必須指向同一人員，且三者長度一致。
- [ ] 11.5 前端圖表不得自行排序或格式化人員名稱，應直接依後端 `x_axis.labels` 呈現；下方表格依 `table.rows` 呈現。
- [ ] 11.6 告警指標仍只使用符合既有告警 predicate 的 facts 建立名冊與計數；Jira、支付域名的 Excel 匯入來源規則及零值名冊維持不變。
- [ ] 11.7 圖表 tooltip 必須顯示與 x 軸及明細表相同的完整人員名稱，不因 axis label 截斷而改變資料名稱。
- [ ] 11.8 CSV 的人員名稱與列順序必須和畫面圖表、明細表一致。

#### 驗收情境

場景：告警統計使用班別人員標籤
假設：Eric 的有效班別為中班，且具有符合既有告警 predicate 的 Ticket
當：執行告警數量固定 OP 月報
那麼：圖表與明細表均顯示 `中-Eric`，不得顯示純 `Eric`

場景：三種固定 OP 指標使用相同排序
假設：資料包含早班 Chenmin、中班 Eric、晚班 ARK
當：分別執行告警、Jira 與支付域名固定 OP 月報
那麼：各報表內的人員均依早、中、晚順序呈現，且每張圖與其下方明細表順序相同

場景：統一顯示不改告警統計範圍
假設：某人只有一般 Ticket，沒有符合告警 predicate 的 Ticket
當：執行告警數量固定 OP 月報
那麼：該人員不因顯示規則統一而新增至告警名冊，既有告警總數保持不變

#### 驗證需求

- 三個固定 OP 指標的班別標籤、班別排序、x 軸／series／table index 對齊單元測試。
- 告警 predicate、告警名冊範圍、Jira／支付來源規則與零值人員回歸測試。
- 預覽 payload 與 CSV 人員名稱、列序一致性測試。
- `go test ./internal/report`、`go test ./...`、`npm test`、`npm run typecheck`、`npm run build`、瀏覽器三圖檢查與 `git diff --check`。

### 需求 12：固定 OP 月報直接依 Ticket 穩定欄位統計

#### 文件定位

本需求依 2026-08-07 報表查驗結果修正需求 9 的 Excel 匯入來源限制。報表中心的三個固定 OP 月報均以 `tickets` 及其關聯主檔為資料來源，與 Jira Report 或 Excel 匯入 provenance 無關。

#### 背景與問題

- `op_jira_issue_count` 目前額外要求 importer 建立活動，導致一般建立但已有 `external_ref` 的 Ticket 被排除。
- `op_payment_domain_change_count` 同樣要求 importer 建立活動，導致資源代碼為 `business_domain_change` 的一般 Ticket 被排除。
- `op_alert_notification_count` 目前以標題、類型、資源及子專案顯示文字包含 `alert`／`告警` 判斷，可能誤納文字相符但 `resource_type` 不是 `alert` 的 Ticket，也可能漏納名稱不含關鍵字的告警資源。

#### 目標

- 三個固定 OP 月報只依 Ticket 穩定欄位統計，不依賴 `ticket_activities` 的匯入內容。
- Jira 開單依 `tickets.external_ref` 去除前後空白後非空判斷。
- 告警通知依 `ticket_resources.resource_type = alert` 判斷。
- 支付或業務項目域名更換依 `ticket_resources.code = business_domain_change` 判斷。
- 人員名冊只包含符合所選指標的 Ticket 建立人員，圖表、明細表與 CSV 共用相同集合及數字。

#### 非目標

- 不修改 Jira Report 的 CSV／Excel 匯入功能、`jira_issues` 或 importer。
- 不自動修正既有 `ticket_resources.resource_type` 主檔資料。
- 不修改模式 B、C、D、E、一般模式 A 指標或前端圖表呈現。
- 不改變專案、子專案、日期、時區、班別與建立人員的既有查詢範圍。

#### 驗收條件

- [x] 12.1 `op_jira_issue_count` 必須計入日期範圍內所有 `external_ref` 非空白的 Ticket，不得查驗 Excel 匯入活動。
- [x] 12.2 `op_payment_domain_change_count` 必須計入日期範圍內所有資源代碼為 `business_domain_change` 的 Ticket，不得查驗 Excel 匯入活動。
- [x] 12.3 `op_alert_notification_count` 必須只依 `ticket_resources.resource_type = alert` 判斷，不得以標題或顯示名稱關鍵字作為主要規則。
- [x] 12.4 三個指標的人員名冊必須由符合各自 predicate 的 Ticket 建立，不得因匯入 provenance 產生空圖或無關零值人員。
- [x] 12.5 Repository 必須查出 `ticket_resources.resource_type`，並保留既有資源代碼、班別、人員、專案、子專案與日期欄位。
- [x] 12.6 `x_axis.labels`、`series`、明細表與 CSV 必須使用相同聚合結果，既有班別前綴與排序維持不變。
- [x] 12.7 一般 external reference、一般 `business_domain_change` Ticket 及 `resource_type = alert` Ticket 必須具正向測試；只有關鍵字但資源類型不是 `alert` 的 Ticket 必須具反向測試。

#### 驗證需求

- `TicketFact` 資源類型映射測試。
- 三個固定 OP predicate 與名冊測試。
- 圖表、明細表、CSV 聚合一致性回歸測試。
- `go test ./internal/report` 與 `git diff --check`。

### 需求 13：固定 OP 圖表與明細表寬度對齊

#### 目標

- 指標導向月報的固定 OP 報表，上方圖表與下方明細表必須放在同一個 RWD 內容容器中。
- 容器在窄螢幕使用全部可用寬度，在寬螢幕最大寬度為 1120px，並保持靠左排列。
- 圖表與明細表共用相同左右邊界與區塊間距，不各自維護互相衝突的最大寬度。

#### 驗收條件

- [x] 13.1 當 payload 符合固定 OP `person`、`total`、`week_*` 契約時，`ReportChartPanel` 與 `ReportDetailTable` 必須由同一父層容器包覆。
- [x] 13.2 共用容器使用 `width: 100%`、`max-width: 1120px`，小於 1120px 時隨外層縮小，不產生額外頁面水平捲動。
- [x] 13.3 下方 `FixedOPDetailTable` 不再自行設定重複的 1120px 上限，寬度由共同父層負責。
- [x] 13.4 一般模式 A、模式 B／C／D、值班統計與 V3 報表維持原本可用寬度，不套用固定 OP 容器。

### 需求 14：OP 專案開單數量年報

#### 使用者故事

身為值班組長，我希望選擇一個年份後查看全部主專案每月開單數量，以主專案堆疊呈現，並同時取得各月與全年總計，方便製作年度紀錄表。

#### 驗收條件

- [ ] 14.1 Report 中心的「報表模式」新增「年報」選項；選取年報後，「數值」提供「OP 專案開單數量」。
- [ ] 14.2 年報以 `Asia/Taipei` 的曆年為統計範圍，橫軸固定顯示該年 `01` 至 `12` 月，格式為 `YYYY/MM`；尚無資料的月份顯示 `0`，不得省略月份。
- [ ] 14.3 「OP 專案開單數量」以 Ticket 建立時間計數，每張符合專案範圍的 Ticket 計一次，不依賴 `ticket_activities` 或匯入來源。
- [ ] 14.4 圖表使用堆疊長條圖；每個系列代表一個主專案，系列名稱取自 `projects`，同一月份所有系列加總等於該月總數。
- [ ] 14.5 明細表的欄為 12 個月份與「總計」，列為各主專案及「總數」；主專案列總計為 12 個月加總，總數列為各欄所有主專案加總，右下角為全年總數。
- [ ] 14.6 年報統計範圍為所有未刪除且狀態為啟用中的主專案；停用與封存主專案及其 Ticket 不得納入。年報入口仍須通過 Report `read` 權限，主專案與子專案選擇不得縮限或改變年報統計範圍。
- [ ] 14.7 後端回傳完整圖表與明細 payload，前端只負責呈現，不得自行補算月份、主專案或總計。
- [ ] 14.8 查無資料時仍回傳 12 個月份及全為 `0` 的總數資料，並以 `meta.empty = true` 標示空資料。
- [ ] 14.9 年報預覽與 CSV 匯出必須共用相同統計口徑、月份順序、主專案順序及總計。
- [ ] 14.10 年份輸入需驗證合理範圍，所有查詢使用參數化 SQL，並沿用 Report `read` 權限與專案範圍檢查。

### 需求 15：MS-指標導向月報

#### 使用者故事

身為 MS 專案的值班組長，我希望使用專用的指標導向月報模式，只選擇既定 OP 指標並取得一致的月內週區間、人員班別圖表與明細表，避免與一般 Ticket 統計指標混用。

#### 驗收條件

- [x] 15.1 僅在目前主專案代碼為 `MS` 的 Report 中心顯示「MS-指標導向月報」模式（`report_mode = G`）；離開 MS 專案時介面回復為模式 A。
- [x] 15.2 模式 G 的「數值」僅提供 `op_jira_issue_count`、`op_alert_notification_count` 與 `op_payment_domain_change_count`，預設為 `op_jira_issue_count`。
- [x] 15.3 模式 G 固定 `person_basis = created_by`，沿用模式 A 的月內週區間、班別人員標籤、圖表、明細表與 CSV 聚合結果。
- [x] 15.4 後端必須拒絕模式 G 使用一般 Ticket 指標、兩個以上指標，或 `metrics` 與 `indicators` 不一致的設定；未指定指標時預設 OP Jira 開單數量。
- [x] 15.5 模式 A「指標導向月報」的數值選單不得顯示三個固定 OP 指標；三個 OP 指標只在模式 G「MS-指標導向月報」提供。由模式 G 切換至模式 A 時，數值需回復一般指標。
