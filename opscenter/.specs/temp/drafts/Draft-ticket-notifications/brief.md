# Draft: Ticket Notifications

Status: Draft

## Source

- `.kiro/specs/2026-07-01_10-51_Ticket_notifications`

## Intent

重新規劃 Ticket 事件通知、訂閱對象、通知內容、站內未讀數與發送可靠性。

## Bounded Context

包含：

- Ticket domain event
- Notification inbox
- 未讀數 API
- 通知對象解析

不包含：

- Email provider 完整整合
- 外部 chat platform connector

## Promotion Notes

Promote 前需先固定哪些 Ticket 狀態變化會產生事件，以及通知失敗是否影響 Ticket 主交易。

