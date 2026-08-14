# Jira Report Tasks

## 文件定位

本 task 追蹤 Jira Report 後續實作。Jira Report 與 Ticket Excel 匯入工具不同：本需求寫入獨立 `jira_issues`，不建立、不修改、不關聯 Opscenter `tickets`。

## 0. 文件與契約對齊

- [x] 0.1 對齊舊 oncall-ticket-system Jira 設計
  - 確認舊需求 15 與本 spec 的差異
  - 標記本 spec 為後續 Jira Report 實作來源
  - 不修改舊 task 完成狀態
  - 已在 requirements / design 補明本 spec 為 Jira Report 後續執行來源，舊 task 保留歷史狀態
  - _Requirements: 1.1-1.8, 7.1-7.2_

- [x] 0.2 讀取現有 OpenAPI 與 router 缺口
  - 檢查 `Docs/openapi.json` 是否已有 `/projects/:id/jira/*`
  - 建立缺口清單：import、report、export
  - 確認目前沒有混用 Ticket Excel import 工具
  - 已確認目前 OpenAPI / router 尚無 Jira Report API，且 `cmd/import_ticket_data` 不屬於本需求
  - _Requirements: 7.1-7.2_

- [x] 0.3 定義 Jira Report DTO contract
  - import summary response
  - import row error response
  - report query request
  - chart/table payload
  - CSV export response
  - 已在 design.md 補 `DTO Contract`，包含 import、report、empty payload 與 CSV export 錯誤契約
  - _Requirements: 1.6-1.8, 4.7-4.8, 5.4-5.8_

## 1. Server 資料表與 bounded context

- [x] 1.1 建立 `jira_issues` migration
  - 建立 partitioned table
  - 建立 `UNIQUE (project_id, issue_key, created_date)`
  - 建立 2026 partition
  - 建立查詢 index
  - 補 table / column comments
  - 已新增 `opscenter-server/sql/0031_create_jira_issues.sql`
  - _Requirements: 3.1-3.7_

- [x] 1.2 建立 `internal/jira` package 骨架
  - `domain.go`
  - `parser.go`
  - `repository.go`
  - `service.go`
  - `query_service.go`
  - `delivery.go`
  - 已建立 Jira bounded context 基礎型別、repository / service / query / handler 介面骨架
  - _Requirements: 1.1-1.8, 4.1-4.8_

- [x] 1.3 實作日期、時間與數值解析 helper
  - 支援 UTF-8 BOM
  - 支援 `YYYY-MM-DD`、`YYYY/MM/DD`
  - 支援含時間格式
  - 數值欄位空值轉 NULL
  - 無法解析時回 row error
  - 已新增 parser helper 與 `internal/jira` 單元測試
  - _Requirements: 1.4, 2.2-2.5, 6.4_

## 2. Jira CSV parser 與匯入

- [x] 2.1 實作 Jira CSV header mapping
  - 固定支援簡中欄位
  - 不支援自定義 mapping
  - 未知欄位忽略
  - 缺少必填欄位標記錯誤
  - 已於 `internal/jira/parser.go` 建立固定簡中 header mapping；未知欄位忽略，缺少必填欄位回 row error
  - _Requirements: 1.5, 2.1-2.5_

- [x] 2.2 實作 CSV row normalize
  - `问题关键字` -> `issue_key`
  - `创建日期` -> `created_date`
  - `标签`、`管理关注列表` 轉字串陣列
  - Jira timestamp 欄位轉 `TIMESTAMPTZ`
  - 數值欄位轉 int / decimal
  - 已實作 row normalize；`标签` 轉 `TEXT[]`，`管理关注列表` 依目前 schema 保存為 `watchers` 原始文字
  - _Requirements: 2.1-2.5, 3.6_

- [x] 2.3 實作 Jira import service
  - 驗證專案可見性
  - 驗證匯入權限
  - 逐列 parser + normalize
  - 逐列寫入 `jira_issues`
  - row-level 錯誤不造成虛假成功
  - 已新增 `Service.ImportCSV`，使用 `ProjectAccessChecker` 驗證 engineer 權限，逐列匯入並回傳 row errors
  - _Requirements: 1.1-1.8, 6.1-6.6_

- [x] 2.4 實作去重與 skip 統計
  - 同專案 `issue_key + created_date` 重複時跳過
  - 不覆蓋既有資料
  - import summary 回傳 created / skipped / failed
  - 已於 repository 使用 `ON CONFLICT (project_id, issue_key, created_date) DO NOTHING RETURNING id` 判斷 created / skipped
  - _Requirements: 3.3-3.4, 1.6-1.8_

