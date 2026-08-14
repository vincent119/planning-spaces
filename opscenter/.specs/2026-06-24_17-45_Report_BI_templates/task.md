# Report BI Templates Tasks

## 文件定位

本文件追蹤 Report 範本 BI 化改造。此工作是 `.kiro/specs/2026-06-22_11-06_Report` 的延伸，不得改變既有內建月報 A / B / C / E 的 API 契約。

## 0. 契約與相容性

- [x] 0.1 盤點現有 `ReportTemplateConfig`
  - 檢查 server domain DTO
  - 檢查 repository 儲存格式
  - 檢查 frontend type 與 form 欄位
  - 列出 V1 到 V2 的欄位對應
  - 完成：已新增 `contract.md`，盤點 server `TemplateConfig`、repository JSONB 儲存、frontend `ReportTemplateConfig` 與現有 dialog 欄位，並列出 V1 到 V2 對應
  - _Requirements: 1.1-1.4, 6.4-6.5_

- [x] 0.2 定義 `ReportTemplateConfigV2`
  - 增加 `version`
  - 分離 `dataset`
  - 分離 `date`
  - 分離 `query`
  - 分離 `visualization`
  - 完成：已在 server `domain.go` 定義 `TemplateConfigV2`，並在 frontend `types.ts` 定義 `ReportTemplateConfigV2`
  - _Requirements: 1.1-1.4_

- [x] 0.3 定義 V1 相容讀取策略
  - 缺少 `version` 視為 V1
  - 可轉換範本回傳 V2 view model
  - 不可轉換範本標示為 legacy
  - 不自動覆寫使用者原始 config
  - 完成：已於 `contract.md` 定義缺少 `version` 視為 V1、不自動覆寫原始 config、可轉換 view model 與 legacy 標示策略
  - _Requirements: 6.4-6.5_

- [x] 0.4 定義 V2 config 錯誤碼
  - 未知 dataset
  - 未知 dimension
  - 未知 metric
  - 未知 filter
  - 無效組合
  - 完成：已在 server `domain.go` 定義 `ConfigErrorCode` 常數，在 frontend `types.ts` 定義 `ReportBIConfigErrorCode`，並於 `contract.md` 列出錯誤碼語意
  - _Requirements: 2.4-2.5, 3.6_

## 1. Server Dataset Metadata

- [x] 1.1 建立 dataset definition 型別
  - `DatasetDefinition`
  - `DimensionDefinition`
  - `MetricDefinition`
  - `FilterDefinition`
  - `SortDefinition`
  - `ChartTypeDefinition`
  - 完成：已在 server `domain.go` 定義 dataset definition 型別，供 metadata、validator 與 query planner 共用
  - _Requirements: 2.1-2.4_

- [x] 1.2 定義 Ticket 運維事件 dataset
  - dataset code：`ticket_events`
  - 顯示名稱
  - 描述
  - 可用日期欄位
  - 支援圖表類型
  - 完成：已在 `dataset_metadata.go` 建立 `ticket_events` registry，包含名稱、描述、日期能力與圖表類型
  - _Requirements: 2.1-2.3_

- [x] 1.3 定義 MVP 維度白名單
  - 日期
  - 人員
  - 子專案
  - 事件類型
  - 狀態
  - 優先級
  - 完成：已定義 `date`、`person`、`sub_project`、`ticket_type`、`status`、`priority`
  - _Requirements: 3.1-3.5_

- [x] 1.4 定義 MVP 指標白名單
  - 事件數量
  - 保留後續加總與平均擴充欄位
  - 完成：已定義 `ticket_count` 指標與 `count` aggregation
  - _Requirements: 3.3_

- [x] 1.5 定義 MVP 篩選白名單
  - 狀態
  - 事件類型
  - 子專案
  - 優先級
  - 人員
  - 完成：已定義人員、子專案、事件類型、狀態、優先級篩選欄位，operator 目前支援 `in`
  - _Requirements: 4.1_

- [x] 1.6 定義 MVP 排序白名單
  - 維度名稱升冪
  - 維度名稱降冪
  - 日期升冪
  - 日期降冪
  - 總量升冪
  - 總量降冪
  - 完成：已定義 `dimension`、`date`、`total`，方向支援 `asc` / `desc`
  - _Requirements: 4.3_

