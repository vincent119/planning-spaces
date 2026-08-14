# Draft: Jira Report Integration

Status: Draft

## Source

- `.kiro/specs/2026-06-22_1832_jira_report`

## Intent

重新規劃 Jira 資料接入、欄位映射、報表統計與前端展示。

## Bounded Context

包含：

- Jira issue 欄位映射
- 匯入或同步資料來源
- Jira 報表統計
- 權限與資料可見範圍

不包含：

- 通用 Report Builder
- BI 模板管理
- Jira webhook 完整平台化

## Promotion Notes

Promote 前需確認 Jira 資料來源是同步、匯入還是查詢時即時讀取。

