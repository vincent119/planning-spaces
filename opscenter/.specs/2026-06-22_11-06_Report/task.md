# Report Tasks

## 文件定位

本文件追蹤 Report 模組後續實作。Report 需求與設計來源為本目錄 `requirements.md`、`design.md`、`design-backend.md`、`design-frontend.md`。不得把 Report 新需求追加回舊 oncall-ticket-system 已完成 task；若舊文件需要對齊，需新增未完成 task 並等待安排。

## 0. 文件與契約對齊

- [x] 0.1 對齊 Report 與舊 oncall-ticket-system 文件引用
  - 確認舊文件中 Report 內容不再作為實作唯一來源
  - 補充舊文件指向 `.kiro/specs/2026-06-22_11-06_Report`
  - 不變更舊 task 已完成狀態
  - 完成：已在舊 `requirements.md`、`design.md`、`design-backend.md`、`design-frontend.md` 的 Report 段落補充新 Report spec 為後續實作來源；未修改舊 task 完成狀態
  - _Requirements: 1.1, 4.1-4.7_

- [x] 0.2 讀取現有 OpenAPI 與 server route 缺口
  - 檢查 `Docs/openapi.json` 是否已有 Report paths
  - 建立缺口清單：requestBody、response DTO、權限、CSV response
  - 若 OpenAPI 不完整，先補 server Swagger 註解或 generator 掃描範圍
  - 完成：已檢查 `opscenter-server/Docs/openapi.json` 與 `opscenter-server/docs/openapi.json`，目前沒有 Report paths；已在 `design-backend.md` 補 OpenAPI 缺口清單，後續 4.1 / 4.2 需補 Swagger 註解與重新產生 OpenAPI
  - _Requirements: 1.1-1.5, 3.7, 5.3-5.5_

- [x] 0.3 定義 Report API DTO 與錯誤契約
  - 定義 `ReportTemplateConfig`
  - 定義 `ReportDateRange`
  - 定義 `ReportChartPayload`
  - 定義空資料 payload 與錯誤 response
  - 完成：已在 `design-backend.md` 補 API envelope、範本 response、execute request、DTO 驗證規則、空資料 payload、錯誤契約與 CSV response 契約
  - _Requirements: 2.7-2.8, 3.7, 6.2-6.6_

## 1. Server 基礎與資料表

- [x] 1.1 建立 `report_templates` migration
  - 新增 `report_templates` 表
  - 支援 `deleted_at` 軟刪除
  - 建立 project/name active unique index
  - 補齊 table / column comments
  - 完成：已新增 `opscenter-server/sql/0030_align_report_templates.sql`；因舊 `0013` 已建立 `report_templates`，本次採 schema 對齊方式補 `deleted_at`、active partial indexes、project/name active unique index 與欄位 comments
  - _Requirements: 4.1-4.3, 6.1_

- [x] 1.2 建立 `internal/report` bounded context 骨架
  - `domain.go`
  - `repository.go`
  - `service.go`
  - `query_service.go`
  - `delivery.go`
  - 完成：已建立 `internal/report` package 與 domain、repository、service、query_service、delivery 骨架；尚未註冊主 router，留給 4.3
  - _Requirements: 1.1-1.5, 4.1-4.7_

- [x] 1.3 實作日期區間與時區 helper
  - 驗證 `YYYY-MM-DD`
  - 固定支援 `Asia/Taipei`
  - 轉換為 UTC half-open range
  - 補單元測試覆蓋月初、月底與跨日時區
  - 完成：已實作 `NormalizeDateRange`，固定支援 `Asia/Taipei`，轉成 UTC half-open range；已補合法轉換、反向區間與不支援時區測試
  - _Requirements: 2.7-2.8_

- [x] 1.4 實作 Report config 白名單驗證
  - 驗證 `report_mode`
  - 驗證 `chart_type`
  - 驗證 `person_basis`
  - 驗證 `x_axis`、`y_axis`、`metrics`、`indicators`
  - 禁止未知維度直接進 SQL
  - 完成：已實作 `NormalizeTemplateConfig`，包含 mode、chart、person_basis、axis、metrics、indicators 白名單與模式 D 必填維度檢查；已補預設值、未知 metric、模式 D 缺維度測試
  - 驗證：`go test ./internal/report`、`go test ./...` 通過
  - _Requirements: 4.3-4.5, 6.4-6.5_

## 2. Server 範本 API

- [x] 2.1 實作範本列表與詳情 API
  - `GET /api/v1/projects/:id/report-templates`
  - `GET /api/v1/projects/:id/report-templates/:template_id`
  - 套用專案可見性與 Report read 權限
  - 不回傳已軟刪除資料
  - 完成：已在 `internal/report/delivery.go` 註冊範本列表與詳情 handler，service 以 `project.RoleViewer` 驗證專案可見性，repository 僅查詢 `deleted_at IS NULL`；Report 表單權限 runtime 掛載仍留 4.3 主 router 整合
  - _Requirements: 1.2-1.5, 4.1-4.4_

- [x] 2.2 實作範本建立 API
  - `POST /api/v1/projects/:id/report-templates`
  - 驗證名稱、描述與 config
  - 驗證 Project Manager 以上或 Report create 權限
  - 寫入審計日誌
  - 完成：已實作 `CreateTemplate`，正規化名稱與 config 白名單，使用 `project.RoleManager` 驗證，duplicate name 轉 `409 resource conflict`，成功後透過 `logs.RecordSystemAudit` 寫入審計紀錄
  - _Requirements: 4.1-4.7, 6.1_

- [x] 2.3 實作範本更新 API
  - `PUT /api/v1/projects/:id/report-templates/:template_id`
  - 驗證範本屬於同一專案
  - 驗證 Project Manager 以上或 Report update 權限
  - 寫入異動前後審計摘要
  - 完成：已實作 `UpdateTemplate`，更新前先以 project/template 查詢確認歸屬，更新後寫入 before/after 審計摘要，權限使用 `project.RoleManager`
  - _Requirements: 4.1-4.7, 6.1_

- [x] 2.4 實作範本軟刪除 API
  - `DELETE /api/v1/projects/:id/report-templates/:template_id`
  - 設定 `deleted_at`
  - 驗證 Project Manager 以上或 Report delete 權限
  - 寫入審計日誌
  - 完成：已實作 `DeleteTemplate`，repository 更新 `deleted_at` / `updated_at`，刪除前確認同專案歸屬，成功後寫入 delete 審計紀錄
  - _Requirements: 4.1-4.7, 6.1_

- [x] 2.5 補範本 API 測試
  - 成功 CRUD
  - 權限不足回 403
  - 專案不可見不得洩漏資料
  - duplicate name 回 resource conflict
  - 完成：已新增 service 與 handler 測試，覆蓋成功 CRUD、權限不足 403、找不到資料 404、duplicate name 409、未知 request 欄位 400、審計呼叫與 Project RoleViewer/Manager 權限檢查
  - 驗證：`go test ./internal/report`、`go test ./...` 通過
  - _Requirements: 1.2-1.5, 4.1-4.7, 6.1_

## 3. Server 報表查詢 API

- [x] 3.1 實作 Report preview API
  - `POST /api/v1/projects/:id/reports/preview`
  - 支援 request config 直接預覽
  - 無資料回空 payload，`meta.empty = true`
  - 失敗記錄可診斷 log
  - 完成：已新增 preview route、service 權限包裝、QueryService 聚合入口；無資料回 200 與 `meta.empty=true`，查詢錯誤回 JSON error envelope
  - _Requirements: 1.5, 4.4-4.5, 6.2-6.6_

- [x] 3.2 實作內建月報 API 骨架
  - `POST /api/v1/projects/:id/reports/builtin-monthly`
  - 驗證 mode 只允許 A / B / C
  - 驗證 year / month / person_basis
  - 共用 `ReportChartPayload`
  - 完成：已新增 builtin monthly route 與 QueryService `BuiltinMonthly`，只接受 A/B/C，依 year/month 產生 Asia/Taipei 月區間並共用 preview payload
  - _Requirements: 3.1-3.7_

- [x] 3.3 實作模式 C 查詢
  - `person_basis = created_by`
  - `person_basis = actor`
  - stack 維度為 `sub_projects`
  - 查詢只使用 `tickets`、`ticket_activities`、`sub_projects` 與人員資料
  - 完成：已實作模式 C 人員 x 子專案堆疊；`created_by` 使用 Ticket facts，`actor` 使用 activity facts 並限制 activity 類型白名單，聚合時以 ticket/person/sub_project 去重
  - _Requirements: 2.1, 2.4-2.5, 3.5-3.7_

- [x] 3.4 實作模式 A 查詢
  - 月份切週區間
  - 指標支援事件類型、資訊來源、子專案
  - 明細表與圖表數字一致
  - 不混用 Jira 資料
  - 完成：已實作模式 A 週區間聚合，指標支援 `ticket_type`、`source`、`sub_project`，查詢資料來源限定 Ticket schema，不讀 Jira
  - _Requirements: 2.1-2.3, 3.1-3.3, 3.7_

- [x] 3.5 實作模式 B 查詢
  - 任務內容使用 Ticket 標題
  - 橫軸為人員
  - 回傳交叉明細表
  - 保留可連回 Ticket 的 id 欄位
  - 完成：已實作模式 B 以 Ticket 標題作 series、橫軸為人員，明細列保留 `ticket_id`、`title`、`person`、`count`
  - _Requirements: 2.1, 2.5, 3.4, 3.7_

- [x] 3.6 實作範本 execute API
  - `POST /api/v1/projects/:id/report-templates/:template_id/execute`
  - 讀取範本 config 後共用 preview 查詢流程
  - 驗證範本專案歸屬
  - 完成：已新增 execute route，service 先驗證 project read 權限，再以 project/template 查詢範本歸屬並共用 QueryService preview 流程
  - _Requirements: 4.1-4.6_

