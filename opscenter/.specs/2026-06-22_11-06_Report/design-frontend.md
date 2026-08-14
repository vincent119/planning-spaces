# Report Frontend Design

## Routes

```text
/projects/:id/reports                         報表中心
/projects/:id/reports/templates/:templateId   範本執行與詳情
/projects/:id/reports/designer                報表設計器，Phase 3 完整化
```

側邊欄入口由選單樹 `reports` 的 read 權限決定。若使用者沒有 `reports` read 權限，不顯示入口；直接開 URL 時仍以後端 API 回應為準。

## Feature Structure

```text
src/features/report/
├── api/
│   ├── reportTemplates.ts
│   ├── reportPreview.ts
│   └── reportExport.ts
├── components/
│   ├── ReportProjectHeader.tsx
│   ├── ReportDateRangePicker.tsx
│   ├── ReportModeSelector.tsx
│   ├── ReportChartPanel.tsx
│   ├── ReportDetailTable.tsx
│   ├── ReportEmptyState.tsx
│   └── ReportErrorState.tsx
├── charts/
│   ├── ModeAIndicatorChart.tsx
│   ├── ModeBTaskStackChart.tsx
│   ├── ModeCPersonSubProjectChart.tsx
│   ├── DailyShiftExecutionMatrix.tsx
│   └── CustomReportChart.tsx
├── pages/
│   ├── ReportCenterPage.tsx
│   ├── ReportTemplatePage.tsx
│   └── ReportDesignerPage.tsx
├── types.ts
└── queryKeys.ts
```

## Report Center Page

報表中心第一屏不做行銷式 hero，直接提供可操作工作區：

```text
Breadcrumb：Reports / 專案名稱

專案摘要列
  專案名稱、專案代碼、狀態、目前日期區間
  主專案下拉選單、子專案下拉選單

工具列
  日期區間快捷：本週 / 上週 / 本月 / 上月 / 自訂
  報表模式：A / B / C / 值班統計 / D
  數值：值班統計時顯示告警通知 / 域名更換 / 支付域名更換
  person_basis：建立人 / 操作人
  搜尋或篩選：事件類型、資訊來源、子專案
  操作：預覽、匯出 CSV、儲存為範本

主內容
  頁籤：報表 / 範本
  報表頁籤：圖表、矩陣、明細表、預覽狀態
  範本頁籤：範本列表、套用、編輯、刪除
```

## UI Rules

- 使用既有深色操作型介面風格，避免大型裝飾卡片。
- 報表工具列需可換行，自適應寬度，不得在窄螢幕擠壓重疊。
- 圖表區需保留穩定高度，載入、空資料、錯誤時不造成版面大幅跳動。
- 所有 Table / Data Grid 文字需接 i18n，包括欄位選單、排序、篩選、分頁、空資料、錯誤與載入。
- Toast 成功 / 失敗文案皆放在 `report.json` namespace。
- API 失敗不得顯示假資料。
- 報表中心不得把範本列表與報表預覽左右擠在同一工作區；需使用「報表 / 範本」頁籤分流。
- 切換「報表 / 範本」頁籤時，不得清空目前專案範圍、日期區間、報表模式與未儲存設定。

## Report Center Tabs

報表中心主內容使用兩個頁籤：

- 報表：顯示報表條件、預覽、圖表、矩陣、明細表、匯出與儲存範本操作。
- 範本：顯示範本管理，包含搜尋、列表、套用、編輯與刪除。

互動規則：

- 預設停在「報表」頁籤。
- 從「範本」頁籤套用範本後，帶入範本設定並切回「報表」頁籤。
- 值班統計矩陣需在「報表」頁籤取得完整可用寬度，不被範本列表壓縮。
- 範本頁籤列表可用 DataGrid 呈現；資料集、報表模式、人員基準與圖表類型可用 chip 顯示。

## Project Scope Switcher

Report 中心 header 需提供「主專案」與「子專案」兩個下拉選單。

設計細節：

