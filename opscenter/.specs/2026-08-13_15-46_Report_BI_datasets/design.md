# Report BI 資料集設計

## 設計原則

資料集是後端管理的語意模型，不等同單一資料表。每個資料集將來源表、關聯、權限、維度與指標封裝在後端；前端只取得 metadata 並組合受允許的查詢。

```text
資料來源與關聯主檔
  -> 資料集定義與指標口徑
  -> metadata API
  -> V3 區塊查詢設定
  -> KPI／圖表／表格／CSV
```

## 資料集定義

每個資料集應具有下列概念模型：

```text
DatasetDefinition
├── code、名稱、說明、version
├── dimensions[]
├── metrics[]
├── filters[]
├── supportedVisualizations[]
├── queryLimits
└── accessPolicy
```

`dimensions` 與 `metrics` 必須使用穩定代碼。資料庫欄位與 SQL expression 僅存在後端 mapping，不能由 request 直接傳入。

## 首批資料集設計

| 資料集 | 主要維度 | 主要數值 | 特別規則 |
| --- | --- | --- | --- |
| Ticket 運維事件 | 日期、專案、子專案、人員、班別、來源、類型、狀態 | Ticket 數、固定 OP 指標、已解決／未結案數 | 沿用既有 Ticket 與 OP predicate |
| 值班執行統計 | 日期、星期、班別、人員、子專案 | 告警、域名、支付域名處理數 | 先重用既有每日矩陣聚合 |
| 人員工作量 | 日期、人員、班別、專案、子專案 | 開單、指派、活動處理、未結案數 | 人員口徑須由產品決策確定 |
| 主專案營運總覽 | 月份、主專案、狀態 | 開單、未結案、固定 OP 指標 | 跨主專案範圍須由產品決策確定 |

## 查詢與 V3 整合

每個 V3 資料區塊保留一個 `dataset`。查詢規劃器依該資料集的 metadata 驗證維度、數值、篩選、排序與圖表類型，再轉換為參數化查詢。

共用參數由區塊明確繫結。參數變動時只使繫結該參數的區塊過期並重新查詢；未繫結區塊保持現有結果。

## 效能與實體來源

資料集代碼與實體來源解耦。`DatasetDefinition` 負責對外的維度、數值及口徑；每個數值或查詢規劃可在後端選擇合適來源：

```text
V3 block query
  -> DatasetDefinition 與權限驗證
  -> Query planner
      ├── 原始 table + index：即時明細、可下鑽、動態篩選
      ├── 一般 view：共用關聯與可重用 SQL 邊界
      └── materialized view／預聚合表：高頻固定粒度彙總
  -> 同一份標準化 payload
```

一般 view 不作為效能優化承諾；其作用是降低重複 join 與統計口徑散落的風險。materialized view 或預聚合表需在資料量、查詢頻率及可接受資料延遲均有量測依據後才導入。

| 資料集 | 預設來源 | 評估彙總來源 | 主要原因 |
| --- | --- | --- | --- |
| `ticket_events` | 原始 `tickets` 與關聯表 | 不預設；先做索引與 query plan 優化 | 篩選與下鑽需求高，日期區間彈性大 |
| `shift_execution` | 現有值班聚合 | 每日人員／班別預聚合表 | 每日矩陣可能高頻重複讀取 |
| `person_workload` | Ticket 與活動資料 | 日／月人員工作量 materialized view | 人員與期間彙總可重用，但需先定義活動去重口徑 |
| `project_operations` | Ticket 與專案資料 | 主專案月度 materialized view | 跨專案趨勢及管理 KPI 具有固定月粒度 |

彙總來源的每一筆資料必須包含或可推導 `refreshed_at`。metadata／payload 應能提供資料新鮮度狀態；資料過舊或刷新失敗時，區塊應顯示明確警告，不得以空資料或即時資料假裝成功。

所有 materialized view、預聚合表、索引、刷新 job 與權限均以新 migration、pre-deploy hook 或受控 scheduler 建立。回滾時優先切回原始 query planner，保留彙總資料供診斷，確認後再由新的 migration 清理。

## 已確認的產品決策

- `project_operations` 查詢範圍為使用者具 Report `read` 且可見的全部主專案，UI 使用主專案下拉複選。權限驗證在後端先於篩選器執行，未授權主專案不得出現在 metadata、選項或聚合結果。
- `person_workload` 提供 `created_by`、`assignee`、`actor` 三種人員口徑。每個 metric definition 必須宣告口徑及去重策略；活動操作人員的數值以明確活動類型白名單與 Ticket／人員去重規則計算。
- `shift_execution` 保留每日矩陣，同時以相同聚合模型提供長條圖、折線圖及表格 payload，不在前端轉換或重算。
- 範本新增 `draft` 與 `published` lifecycle。草稿只能由建立者與具範本管理權限者列出、預覽、執行或修改；發布後依既有 Report read 與專案可見性提供一般成員執行。

## 權限與安全

- metadata API 與 query API 均執行 Report `read`、主專案可見性及子專案歸屬驗證。
- 所有維度、數值、運算子及排序鍵採白名單 mapping。
- SQL 僅在後端組裝並使用參數化條件。
- 指標及範本異動寫入摘要審計紀錄，不寫入篩選值或查詢結果。

## 相容策略

- V1、V2、V3 config 依既有版本 parser 分流。
- 固定模式 A 至 G 繼續使用現有 query path，不強制遷移至資料集。
- 資料集新增與指標口徑變更均需版本與遷移說明；任何變更不得默默改變既有範本結果。