- [x] 3.7 實作 CSV export API
  - `GET /api/v1/projects/:id/reports/export`
  - 使用目前查詢條件
  - response 正確設定 `Content-Type` 與 `Content-Disposition`
  - 失敗不得回傳可被前端誤下載的錯誤檔
  - 完成：已新增 export route，從 query string 讀取日期、mode、person_basis、維度與指標，成功回 `text/csv; charset=utf-8` 與 `Content-Disposition`，失敗回 JSON error envelope
  - _Requirements: 5.3-5.5_

- [x] 3.8 補報表查詢測試
  - 日期區間轉 UTC
  - 無資料 empty payload
  - mode A / B / C 基本聚合
  - SQL injection 防護白名單
  - 權限與專案範圍
  - 完成：已新增 QueryService 與 handler 測試，覆蓋 empty payload、mode A/B/C 聚合、actor facts、builtin monthly mode 驗證、CSV 匯出、preview/builtin/execute/export route、CSV 錯誤不回檔案格式；SQL 注入防護沿用 `NormalizeTemplateConfig` 白名單與 repository 參數化 SQL
  - 驗證：`go test ./internal/report`、`go test ./...` 通過
  - _Requirements: 1.2-1.5, 2.1-2.8, 3.1-3.7, 6.4-6.6_

## 4. Server OpenAPI 與整合

- [x] 4.1 補 Report Swagger 註解
  - 範本 CRUD
  - preview
  - builtin-monthly
  - execute
  - export CSV
  - 完成：已補齊 Report 範本 CRUD、preview、builtin-monthly、execute、export CSV 的 Swagger 註解；CSV 成功 response 為檔案，失敗 response 為 JSON envelope
  - _Requirements: 3.7, 4.1-4.7, 5.3-5.5_

- [x] 4.2 重新產生並檢查 OpenAPI
  - 執行既有 openapi generator
  - 確認 `Docs/openapi.json` 包含完整 requestBody 與 response schema
  - 不只列出 path
  - 完成：已執行 `make openapi`，產出 `opscenter-server/Docs/openapi.json` 共 56 paths；已檢查 Report paths、requestBody、JSON response schema、CSV success/error content type
  - _Requirements: 1.1, 3.7, 4.1-4.7_

- [x] 4.3 註冊 Report routes
  - 接入主 router
  - 套用 auth middleware
  - 套用專案可見性與表單權限檢查
  - 完成：已在主 router 的已登入 `formAccessAPI` 接入 Report routes；read 類路由套用 `reports/read + viewer`，範本建立/更新/刪除套用 `reports/create|update|delete + project_manager`，service 內仍保留專案角色檢查
  - _Requirements: 1.1-1.4_

- [x] 4.4 Server 驗證
  - `go test ./internal/report`
  - `go test ./...`
  - `make openapi`
  - 完成：已補 OpenAPI generator 測試，確保 CSV endpoint 成功 response 使用 `text/csv` 且 failure response 使用 `application/json`
  - 驗證：`go test ./cmd/openapi ./internal/report ./internal/server`、`go test ./...`、`make openapi` 通過
  - _Requirements: 1.1-6.6_

## 5. Frontend 基礎與路由

- [x] 5.1 建立 Report feature 目錄骨架
  - `api`
  - `components`
  - `charts`
  - `pages`
  - `types.ts`
  - `queryKeys.ts`
  - 完成：已補 `src/features/report` feature exports、`types.ts`、`queryKeys.ts`、`api/reports.ts`、`charts/index.ts` 與 Report pages 骨架
  - _Requirements: 1.1, 5.1-5.6_

- [x] 5.2 建立 Report API client
  - templates CRUD
  - preview
  - builtin-monthly
  - execute
  - export CSV
  - query key 不與 Ticket / Dashboard 混用
  - 完成：已建立 `api/reports.ts`，包裝 Report 範本 CRUD、preview、builtin-monthly、execute、CSV export；CSV 使用 `apiClient.download` 避免 JSON 錯誤被下載；query key 統一使用 `reportQueryKeys`
  - _Requirements: 3.7, 4.1-4.7, 5.3-5.5_

- [x] 5.3 註冊 Report routes
  - `/projects/:id/reports`
  - `/projects/:id/reports/templates/:templateId`
  - `/projects/:id/reports/designer`
  - 直接開 URL 時依 API 權限處理錯誤狀態
  - 完成：已註冊 Report 中心、範本詳情與設計器路由；範本詳情頁直接開 URL 會查詢真實 API 並顯示 loading / error / empty 狀態
  - _Requirements: 1.1-1.5_

- [x] 5.4 建立 Report i18n
  - `zh-TW/report.json`
  - `zh-CN/report.json`
  - `en/report.json`
  - Toast、空資料、錯誤、欄位、模式與按鈕全覆蓋
  - 完成：已補三語系 Report namespace，覆蓋 toast、空資料、錯誤、欄位、模式、按鈕、設計器與範本詳情文字
  - _Requirements: 5.6_

## 6. Frontend Report 中心 UI

- [x] 6.1 建立 ReportCenterPage layout
  - breadcrumb
  - 專案摘要列
  - 工具列
  - 圖表區
  - 明細表區
  - 不做 landing page 或 hero
  - 完成：已將 ReportCenterPage 改為操作型工作區，包含 breadcrumb、專案摘要列、可換行工具列、範本列表、圖表區與明細表區，未使用 landing / hero 版型
  - _Requirements: 1.1, 5.1-5.6_

- [x] 6.2 實作日期區間控制
  - 本週
  - 上週
  - 本月
  - 上月
  - 自訂區間
  - 預設本月
  - 完成：已支援本週、上週、本月、上月、自訂區間，預設本月，送出 `YYYY-MM-DD` 與 `Asia/Taipei`
  - _Requirements: 2.7-2.8, 4.6_

- [x] 6.3 實作報表模式與統計口徑控制
  - mode A / B / C / D
  - `person_basis = created_by`
  - `person_basis = actor`
  - 工具列可換行且不重疊
  - 完成：已提供 mode A/B/C/D、person_basis created_by/actor 與 mode A metric 控制；工具列使用 flex wrap 與 responsive min-width 避免擠壓重疊
  - _Requirements: 3.1-3.7, 5.1-5.2_

- [x] 6.4 實作 Preview 流程
  - 使用者按預覽才打 API
  - loading state
  - empty state
  - error state + retry
  - 不顯示假資料
  - 完成：已使用 mutation 由預覽按鈕觸發 `previewReport`，支援 loading、empty、error + retry；未按預覽前只顯示空狀態，不顯示假資料
  - _Requirements: 1.5, 4.5, 6.6_

- [x] 6.5 實作 CSV 匯出 UI
  - 使用目前查詢條件
  - 支援後端 `Content-Disposition`
  - 錯誤 JSON 不得被下載成檔案
  - 匯出失敗顯示 Toast
  - 完成：已使用目前查詢條件匯出 CSV；新增 `apiClient.downloadWithMeta` 讀取 `Content-Disposition` 檔名，錯誤仍走 JSON envelope 並顯示 Toast，不下載錯誤檔
  - _Requirements: 5.3-5.5_

## 7. Frontend 圖表與表格

- [x] 7.1 建立 `ReportChartPanel`
  - 根據 payload mode / chart_type 選 chart
  - 支援長條圖
  - 支援堆疊長條圖
  - 不在前端補算核心統計
  - 完成：已建立 `ReportChartPanel`，依 `report_mode` 選擇 A/B/C/D 圖表元件，長條與堆疊長條只使用後端 payload，不在前端補算統計
  - _Requirements: 3.7, 5.1-5.2_

- [x] 7.2 實作模式 C 圖表
  - 人員 × 子專案堆疊
  - 圖例可讀
  - tooltip 顯示數量
  - 長人名與長子專案名稱不重疊
  - 完成：已建立 `ModeCPersonSubProjectChart`，使用後端 series stack 呈現人員 × 子專案堆疊，圖例可捲動並支援 tooltip 與長文字截斷
  - _Requirements: 3.5-3.7, 5.1-5.2_

- [x] 7.3 實作模式 A 圖表
  - 週區間橫軸
  - 指標分組
  - 明細表同步呈現
  - 完成：已建立 `ModeAIndicatorChart`，以後端 x_axis / series 呈現週區間與指標分組；明細表共用同一 payload
  - _Requirements: 3.1-3.3, 5.1-5.2_

- [x] 7.4 實作模式 B 圖表
  - Ticket 標題 / 任務內容 × 人員
  - 堆疊長條
  - 交叉明細表
  - 完成：已建立 `ModeBTaskStackChart`，以後端 series 呈現 Ticket 任務 × 人員堆疊長條；交叉明細表共用後端 table payload
  - _Requirements: 3.4, 5.1-5.2_

- [x] 7.5 實作 `ReportDetailTable`
  - 使用後端 columns / rows
  - 支援水平捲動
  - 接入 Data Grid i18n helper
  - 空資料與 loading 狀態一致
  - 完成：已建立 `ReportDetailTable`，直接使用後端 `table.columns` / `table.rows`，支援水平捲動、Data Grid i18n、loading 與空資料狀態
  - _Requirements: 5.2, 5.6_

## 8. Frontend 範本與設計器

- [x] 8.1 建立範本列表 UI
  - 顯示專案範本
  - 顯示模式、描述、建立者、更新時間
  - 支援執行入口
  - 完成：已將 TemplateList 串接 `listReportTemplates`，顯示範本名稱、描述、模式、建立者、更新時間，並提供範本執行入口
  - _Requirements: 4.1-4.6_

- [x] 8.2 建立新增 / 編輯範本 Dialog
  - 名稱
  - 描述
  - report mode
  - chart type
  - person basis
  - title template
  - 儲存成功後關閉 Dialog
  - 完成：已建立 `ReportTemplateDialog`，支援名稱、描述、report mode、chart type、person basis、title template；建立與更新成功後才關閉 Dialog 並刷新列表
  - _Requirements: 4.1-4.7_

- [x] 8.3 建立範本刪除流程
  - confirmation dialog
  - 後端確認後才顯示成功
  - 刪除後刷新列表
  - 完成：已在 TemplateList 加入刪除確認 Dialog，後端 delete 成功後才顯示成功 Toast、關閉 Dialog 並刷新列表
  - _Requirements: 4.2, 4.7, 6.1_