- [x] 1.7 實作 dataset 列表 API
  - `GET /api/v1/projects/:id/reports/datasets`
  - 套用 Report read 權限
  - 套用專案可見性
  - 回傳可用 dataset 摘要
  - 完成：已在 Report handler/service 新增 dataset list API，secured route 使用 `reports/read`，service 內以 Project Viewer 驗證專案可見性
  - _Requirements: 2.1-2.4, 6.2_

- [x] 1.8 實作 dataset metadata API
  - `GET /api/v1/projects/:id/reports/datasets/:dataset/metadata`
  - 回傳維度、指標、篩選、排序與圖表類型
  - 未知 dataset 回 400
  - 完成：已新增 metadata API，未知 dataset 透過 `ErrInvalidInput` 回 400
  - _Requirements: 2.2-2.5_

- [x] 1.9 補 metadata API 測試
  - 有權限取得 dataset
  - 無權限回 403
  - 不可見專案不得洩漏 metadata
  - 未知 dataset 回 400
  - 完成：已補 service 與 handler 測試，覆蓋列表、metadata、未知 dataset、權限不足與 viewer 權限檢查
  - _Requirements: 2.2-2.5, 6.2_

## 2. Server Config Validator

- [x] 2.1 實作 V2 config parser
  - 接受 JSON raw message
  - 判斷 version
  - V1 與 V2 分流
  - 回傳正規化 config
  - 完成：已新增 `ParseTemplateConfig`，支援缺少 version 的 V1、明確 `version = 1` 與 `version = 2`，V1 回傳 legacy 標記
  - _Requirements: 1.1-1.4, 6.5_

- [x] 2.2 驗證 dataset
  - dataset 必填
  - dataset 必須存在於 registry
  - dataset 必須支援指定 chart type
  - 完成：`NormalizeTemplateConfigV2` 會檢查 dataset registry 與 chart type 支援度，錯誤碼為 `unknown_dataset` 或 `unsupported_chart_type`
  - _Requirements: 2.1-2.5_

- [x] 2.3 驗證日期設定
  - preset 白名單
  - timezone 固定 `Asia/Taipei`
  - grain 白名單
  - date dimension 才允許 grain
  - 完成：已驗證 preset、timezone、grain，且只有使用 date 維度時允許 grain
  - _Requirements: 3.4, 4.2_

- [x] 2.4 驗證維度與 series
  - x_axis 必填
  - series 可為空
  - x_axis 與 series 不得相同
  - 人員維度要求 person_basis
  - 完成：已驗證 x_axis / series 白名單、重複維度組合，以及 person 維度的 `person_basis`
  - _Requirements: 3.1-3.6_

- [x] 2.5 驗證指標
  - metric 必填
  - metric 必須存在
  - metric aggregation 不由前端指定
  - 完成：已驗證 metric 必須存在於 dataset definition，aggregation 只從後端 definition 取得
  - _Requirements: 3.3, 6.4_

- [x] 2.6 驗證 filters
  - field 必須存在
  - operator 必須符合欄位允許值
  - values 型別正確
  - 空 values 視為無效
  - 完成：已驗證 filter field、operator 與 values，未知欄位或注入型欄位字串不會通過白名單
  - _Requirements: 4.1, 6.4-6.5_

- [x] 2.7 驗證 sort 與 limit
  - sort field 白名單
  - direction 白名單
  - limit 只允許全部、10、20、50
  - 完成：已驗證 sort field、direction 與 Top N limit；nil 代表全部，數值只允許 10 / 20 / 50
  - _Requirements: 4.3-4.4_

- [x] 2.8 補 validator 測試
  - 合法 V2 config
  - 未知 dataset
  - 未知 dimension
  - 未知 metric
  - filter SQL injection 字串
  - x_axis 與 series 重複
  - 人員維度缺 person_basis
  - 無效 limit
  - 完成：已新增 `config_v2_test.go`，覆蓋合法 V2、V1 legacy、明確 V1、未知 dataset/dimension/metric、filter 注入字串、重複維度、人員口徑、非法 limit、非日期維度 grain 與未知 version
  - _Requirements: 2.4-2.5, 3.6, 6.4-6.5_