- 位置：專案摘要列右側，與時區、目前報表模式 chip 同一操作區。
- 主專案資料來源：沿用 Ticket 工作區的可見主專案列表查詢。
- 主專案選項內容：主專案名稱、專案代碼、狀態。
- 主專案行為：選取其他主專案後導向 `/projects/:id/reports`，並重設子專案為空值。
- 子專案資料來源：目前主專案下的可見子專案列表。
- 子專案選項內容：子專案名稱、子專案 key、狀態。
- 子專案預設值：空值，顯示為「全部子專案」或等價文案。
- 子專案查詢語意：空值代表使用目前主專案查詢；有值代表限定該子專案查詢。
- 載入中：停用對應下拉選單。
- 載入失敗：停用對應下拉選單並顯示可理解錯誤，不顯示假資料。
- 響應式：手機寬度下兩個下拉選單可換行到獨立一列，文字不可擠壓或重疊。
- i18n：label、錯誤訊息、狀態文字皆需使用既有 namespace，不得硬編。

## Components

### ReportDateRangePicker

- 支援本週、上週、本月、上月、自訂。
- 預設為本月。
- 送出格式為 `YYYY-MM-DD`。
- 顯示時使用 `Asia/Taipei`。

### ReportModeSelector

- A：指標導向月報。
- B：任務導向月報。
- C：人員 × 子專案月報。
- 值班統計：日期 × 指標群組 × 班別/人員的 Excel 矩陣表。
- G：僅在主專案代碼為 `MS` 時顯示的 MS-指標導向月報；數值固定提供 OP Jira 開單、OP 告警通知及 OP 支付或業務項目域名更換，預設 OP Jira 開單。
- D：自訂維度，MVP 可顯示為預留或有限功能。

模式 A 的數值選單只提供一般指標（Ticket 數量、事件類型、資訊來源、子專案）。三個固定 OP 指標不得出現在模式 A，僅由模式 G 提供；由模式 G 切回模式 A 時，數值重設為事件類型。

### ReportChartPanel

- 根據 `ReportChartPayload.report_mode` 與 `chart_type` 選擇 chart component。
- 圖表資料只使用後端 payload，不在前端重算核心統計。
- 圖例、軸標籤、tooltip 需支援 i18n 與長文字截斷。
- 模式 A 固定 OP 指標需支援上方柱狀圖、下方週區間明細表；柱狀圖 x 軸顯示值班人員，表格欄位依後端 `table.columns` 順序呈現。
- 告警、Jira 開單、支付域名更換三個固定 OP 圖表均直接使用後端 `x_axis.labels`；不得在前端移除班別前綴、轉換大小寫或重新排序。圖表 tooltip 使用 payload 的完整 label，下方明細依 `table.rows` 同序呈現。

### ReportDetailTable

- 顯示後端 `table.columns` 與 `table.rows`。
- 支援水平捲動。
- 數字欄位靠右，文字欄位靠左或置中依既有 Data Grid 規範。
- 空資料顯示「此區間沒有可呈現的報表資料」。

### 固定 OP 月報 Excel 基準呈現

- 報表中心仍以模式 A 搭配 `op_jira_issue_count` 或 `op_payment_domain_change_count` 發出預覽及匯出請求；不得改用模式 E。
- 標題、班別人員標籤、總計、週區間及零值均直接使用後端 payload，前端不得以 Ticket 標題、顯示名稱或日期重新聚合。
- 2026 年 7 月明細表應顯示五個週區間欄位，不顯示獨立的 `7/31-7/31` 欄位。
- 支付域名更換數量為 0 的值班人員仍需顯示於圖表與明細表，不因 falsy 判斷或前端過濾而消失。
- CSV 匯出沿用同一組 project、日期、模式與指標參數，欄位順序及數字需與畫面預覽一致。
- 若後端回傳 warning 或空資料，沿用既有 loading、empty、error 呈現；不得以假資料補齊 Excel 數字。

#### FixedOPDetailTable 緊湊全展開設計

`ReportDetailTable` 保留作為一般報表的 Data Grid renderer。當 payload 為模式 A 或 G，且 `table.columns` 符合 `person`、`total`、一個以上 `week_*` 的固定 OP 契約時，改交由獨立 `FixedOPDetailTable` 呈現，避免全域修改影響大量資料報表。