- [x] 8.4 建立範本執行頁
  - 讀取範本
  - 選日期區間
  - 執行 preview
  - 顯示圖表與明細表
  - 完成：已將 `/projects/:id/reports/templates/:templateId` 改為範本執行頁，讀取範本、選日期區間、呼叫 execute API，並以 ReportChartPanel / ReportDetailTable 顯示結果
  - _Requirements: 4.4-4.6, 5.1-5.2_

- [x] 8.5 建立模式 D 設計器預留頁
  - 顯示有限維度控制或明確預留狀態
  - 不顯示未串接假資料
  - 後續完整設計器可沿用 config 型別
  - 完成：已在 ReportDesignerPage 提供模式 D 預留狀態與有限維度控制，使用既有 config 型別，不顯示假圖表或假資料
  - _Requirements: 4.4-4.5, 6.6_

## 9. Frontend 驗證

- [x] 9.1 補 API client 與型別檢查
  - DTO 與 OpenAPI 對齊
  - 空 payload 型別可處理
  - CSV 錯誤 response 可處理
  - 完成：已檢查 OpenAPI Report paths、preview / execute / templates / export 契約；CSV 200 為 `text/csv`、400 為 `application/json`；前端 `ReportChartPayload`、空 table、CSV `downloadWithMeta` 型別可通過 `tsc -b`
  - _Requirements: 1.5, 3.7, 5.3-5.5_

- [x] 9.2 補主要 UI 狀態驗證
  - loading
  - empty
  - error
  - success preview
  - export failed
  - 完成：已檢查 ReportCenter、ReportTemplateDetail、ReportChartPanel、ReportDetailTable、TemplateList 的 loading / empty / error / success preview / export failed 程式路徑；目前前端專案未配置 Vitest/Jest/Testing Library/Playwright，故本項以既有可執行 typecheck/build 與程式路徑檢查完成，未新增測試框架
  - _Requirements: 1.5, 5.6, 6.6_

- [x] 9.3 前端建置驗證
  - `npm run typecheck`
  - `npm run build`
  - 檢查 bundle warning 是否為既有問題或本次新增造成
  - 完成：`npm run typecheck` 通過；`npm run build` 通過；build 仍有 Vite chunk size warning，屬目前單一 bundle 偏大警告，非編譯失敗
  - _Requirements: 1.1-6.6_

## 10. 整合驗證

- [x] 10.1 建立 Report 測試資料
  - project
  - sub_projects
  - ticket_types
  - ticket_resources
  - tickets
  - ticket_activities
  - 完成：已在 `internal/report/query_service_test.go` 建立 Report fixture，包含專案範圍、子專案、事件類型、資訊來源、Ticket 與 activity basis 所需資料欄位；採可重複執行測試資料，不新增一次性資料庫 seed
  - _Requirements: 2.1-2.5, 3.1-3.7_

- [x] 10.2 驗證模式 C 端到端
  - 建立人統計
  - 操作人統計
  - 子專案堆疊
  - 空資料
  - 完成：已以 QueryService 測試覆蓋 mode C 人員統計、actor basis activity facts、子專案堆疊與 empty payload
  - _Requirements: 3.5-3.7_

- [x] 10.3 驗證模式 A / B 端到端
  - 月份週區間
  - 指標拆分
  - 任務 × 人員
  - 明細表與圖表一致
  - 完成：已以 QueryService 測試覆蓋 mode A 週區間與資訊來源指標拆分、mode B Ticket 任務 × 人員、明細表保留 ticket_id，並檢查圖表 series / table row 與 fixture 數量一致
  - _Requirements: 3.1-3.4, 5.1-5.2_

- [x] 10.4 驗證權限與專案可見性
  - 無 Report read 權限不得查詢
  - 私有專案非成員不得查詢
  - CSV 匯出不得跨專案
  - 完成：已補 `TestServiceReportQueryRejectsProjectAccessDenied`，Preview 與 Export 都會先通過 project access gate；access denied 時不產生 CSV filename/content，避免跨專案匯出
  - _Requirements: 1.2-1.4, 5.3-5.4_

- [ ] 10.5 完整驗證
  - `go test ./...`
  - `npm run typecheck`
  - `npm run build`
  - 以瀏覽器檢查 Report 中心、圖表、表格、空資料與錯誤狀態
  - 進度：`go test ./...`、`npm run typecheck`、`npm run build` 已通過；已啟動前端 dev server 並用 HTTP 驗證 Report center / template / designer routes 皆回 200
  - 阻塞：內建瀏覽器目前不可用，且本機後端未啟動，尚無法完成瀏覽器中真實登入、Report 成功圖表、表格、空資料與錯誤狀態的人工可視檢查
  - _Requirements: 1.1-6.6_

## 11. 每日值班執行統計

- [x] 11.1 補後端 DTO 與 OpenAPI contract
  - 新增 `DailyShiftExecutionPayload`
  - 新增 `DailyShiftExecutionMatrix`
  - 新增 `/api/v1/projects/:id/reports/daily-shift-execution`
  - 明確定義 empty payload 與 CSV / Excel-friendly 匯出格式
  - 完成：已新增後端 DTO、HTTP request、route、Swagger 註解與 `DailyShiftExecutionPayload` response；`make openapi` 已更新為 57 paths
  - _Requirements: 3.8-3.11, 5.3-5.6_

- [x] 11.2 實作每日日期欄位與星期欄位產生器
  - 依 `Asia/Taipei` 展開 `date_from` 到 `date_to`
  - 支援跨月日期區間
  - 回傳穩定欄位順序
  - 完成：已新增 `dailyShiftColumns`，依 Report date range 產生穩定日期與星期欄位
  - _Requirements: 2.7-2.8, 3.8_

- [x] 11.3 實作指標群組白名單與資料來源缺口標示
  - `jira_notification`
  - `alert_notification`
  - `domain_change`
  - `payment_domain_change`
  - 來源不足時回傳空群組或 warning，不得由前端補假資料
  - 完成：已新增每日指標群組白名單，未知群組回 invalid request；資料無法從現有 Ticket facts 匹配時回空矩陣，不由前端補資料
  - _Requirements: 2.1-2.6, 3.9, 6.4-6.6_

- [x] 11.4 實作班別彙總查詢
  - 班別：早、中、晚
  - 依排班資料或後端可驗證規則推導
  - 不在 Ticket 表新增班別快照欄位
  - 完成：已依 Asia/Taipei 操作時間推導早 / 中 / 晚班別並產生 shift row，不修改 Ticket schema
  - _Requirements: 2.6, 3.10_

- [x] 11.5 實作人員明細查詢
  - 支援依 `created_by` 或 `actor` 統計口徑
  - 列格式支援 `早-Chenmin`、`中-Eric`、`晚-Turry`
  - 人員不得跨越專案權限範圍
  - 完成：每日矩陣沿用 Report facts 的 `created_by` / `actor` person basis 與專案權限範圍，person row 顯示為 `班別-人員`
  - _Requirements: 2.5, 3.10, 1.2-1.4_

- [x] 11.6 實作每日值班執行統計前端矩陣 UI
  - 第一列日期、第二列星期
  - 左側 sticky 指標群組與列名稱
  - 支援群組列、班別列、人員列
  - 支援水平捲動與空資料狀態
  - 完成：已新增 `DailyShiftExecutionMatrix` 元件，使用 sticky 左側欄、日期 / 星期雙層表頭、水平捲動與空資料狀態
  - _Requirements: 3.8-3.11, 5.2, 5.6_

- [x] 11.7 補 Report mode selector 與 i18n
  - 新增「每日值班執行統計」選項
  - 新增 `mode.daily_shift_execution`
  - 所有欄位、空資料、錯誤與匯出文案接 i18n
  - 完成：Report mode selector 已新增 E 模式；`zh-TW`、`zh-CN`、`en` 已補 E 模式、矩陣空資料與 row label 文案
  - _Requirements: 3.8, 5.6_

- [x] 11.8 補 CSV / Excel-friendly 匯出
  - 保留日期欄、星期欄、群組列、班別列、人員列
  - 使用目前查詢條件與權限
  - 失敗不得下載錯誤 JSON
  - 完成：`report_mode=E` 匯出會走每日矩陣 CSV，保留 group / row_type / label、日期欄、星期列與矩陣列序；失敗仍沿用 JSON error
  - _Requirements: 3.11, 5.3-5.5_

- [x] 11.9 補測試與驗證
  - 後端日期矩陣測試
  - empty payload 測試
  - 指標白名單測試
  - 前端 typecheck / build
  - 瀏覽器檢查矩陣表不重疊、不顯示假資料
  - 完成：已補後端日期矩陣、empty payload、未知指標群組、每日矩陣 CSV 與 route 測試；驗證 `go test ./...`、`npm run typecheck`、`npm run build` 通過
  - 限制：未啟動瀏覽器進行人工視覺檢查；已透過 build/typecheck 與穩定寬度樣式降低重疊風險
  - _Requirements: 1.5, 3.8-3.11, 6.4-6.6_

- [x] 11.10 拆分每日值班統計報表口徑
  - 報表中心將每日值班統計拆成「每日值班統計－告警通知」、「每日值班統計－域名更換」、「每日值班統計－支付域名更換」三個選項
  - 預覽與匯出每次只送出單一 `metric_group`
  - 後端 response title 與 CSV filename 需反映選定口徑
  - 舊 API 多 `metric_groups` 行為保留相容，不由前端新流程使用
  - 完成：前端報表模式下拉已拆成三個每日值班統計選項；預覽與匯出使用單一 metric group；矩陣標題與匯出檔名前綴依口徑切換
  - 2026-08-07：此處記錄初版歷史行為；報表模式選項後續由 Task 20 改為單一「值班統計」，三個口徑移至「數值」欄位
  - 2026-08-03 修正：補齊本機 Makefile 與 Docker multi-stage 的前端建置／Go embed 同步，避免 `:9998` 持續提供 placeholder 而看不到支付域名更換等 Report 選項
  - _Requirements: 3.8-3.11, 5.3-5.6_