## 3. Server Query Planner

- [x] 3.1 建立 Query Planner 介面
  - input 為正規化 V2 config
  - input 包含專案範圍與日期區間
  - output 為 chart payload 與 table payload
  - 完成：已新增 `QueryPlannerInput` 與 `QueryService.PlanV2`，輸入包含專案、子專案、日期區間、V2 config 與操作者，輸出 `ChartPayload`
  - _Requirements: 1.1-1.3, 5.1-5.5_

- [x] 3.2 建立 Ticket dataset SQL mapping
  - 每個 dimension 對應固定 expression
  - 每個 filter 對應固定 where builder
  - 每個 sort 對應固定 order expression
  - 禁止外部字串成為 SQL expression
  - 完成：已擴充 `TicketFact` 固定欄位與 repository SELECT；V2 planner 只使用後端白名單 dimension/filter/sort mapping，不接受外部 SQL expression
  - _Requirements: 2.4-2.5, 6.4-6.5_

- [x] 3.3 實作日期維度查詢
  - 日粒度
  - 週粒度
  - 月粒度
  - Asia/Taipei 邊界
  - 完成：已在 `query_planner_v2.go` 實作日、週、月日期 label，使用 `Asia/Taipei` local time
  - _Requirements: 2.7-2.8, 3.4_

- [x] 3.4 實作人員維度查詢
  - `created_by`
  - `actor`
  - 人員顯示名稱
  - 空人員 fallback
  - 完成：`PlanV2` 依 `person_basis` 選擇 ticket facts 或 activity facts，人員空值 fallback 為 `未指派`
  - _Requirements: 3.1, 3.5_

- [x] 3.5 實作子專案維度查詢
  - active 子專案名稱
  - historical snapshot fallback
  - 未分類 bucket
  - 完成：子專案維度使用 facts 內固定 `SubProject` 欄位，空值 fallback 為 `未分類`
  - _Requirements: 3.1, 4.1_

- [x] 3.6 實作事件類型維度查詢
  - 使用 `ticket_types`
  - 不直接用顯示文字當主鍵
  - 停用類型保留歷史顯示
  - 完成：repository 固定 join `ticket_types` 產出 fact 欄位，planner 只使用該白名單欄位並提供空值 fallback
  - _Requirements: 3.1, 4.1_

- [x] 3.7 實作狀態與優先級維度查詢
  - 狀態白名單
  - 優先級白名單
  - 空值 bucket
  - 完成：已擴充 `TicketFact`、ticket/activity repository SQL 與 V2 dimension mapping，支援 `status`、`priority` 與空值 bucket
  - _Requirements: 3.1, 4.1_

- [x] 3.8 實作 series 分組
  - 無 series
  - 子專案 series
  - 事件類型 series
  - 狀態 series
  - 優先級 series
  - 人員 series
  - 完成：V2 planner 支援無 series 的 `總計`，以及所有白名單維度作為 series
  - _Requirements: 3.2_

- [x] 3.9 實作事件數量 metric
  - count 規則
  - actor 口徑去重規則
  - 與既有 Report 統計口徑對齊
  - 完成：已實作 `ticket_count` 聚合，依 ticket/x/series 去重，actor facts 也避免同一 Ticket 重複計數
  - _Requirements: 3.3, 5.5_

- [x] 3.10 實作 Top N
  - 依總量裁切 X 軸
  - 其他項目是否合併先不實作
  - 明細表與圖表裁切一致
  - 完成：已依 X 軸總量排序後裁切 Top N，圖表 labels 與 table rows 使用同一組裁切結果
  - _Requirements: 4.4, 5.5_

- [x] 3.11 實作 chart payload builder
  - stacked bar
  - grouped bar
  - line
  - table only 空 chart
  - 完成：已依 V2 visualization 建立 chart payload；stacked bar 設定 stack，grouped bar / line 不設定 stack，table 模式不回傳 series
  - _Requirements: 5.1-5.2_