桌面版採 MUI 語意化 `Table`，不設定固定高度、不啟用 pagination，也不建立 `overflow: auto` 的內部 viewport。表格使用 `table-layout: fixed` 與 `width: 100%`，內容寬度上限為 1120px，避免寬螢幕將少量欄位過度拉開；人員欄寬 128 至 144px、總計欄寬 72 至 88px，其餘週區間平均分配剩餘空間。表頭與資料列高度控制在 36 至 40px，儲存格採緊湊 padding；人員靠左，數字靠右。資料量增加時由外層頁面自然增高。

固定 OP 的 `ReportChartPanel` 與 `ReportDetailTable` 由報表中心共同父層 `Box` 包覆，父層使用 `display: flex`、直向排列、既有區塊間距、`width: 100%` 及 `maxWidth: 1120px`。圖表與明細表本身只使用 `width: 100%`，不再各自決定最大寬度。此方式讓上下 Paper 的左右邊界一致，並在小螢幕自然縮至可用寬度；不得使用 MUI `Container` 的預設 gutter，避免與 Paper padding 疊加。

響應式行為如下：

| 寬度 | 呈現方式 | 捲動策略 |
| --- | --- | --- |
| `>= 1024px` | 單一緊湊 table，完整顯示所有列欄 | 僅允許頁面自然捲動 |
| `< 1024px` | 每位人員一張緊湊卡片，總計與週區間以 label/value grid 排列 | 僅允許頁面自然捲動 |

卡片排列仍依 `table.rows` 順序，卡片內欄位依 `table.columns` 順序；0 必須顯示為 0，不得以 falsy 判斷省略。桌面使用 `th scope="col"` 建立欄標題關聯；卡片以可讀的標籤和值順序呈現。loading、empty、error 仍由既有外層流程處理。

前端只負責版面轉換，不重新計算總計、週區間、人員或排序。模式 B／C／D／V3 繼續使用一般 `ReportDetailTable` 的 Data Grid；模式 E 繼續使用 `DailyShiftExecutionMatrix` 的水平捲動與 sticky 欄位；柱狀圖與 CSV 不受影響。

建議元件邊界：

```text
ReportDetailTable
├── isFixedOPDetailTable(payload) → FixedOPDetailTable
│   ├── desktop: CompactSemanticTable
│   └── narrow: CompactPersonCards
└── default → DataGrid
```

測試需涵蓋固定 OP 判斷、桌面所有列欄、無 pagination／內部 overflow、窄螢幕卡片、欄位順序、0 值，以及一般 Data Grid 與模式 E 的回歸。不得只以 CSS 隱藏捲軸，內容必須實際存在於頁面流程並可被鍵盤與螢幕閱讀器讀取。

預期影響範圍以 `ReportCenterPage` 的請求參數、`ReportChartPanel`、`ReportDetailTable`、Report 型別及相關測試為限。若現有元件已正確消費 payload，前端僅補回歸測試，不進行額外重構。

### DailyShiftExecutionMatrix

- 專用於「值班統計」。
- 第一列顯示日期，第二列顯示星期。
- 左側固定顯示指標群組、列標籤與縮排層級。
- 後端仍支援群組標題列、班別列、人員列；告警通知、域名更換及支付域名更換在前端只顯示群組與人員列，隱藏班別彙總列。
- 上述三種數值在第一個日期前顯示總計欄，依 `matrix.columns` 將該列每日值相加，缺值以 `0` 計算。
- 群組列不可自行補算；若後端未回傳數值，顯示空值或 0 需依 payload 設定。
- 需支援水平捲動與 sticky 左側欄位，避免 30 天欄位擠壓。
- 匯出 CSV / Excel-friendly 檔案時保留同樣列序與日期欄。

## State Management

TanStack Query key：

```typescript
export const reportQueryKeys = {
  templates: (projectId: string) => ['reports', projectId, 'templates'] as const,
  template: (projectId: string, templateId: string) => ['reports', projectId, 'templates', templateId] as const,
  preview: (projectId: string, payload: ReportPreviewRequest) => ['reports', projectId, 'preview', payload] as const,
};
```

- 預覽查詢由使用者按「預覽」觸發，不在每次欄位輸入時自動打 API。
- 範本新增、修改、刪除成功後 invalidate templates query。
- 匯出 CSV 使用 mutation 或直接下載流程，失敗需顯示 Toast。