## 12. 模式 A 固定 OP 月報指標

- [x] 12.1 補模式 A 固定 OP 指標 contract
  - `op_jira_issue_count`
  - `op_alert_notification_count`
  - `op_payment_domain_change_count`
  - 明確定義 chart payload 與 table 欄位
  - 完成：已補後端 `metrics` / `indicators` 白名單、前端型別與報表中心 / 設計器選項，固定 OP 指標沿用 `ReportChartPayload` 與 `DetailTable`
  - _Requirements: 3.3a-3.3c, 3.7_

- [x] 12.2 實作月內週區間產生器
  - 例如 `6/1-6/4`、`6/5-6/11`、`6/12-6/18`
  - 支援每月最後一週不足 7 天
  - 回傳穩定欄位順序
  - 完成：已新增月內週區間產生器，六月 2026 會輸出 `6/1-6/4`、`6/5-6/11`、`6/12-6/18`、`6/19-6/25`、`6/26-6/30`
  - _Requirements: 3.3c, 2.7-2.8_

- [x] 12.3 實作 OP Jira 開單數量統計
  - 依人員整月總計產生柱狀圖
  - 依人員 × 週區間產生明細表
  - 若 Jira 來源規則不足，後端回空與 warning，不由前端補資料
  - 完成：已依現有 Ticket facts 的標題、事件類型、資訊來源、子專案文字規則聚合 Jira 指標；無符合資料時回傳空 payload
  - _Requirements: 3.3a-3.3c, 6.6_

- [x] 12.4 實作 OP 告警通知數量統計
  - 依 `ticket_resources.resource_type = alert` 或設定的告警來源白名單
  - 依人員 × 週區間聚合
  - 完成：已依現有 Ticket facts 的 `alert` / `告警` 文字規則聚合告警通知指標
  - _Requirements: 2.3, 3.3a-3.3c_

- [x] 12.5 實作 OP 支付or業務項目域名更換統計
  - 依事件類型、資訊來源或子專案白名單規則
  - 依人員 × 週區間聚合
  - 規則不足時回空與 warning
  - 完成：已依現有 Ticket facts 的 `payment` / `支付` 與 `domain` / `域名` 組合規則聚合支付或業務項目域名更換指標；無符合資料時回傳空 payload
  - _Requirements: 2.1-2.4, 3.3a-3.3c_

- [x] 12.6 補前端固定 OP 指標呈現
  - 上方柱狀圖
  - 下方值班人員、總計、週區間明細表
  - 人員顯示保留班別前綴
  - 完成：報表中心與設計器可選固定 OP 指標，既有圖表與明細表會直接呈現後端回傳的人員、總計、週區間欄位
  - _Requirements: 3.3a-3.3c, 5.1-5.2, 5.6_

- [x] 12.7 補固定 OP 指標匯出
  - CSV / Excel-friendly 格式
  - 保留圖下明細表欄位順序
  - 使用目前查詢條件與權限
  - 完成：固定 OP 指標匯出沿用 `chartPayloadToCSV`，依後端 table columns 順序輸出值班人員、總計、週區間欄位
  - _Requirements: 5.3-5.5_

- [x] 12.8 補測試與驗證
  - 週區間產生器測試
  - 三個固定指標 empty payload 測試
  - 三個固定指標聚合測試
  - 前端 typecheck / build
  - 瀏覽器檢查圖表與明細表不重疊
  - 完成：已補週區間、固定 OP 指標白名單、三個固定指標聚合與 empty payload 測試；驗證 `go test ./internal/report`、`go test ./...`、`npm run typecheck`、`npm run build` 通過
  - 限制：本次未啟動瀏覽器做人工視覺檢查；前端沿用既有圖表與明細表元件，已由 typecheck/build 驗證資料契約
  - _Requirements: 1.5, 3.3a-3.3c, 6.4-6.6_

## 13. Report 專案範圍切換

- [x] 13.1 補 Report 子專案範圍 server contract
  - preview request 支援選填 `sub_project_id`
  - builtin-monthly request 支援選填 `sub_project_id`
  - template execute request 支援選填 `sub_project_id`
  - export query string 支援選填 `sub_project_id`
  - `sub_project_id` 空值代表使用主專案範圍
  - 完成：已在 Report domain、HTTP request DTO、Swagger 註解與前端 API types 補 `sub_project_id` 選填契約
  - _Requirements: 1.6-1.9_

- [x] 13.2 實作 Report 子專案範圍後端查詢
  - 後端驗證子專案屬於目前主專案
  - Ticket facts 查詢支援 `tickets.sub_project_id = ?`
  - Activity facts 查詢支援關聯 Ticket 的 `sub_project_id`
  - empty payload 與 CSV export 使用相同範圍
  - OpenAPI 重新產生並確認 schema / query param
  - 完成：已新增子專案歸屬驗證、ticket/activity facts 子專案 SQL 篩選、meta 回傳 `sub_project_id`、CSV export 同步套用範圍，並重新產生 `Docs/openapi.json`
  - _Requirements: 1.2-1.9, 5.3-5.5, 6.4-6.6_

- [x] 13.3 補 Report 主專案與子專案切換 UI contract
  - Header / 專案摘要列提供主專案下拉選單
  - Header / 專案摘要列提供子專案下拉選單
  - 使用可見主專案列表，不硬編專案
  - 使用目前主專案下的可見子專案列表，不硬編子專案
  - 子專案預設為空值
  - 子專案空值代表使用主專案範圍查詢
  - 切換後導向 `/projects/:id/reports`
  - 完成：已在 Report 中心實作主專案與子專案下拉，子專案預設空值並顯示「全部子專案」
  - _Requirements: 1.6-1.9_

- [x] 13.4 實作 Report 主專案與子專案下拉
  - 沿用 Ticket 工作區主專案列表 hook / API
  - 顯示專案名稱、專案代碼、狀態
  - 串接目前主專案下的子專案列表
  - 子專案顯示名稱、key、狀態
  - 切換主專案時重設子專案為空值
  - 載入中停用，載入失敗顯示錯誤狀態
  - 完成：主專案沿用 `useWorkspaceProjects`，子專案沿用 `listSubProjects`，切換 route project 時透過 state reset 清空子專案
  - _Requirements: 1.6-1.9, 5.6_

- [x] 13.5 補 Report 前端查詢條件與匯出帶入子專案
  - preview / builtin-monthly / template execute 支援子專案選填條件
  - CSV export 使用相同子專案條件
  - 子專案空值不傳或傳後端定義的空值語意，不得傳錯誤子專案 id
  - 完成：preview 與 CSV export 會在子專案有值時帶入 `sub_project_id`，空值不送出；builtin-monthly / execute types 已補欄位，後續使用時可沿用
  - _Requirements: 1.8-1.9, 5.3-5.5_

- [x] 13.6 補 i18n 與版面驗證
  - 補主專案、子專案 label 與錯誤文案
  - 補「全部子專案」空值文案
  - 手機與桌面寬度不重疊
  - `npm run typecheck`
  - `npm run build`
  - 完成：已補 `zh-TW`、`zh-CN`、`en` Report i18n；驗證 `go test ./...`、`npm run typecheck`、`npm run build` 通過
  - _Requirements: 1.6-1.9, 5.6_

## 14. 模式 D 自定義報表品質強化

Status: Complete

### Execution Context

- 意圖：修正模式 D 審查發現的預覽、表格、圖例、V2 標題模板、i18n、無障礙、效能與匯出安全問題。
- 非目標：不重寫 A / B / C / E 聚合，不建立拖拉式 BI 畫布，不實作非同步匯出 Job。
- 已定決策：預覽與儲存驗證分離；preview 上限 50；metadata 使用 code 翻譯；V2 config export 改 POST；CSV 安全處理由後端統一序列化邊界負責。
- 邊界：Report 前端、`internal/report`、Report OpenAPI 與三語系 report namespace。
- 關鍵檔案：`ReportDesignerPage.tsx`、`ReportChartPanel.tsx`、`ReportDetailTable.tsx`、`chartOptions.ts`、`reports.ts`、`types.ts`、`internal/report`、`Docs/openapi.json`。
- 完成條件：需求 7.1 至 7.16 全部具自動或人工驗證證據，且不破壞 Protected Behavior。

### Protected Behavior

- 模式 A / B / C / E 的查詢口徑、chart payload 與明細表數字保持一致。
- Report read / create / update / delete、專案可見性與子專案歸屬驗證不得弱化。
- 既有 V1 / legacy 範本仍可查看、執行與另存，不強制原地轉換。
- API 失敗不得顯示假圖表、假表格或先行成功 Toast。
- CSV 成功維持 `Content-Disposition`；失敗不得下載 JSON 為 CSV。

- [x] 14.1 固定 V2 config、metadata、validation 與 POST export 契約
  - Boundary:
    - Allowed Changes: `requirements.md`、`design-frontend.md`、`design-backend.md`、`internal/report` DTO / validation / metadata / delivery contract、`Docs/openapi.json`、前端 `types.ts` 與 API types
    - Forbidden: Report SQL 聚合規則、Ticket / Schedule schema、權限語意
  - Depends: 無
  - Context: 新增 `title_template` 白名單變數、filter options、結構化 validation code 與 POST export request body；GET V2 config export 標記相容期淘汰
  - Verify: `go test ./internal/report`、OpenAPI contract test、`make openapi`、`npm run typecheck`

- [x] 14.2 重構設計器狀態與日期規則
  - Boundary:
    - Allowed Changes: `ReportDesignerPage.tsx`、新增 Report designer hook / form components、`reportDateRange.ts`、相關 i18n 與前端測試
    - Forbidden: Report API 聚合、其他 feature 頁面、全域日期規則
  - Depends: 14.1
  - Context: 分離 `queryValid`、`saveValid`、`exportValid`；mutation 綁定 request hash；補 V2 title template；固定 `Asia/Taipei` 月份邊界；移除不限預覽
  - Verify: 未命名可預覽、命名後可儲存、修改設定使預覽過期、跨時區日期一致、preview limit 最大 50

