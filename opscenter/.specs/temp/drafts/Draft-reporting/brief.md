# Draft: Reporting

Status: Draft

## Source

- `.kiro/specs/2026-06-22_11-06_Report`

## Intent

重新規劃報表查詢、篩選、匯出、圖表展示、權限與大量資料處理。

## Bounded Context

包含：

- Report 查詢 API
- 前端報表頁
- 匯出與欄位格式
- 報表權限與資料範圍

不包含：

- BI 模板管理
- Jira 專屬資料映射
- Business Metrics 的指標治理

## Promotion Notes

Promote 時應先固定指標口徑與 timezone，避免與 Dashboard、BI、Jira 報表結果不一致。