## API Types

```typescript
export type ReportMode = 'A' | 'B' | 'C' | 'D' | 'E';
export type ChartType = 'bar' | 'stacked_bar';
export type PersonBasis = 'created_by' | 'actor';

export interface ReportDateRange {
  date_from: string;
  date_to: string;
  timezone?: 'Asia/Taipei';
}

export interface ReportProjectScope {
  project_id: string;
  sub_project_id?: string;
}

export interface ReportTemplateConfig {
  report_mode: ReportMode;
  x_axis?: string;
  y_axis?: string;
  metrics?: string[];
  chart_type: ChartType;
  person_basis?: PersonBasis;
  title_template?: string;
  indicators?: string[];
}

export interface ReportChartPayload {
  title: string;
  report_mode: ReportMode;
  chart_type: ChartType;
  x_axis: {
    type: 'category' | 'value';
    labels: string[];
  };
  series: Array<{
    name: string;
    stack?: string;
    data: number[];
  }>;
  table?: {
    columns: Array<{ field: string; header_name: string; type?: string }>;
    rows: Array<Record<string, unknown>>;
  };
  meta: {
    project_id: string;
    sub_project_id?: string;
    date_from: string;
    date_to: string;
    timezone: string;
    person_basis?: PersonBasis;
    empty: boolean;
    generated_at: string;
  };
}

export interface DailyShiftExecutionPayload {
  title: string;
  report_mode: 'E';
  matrix: {
    columns: Array<{ date: string; weekday: string }>;
    rows: Array<{
      id: string;
      group: string;
      label: string;
      row_type: 'group' | 'shift' | 'person';
      shift?: string;
      person_id?: string;
      person_name?: string;
      level: number;
      values: Record<string, number>;
    }>;
  };
  meta: ReportChartPayload['meta'];
}
```

前端送出 `preview`、`builtin-monthly`、`template execute` 與 CSV export 時需套用相同專案範圍規則：主專案由 route `projectId` 決定；子專案為空值時不送出 `sub_project_id` 或送出後端定義的空值；子專案有值時送出該 `sub_project_id`。

## i18n

新增 `report.json`，至少包含：

```text
nav.reports
title.center
action.preview
action.export_csv
action.save_template
mode.a
mode.a.op_jira_issue_count
mode.a.op_alert_notification_count
mode.a.op_payment_domain_change_count
mode.b
mode.c
mode.daily_shift_execution
mode.d
person_basis.created_by
person_basis.actor
empty.no_data
error.load_failed
error.preview_failed
toast.template_created
toast.template_updated
toast.template_deleted
toast.export_failed
```

## CSV Export

- 匯出按鈕使用目前查詢條件。
- 檔名格式：`report_{project_key}_{date_from}_{date_to}_{mode}.csv`。
- 後端回傳 `Content-Disposition` 時以前端下載檔名為備援，不覆蓋後端檔名。
- 匯出失敗顯示 Toast，不得下載錯誤 JSON。

## 模式 D 品質強化設計

### 文件定位與已知契約

本節接續需求 7 與既有 `ReportDesignerPage`、`ReportChartPanel`、`ReportDetailTable`、`ReportTemplateConfigV2`。既有 preview payload、範本 V1 相容讀取、Report 權限與 A / B / C / E renderer 為受保護行為。

目前已知狀態：

- preview 使用 POST JSON body，後端回傳 `ReportChartPayload`。
- V2 config 已包含 dataset、date、query、visualization，但缺少 `title_template`。
- `show_table` 與 `legend.wrap` 已存在 config，現有 preview renderer 尚未完整套用。
- metadata 的 code 穩定，但 label / description 目前為後端固定文案。
- V2 CSV export 目前把完整 config 序列化至 GET query，將由後端新 POST 契約取代。

### Bounded Context

包含：模式 D 表單狀態、preview freshness、V2 config 顯示設定、圖表與明細表 renderer、metadata 翻譯、前端 response 驗證、POST Blob 下載、可存取性與預覽資料量保護。

不包含：Report SQL 聚合、任意公式、拖拉畫布、跨資料集 join、非同步匯出 Job、A / B / C / E 業務統計重寫。

### 狀態模型

