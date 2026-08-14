# Jira Report 需求

## 背景

本需求參考舊設計目錄 `.kiro/specs/2026-06-01_10-22_oncall-ticket-system` 中的「需求 15：Jira CSV 匯入與統計報表」與 `jira_issues` 資料表設計，獨立整理為 Jira Report 後續實作來源。

Jira Report 與目前 Ticket Excel 匯入工具不同：

- Ticket Excel 匯入工具會把 Excel 開單紀錄轉成 Opscenter `tickets`。
- Jira Report 匯入 Jira 匯出的 CSV，完整保存到獨立 `jira_issues` 資料表。
- Jira Report 匯入資料與 Opscenter `tickets` 完全獨立，不建立關聯，也不回寫 Ticket。

## 目標

建立 Jira CSV 匯入與統計報表能力，讓值班組長可匯入 Jira 匯出資料，產出獨立的 Jira 執行統計報表。

## 範圍

本需求包含：

- 上傳或提交 Jira CSV 檔案。
- 解析 Jira 簡中匯出欄位。
- 將 Jira Issue 完整保存到獨立 `jira_issues` 資料表。
- 依 Issue Key 與创建日期去重。
- 查詢 Jira 統計報表。
- 匯出 Jira 報表 CSV。
- 前端提供 Jira 匯入與統計報表頁面。

本需求不包含：

- 串接 Jira API。
- 從 Jira 自動同步。
- 將 Jira Issue 建立成 Opscenter Ticket。
- Jira Issue 與 Opscenter Ticket 的關聯。
- 自訂欄位 mapping 設定。
- 修改現有 Ticket Excel 匯入工具。

## 文件關係

- 舊 `oncall-ticket-system` 的需求 15 與 Jira 設計視為歷史來源，用來說明原始意圖與欄位範圍。
- 本目錄 `.kiro/specs/2026-06-22_1832_jira_report` 是後續 Jira Report 的需求、設計與 task 執行來源。
- 舊 `oncall-ticket-system` 內尚未完成的 Jira task 不因本 spec 建立而改為完成，也不作為後續執行入口。
- `Import_ticket_tools` 屬於 Ticket Excel 匯入工具，資料流向是 `tickets`；Jira Report 的資料流向是 `jira_issues`，兩者不得混用。

## 使用者故事

身為值班組長，我希望能匯入 Jira 匯出的 CSV 檔案，讓系統儲存並統計運維執行的 Jira Ticket 數量，產出獨立的 Jira 執行報表。

## 需求 1：Jira CSV 匯入

### 驗收條件

- [ ] 1.1 系統提供 Jira CSV 檔案上傳或匯入入口。
- [ ] 1.2 所有可存取專案的專案成員皆可執行 Jira 匯入。
- [ ] 1.3 匯入前需驗證檔案格式為 CSV。
- [ ] 1.4 CSV 編碼需支援 UTF-8；若來源有 BOM 需正常處理。
- [ ] 1.5 匯入必須解析 Jira 簡中匯出欄位，不提供使用者自定義欄位 mapping。
- [ ] 1.6 匯入結果需回傳總筆數、新增筆數、跳過重複筆數、失敗筆數。
- [ ] 1.7 匯入失敗列需回傳列號、Issue Key、錯誤原因。
- [ ] 1.8 匯入不得顯示虛假成功狀態；只要有失敗列，結果需明確標示。

## 需求 2：Jira CSV 欄位契約

### 驗收條件

- [ ] 2.1 固定支援以下 Jira 簡中欄位：
  - 概要
  - 问题关键字
  - 问题ID
  - 问题类型
  - 状态
  - 项目关键字
  - 项目名称
  - 项目类型
  - 项目主管
  - 项目描述
  - 项目URL
  - 优先级
  - 解决结果
  - 经办人
  - 报告人
  - 创建者
  - 创建日期
  - 已更新
  - 最近查看的
  - 已解决
  - 到期日
  - 表决
  - 标签
  - 描述
  - 环境
  - 管理关注列表
  - 初始预估
  - 剩余的估算
  - 耗费时间
  - 工作量比率
  - Σ 原预估时间
  - Σ 预估剩余时间
  - Σ 耗费时间
  - 安全级别
  - 自定义字段(Epic状态)
  - 自定义字段(Epic颜色)
  - 自定义字段(Story Point)
  - 自定义字段(史诗名称)
  - 自定义字段(史诗链接)
  - 自定义字段(等级)
