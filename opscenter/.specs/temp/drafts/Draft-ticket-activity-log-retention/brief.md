# Draft: Ticket Activity Log Retention

Status: Draft

## Source

- `.kiro/specs/2026-07-03_11-30_ticket-activity-log-retention`

## Intent

重新規劃 Ticket 活動紀錄保留、清理排程、驗證紀錄與資料庫維護。

## Bounded Context

包含：

- Activity log retention policy
- Cleanup job
- Verification report
- Query performance protection

不包含：

- Ticket activity event schema 全面重寫
- Ticket partition scheduler 的全部管理流程

## Promotion Notes

Promote 時需與 Ticket Partition Draft 對齊，避免兩個排程同時改動相同資料表而缺少順序控制。

