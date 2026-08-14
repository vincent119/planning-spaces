# Draft: Fixed Shift Leave Management

Status: Draft

## Source

- `.kiro/specs/2026-06-25_11-04_fixed-shift-schedule-leave-management`

## Intent

重新規劃三班固定制排班、假勤、排班例外、管理頁與查詢 API。

## Bounded Context

包含：

- 班別與排班規則
- 假勤申請與管理
- 當前班別查詢
- 專案與成員資料依賴

不包含：

- 通用 HR 系統
- 跨公司薪資與工時計算

## Promotion Notes

Promote 時需明確定義 `/schedule/current-shift-group` 是否屬於 ticket 頁讀取必需資料，以及缺權限時前端是否應略過呼叫。