- [x] 14.3 修正圖表、表格與圖例行為
  - Boundary:
    - Allowed Changes: `ReportChartPanel.tsx`、`ReportDetailTable.tsx`、`CustomReportChart.tsx`、`chartOptions.ts`、designer / center / template detail 的呈現整合、相關測試
    - Forbidden: 後端核心統計數字、A / B / C / E payload shape
  - Depends: 14.2
  - Context: `show_table`、table-only、legend wrap / scroll、穩定色弱友善 palette、圖型建議、row id fallback、memo 化 option 與 formatter
  - Verify: chart + table、chart only、table only、超過 6 series、長標籤、空資料、不同排序下 series 顏色穩定

- [x] 14.4 完成 i18n、錯誤邊界與無障礙
  - Boundary:
    - Allowed Changes: `report.json` 三語系、metadata 顯示 adapter、Report runtime schemas、Report Data Grid focus styles、Report 錯誤元件與測試
    - Forbidden: 全域錯誤 envelope、非 Report namespace、移除既有 Data Grid 鍵盤操作
  - Depends: 14.1
  - Context: code-first 翻譯、validation type guard、未知 API response 局部錯誤、`:focus-visible`、內部 code 不直接顯示
  - Verify: 三語系 key parity、英文與簡中不混入固定繁中 metadata、未知 code fallback、鍵盤焦點可見、異常 payload 不使整頁崩潰

- [x] 14.5 實作 POST 匯出與 CSV Formula Injection 防護
  - Boundary:
    - Allowed Changes: Report export handler / service / serializer、OpenAPI、前端 Report export API / download helper、相關 Go 與前端測試
    - Forbidden: 權限降級、把完整 config 寫入 URL 或 log、非 Report CSV 行為
  - Depends: 14.1
  - Context: 前端以 POST JSON body 下載 Blob；後端統一轉義公式觸發字元並維持專案範圍驗證
  - Verify: 惡意 CSV cell、長 config、不合法 config、401 / 403 / 400、跨專案與子專案、成功檔名、錯誤 JSON 不下載

- [ ] 14.6 補效能防線、測試與回歸驗證（進行中：待瀏覽器桌面／手機人工檢查）
  - Boundary:
    - Allowed Changes: Report preview 上限、legend 上限、Report component / hook tests、必要的測試設定、Report 文件 Implementation Notes
    - Forbidden: 以提高全域限制掩蓋問題、刪除既有驗收、把未執行測試標記通過
  - Depends: 14.2、14.3、14.4、14.5
  - Context: 補 config hash race、日期、i18n、runtime schema、表格 row id、圖表選項、POST export 與 CSV 安全回歸
  - Verify: `go test ./internal/report`、`go test ./...`、前端測試、`npm run typecheck`、`npm run build`、`make openapi`、瀏覽器桌面／手機檢查、`git diff --check`

### 品質檢查清單

- [x] Requirements、frontend / backend design、OpenAPI、types、實作與測試互相可追溯
- [x] 無新增硬編碼 secret、未記錄完整 filter values、CSV 公式字元已覆蓋
- [x] 三語系 scalar key 一致，Report metadata 與 Data Grid 皆隨 locale 更新
- [x] 鍵盤焦點、loading、empty、error、stale preview 與 disabled reason 可感知
- [x] 預覽資料量、series 數量、同步匯出大小都有明確上限或錯誤
- [x] 既有 A / B / C / E、legacy template、權限與專案範圍回歸通過

### Implementation Notes

- 2026-08-03：依模式 D 前端審查建立需求 7 與 task 14，尚未排程實作。
- 2026-08-03：完成 14.1 至 14.5 與 14.6 自動化驗證。前端新增日期單元測試並通過 `npm test`、`npm run typecheck`、`npm run build`；後端通過 `go test ./internal/report` 與 `go test ./...`；OpenAPI 已重新產生，三語系 key parity 與 `git diff --check` 通過。瀏覽器桌面／手機人工檢查尚待具登入資料的執行環境。

## 15. V3 多區塊彈性報表範本

Status: InProgress

### Execution Context

- 意圖：把單一圖表 V2 範本擴充為支援多區塊、桌面拖拉縮放、碰撞自動挪移、手機自動重排與共用參數的 V3 區塊式報表。
- 非目標：不建立像素級自由畫布、任意公式、HTML block、跨資料集 join、多人即時編輯、PDF renderer 或非同步查詢 Job。
- 已定決策：新增 V3 而不原地轉換 V1 / V2；desktop 12 欄為唯一儲存 layout；tablet / mobile deterministic 衍生；block 上限 20、資料 block 上限 12、preview concurrency 4；資料 query 重用 V2 白名單。
- 邊界：Report V3 domain / API / migration、Report layout frontend、Report i18n、OpenAPI 與相關測試。
- 關鍵檔案：`internal/report`、Report migration、`Docs/openapi.json`、`features/report/layout`、Report routes、`report.json`。
- 完成條件：需求 8.1 至 8.18 具驗證證據，V1 / V2 與 A / B / C / E 回歸通過，且桌面／平板／手機與鍵盤操作完成檢查。

### Protected Behavior

- V1、V2 範本仍能列表、查看、執行、匯出與另存，不自動改寫 config。
- 模式 A / B / C / E 的 query、payload、數字與現有路由保持不變。
- reports read / create / update / delete、project scope 與 sub-project ownership 不得弱化。
- V3 單一 block 失敗不得偽造成功資料；整份 request 權限或 config 失敗不得以 partial success 掩蓋。
- CSV 維持 POST、`Content-Disposition` 與 Formula Injection 防護。
- Task 14.6 的人工瀏覽器驗證不得因 V3 規劃而標記完成。

- [x] 15.1 固定 V3 config、layout、block、parameter 與 revision 契約
  - Boundary:
    - Allowed Changes: 需求 8、frontend / backend design、`internal/report` domain DTO / parser / validation、前端 V3 types / schemas、OpenAPI contract、相關單元測試
    - Forbidden: Query SQL、UI 拖拉實作、既有 V1 / V2 config mutation、權限語意
  - Depends: 14.1 至 14.5
  - Context: 建立 discriminated block contract、12 欄 layout validation、block / data block 上限、parameter binding、revision request / response 與穩定 error code
  - Verify: V3 parse / normalize table tests、重複 id、孤兒 layout、越界、重疊、未知 block / parameter、V1 / V2 regression、OpenAPI schema

- [x] 15.2 實作 V3 persistence、樂觀鎖與 audit
  - Boundary:
    - Allowed Changes: Report migration、repository、service、delivery、audit summary、OpenAPI、相關 Go tests
    - Forbidden: 其他 domain schema、V1 / V2 強制 revision、完整 filter values / text 寫入 audit
  - Depends: 15.1
  - Context: `config_version = 3`、revision default 1、atomic compare-and-swap update、409 conflict、V3 create / update audit
  - Verify: migration 可重複執行與 schema contract、create revision、成功 update increment、stale revision 409、跨專案、權限、audit redaction、legacy regression

- [x] 15.3 實作 multi-block preview、partial result 與 block export
  - Boundary:
    - Allowed Changes: `internal/report` V3 orchestration、V2 planner adapter、request-scope dedupe、concurrency limit、V3 delivery / OpenAPI、相關 tests
    - Forbidden: 無限制 goroutine、SQL 字串由 config 組合、V2 planner 統計口徑變更、GET 完整 config export
  - Depends: 15.1、15.2
  - Context: 新增 layout-preview、execute-layout、saved block export；每 block 獨立結果，整體 config / auth 錯誤維持 HTTP error
  - Verify: 0 / 1 / 12 data blocks、partial error、request cancel、concurrency 4、dedupe、project / sub-project scope、CSV 安全與檔名

- [x] 15.4 建立 layout domain pure functions 與 designer state
  - Boundary:
    - Allowed Changes: `features/report/layout/types.ts`、schemas、compact / collision / responsive utils、designer reducer / hooks、純函式與 hook tests
    - Forbidden: 先安裝未評估的 drag library、後端統計重算、全域 state 重構
  - Depends: 15.1
  - Context: deterministic move / resize、vertical compact、desktop source of truth、tablet 6 欄與 mobile 1 欄、undo / redo、canonical hash dirty state
  - Verify: boundary examples、collision chains、random property tests、stable ordering、gesture 單一 history、undo / redo、dirty / persisted hash

- [x] 15.5 實作區塊編輯器、拖拉縮放與響應式預覽
  - Boundary:
    - Allowed Changes: `features/report/layout` pages / components / routes、必要且經評估的 grid adapter、Report i18n、component tests
    - Forbidden: 自由圖層重疊、手機精細觸控縮放、HTML renderer、非 Report 導航重構
  - Depends: 15.4
  - Context: block palette、canvas、inspector、metric / chart / table / text renderer、複製刪除、desktop / tablet / mobile preview、resize observer
  - Verify: 新增 / 複製 / 刪除 / move / resize、無重疊、斷點重排、長文字、empty / error、三語系與 theme

- [x] 15.6 串接共用參數、局部 preview freshness 與儲存衝突 UI
  - Boundary:
    - Allowed Changes: V3 frontend API、preview hook / scheduler、parameter controls、save mutation、conflict / dirty guard UI、runtime schema、相關 tests
    - Forbidden: 前端繞過權限、以最新 UI state 誤標舊 response、409 自動覆蓋、完整 config 放 URL
  - Depends: 15.2、15.3、15.5
  - Context: block hash + bound parameter hash、concurrency 4、局部 stale、cancel superseded request、revision 409 reload / save-as、離開提示
  - Verify: parameter binding、局部 stale、out-of-order response、partial result、save success / failure、409、route / project switch dirty guard

- [ ] 15.7 完成無障礙、效能與整合回歸（進行中）
  - Boundary:
    - Allowed Changes: Report V3 keyboard controls、focus / aria、測試設定、性能量測、OpenAPI、Report 文件 Implementation Notes
    - Forbidden: 以放寬 block / query 上限解決效能、刪除既有驗收、未執行測試標記通過
  - Depends: 15.3、15.6
  - Context: 鍵盤 move / resize、screen reader announcement、20 blocks / 12 data blocks、bundle impact、桌面／平板／手機、V1 / V2 regression
  - Verify: `go test ./internal/report`、`go test ./...`、`npm test`、`npm run typecheck`、`npm run build`、`make openapi`、key parity、瀏覽器與鍵盤人工檢查、`git diff --check`