- [x] 3.12 實作 dynamic table payload builder
  - 欄位依維度與 series 產生
  - row key 穩定
  - 總計欄
  - CSV 匯出共用
  - 完成：已依 x_axis、series 與 metric 產生動態 table payload，後續第 4 段 export 可共用此 table
  - _Requirements: 5.2, 5.5_

- [x] 3.13 補 Query Planner 測試
  - 日期 × 事件類型
  - 人員 × 子專案
  - 子專案 × 狀態
  - 狀態 × 優先級
  - actor 口徑
  - created_by 口徑
  - Top 10
  - 空資料
  - 完成：已新增 `query_planner_v2_test.go`，覆蓋人員 × 子專案、日期 × 事件類型、子專案 × 狀態、狀態 × 優先級、actor 去重、Top N 與空資料
  - _Requirements: 3.1-3.6, 4.1-4.5, 5.1-5.5_

## 4. Server API 整合

- [x] 4.1 升級 preview API 支援 V2
  - request 接受 V2 config
  - V1 走既有流程
  - V2 走 Query Planner
  - 錯誤回傳欄位代碼
  - 完成：`PreviewReport` 已改為 raw config parser，V1 走既有 preview，`version=2` 走 `PlanV2`；`ConfigValidationError` 回 400 並使用錯誤碼作 message
  - _Requirements: 1.2, 2.5_

- [x] 4.2 升級 template create API
  - 接受 V2 config
  - 儲存前正規化
  - duplicate name 保持既有行為
  - 寫入審計
  - 完成：template create 支援 V1/V2 raw config，V2 儲存前執行 `NormalizeTemplateConfigV2`，repository 依版本 marshal；duplicate 與審計流程沿用既有行為
  - _Requirements: 1.1-1.4, 6.1-6.3_

- [x] 4.3 升級 template update API
  - 接受 V2 config
  - 比對 before / after 摘要
  - 寫入審計
  - 完成：template update 支援 V2 config，repository 更新時保留 V2 JSON，審計摘要會依版本記錄 V1 或 V2 config
  - _Requirements: 6.1-6.3_

- [x] 4.4 升級 template execute API
  - V2 execute 讀取範本 config
  - 允許覆蓋日期
  - 不允許覆蓋 dataset / dimension / metric
  - 完成：execute 讀取範本後若為 V2 config 會走 `PlanV2`，request 只接受日期與子專案覆蓋，不接受覆蓋 dataset / dimension / metric
  - _Requirements: 1.3, 6.2_

- [x] 4.5 升級 CSV export API
  - 支援 V2 config query
  - 支援 template id export
  - 使用 table payload 產 CSV
  - 失敗不得回傳 CSV content type
  - 完成：export 支援 `config` V2 JSON query 與 `template_id`，使用 chart table payload 產 CSV；失敗仍走 JSON error envelope，不回 CSV content type
  - _Requirements: 4.5, 5.2, 6.2_

- [x] 4.6 補 OpenAPI 註解
  - dataset metadata
  - V2 config schema
  - preview V2 request
  - execute override date request
  - export query
  - 完成：已補 dataset metadata routes、`PreviewV2Request`、`TemplateV2Request`、execute request 與 export `template_id` / `config` query 註解
  - _Requirements: 2.2, 6.4_

- [x] 4.7 重新產生 OpenAPI
  - 執行既有 generator
  - 檢查 schema
  - 檢查 CSV success 與 JSON error
  - 完成：已執行 `make openapi`，產出 `Docs/openapi.json` 共 62 paths，並檢查 Report dataset、template、preview、export path 內容
  - _Requirements: 2.2, 4.5_

- [x] 4.8 補 API handler 測試
  - metadata
  - preview V2
  - create V2
  - update V2
  - execute V2
  - export V2
  - 權限不足
  - 完成：已補 V2 preview/create/execute/export handler 測試與 V2 config error code 測試；metadata 與權限不足測試已在第 1 段完成
  - _Requirements: 1.1-6.5_

## 5. Frontend API 與型別

