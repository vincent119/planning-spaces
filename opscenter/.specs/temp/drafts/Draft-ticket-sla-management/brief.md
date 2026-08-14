# Draft: Ticket SLA Management

Status: Draft

## Source

- `.kiro/specs/2026-07-08_10-00_ticket-sla-management`

## Intent

重新規劃 SLA 規則、專案 SLA 讀取、SLA checker、違規判斷、計算時間與權限。

## Bounded Context

包含：

- Project SLA API
- SLA rule CRUD
- SLA checker job
- Ticket SLA 狀態更新

不包含：

- Dashboard 圖表完整實作
- 通知發送完整流程

## Promotion Notes

Promote 時需明確寫出一般讀取應走群組表單 read 權限，不應被 project membership 查詢誤擋，除非正式需求另有例外。