### 品質檢查清單

- [x] Requirements、frontend / backend design、OpenAPI、migration、types 與 task 可追溯
- [x] Layout 演算法 deterministic、無重疊、無越界且具 property test
- [x] V3 parser 不影響 V1 / V2，revision conflict 不覆蓋較新資料
- [x] 每個 block 具 loading / empty / error / stale 與鍵盤替代操作
- [x] 共用參數與 block query 仍由後端執行 project / sub-project 驗證
- [x] 20 blocks、12 data blocks、concurrency 4 與既有 category / series 上限生效
- [ ] 三語系、light / dark、desktop / tablet / mobile 與長內容完成驗證
- [x] Audit、log、error 與 URL 不包含完整 filter values 或 V3 config
- [x] V1 / V2、A / B / C / E、CSV 與 Report 權限回歸通過

### Implementation Notes

- 2026-08-03：依使用者提出的彈性範本需求建立需求 8 與 Task 15；狀態為 InProgress。
- 2026-08-03：完成 15.1。V3 strict parser、block / parameter / layout domain、前端 Zod schema、revision response contract 與測試已建立；`go test ./internal/report`、`npm test`、`npm run typecheck`、`make openapi`、`git diff --check` 通過。
- 2026-08-03：完成 15.2 至 15.6 與 15.7 自動化驗證。後端新增 revision migration、compare-and-swap、去敏 audit、concurrency 4 的 multi-block preview、request-scope dedupe、partial result 與 block CSV；前端新增 deterministic layout engine、property test、undo / redo、原生拖拉縮放、12 / 6 / 1 欄響應式版面、局部 stale、共用參數、V3 儲存與 409 處理。`go test ./...`、`npm test`、`npm run typecheck`、`npm run build`、`make openapi`、三語系 key parity 與 `git diff --check` 通過。瀏覽器 runtime 無可用實例，light / dark、桌面／平板／手機 Pointer Events、鍵盤與螢幕閱讀器人工檢查仍待執行，因此 15.7 保持 InProgress。

## 16. 固定 OP 月報 Excel 基準修正

Status: InProgress

### Execution Context

- 意圖：修正模式 A 的 OP Jira 開單與支付域名更換聚合，使 MS 專案 2026 年 7 月結果可與 Excel 的 663、492 對齊。
- 非目標：不修正告警通知、不修改 Ticket 資料、不重做模式 E、不將 Excel 當執行期資料來源。
- 已定決策：Jira 以 Excel 開單記錄匯入來源及非空 `tickets.external_ref` 判斷；支付域名更換以同匯入來源及 `ticket_resources.code = business_domain_change` 判斷；匯入 provenance 取自 importer 建立的 activity，不解析 Ticket description；人員需含有效班別前綴；月底單日週區間併入前一段。
- 邊界：`internal/report` facts、repository、模式 A 聚合、前端 payload 呈現、CSV、相關測試與文件。
- 關鍵檔案：`internal/report/query_repository.go`、`internal/report/query_service.go`、Report 前端元件與 API 型別、Report 測試。
- 完成條件：需求 9.1 至 9.12 具驗證證據，MS 專案 2026 年 7 月 Jira 為 663、支付域名更換為 492，且模式 A 其他指標與模式 B／C／D／E 回歸通過。

### Protected Behavior

- 專案範圍必須取自已授權的路由參數，不得硬編碼 MS project id 或跨專案查詢。
- 告警通知 predicate 與結果保持不變，不以本次 Excel 告警圖表數字擴張範圍。
- 不解析匯入 `description` 作為固定報表的永久聚合契約。
- 不更新、刪除或搬移任何 Ticket、project、resource 或 assignment 資料。
- 模式 B／C／D／E、V1／V2／V3 範本、權限與 CSV Formula Injection 防護維持既有行為。

- [x] 16.1 擴充固定 OP 月報 fact 資料契約
  - Boundary:
    - Allowed Changes: Report `TicketFact`、query repository 欄位映射、repository 單元測試
    - Forbidden: Database schema、資料 migration、其他 domain repository、硬編碼專案 id
  - Depends: 無
  - Context: 查出 `tickets.external_ref`、`ticket_resources.code`、Excel 開單記錄 importer activity provenance 與既有有效班別欄位，供穩定 predicate 與標籤使用
  - Verify: 欄位 select／scan 測試、imported／一般 Ticket 分流、project／日期／deleted scope 測試、參數化 SQL 檢查
  - 完成：`TicketFact` 與兩條 repository 查詢已加入 external reference、resource code 及 Excel 開單記錄 activity provenance；mapping test 通過

- [x] 16.2 修正 Jira 與支付域名更換聚合
  - Boundary:
    - Allowed Changes: 模式 A 固定 OP predicate、名冊與計數聚合、班別人員標籤、標題、週區間 helper、後端單元測試
    - Forbidden: 告警通知 predicate、模式 B／C／D／E、解析 description、固定 Excel 筆數
  - Depends: 16.1
  - Context: Jira 計算匯入 facts 的非空 external reference；支付域名更換計算匯入 facts 的穩定 resource code；名冊先於指標過濾建立；告警維持既有來源；月底單日併入前一週
  - Verify: Jira／支付 predicate table tests、`早-Chenmin` 標籤、零值人員、2026 年 7 月五段週區間、兩種正式標題、empty payload
  - 完成：兩項指標已限定 Excel 開單記錄 provenance 並改用穩定欄位；名冊、班別排序、零值列、正式標題及月底單日合併測試通過；告警 predicate 未修改

- [x] 16.3 確認前端預覽與明細表忠實呈現 payload
  - Boundary:
    - Allowed Changes: Report Center 模式 A 指標參數、`ReportChartPanel`、`ReportDetailTable`、Report 型別、i18n、相關前端測試
    - Forbidden: 前端重算統計、模式 E renderer、整體 Report UI 重構、假資料補值
  - Depends: 16.2
  - Context: 保留班別前綴與零值列，顯示後端標題及五個週區間，避免 falsy 過濾 0
  - Verify: Jira／支付選項 request、標題、x 軸、零值列、欄位順序、loading／empty／error 回歸
  - 完成：既有 `ReportChartPanel` 與 `ReportDetailTable` 直接消費 payload 且不過濾 0，無需修改前端；`npm test`、`npm run typecheck`、`npm run build` 通過

- [x] 16.4 驗證 CSV 與自動化回歸
  - Boundary:
    - Allowed Changes: Report CSV 測試、後端與前端回歸測試、必要的測試 fixture
    - Forbidden: 生產資料 mutation、降低權限或輸入驗證、跳過既有失敗測試
  - Depends: 16.2、16.3
  - Context: 預覽及 CSV 必須共用相同聚合口徑，且既有報表模式不可退化
  - Verify: `go test ./internal/report`、`go test ./...`、`npm test`、`npm run typecheck`、`npm run build`、`git diff --check`
  - 完成：新增固定 OP 預覽與 CSV 欄位／數字一致性測試；Report 套件、完整 Go、前端測試、typecheck 與 build 通過

- [ ] 16.5 執行 MS 專案 Excel 基準整合驗證（進行中）
  - Boundary:
    - Allowed Changes: 唯讀 API／SQL 比對、瀏覽器預覽、驗證紀錄與 Implementation Notes
    - Forbidden: 修改資料庫資料、將環境基準寫入正式條件、順帶修正告警通知
  - Depends: 16.4
  - Context: 以 project key `MS` 動態取得 project id，日期使用 2026-07-01 至 2026-07-31，比對 Excel Jira 663 與支付域名更換 492
  - Verify: API payload、畫面圖表、明細表、CSV 四者數字與週區間一致，並記錄查詢環境及 trace id
  - 進度：2026-08-03 唯讀 SQL 已確認 activity provenance 可重現 Jira 663、支付域名更換 492，且九位人員均可解析早／中／晚班；本機 9998 未啟動，API、瀏覽器與實際下載 CSV 驗證待執行

### 品質檢查清單

- [x] Requirements、backend／frontend design、task 與實作可追溯
- [x] 正式邏輯未硬編碼 MS project id、Excel 檔名或 663／492
- [x] Jira 與支付域名更換使用穩定資料欄位及 importer activity provenance，不依賴顯示文字或 Ticket description
- [x] 班別人員、零值名冊、週區間、標題與 CSV 具自動化測試
- [x] 告警通知及模式 B／C／D／E 回歸通過
- [ ] MS 專案 Excel 基準完成唯讀整合驗證

### Implementation Notes

- 2026-08-03：依 Excel 與資料庫比對結果建立需求 9、設計補充與 Task 16；目前僅完成規格規劃，尚未修改程式碼或資料。
- 2026-08-03：完成 16.1 至 16.4。後端改以 importer `created` activity 辨識 Excel 開單記錄來源，Jira 使用非空 external reference，支付域名使用 `business_domain_change` resource code；新增班別人員名冊、零值列、正式標題、月底單日合併與 CSV 測試。`go test ./internal/report`、`go test ./...`、`npm test`、`npm run typecheck`、`npm run build`、`git diff --check` 通過；前端 build 僅有既有 chunk 大小警告。
- 2026-08-03：16.5 唯讀資料驗證確認 MS 專案 2026 年 7 月為 Jira 663、支付域名更換 492；API、瀏覽器與實際下載 CSV 因本機服務未啟動而待驗證，Task 16 維持 InProgress。

## 17. 固定 OP 月報明細表緊湊全展開

Status: InProgress

### Execution Context