- [x] 5.1 定義 V2 frontend types
  - dataset metadata
  - V2 config
  - builder form state
  - validation error
  - 完成：已補 `ReportDatasetMetadata`、metadata item 型別、`ReportTemplateConfigAny`、V2 payload、builder form state 與 validation error 型別
  - _Requirements: 2.2-2.3_

- [x] 5.2 實作 metadata API client
  - list datasets
  - get dataset metadata
  - error envelope mapping
  - 完成：已新增 `listReportDatasets` 與 `getReportDatasetMetadata`，沿用既有 `apiClient` error envelope 與 auth refresh 流程
  - _Requirements: 2.2-2.3_

- [x] 5.3 更新 report API client
  - preview V2
  - create template V2
  - update template V2
  - execute V2 date override
  - export V2
  - 完成：`previewReport`、template create/update payload 已支援 V1/V2 union，並新增 V2 明確函式；CSV export 支援 V2 config 與 template id export
  - _Requirements: 1.1-1.4, 4.5_

- [x] 5.4 更新 query keys
  - datasets
  - dataset metadata
  - preview config hash
  - template execute
  - 完成：已新增 datasets、dataset metadata、template execute query key，preview key 改用穩定排序後的 config hash
  - _Requirements: 2.2, 1.2_

## 6. Frontend Builder UI

- [x] 6.1 將範本表單改為抽屜或完整頁面
  - 避免窄 modal
  - 表單與預覽分區
  - 行動版可單欄
  - 完成：新增範本流程改走 `/reports/designer` 完整頁，左側設定、右側預覽，行動版自動單欄
  - _Requirements: 1.1-1.4_

- [x] 6.2 建立基本資料區塊
  - 名稱
  - 描述
  - 可見範圍提示
  - legacy 狀態提示
  - 完成：已建立名稱、描述與專案共用提示；legacy 編輯提示留待第 7 段列表與執行流程處理
  - _Requirements: 1.4, 6.5_

- [x] 6.3 建立資料集選擇區塊
  - dataset select
  - metadata loading
  - metadata error state
  - metadata 未載入時停用後續欄位
  - 完成：已串 `listReportDatasets` 與 `getReportDatasetMetadata`，metadata 載入中與失敗狀態會反映到欄位停用與錯誤提示
  - _Requirements: 2.2-2.3_

- [x] 6.4 建立日期設定區塊
  - preset
  - grain
  - timezone 顯示
  - 自訂日期 override 說明
  - 完成：已加入 preset、grain、preview 日期 override 與 Asia/Taipei 說明；修正 preset 變更會同步更新起迄日期，手動調整起迄日期會切回自訂
  - _Requirements: 3.4, 4.2_

- [x] 6.5 建立維度設定區塊
  - X 軸維度
  - series 維度
  - 選項來自 metadata
  - 禁止重複維度
  - 完成：X 軸與 series 皆使用 metadata options，series 會停用目前 X 軸並顯示重複維度錯誤
  - _Requirements: 3.1-3.6_

- [x] 6.6 建立人員口徑設定
  - `created_by`
  - `actor`
  - 只有人員維度或 series 需要時顯示
  - 完成：只有 metadata 標示需要人員口徑的維度被選取時顯示 created_by / actor
  - _Requirements: 3.5_

- [x] 6.7 建立指標設定區塊
  - metric select
  - metric 說明
  - 預設事件數量
  - 完成：metric select 使用 metadata metrics，預設 `ticket_count`，並顯示 aggregation 說明
  - _Requirements: 3.3_

- [x] 6.8 建立篩選設定區塊
  - 狀態
  - 事件類型
  - 子專案
  - 優先級
  - 人員
  - 多選與清除
  - 完成：依 metadata filters 渲染多選欄位；子專案使用既有子專案 API 選項，其餘欄位支援 free solo 多值輸入與清除
  - _Requirements: 4.1_

- [x] 6.9 建立排序與 Top N 區塊
  - sort field
  - sort direction
  - limit
  - 無 limit 顯示全部
  - 完成：已加入 sort field、sort direction、Top N 與全部選項
  - _Requirements: 4.3-4.4_

