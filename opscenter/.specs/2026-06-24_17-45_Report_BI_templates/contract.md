# Report BI Templates Contract

## 目的

本文件記錄 Report 範本 V1 與 V2 契約差異、相容策略與錯誤碼。`task.md` 的 `0. 契約與相容性` 以此文件作為實作依據。

## 現有 V1 盤點

### Server domain DTO

位置：[domain.go](/Users/vincent/Documents/git_home/vin/opscenter/opscenter-server/internal/report/domain.go)

V1 型別為 `TemplateConfig`，欄位如下：

| 欄位 | 用途 | 備註 |
| ---- | ---- | ---- |
| `report_mode` | A / B / C / D / E 報表模式 | 目前是主要分流依據 |
| `x_axis` | 自訂報表 X 軸 | V1 只支援有限模式 |
| `y_axis` | 自訂報表 Y 軸 | 模式 D 使用 |
| `metrics` | 指標清單 | 模式 A / D 使用 |
| `chart_type` | 圖表類型 | 目前為 `bar` / `stacked_bar` |
| `person_basis` | 人員口徑 | `created_by` / `actor` |
| `title_template` | 標題模板 | 前端表單可設定 |
| `indicators` | 指標群組 | 模式 A / E 使用 |

### Repository 儲存格式

位置：[repository.go](/Users/vincent/Documents/git_home/vin/opscenter/opscenter-server/internal/report/repository.go)

`report_templates.config` 使用 JSONB 儲存。repository 目前直接將 `TemplateConfig` marshal / unmarshal：

- 建立範本：`json.Marshal(template.Config)`
- 更新範本：`json.Marshal(template.Config)`
- 讀取範本：`json.Unmarshal(r.Config, &config)`

因此 V1 config 沒有 `version` 欄位。後續讀取時若缺少 `version`，必須視為 V1。

### Frontend type 與 form 欄位

位置：[types.ts](/Users/vincent/Documents/git_home/vin/opscenter/opscenter-frontend/src/features/report/types.ts)、[ReportTemplateDialog.tsx](/Users/vincent/Documents/git_home/vin/opscenter/opscenter-frontend/src/features/report/components/ReportTemplateDialog.tsx)

前端 `ReportTemplateConfig` 對應 V1。既有 dialog 可設定：

- 範本名稱
- 描述
- 報表模式
- 圖表類型
- 人員口徑
- 標題模板

目前其他欄位由前端依模式補預設值：

- `x_axis`
- `y_axis`
- `metrics`
- `indicators`

## V2 契約

V2 型別已先定義於 server `TemplateConfigV2` 與 frontend `ReportTemplateConfigV2`。

```json
{
  "version": 2,
  "dataset": "ticket_events",
  "date": {
    "preset": "this_month",
    "timezone": "Asia/Taipei"
  },
  "query": {
    "x_axis": "person",
    "series": "sub_project",
    "metric": "ticket_count",
    "person_basis": "created_by",
    "filters": [],
    "sort": {
      "field": "total",
      "direction": "desc"
    },
    "limit": 20
  },
  "visualization": {
    "chart_type": "stacked_bar",
    "show_table": true,
    "legend": {
      "placement": "bottom",
      "wrap": true
    }
  }
}
```

### V2 結構

| 區塊 | 用途 |
| ---- | ---- |
| `version` | 契約版本，目前固定為 `2` |
| `dataset` | 資料集，MVP 固定為 `ticket_events` |
| `date` | 日期預設、時區與粒度 |
| `query` | 維度、series、指標、篩選、排序與 Top N |
| `visualization` | 圖表類型、明細表與圖例設定 |

## V1 到 V2 欄位對應

| V1 欄位 | V2 欄位 | 對應策略 |
| ---- | ---- | ---- |
| `report_mode` | 無直接對應 | 內建 A / B / C / E 保留 V1 流程；模式 D 可轉 BI config |
| `x_axis` | `query.x_axis` | 若值存在且符合 V2 白名單才可轉換 |
| `y_axis` | `query.series` | `sub_project` 可轉換；其他值需依 metadata 判斷 |
| `metrics[0]` | `query.metric` | 只接受 `ticket_count` |
| `chart_type` | `visualization.chart_type` | `bar` / `stacked_bar` 可直接轉換 |
| `person_basis` | `query.person_basis` | `created_by` / `actor` 可直接轉換 |
| `title_template` | 暫無 | 保留在 V1 原始 config，不自動帶入 V2 |
| `indicators` | 暫無 | OP 固定指標與每日統計保留 V1 流程 |

## 相容讀取策略

- 缺少 `version` 的 `report_templates.config` 一律視為 V1。
- V1 config 不在讀取時自動覆寫成 V2，避免破壞既有資料。
- 可轉換的 V1 範本可在 response view model 額外提供 V2 預覽設定。
- 不可轉換的 V1 範本標示為 legacy，仍允許執行既有流程。
- 編輯 legacy 範本時，前端應提示另存為 V2 範本。
- 內建月報 A / B / C / E API 契約不因 V2 改造而改變。

## V2 Config 錯誤碼

| 錯誤碼 | 語意 |
| ---- | ---- |
| `unknown_dataset` | dataset 不存在 |
| `unknown_dimension` | x_axis 或 series 不在白名單 |
| `unknown_metric` | metric 不在白名單 |
| `unknown_filter` | filter field 不在白名單 |
| `unknown_sort` | sort field 不在白名單 |
| `unsupported_chart_type` | dataset 不支援指定圖表 |
| `invalid_date_config` | 日期 preset、timezone 或 grain 無效 |
| `invalid_dimension_combination` | 維度組合無效，例如 x_axis 與 series 相同 |
| `invalid_person_basis` | 人員維度需要但未提供合法 person_basis |
| `invalid_filter_values` | filter values 空值或型別錯誤 |
| `invalid_limit` | limit 不在允許值內 |
| `legacy_config` | config 是舊版且無法轉換為 V2 |
