# Draft: Report BI Templates

Status: Draft

## Source

- `.kiro/specs/2026-06-24_17-45_Report_BI_templates`

## Intent

重新規劃 BI 報表模板、模板 contract、資料遷移、模板套用與前端管理體驗。

## Bounded Context

包含：

- Template contract
- Migration plan
- Template CRUD
- Report 頁面的模板套用

不包含：

- 所有報表查詢 API 重寫
- 外部 BI 工具整合

## Promotion Notes

Promote 時應引用既有 `contract.md` 與 `migration.md`，並把不可變更的資料 contract 寫入 design。