- [x] 6.10 建立視覺化設定區塊
  - chart type
  - show table
  - legend wrap
  - 不支援的 chart type 停用
  - 完成：chart type 使用 metadata chart types，並加入 show table 與 legend wrap 切換
  - _Requirements: 5.1-5.4_

- [x] 6.11 建立前端表單驗證
  - 必填欄位
  - 無效組合
  - metadata 缺失
  - API validation error 顯示到欄位
  - 完成：已驗證範本名稱、日期區間、重複維度與 metadata 缺失；API validation error 會顯示在預覽區
  - _Requirements: 2.3, 3.6_

- [x] 6.12 建立預覽後儲存流程
  - 未預覽不可儲存
  - config 變更後需重新預覽
  - 預覽成功顯示摘要
  - 儲存成功刷新列表
  - 完成：preview hash 與目前 config 比對，未預覽或變更後不可儲存；預覽成功顯示摘要，儲存成功 invalidate 範本列表並返回報表中心
  - _Requirements: 1.2, 1.4_

## 7. Frontend 範本列表與執行

- [x] 7.1 更新範本列表欄位
  - 名稱
  - dataset
  - X 軸
  - series
  - metric
  - chart type
  - 更新時間
  - 完成：範本列表已顯示名稱、版本、描述、dataset、模式、X 軸、series、metric、chart type 與更新時間
  - _Requirements: 1.4_

- [x] 7.2 支援 legacy 範本顯示
  - 標示舊版
  - 允許執行
  - 編輯時提示另存新範本
  - 完成：列表與詳情頁會標示 legacy；legacy 範本仍可執行與匯出；編輯 legacy 時顯示另存新範本提示並走 create 流程
  - _Requirements: 6.5_

- [x] 7.3 更新範本執行流程
  - 選擇日期 override
  - 執行成功更新圖表
  - 執行失敗顯示錯誤
  - 完成：範本詳情頁支援日期 override，執行成功更新圖表與明細，失敗顯示 toast 與圖表區錯誤
  - _Requirements: 1.3, 6.2_

- [x] 7.4 更新 CSV 匯出流程
  - 範本 export
  - 目前 preview config export
  - 錯誤時不下載檔案
  - 完成：範本詳情頁支援 template id 匯出，designer 頁支援目前 preview config 匯出；錯誤沿用 JSON error envelope，不會下載檔案
  - _Requirements: 4.5, 6.2_

- [x] 7.5 報表中心改為報表 / 範本頁籤
  - 報表中心主內容需拆成「報表」與「範本」兩個頁籤
  - 「報表」頁籤保留報表條件、預覽、圖表 / 矩陣、明細表、匯出與儲存範本操作
  - 「範本」頁籤集中呈現範本列表、搜尋、套用、編輯與刪除操作
  - 移除報表預覽旁的範本列表側欄，避免每日值班執行統計矩陣被壓縮
  - 從「範本」頁籤套用範本後，需帶入範本設定並切回「報表」頁籤
  - 切換頁籤不得清空專案範圍、子專案、日期區間、報表模式與未儲存設定
  - 範本頁籤列表建議使用 DataGrid；資料集、報表模式、人員基準與圖表類型可用 chip 輔助辨識
  - 完成：報表中心加入「報表 / 範本」tabs；報表條件與預覽改在報表 tab 內使用完整寬度；範本列表移到範本 tab；範本列表在報表中心模式下的執行按鈕改為套用範本並切回報表 tab
  - 驗證：`npm run typecheck`、`npm run build` 通過
  - _Requirements: 7.1-7.6; Design: Report Center Tabs_

## 8. Frontend Chart 與 Table

- [x] 8.1 擴充 chart renderer
  - stacked bar
  - grouped bar
  - line
  - table only
  - 完成：chart option 依 payload `chart_type` 支援 stacked bar、grouped bar、line；table only 以表格模式提示呈現，不再渲染空圖
  - _Requirements: 5.1-5.2_

- [x] 8.2 強化圖例呈現
  - 多列顯示
  - 不使用分頁隱藏主要 series
  - 長名稱截斷與 tooltip
  - 完成：legend 改用 plain 多列顯示，依 series 數量增加高度；長名稱截斷並保留 tooltip
  - _Requirements: 5.3_

