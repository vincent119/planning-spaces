# Draft: Op Admin Cross Project Read Access

Status: Draft

## Source

- `.kiro/specs/2026-07-22_15-35_op-admin-cross-project-read-access`

## Intent

重新規劃 op_admin 跨專案讀取權限、限制邊界、API 行為與回歸驗收。

## Bounded Context

包含：

- op_admin read access
- Project scoped read API
- Forbidden write escalation
- Audit and test cases

不包含：

- 任意角色升權
- Project membership 資料模型重寫

## Promotion Notes

Promote 時需明確列出 op_admin 能讀哪些 project resource，以及哪些 write action 必須仍被拒絕。

