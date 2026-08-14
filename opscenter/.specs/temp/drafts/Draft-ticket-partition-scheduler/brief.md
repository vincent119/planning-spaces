# Draft: Ticket Partition Scheduler

Status: Draft

## Source

- `.kiro/specs/2026-07-01_12-12_ticket-partition-scheduler`

## Intent

重新規劃 Ticket 分區、分區維護、排程、保留策略與查詢效能保護。

## Bounded Context

包含：

- Ticket table partition strategy
- Scheduler lifecycle
- Retention 與 maintenance job
- 運維驗證與告警

不包含：

- Ticket 業務狀態重寫
- Activity log retention 的完整規則

## Promotion Notes

Promote 時應以 `Chore` 或 `Refactor` 為主，任務要包含 migration rollback 與大量資料驗證。