- [ ] 2.2 `问题关键字` 必填，對應 `jira_issues.issue_key`。
- [ ] 2.3 `创建日期` 必填，對應 `jira_issues.created_date`。
- [ ] 2.4 缺少必填欄位的列不得寫入。
- [ ] 2.5 未知欄位可忽略，但不得影響已知欄位解析。

## 需求 3：資料保存與去重

### 驗收條件

- [ ] 3.1 Jira Issue 必須寫入獨立 `jira_issues` 資料表。
- [ ] 3.2 `jira_issues` 使用年份分區，分區鍵為 `created_date`。
- [ ] 3.3 系統以 `issue_key + created_date + project_id` 作為同一專案內去重依據。
- [ ] 3.4 重複匯入相同資料時預設跳過，不覆蓋既有資料。
- [ ] 3.5 匯入資料與 Opscenter `tickets` 不建立關聯。
- [ ] 3.6 外部 Jira Issue Key 維持原始文字格式，不轉 ULID。
- [ ] 3.7 匯入時間需記錄於 `imported_at`。

## 需求 4：Jira 報表查詢

### 驗收條件

- [ ] 4.1 系統提供 Jira 執行統計報表。
- [ ] 4.2 報表可依经办人統計 Issue 數量，並以 Jira 專案作為堆疊 series。
- [ ] 4.3 報表可依创建日期篩選時間區間。
- [ ] 4.4 報表可依 Jira 项目关键字篩選。
- [ ] 4.5 報表可依状态篩選。
- [ ] 4.6 報表可依优先级篩選。
- [ ] 4.7 報表回傳需包含圖表資料與明細表資料。
- [ ] 4.8 無資料時回傳空 payload，不得由前端產生假資料。

## 需求 5：Jira 報表呈現與匯出

### 驗收條件

- [ ] 5.1 前端提供 `/projects/:id/jira` Jira 匯入與統計報表頁。
- [ ] 5.2 Jira 報表以堆疊長條圖呈現個人 × Jira 專案 × 數量。
- [ ] 5.3 報表下方提供明細表。
- [ ] 5.4 使用者可匯出目前查詢條件的 CSV。
- [ ] 5.5 匯出 CSV 需保留查詢條件與排序結果。
- [ ] 5.6 匯出失敗不得回傳可被前端誤下載的錯誤檔。
- [ ] 5.7 匯出 Jira 報表 CSV 按鈕需依使用者權限顯示或停用；無匯出權限時不得顯示可操作按鈕。
- [ ] 5.8 即使前端隱藏或停用匯出按鈕，後端 `GET /api/v1/projects/:id/jira/export` 仍必須再次檢查匯出權限。

## 需求 6：權限與安全

### 驗收條件

- [ ] 6.1 使用者必須登入才能匯入與查詢 Jira 報表。
- [ ] 6.2 使用者必須可存取指定主專案。
- [ ] 6.3 Jira 匯入、查詢與匯出不可只依前端按鈕隱藏控制；後端必須驗證權限。
- [ ] 6.4 CSV parser 不得執行公式、巨集或外部連結。
- [ ] 6.5 錯誤訊息不得洩漏 DB DSN、檔案系統路徑或 credentials。
- [ ] 6.6 匯入與報表 API 需使用參數化查詢或 ORM binding，不得拼接使用者輸入 SQL。

## 需求 7：OpenAPI 與測試

### 驗收條件

- [ ] 7.1 OpenAPI 需包含 Jira import、report、export paths。
- [ ] 7.2 OpenAPI 需定義 requestBody、response schema 與錯誤格式。
- [ ] 7.3 後端需補 CSV parser、去重、報表聚合與權限測試。
- [ ] 7.4 前端需補 typecheck / build 驗證。

## Glossary

| 名稱 | 說明 |
| --- | --- |
| Jira Issue | 從 Jira CSV 匯入的外部工單資料，儲存於 `jira_issues`，與 Opscenter Ticket 無關聯 |
| Issue Key | Jira 工單識別碼，例如 `QIEZ-1648` |
| 创建日期 | Jira Issue 建立日期，用於篩選、分區與去重 |
| 经办人 | Jira Assignee，用於個人統計 |
| Jira Report | 依 `jira_issues` 產生的獨立統計報表 |