設計器需分離三種驗證：

```text
queryValid  = 日期、metadata、維度、指標、篩選、排序與上限有效
saveValid   = queryValid + 範本名稱有效 + preview hash 為目前版本
exportValid = queryValid + preview hash 為目前版本
```

preview mutation 必須帶入 `{ payload, hash }` 變數，成功時只能把該次 request 的 hash 標記為已預覽。表單在 request 期間仍可編輯，但舊 request 完成不得更新成新 config 的 current 狀態。

### V2 Config 擴充

```typescript
interface ReportTemplateConfigV2 {
  version: 2;
  dataset: ReportBIDataset;
  title_template?: string;
  date: ReportBIDateConfig;
  query: ReportBIQueryConfig;
  visualization: ReportBIVisualizationConfig;
}
```

`title_template` 只允許純文字與白名單變數 `{{project_name}}`、`{{date_from}}`、`{{date_to}}`。前端提供可用變數說明，後端負責最終驗證與替換；不得使用 `dangerouslySetInnerHTML` 呈現結果。

### 預覽與表格呈現

- `chart_type = table`：不建立 ECharts instance，直接顯示 `ReportDetailTable`。
- 其他圖型且 `show_table = true`：上方顯示圖表，下方顯示明細表。
- 其他圖型且 `show_table = false`：只顯示圖表。
- table row id 需先展開 row，再覆寫安全 fallback id，避免空 id 覆蓋前端 fallback。
- `legend.wrap = true` 且 series 不超過 6 個時可使用多列 plain legend；series 超過 6 個或 wrap=false 時使用 scroll legend，圖表高度設定上限。

### 視覺化決策

圖表選擇提示依資料關係產生，不強制覆寫有效設定：

| 維度關係 | 建議圖型 | 原因 |
| --- | --- | --- |
| 日期趨勢 | line | 保留時間順序與變化方向 |
| 人員／子專案排名 | bar 或 grouped_bar | 容易比較 3 至 12 個類別 |
| 多系列組成 | stacked_bar | 顯示總量與組成，但 series 建議不超過 6 |
| 純明細 | table | 不建立無意義圖形 |

categorical palette 使用穩定 hash 將 series key 對應色彩；色盤避免只用紅綠差異，圖表 tooltip、資料表與圖例文字共同提供辨識，不以色彩作為唯一訊息。

### i18n 與錯誤

- metadata 顯示採 `code -> report.json -> API label -> code` fallback。
- validation code 以 type guard 限定 `ReportBIConfigErrorCode`；未知錯誤顯示 `error.invalid_request` 或 `error.unknown`。
- API 原始 message 與 trace ID 可供診斷，但不得直接拼成翻譯 key。
- 圖表摘要不得直接顯示 `stacked_bar` 等內部 code。
- Data Grid 保留 MUI 預設焦點或提供主題色 `:focus-visible` 外框，不使用無替代樣式的 `outline: none`。

### 目標結構與受影響檔案

```text
features/report/
├── components/
│   ├── ReportDesignerForm.tsx
│   ├── ReportDesignerPreview.tsx
│   ├── ReportChartPanel.tsx
│   └── ReportDetailTable.tsx
├── hooks/
│   └── useReportDesigner.ts
├── schemas/
│   └── reportSchemas.ts
├── utils/
│   ├── reportDateRange.ts
│   ├── reportErrors.ts
│   └── reportFormatting.ts
├── charts/chartOptions.ts
├── api/reports.ts
├── pages/ReportDesignerPage.tsx
└── types.ts
```

共用 Blob 下載集中至單一 helper；圖表 option 與數字 formatter 依 payload、locale、theme 使用 `useMemo`。自定義預覽移除 unlimited 選項，最多 50 category；完整資料透過匯出取得。

### 風險與處理方式

- POST export 與舊 GET 契約切換：後端先提供 POST，再切換前端，最後決定 GET deprecation 時程。
- metadata 新增 options 可能增加 payload：只回傳白名單列舉；大量動態值改用分頁 options API。
- runtime schema 導入造成既有異常資料顯示錯誤：提供局部可恢復錯誤狀態，不讓整頁崩潰。
- 圖表顏色改動影響既有視覺：以 series key 穩定映射並加入視覺回歸檢查。

