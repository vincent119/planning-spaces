# 報表範本表單權限修正需求

## 文件定位

本規格接續既有「Report BI Templates」與「Report BI 資料集」規格，補齊已上線報表範本 API 未納入表單選單權限矩陣的缺口。本規格不重寫資料集、BI 查詢、範本生命週期或既有固定報表的資料口徑。

## 背景

報表範本目前使用 `reports` 表單節點授權，報表中心的通用讀取權限會使範本列表與執行 API 可被不應管理範本的使用者存取。表單管理介面也沒有可獨立設定「報表範本」的節點，無法表達管理者全權、其餘既有群組唯讀的已確認矩陣。

## 目標

- 建立可在表單選單樹與權限矩陣管理的 `reports/templates` 節點。
- 將報表範本 CRUD 與執行／匯出 API 改以該節點的操作權限判斷。
- 初始化既有五個權限群組的已確認矩陣，並同步有效 Casbin policy。
- 前端依有效表單權限顯示或隱藏建立、編輯、刪除等管理操作。

## 非目標

- 不調整 `reports` 節點及內建月報、資料集 metadata、一般報表預覽與匯出 API 的既有授權。
- 不新增、刪除或重新命名既有權限群組。
- 不改變主專案可見性、資料集欄位白名單、範本草稿／發布資料模型。
- 不由 migration 覆寫管理者日後透過權限矩陣手動調整的其他表單節點。

## 已確認權限矩陣

`reports/templates` 的初始權限如下；`all` 對應 `read`、`create`、`update`、`delete` 四種操作。範本執行、區塊執行與 CSV 匯出沿用 `read`。

| 群組代碼 | 名稱 | read | create | update | delete |
| --- | --- | --- | --- | --- | --- |
| `ops_admin` | 系統管理權限群組 | 是 | 是 | 是 | 是 |
| `ops_op_admin` | 運維管理權限群組 | 是 | 是 | 是 | 是 |
| `engineers` | 值班工程師 | 是 | 否 | 否 | 否 |
| `ops_member` | 一般成員權限群組 | 是 | 否 | 否 | 否 |
| `ops_op_member` | 維運成員權限群組 | 是 | 否 | 否 | 否 |

未具備上述群組有效權限的使用者不得存取報表範本 API；回應維持既有授權失敗語意。

## 使用情境與驗收情境

### 情境 1：權限矩陣可管理報表範本

測試：migration 驗證與表單權限 repository／delivery 測試。

假設：資料庫已有 `reports` 表單節點及五個既有群組。

當：部署本修正 migration。

那麼：表單樹出現啟用中的 `reports/templates`「報表範本」節點，且五個群組的資料列符合已確認權限矩陣與有效 Casbin policy。

### 情境 2：管理群組可完整管理範本

測試：`go test ./internal/report`。

假設：使用者為 `ops_admin` 或 `ops_op_admin` 的有效成員，且具既有專案可見性。

當：使用者呼叫範本列表、建立、讀取、更新、刪除、執行或匯出 API。

那麼：各 API 依對應的 `reports/templates` 操作授權通過，並仍保留既有專案範圍檢查。

### 情境 3：唯讀群組只能檢視與執行

測試：`go test ./internal/report` 與前端元件測試。

假設：使用者只具有 `engineers`、`ops_member` 或 `ops_op_member` 的有效 `reports/templates` 唯讀權限。

當：使用者檢視或執行範本。

那麼：請求成功；嘗試建立、更新或刪除時必須被拒絕，前端不得顯示對應管理操作。

### 情境 4：無授權使用者不得繞過表單矩陣

測試：`go test ./internal/report`。

假設：使用者不具 `reports/templates` 對應操作權限。

當：使用者直接呼叫任一範本 API。

那麼：後端拒絕請求，不得因具 `reports` 通用讀取權限而通過。

## 驗收條件

- [ ] `reports/templates` 是 `reports` 下的獨立表單節點，具正確顯示名稱、排序、啟用狀態與可重跑 migration 行為。
- [ ] 初始群組權限完全符合本文件的已確認矩陣，並同步 Casbin policy。
- [ ] 所有 report template CRUD、execute、execute-layout 與 block export 路由都使用 `reports/templates`，不再使用通用 `reports` 節點。
- [ ] 後端授權與既有專案可見性檢查均被測試覆蓋。
- [ ] 前端範本列表與設計器的管理按鈕使用 `reports/templates` 有效權限；唯讀使用者仍能檢視及執行。
- [ ] 不影響其他 `reports` API、固定月報、資料集 API 與既有表單節點。

## 驗證需求

- 後端：相關 package 測試、migration 對既有資料可重跑驗證。
- 前端：型別檢查、相關元件測試或既有驗證命令。
- 整合：以五個群組與無權限使用者驗證 API 狀態碼及 UI 操作可見性。
- 品質：`git diff --check`。