- [x] 8.3 強化主題色
  - 文字跟隨 theme
  - 座標軸跟隨 theme
  - 格線跟隨 theme
  - 空資料提示跟隨 theme
  - 完成：圖表文字、座標軸、格線、tooltip、背景與空資料提示皆取 MUI theme 色彩
  - _Requirements: 5.4_

- [x] 8.4 強化 y 軸與圖表留白
  - y 軸 label 不重疊
  - 圖例不壓到圖表
  - data zoom 與 legend 不重疊
  - 完成：grid left 依數字長度保留空間，legend 與 data zoom 分層計算 bottom，chart height 依 legend rows 與 data zoom 動態增加
  - _Requirements: 5.3-5.4_

- [x] 8.5 實作動態明細表
  - 欄位依 payload 產生
  - 數字格式化
  - 空資料狀態
  - 與 CSV 欄位一致
  - 完成：明細表使用 payload table columns/rows 順序，數字依 locale 加千分位，長文字以 tooltip 顯示完整值，空資料沿用既有 DataGrid 狀態
  - _Requirements: 5.2, 5.5_

## 9. Migration 與資料相容

- [x] 9.1 檢查是否需要 schema migration
  - `report_templates.config` 是否足夠
  - 是否需新增 `config_version`
  - 是否需新增 metadata 欄位加速列表
  - 完成：已確認 `report_templates.config` JSONB 足以保存 V1/V2；`config_version` 與 `config_legacy` 由 server parser 推導，metadata 顯示欄位目前不需 DB filter / sort，因此不新增欄位
  - _Requirements: 6.5_

- [x] 9.2 若需要，建立 migration
  - 新增欄位
  - backfill config version
  - 建立必要 index
  - 完成：評估後不需要新增 migration；既有 `0030_align_report_templates.sql` active index 與 unique index 足夠目前 V2 範本 CRUD / execute / export 使用
  - _Requirements: 1.4, 6.5_

- [x] 9.3 撰寫資料回滾說明
  - 不刪除既有範本
  - V2 config 可保留但舊版程式不可編輯
  - 回滾風險說明
  - 完成：已新增 `migration.md`，記錄不刪除 V1/V2 範本、回滾到不支援 V2 程式的風險、暫停 V2 範本的軟刪除 SQL 與驗證查詢
  - _Requirements: 6.5_

## 10. 驗證

- [x] 10.1 Server 單元測試
  - config validator
  - dataset metadata
  - query planner
  - CSV builder
  - 完成：已執行 `go test ./internal/report`，覆蓋 config validator、dataset metadata、V2 query planner 與 CSV 相關測試
  - _Requirements: 2.4-2.5, 6.4-6.5_

- [x] 10.2 Server 整合測試
  - metadata API
  - preview V2
  - template CRUD V2
  - execute V2
  - export V2
  - 完成：已執行 `go test ./...`，report handler 測試覆蓋 metadata API、preview V2、create/execute/export V2 與 template id export
  - _Requirements: 1.1-6.5_

- [x] 10.3 Frontend 型別與建置
  - `npm run typecheck`
  - `npm run build`
  - 完成：已執行 `npm run typecheck` 與 `npm run build`；build 成功，僅保留既有 chunk size 警告
  - _Requirements: 1.1-6.5_

- [ ] 10.4 Frontend 手動驗收
  - 建立人員 × 子專案堆疊報表
  - 建立日期 × 事件類型折線報表
  - 建立子專案 × 狀態群組長條報表
  - 儲存範本
  - 執行範本
  - 匯出 CSV
  - _Requirements: 3.1-5.5_

- [ ] 10.5 主題與版面驗收
  - 淺色主題文字可讀
  - 深色主題文字可讀
  - 圖例換行
  - y 軸不重疊
  - 小螢幕不破版
  - _Requirements: 5.3-5.4_

- [ ] 10.6 回歸驗收
  - 內建月報 A
  - 內建月報 B
  - 內建月報 C
  - 每日值班執行統計
  - 舊版範本執行
  - _Requirements: 6.4-6.5_