## V3 多區塊報表範本前端設計

### 文件定位與已知契約

本設計對應需求 8，接續既有 `ReportTemplateConfigV2`、Report Designer、chart / table renderer、POST CSV export 與 response runtime schema。V3 是新增契約，不修改 V1、V2 的判斷與 renderer。

已知契約：

- V2 query 已具 dataset、date、dimension、metric、filter、sort、limit 與 visualization 白名單，可作為 V3 資料 block 的內部 query 契約。
- 現有範本 CRUD 以 `config.version` 辨識 V2；V3 需增加明確 type guard 與 runtime schema。
- 現有單圖 preview 只有單一 loading / error；V3 必須改為以 block id 管理局部狀態。
- 現有權限仍由後端 Report read / create / update / delete 與 project scope 控制，前端 disabled 狀態不構成授權。

### Bounded Context

包含：V3 layout designer、block palette、格線拖拉縮放、碰撞與 compact、響應式衍生 layout、共用參數、局部 preview、dirty guard、revision conflict UI、V3 runtime schema 與資料 block CSV 匯出。

不包含：自由圖層畫布、任意公式、HTML block、跨資料集 join、多人即時游標、PDF 排版器、V1 / V2 自動 migration。

### V3 前端契約

```typescript
type ReportBlockType = 'metric_card' | 'chart' | 'table' | 'text';

interface ReportTemplateConfigV3 {
  version: 3;
  parameters: ReportParameterDefinition[];
  layout: {
    columns: 12;
    row_height: 32;
    gap: 12;
    items: ReportLayoutItem[];
  };
  blocks: ReportBlock[];
}

interface ReportLayoutItem {
  block_id: string;
  x: number;
  y: number;
  w: number;
  h: number;
  min_w: number;
  min_h: number;
  max_w?: number;
  max_h?: number;
}

type ReportBlock =
  | { id: string; type: 'metric_card'; title: string; query: ReportBlockQuery; parameter_bindings: string[] }
  | { id: string; type: 'chart'; title: string; query: ReportBlockQuery; visualization: ReportBIVisualizationConfig; parameter_bindings: string[] }
  | { id: string; type: 'table'; title: string; query: ReportBlockQuery; parameter_bindings: string[] }
  | { id: string; type: 'text'; title: string; content: string };
```

`ReportBlockQuery` 重用 V2 dataset 與 query 白名單，但日期與子專案由 runtime parameter binding 注入，不把可執行字串存入 config。text content 以純文字呈現。

### 編輯器資訊架構

```text
頂部工具列
├── 範本名稱、revision、dirty 狀態
├── Undo / Redo、預覽、儲存、離開
└── viewport：desktop / tablet / mobile preview

左側 Block Palette
├── 指標卡
├── 圖表
├── 明細表
└── 文字說明

中央 12 欄 Grid Canvas
└── draggable / resizable blocks

右側 Inspector
├── block 基本資料
├── query / filters
├── visualization
├── parameter bindings
└── layout size constraints
```

桌面寬度不足時 Inspector 使用 Drawer；手機僅提供預覽與簡化排序，不在第一階段提供觸控精細縮放。

### Layout 演算法

- 儲存座標只使用 desktop 12 欄整數格線。
- move / resize 先 clamp 至邊界，再依 `y -> x -> block_id` 排序處理碰撞。
- 發生碰撞時固定將被碰撞項目往下推，不左右猜測，以維持 deterministic。
- 所有移動結束後執行 vertical compact，但不得越過上方已固定 block。
- tablet 由 12 欄比例映射至 6 欄後重新 compact；mobile 固定 1 欄並依 desktop `y -> x -> block_id` 排列。
- `compactReportLayout` 與 `deriveResponsiveLayout` 必須為無副作用 pure functions，便於 property test。

### 編輯狀態與歷史

使用 reducer 管理 `persistedConfig`、`draftConfig`、`undoStack`、`redoStack`、`selectedBlockId`、`revision`。每個完整 drag / resize gesture 只建立一筆 history，不為每個 pointer move 建立紀錄。dirty 由 canonical config hash 比較，不以手動 boolean 推測。

