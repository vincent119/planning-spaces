# Draft: Group Based Operation Permission

Status: Draft

## Source

- `.kiro/specs/2026-07-23_11-00_group-based-operation-permission`
- 近期 op_member 讀取事件類型、SLA、Dashboard 被拒紀錄

## Intent

重新規劃群組設定的表單 read 權限如何套用在一般讀取 API，並定義哪些 API 不應再用 project membership 當唯一讀取判斷。

## Bounded Context

包含：

- Form read permission
- Group permission aggregation
- Read-only operation API
- Middleware policy decision log

不包含：

- 寫入、刪除、管理權限全面重寫
- 使用者與群組資料模型重建

## Promotion Notes

Promote 時需列出所有受影響 endpoint，例如 ticket types、project SLA、dashboard、sub-projects、members、ticket metadata options。每個 endpoint 都要有 allow/deny 驗收案例。

