# Draft: Dashboard

Status: Draft

## Source

- `.kiro/specs/2026-06-17_1619_Dashbard`

## Intent

重新規劃專案 Dashboard 的統計資料、圖表、timezone、查詢效能、空狀態與讀取權限。

## Bounded Context

包含：

- Project dashboard API
- Ticket 統計與趨勢
- SLA 與業務指標的展示引用
- `dashboard:read` 與表單 read 權限關係

不包含：

- Report Builder
- BI 模板編輯
- Ticket 詳情流程

## Promotion Notes

Promote 時需明確寫出：讀 Dashboard 不應因缺 project membership 被拒，除非正式需求定義該限制。