離開頁面或切換專案時，若 dirty 則顯示確認。儲存送出 `{ revision, config }`；409 時保留本地 draft，提供重新載入與另存為新範本，不自動覆蓋。

### 共用參數與局部 Preview

- 第一階段參數只允許 `date_range` 與 `sub_project`。
- block 以 `parameter_bindings` 宣告繼承；未綁定 block 不因該參數變更而 stale。
- query key 至少包含 project id、block id、block config hash、已綁定 parameter hash。
- 預覽 scheduler 同時最多執行 4 個資料 block；其餘排隊，取消已被新 hash 取代的 request。
- response 以 block id 合併；每個 block 獨立呈現 idle / loading / success / empty / error / stale。

### Block 呈現規則

| Block | 預設尺寸 | 最小尺寸 | 行為 |
| --- | --- | --- | --- |
| metric_card | 3 × 3 | 2 × 2 | 顯示單一數值、標題與日期摘要 |
| chart | 6 × 8 | 4 × 5 | 重用 Report chart renderer，尺寸變更後呼叫 resize |
| table | 12 × 10 | 6 × 6 | Data Grid 依容器高度分頁，不建立巢狀水平頁面捲動 |
| text | 4 × 3 | 2 × 2 | 純文字、自動換行，不渲染 HTML |

刪除需確認；複製保留設定但建立新 block id，放在來源 block 下方後 compact。block toolbar 需有可見名稱，且所有 icon action 提供 aria-label 與 Tooltip。

### 目標檔案結構

```text
features/report/layout/
├── api/reportLayouts.ts
├── components/
│   ├── ReportLayoutCanvas.tsx
│   ├── ReportBlockPalette.tsx
│   ├── ReportBlockFrame.tsx
│   ├── ReportBlockInspector.tsx
│   └── blocks/
├── hooks/
│   ├── useReportLayoutDesigner.ts
│   └── useReportLayoutPreview.ts
├── schemas/reportLayoutSchemas.ts
├── utils/
│   ├── compactReportLayout.ts
│   └── deriveResponsiveLayout.ts
├── pages/ReportLayoutDesignerPage.tsx
└── types.ts
```

格線 library 不預設存在。實作前需評估既有 dependency；若新增 library，必須確認 React 19、鍵盤替代操作及 bundle size，並將 layout pure functions留在專案內，不把核心資料規則綁死於 library。

### 風險與處理方式

- 拖拉 library 無障礙不足：block toolbar 提供鍵盤移動、縮放與數值 Inspector 作為完整替代路徑。
- 大量 block 同時查詢：限制資料 block 12、concurrency 4、category / series 沿用既有限制。
- 不同 viewport 儲存結果分歧：第一階段只儲存 desktop source of truth，其他 viewport deterministic 衍生。
- 編輯器元件過大：layout domain、preview orchestration、block renderer 與頁面容器分離。
- V3 config 損壞：API boundary schema 驗證並顯示局部可恢復錯誤，不嘗試猜測修復。

## OP 專案開單數量年報

本節對應需求 14。Report 中心的報表模式新增 `F`，顯示名稱為「年報」。選取後「數值」欄位啟用且固定提供 `op_project_ticket_count`，顯示名稱為「OP 專案開單數量」。日期控制改為年份選擇，送出時轉成該年 `01-01` 至 `12-31`，時區固定 `Asia/Taipei`。

年報使用獨立 renderer 呈現堆疊長條圖與矩陣表：

- 圖表標題格式為 `YYYY年OP專案紀錄表`。
- 橫軸完整顯示 `YYYY/01` 至 `YYYY/12`，圖例為主專案。
- 明細表第一欄為主專案，接續 12 個月份，最後一欄為「總計」。
- 最後一列為「總數」；所有值直接讀取後端 payload，不在 React 元件內重新聚合。
- 小螢幕允許明細表水平捲動，第一欄與總計欄應維持可辨識；圖表不得以 3D 裝飾造成數值判讀失真。
- 年報不受目前 URL 主專案與子專案選擇影響；畫面需顯示「全部主專案」範圍，並停用子專案選擇避免誤解。
- API response schema、TypeScript `ReportMode`、i18n、CSV 匯出參數及空資料狀態需同步支援模式 `F`。