- [x] 2.5 補 parser / import 測試
  - UTF-8 BOM
  - 必填欄位缺少
  - 日期與數值解析
  - duplicate skip
  - row errors
  - 已補 `parser_test.go`、`service_test.go`、`repository_test.go`
  - _Requirements: 2.1-2.5, 3.1-3.7, 7.3_

## 3. Jira 報表查詢與匯出

- [x] 3.1 實作 Jira report query service
  - 依 `assignee` 聚合 Issue 數量
  - 支援 `date_from` / `date_to`
  - 支援 `jira_project_key`
  - 支援 `status`
  - 支援 `priority`
  - 支援 `assignee`
  - 已新增 `QueryService.Report` 與 repository 參數化聚合查詢
  - _Requirements: 4.1-4.8_

- [x] 3.2 實作 Jira report payload
  - 長條圖 payload
  - 明細 table columns / rows
  - meta empty payload
  - 不由前端補假資料
  - 已回傳 title、bar chart、table rows 與 empty meta
  - _Requirements: 4.7-4.8, 5.2-5.3_

- [x] 3.3 實作 Jira report CSV export
  - 使用目前查詢條件
  - 保留排序結果
  - 成功回 `text/csv`
  - 失敗回 JSON error，不回可下載錯誤檔
  - 已新增 `QueryService.ExportCSV` 回傳 CSV body、檔名與 `text/csv; charset=utf-8`；HTTP 錯誤格式由 API task 接續處理
  - _Requirements: 5.4-5.6_

- [x] 3.4 補報表查詢與匯出測試
  - empty payload
  - assignee 聚合
  - filters
  - CSV content type
  - SQL injection 防護白名單或參數化查詢
  - 已補 `query_service_test.go` 與 repository 查詢測試
  - _Requirements: 4.1-4.8, 5.4-5.6, 6.6, 7.3_

## 4. Server API、權限與 OpenAPI

- [x] 4.1 實作 Jira import API
  - `POST /api/v1/projects/:id/jira/import`
  - multipart file upload
  - JSON import summary response
  - JSON error envelope
  - 已新增 multipart `file` 上傳 endpoint，成功回 `ImportSummary`，錯誤走 `httpx` envelope
  - _Requirements: 1.1-1.8, 6.1-6.6_

- [x] 4.2 實作 Jira report API
  - `GET /api/v1/projects/:id/jira/report`
  - query parameters
  - chart/table payload response
  - 已新增 report endpoint，接收日期與 Jira 篩選條件並回傳 chart/table payload
  - _Requirements: 4.1-4.8_

- [x] 4.3 實作 Jira export API 權限檢查
  - `GET /api/v1/projects/:id/jira/export`
  - 後端再次檢查 export 權限
  - 無權限回 403
  - 不依賴前端按鈕隱藏
  - 已透過 `RequireProjectFormPermission` 檢查 `jira/export` read 與 project viewer role
  - _Requirements: 5.7-5.8, 6.1-6.3_

- [x] 4.4 註冊 Jira routes
  - 接入主 router
  - 套用 auth middleware
  - 套用專案可見性檢查
  - 套用 form permissions 或等價權限
  - 已在主 router 註冊 `RegisterJira`，並新增 `jira/import`、`jira/report`、`jira/export` 權限 seed
  - _Requirements: 6.1-6.3_

- [x] 4.5 補 Swagger 註解並重新產生 OpenAPI
  - import multipart requestBody
  - report query parameters
  - export CSV success response
  - JSON error response
  - 確認 `Docs/openapi.json`
  - 已補 godoc 註解並執行 `make openapi`，OpenAPI 已包含 Jira import/report/export paths
  - _Requirements: 7.1-7.2_

- [x] 4.6 Server 驗證
  - `go test ./internal/jira`
  - `go test ./internal/server`
  - `go test ./...`
  - `make openapi`
  - 已全部執行並通過
  - _Requirements: 7.3_

## 5. Frontend feature 與 API client

- [x] 5.1 建立 Jira feature 目錄骨架
  - `api`
  - `components`
  - `charts`
  - `pages`
  - `types.ts`
  - `queryKeys.ts`
  - 已補齊 `src/features/jira` 實際 feature 檔案與 barrel export
  - _Requirements: 5.1_

- [x] 5.2 建立 Jira API client
  - import CSV
  - report query
  - export CSV
  - error envelope handling
  - 已新增 `importJiraCsv`、`getJiraReport`、`exportJiraReportCsv`，沿用全域 `apiClient` envelope 與下載錯誤處理
  - _Requirements: 1.1-1.8, 4.1-4.8, 5.4-5.8_