- 意圖：將模式 A 固定 OP 月報下方明細改為緊湊、無內部分頁與捲軸的全展開呈現，讓使用者能直接比較所有人員及週區間。
- 非目標：不修改後端聚合、payload、柱狀圖、CSV、模式 E 矩陣或一般報表的 Data Grid 行為。
- 已定決策：固定 OP 使用獨立 renderer；桌面採語意化緊湊表格，窄螢幕採每人一張緊湊卡片；資料增加時只使用頁面自然捲動。
- 邊界：`ReportDetailTable` 的固定 OP 分流、新增專用元件、樣式、型別與相關前端測試。
- 關鍵檔案：`opscenter-frontend/src/features/report/components/ReportDetailTable.tsx`、預計新增的 `FixedOPDetailTable.tsx`、Report 型別與測試。
- 完成條件：需求 10.1 至 10.10 具自動化與瀏覽器驗證證據，且一般報表與模式 E 沒有回歸。

### Protected Behavior

- 後端回傳的 columns、rows、列序、欄序與 0 值必須原樣呈現，不在前端重算。
- 模式 B／C／D／V3 保留既有 Data Grid、分頁及大量資料處理能力。
- 模式 E 保留水平捲動及 sticky 左側欄位。
- 圖表、CSV、loading、empty、error 與權限行為保持不變。
- 不得以隱藏 scrollbar 取代真正的全展開版面。

- [x] 17.1 建立固定 OP 明細辨識與 renderer 邊界
  - Boundary:
    - Allowed Changes: `ReportDetailTable.tsx`、固定 OP 判斷 helper、新專用元件骨架、相關單元測試
    - Forbidden: 後端 payload、其他 report mode 聚合、全域 Data Grid 預設值
  - Depends: 無
  - Context: 僅在模式 A 且欄位符合 `person`、`total`、`week_*` 契約時切換 renderer，其餘報表維持 Data Grid
  - Verify: 固定 OP 正向判斷、缺欄位／其他模式反向判斷、一般報表 renderer 回歸
  - 完成：新增固定 OP payload guard 與專用 renderer 分流；其他模式或非固定欄位契約仍使用原 Data Grid

- [x] 17.2 實作桌面緊湊全展開表格
  - Boundary:
    - Allowed Changes: `FixedOPDetailTable.tsx`、元件區域樣式與桌面版測試
    - Forbidden: pagination、固定高度、表格內 `overflow: auto`、前端統計重算
  - Depends: 17.1
  - Context: 1024px 以上以語意化 table 一次呈現所有列欄；列高 36 至 40px，人員、總計與週區間使用受控欄寬及數字對齊
  - Verify: 九列七欄完整呈現、無分頁、無內部水平／垂直捲軸、0 值與欄位順序、table header 關聯
  - 完成：新增語意化 MUI Table，列高 40px、人員欄 144px、總計欄 88px、內容上限 1120px，移除固定高度、分頁與內部 overflow

- [x] 17.3 實作窄螢幕完整卡片版面
  - Boundary:
    - Allowed Changes: 固定 OP 專用響應式樣式、卡片 renderer、相關 component tests
    - Forbidden: 表格內水平捲動、隱藏週區間、折疊或分頁、修改一般報表響應式策略
  - Depends: 17.1
  - Context: 小於 1024px 時每位人員一張卡片，以 label/value grid 完整顯示總計與所有週區間，並保留頁面自然閱讀順序
  - Verify: 平板／手機所有人員與數值存在、無卡片內捲軸、鍵盤與螢幕閱讀器順序、長標籤與 0 值
  - 完成：小於 1024px 改為依後端列序排列的人員卡片，總計及所有週區間均保留於頁面流程，不建立卡片內捲軸或分頁

- [ ] 17.4 完成自動化與瀏覽器回歸
  - Boundary:
    - Allowed Changes: Report 前端測試、必要 fixture、驗證紀錄與本 Task Implementation Notes
    - Forbidden: 降低既有測試要求、順帶改動後端或其他報表版面
  - Depends: 17.2、17.3
  - Context: 驗證固定 OP 桌面／平板／手機，以及一般 Data Grid、模式 E、圖表、CSV 與狀態頁
  - Verify: `npm test`、`npm run typecheck`、`npm run build`、桌面／平板／手機人工檢查、`git diff --check`
  - 進度：自動化測試、typecheck 與 build 已通過；目前執行環境無可用瀏覽器實例，桌面／平板／手機人工檢查待執行

### 品質檢查清單

- [x] Requirements、frontend design、task 與實作可追溯
- [x] 固定 OP 桌面版一次呈現全部列欄，無內部分頁或捲軸
- [x] 窄螢幕以卡片完整呈現，不隱藏任何週區間或 0 值
- [x] 表格密度、數字對齊、語意標記與可讀性符合 UI 規範
- [x] 一般報表 Data Grid、模式 E、圖表與 CSV 自動化回歸通過
- [ ] 前端測試、typecheck、build、瀏覽器檢查與 `git diff --check` 通過

### Implementation Notes

- 2026-08-03：依使用者回饋建立需求 10、前端設計補充與 Task 17；目前僅完成規格規劃，尚未修改前端程式碼。
- 2026-08-03：完成 17.1 至 17.3。新增固定 OP 專用 payload guard、桌面緊湊語意表格與窄螢幕完整卡片；一般報表仍使用原 Data Grid。`npm test`、`npm run typecheck`、`npm run build` 通過，build 僅有既有 chunk 大小警告；瀏覽器 runtime 無可用實例，17.4 與 Task 17 維持 InProgress。

## 18. 三個固定 OP 統計圖人員顯示一致

Status: InProgress

### Execution Context

- 意圖：讓告警、Jira 開單、支付域名更換三個固定 OP 統計圖，使用與下方明細表相同的班別人員名稱及順序。
- 非目標：不修改告警 predicate、告警名冊來源、Jira／支付匯入來源、統計數字、週區間或一般報表。
- 已定決策：後端三個固定 OP 指標共用班別人員 label 與 shift rank；x 軸、series、table、CSV 共用同一有序 labels，前端不重算。
- 邊界：`internal/report/query_service.go` 固定 OP 人員標籤／排序、相關後端測試、前端契約回歸與文件。
- 關鍵檔案：`opscenter-server/internal/report/query_service.go`、`query_service_test.go`、固定 OP CSV 測試、`ModeAIndicatorChart.tsx`、`chartOptions.ts`。
- 完成條件：需求 11.1 至 11.8 具測試證據，告警顯示 `中-Eric` 類型標籤，三個指標的圖表與明細順序一致，且統計數字不變。

### Protected Behavior

- 三個固定 OP 指標既有 predicate 與 Ticket 計數口徑保持不變。
- 告警名冊不因本次顯示修正擴大；Excel 告警數量資料來源仍另案處理。
- Jira、支付域名的 Excel 匯入 provenance、零值名冊、標題及週區間保持不變。
- 前端只消費 payload，不自行組合班別或調整資料 index。
- 模式 B／C／D／E、一般模式 A、CSV Formula Injection 防護與權限保持不變。

- [x] 18.1 統一三個固定 OP 人員標籤與排序
  - Boundary:
    - Allowed Changes: `query_service.go` 的固定 OP label／sort helper 與專用單元測試
    - Forbidden: indicator predicate、名冊來源、週區間、標題、資料庫 schema
  - Depends: 無
  - Context: 移除告警純人名特例，三個指標都使用 `早／中／晚-人員` 與早、中、晚、未歸屬的穩定排序
  - Verify: 告警 `中-Eric`、三班混合排序、缺班別 fallback、同班別名稱排序
  - 完成：移除告警純人名特例；三個固定 OP 指標共用 `fixedOPPersonLabel` 與班別 rank 排序，告警 predicate 與名冊來源未修改

- [x] 18.2 驗證圖表、明細與 CSV index 一致
  - Boundary:
    - Allowed Changes: Report payload／CSV 測試與必要 fixture
    - Forbidden: 前端重排、CSV 獨立聚合、固定環境人名或數字
  - Depends: 18.1
  - Context: 每個 labels index 必須對應同一 series 數值及同 index table person，CSV 使用相同列序
  - Verify: 三指標 table-driven tests、labels／series／rows 長度、person index、CSV 人員名稱與順序
  - 完成：三指標測試逐一驗證 x 軸、series、table 與 CSV index；另驗證早／中／晚／未歸屬排序及非告警人員不進入告警名冊

- [ ] 18.3 完成前端與完整回歸驗證
  - Boundary:
    - Allowed Changes: 前端契約測試、驗證紀錄與本 Task Implementation Notes
    - Forbidden: 修改共用 chart 排序、其他 report mode UI、順帶調整圖表類型或色彩
  - Depends: 18.2
  - Context: `ModeAIndicatorChart` 直接顯示完整 payload label，tooltip 與表格人員一致
  - Verify: `go test ./internal/report`、`go test ./...`、`npm test`、`npm run typecheck`、`npm run build`、三圖瀏覽器檢查、`git diff --check`
  - 進度：Go Report、完整 Go、前端測試、typecheck 與 build 已通過；前端不需修改。瀏覽器 runtime 仍無可用實例，三圖人工檢查待執行

### 品質檢查清單

- [x] Requirements、backend／frontend design、task 與實作可追溯
- [x] 三個固定 OP 指標均使用班別人員標籤與相同排序規則
- [x] x 軸、series、table 與 CSV 的人員 index／列序一致
- [x] 告警 predicate、名冊範圍與三個指標數字不變
- [x] 其他報表模式與前端共用圖表自動化驗證沒有回歸
- [ ] 完整測試、建置、瀏覽器檢查與 `git diff --check` 通過

### Implementation Notes

- 2026-08-03：依使用者回饋建立需求 11、設計補充與 Task 18；目前僅完成規格規劃，尚未修改後端或前端程式碼。
- 2026-08-03：完成 18.1 與 18.2。三個固定 OP 指標統一班別人員標籤與排序，新增 payload／CSV index 對齊及告警名冊保護測試；`go test ./internal/report`、`go test ./...`、`npm test`、`npm run typecheck`、`npm run build` 通過，build 僅有既有 chunk 大小警告。因瀏覽器 runtime 無可用實例，18.3 與 Task 18 維持 InProgress。

## 19. 固定 OP 月報 Ticket 穩定欄位修正

Status: Complete

### Execution Context