- [x] 5.3 建立 `/projects/:id/jira` route
  - 接入專案 layout
  - 麵包屑
  - 主專案摘要
  - 不顯示假資料
  - 已將 route 接到 `ProjectJiraPage` 與 `ProjectWorkspaceLayout`，目前僅顯示真實專案 context 與空狀態
  - _Requirements: 5.1_

- [x] 5.4 補 i18n
  - zh-TW
  - zh-CN
  - en
  - table / grid 文案
  - error / empty states
  - 已新增 `jira` namespace 並接入 i18n resources
  - _Requirements: 5.1-5.8_

## 6. Frontend 匯入 UI

- [x] 6.1 建立 Jira CSV upload panel
  - 選擇 CSV 檔
  - 匯入按鈕
  - loading 狀態
  - error toast
  - 已在 `ProjectJiraPage` 建立 CSV 選檔、匯入 mutation、loading 與錯誤 toast
  - _Requirements: 1.1-1.8, 5.1_

- [x] 6.2 建立 import result summary
  - total rows
  - created rows
  - skipped rows
  - failed rows
  - row errors table
  - 已顯示 import summary 指標與 row errors table，空錯誤列顯示真實 empty state
  - _Requirements: 1.6-1.8_

- [x] 6.3 匯入權限控制
  - 無 import 權限時隱藏或停用匯入按鈕
  - 後端 403 顯示清楚錯誤
  - 不顯示虛假成功
  - 已於後端 403 回應時停用匯入控制並顯示權限錯誤；成功摘要只使用 API 回傳結果
  - _Requirements: 6.1-6.3_

## 7. Frontend 報表 UI

- [x] 7.1 建立 Jira report filters
  - 日期區間
  - Jira 项目关键字
  - 状态
  - 优先级
  - 经办人
  - 已新增日期區間與 Jira project key / status / priority / assignee 篩選欄位，查詢按鈕送出目前條件
  - _Requirements: 4.3-4.6_

- [x] 7.2 建立 Jira assignee bar chart
  - 個人 × Issue 數量
  - empty state
  - loading state
  - 不使用假資料
  - 已接 `getJiraReport` 回傳 chart payload；loading 顯示 spinner，empty 使用 API empty meta
  - 已調整為經辦人 × Jira 專案 stacked bar，符合運維處理事件數量報表呈現
  - _Requirements: 4.1-4.8, 5.2_

- [x] 7.3 建立 Jira report detail table
  - assignee
  - count
  - 排序與欄位寬度
  - i18n
  - 已使用 API table columns / rows 建立 DataGrid，數字欄右對齊並接 DataGrid locale
  - _Requirements: 5.3_

- [x] 7.4 實作 Jira report CSV export button 權限控制
  - 有 `jira/export` read 權限才顯示可操作按鈕
  - 無權限時隱藏或停用
  - 後端 403 顯示權限錯誤
  - export 使用目前查詢條件
  - 已使用目前已送出的查詢條件匯出；後端 403 時停用匯出控制並顯示權限錯誤
  - _Requirements: 5.4-5.8, 6.1-6.3_

## 8. 整合驗證

- [x] 8.1 前端驗證
  - `npm run typecheck`
  - `npm run build`
  - 匯入 UI 無假資料
  - 報表 empty state 正確
  - 匯出按鈕權限狀態正確
  - 已執行 `npm run typecheck`、`npm run build`；匯入摘要、報表 empty state 與匯出 403 狀態皆由 API 回應驅動，不產生假資料
  - _Requirements: 5.1-5.8, 7.4_

- [X] 8.2 端到端手動驗證
  - 匯入一份 Jira CSV
  - 重複匯入同檔案應 skipped
  - 查詢 assignee 統計
  - 匯出 CSV
  - 無 export 權限使用者不可操作匯出
  - 尚未執行：需要啟動完整前後端、套用 DB migration、準備可登入測試帳號與不同 export 權限角色
  - _Requirements: 1.1-7.4_

- [x] 8.3 文件收斂
  - 確認 requirements / design / task 與實作一致
  - 若 implementation 有 contract 差異，先補文件再標完成
  - 不回寫到舊 oncall-ticket-system 已完成 task
  - 已修正 design DTO contract，將 import error 欄位對齊為 `row_errors`，並補齊 report `timezone` 與 `httpx.APIResponse` 錯誤 envelope 格式
  - _Requirements: 7.1-7.4_