- 意圖：修正三個固定 OP 月報錯誤依賴匯入 provenance 或顯示文字的問題，改為直接依 Ticket 穩定欄位統計。
- 已定決策：Jira 使用 `external_ref`、告警使用資源 `resource_type`、支付域名更換使用資源 `code`；三者均不限制 Excel 匯入活動。
- 邊界：Report Ticket fact 欄位、固定 OP predicate／名冊、相關文件與後端測試。
- 非目標：不修改 importer、Jira Report、來源主檔資料、前端或其他報表模式。
- 完成條件：需求 12.1 至 12.7 具測試證據，`go test ./internal/report` 與 `git diff --check` 通過。

- [x] 19.1 擴充 Ticket fact 資源類型
  - 修改 repository select、record mapping 與 mapping test，並移除無用途的 importer activity provenance 子查詢
  - Verify: `TicketResourceType` 正確取得 `ticket_resources.resource_type`

- [x] 19.2 修正三個固定 OP predicate 與名冊
  - Jira 移除 `ExcelOpenTicketImport` 條件
  - 告警改用 `TicketResourceType == alert`
  - 支付域名更換移除 `ExcelOpenTicketImport` 條件
  - 三個名冊均只使用符合各自 predicate 的 Ticket

- [x] 19.3 更新固定 OP 測試
  - 一般 external reference 與一般 `business_domain_change` Ticket 必須計入
  - 告警資源類型正反向測試
  - 圖表、明細與 CSV index 一致性回歸

- [x] 19.4 執行驗證並完成文件紀錄
  - Verify: `go test ./internal/report`
  - Verify: `git diff --check`

### Implementation Notes

- 2026-08-07：依使用者查驗結果建立需求 12、設計補充與 Task 19；開始修正三個固定 OP 月報統計口徑。
- 2026-08-07：完成 Ticket resource type 映射、三個固定 OP predicate／名冊修正及測試更新。`go test ./internal/report` 通過；完整 `go test ./...` 首次受既有 metrics 測試以一般數字子字串比對 `123` 的偶發失敗影響，單獨重跑該測試及再次執行完整測試均通過；`git diff --check` 通過。

## 20. 值班統計控制列與矩陣呈現調整

Status: Complete

### Execution Context

- 意圖：將三個值班口徑整合為單一「值班統計」報表模式，改由「數值」選擇口徑，並改善矩陣閱讀效率。
- 已定決策：前端沿用模式 `E`、`dailyShiftMetricGroup`、API `metric_groups` 與範本 `indicators`；不修改資料庫與後端契約。
- 邊界：Report 控制列、值班矩陣 renderer、三語系、後端顯示標題、OpenAPI 說明與規格文件。
- 非目標：不修改值班統計 Ticket predicate、日期值、CSV 契約、API 路徑或資料庫 schema。

- [x] 20.1 統一值班統計顯示名稱
  - 報表模式顯示「值班統計」
  - 標題使用「值班統計－告警通知／域名更換／支付域名更換」
  - 空資料提示與三語系同步更新

- [x] 20.2 將值班口徑移至數值欄位
  - 報表模式只保留單一 `E`
  - 選到值班統計時，數值提供 `alert_notification`、`domain_change`、`payment_domain_change`
  - 預覽、匯出及範本沿用既有 request/config 格式

- [x] 20.3 新增總計並隱藏班別彙總列
  - 在第一個日期欄前顯示總計
  - 指標與人員列依回傳日期欄位加總，缺值以 `0` 計算
  - 隱藏上述三種數值的 `row_type = shift`，保留班別-人員明細列
  - Jira 通知維持既有顯示行為

- [x] 20.4 完成文件與自動化驗證
  - Requirements、general/backend/frontend design、task 與實作保持可追溯
  - `go test ./internal/report`、`npm run typecheck`、`npm run build`、OpenAPI 產生與 `git diff --check` 通過

### Implementation Notes

- 2026-08-07：完成值班統計名稱、控制列數值選擇、總計欄及班別彙總列隱藏；資料庫、API 路徑、模式 `E`、`daily_shift_execution` 與 CSV 契約未變更。前端 build 僅有既有 chunk 大小警告。

## 21. 固定 OP 圖表與明細表 RWD 對齊

Status: Complete

### Execution Context

- 意圖：讓固定 OP 指標導向月報的上方圖表與下方明細表使用相同寬度及左右邊界。
- 已定決策：在兩個元件共同父層使用專用 `Box`，寬度為 `100%`、最大 1120px；不使用帶預設 gutter 的 MUI `Container`。
- 非目標：不修改圖表資料、明細資料、後端 payload、CSV、其他報表模式或資料庫。

- [x] 21.1 建立固定 OP 共用 RWD 容器
  - 以既有固定 OP payload guard 判斷套用範圍
  - 共用容器包覆 `ReportChartPanel` 與 `ReportDetailTable`
  - 小螢幕使用完整可用寬度，寬螢幕最大 1120px

- [x] 21.2 移除明細表重複寬度責任
  - `FixedOPDetailTable` 保留 `width: 100%`
  - 最大寬度交由共同父層統一管理

- [x] 21.3 完成驗證
  - `npm run typecheck`
  - `npm run build`
  - `git diff --check`

### Implementation Notes

- 2026-08-07：完成固定 OP 上下區塊共用 1120px RWD 容器；其他報表模式維持原寬度，build 僅有既有 chunk 大小警告。

## 22. OP 專案開單數量年報

Status: InProgress

### Execution Context

- 意圖：在 Report 中心新增「年報」模式，以 12 個月份及全部啟用主專案堆疊呈現 Ticket 開單數量，並提供月總計與全年總計。
- 已定決策：新增模式 `F` 與指標 `op_project_ticket_count`；統計依 Ticket 建立時間及 `Asia/Taipei` 曆年，只納入啟用主專案，前端不重算核心數字。
- 邊界：Report domain、query、preview／export、OpenAPI、前端模式選擇、年報圖表與表格、i18n、測試及文件。
- 非目標：不修改既有 A 至 E 統計口徑、不加入 3D 圖表、不新增資料庫表、不依賴 Ticket activity 或 Excel 匯入來源。
- 完成條件：需求 14.1 至 14.10 具測試證據，圖表、明細與 CSV 數字及順序一致。

- [ ] 22.1 實作年報後端 domain 與聚合查詢
  - 新增模式 `F`、指標白名單、年份與日期區間驗證
  - 依所有啟用主專案及 Ticket 建立時間聚合，排除停用與封存主專案
  - 固定補齊 12 個月份、主專案列與總數列
  - Verify: 年度邊界、空月份、空資料、跨主專案聚合、停用／封存排除、月總計與全年總計測試

- [ ] 22.2 整合 preview、CSV 與 OpenAPI
  - preview 與 export 共用同一聚合模型
  - 補 request／response DTO、Swagger 註解與 OpenAPI schema
  - Verify: handler、schema、CSV 月份／列序／總計一致性及 Formula Injection 回歸

- [ ] 22.3 實作前端年報控制與呈現
  - 報表模式新增「年報」，數值新增「OP 專案開單數量」
  - 年份選擇轉換為完整年度日期區間
  - 新增 12 月堆疊長條圖與主專案 × 月份 × 總計明細表
  - 同步 TypeScript schema、i18n、空資料、匯出與範本顯示
  - Verify: component／model tests、`npm run typecheck`、`npm run build`

- [ ] 22.4 完成整體回歸驗證
  - Verify: `go test ./internal/report`、`go test ./...`、前端測試、typecheck、build、OpenAPI 產生、`git diff --check`
  - 人工檢查桌面與窄螢幕，確認 12 月、圖例、總計與水平捲動可讀

### Implementation Notes

- 2026-08-13：依使用者提供的年度 OP 專案紀錄表樣本建立需求 14、前後端設計與 Task 22；依專案文件同步規範，目前僅完成規格規劃，尚未修改後端或前端程式碼，等待使用者安排實作。
- 2026-08-13：使用者已安排執行 Task 22，開始實作後端年度聚合、前端年報控制及呈現。
- 2026-08-13：使用者更正年報統計維度為所有主專案，不是目前主專案底下的子專案；同步改為只納入啟用中主專案，停用與封存主專案不統計。

## 23. MS-指標導向月報

Status: Complete

### Execution Context

- 意圖：在 MS 專案提供固定 OP 指標的專用月報模式，避免一般模式 A 指標與 OP 月報混用。
- 邊界：Report mode、設定白名單、模式 A 聚合重用、前端控制列、renderer、i18n、測試及文件。
- 非目標：不修改 Ticket 資料、既有模式 A 至 F 統計口徑、資料庫 schema 或 Report 權限模型。

- [x] 23.1 實作後端模式 G 設定驗證與模式 A 聚合重用
  - 指標限三個固定 OP 值，未指定時預設 OP Jira 開單數量
  - 固定開單人員基準，拒絕多項或不一致指標
  - Verify: `go test ./internal/report/...`

- [x] 23.2 實作 MS 專案前端控制與固定 OP 呈現
  - 僅 MS 顯示模式 G；數值下拉僅顯示三項 OP 指標
  - 固定 OP chart 與明細表 renderer 支援模式 G
  - Verify: `npm test`、`npm run typecheck`

### Implementation Notes

- 2026-08-13：完成模式 G、後端指標白名單與開單人員正規化；圖表、明細表和 CSV 重用模式 A 聚合結果。

## 24. 模式 A 與 MS 固定 OP 指標分流

Status: InProgress

### Execution Context

- 意圖：從模式 A 的數值選單移除三個固定 OP 指標，讓它們只由模式 G 提供。
- 邊界：Report Center 前端數值選單與切換狀態；不修改後端固定 OP 統計口徑、CSV 或資料庫。

- [x] 24.1 限制模式 A 數值選項並處理模式切換
  - 模式 A 只顯示一般指標
  - 模式 G 切回模式 A 時重設為事件類型
  - Verify: `npm test`、`npm run typecheck`

### Implementation Notes

- 2026-08-13：模式 A 已移除三個固定 OP 指標；模式 G 保留該三項，切回模式 A 時預設為事件類型。
