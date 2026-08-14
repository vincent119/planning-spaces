# Draft: Menu Navigation

Status: Draft

## Source

- `.kiro/specs/2026-06-18_17-45_Menu`

## Intent

重新規劃前端選單、路由可見性、權限 gating、導覽層級與操作入口一致性。

## Bounded Context

包含：

- Sidebar 與 top navigation
- Route guard 與 menu item visibility
- 權限不足時的導向與提示
- 管理頁與專案頁入口

不包含：

- 各頁面內部業務流程
- 後端權限 middleware 的完整實作

## Promotion Notes

Promote 時需把「看得到入口」與「API 可通過權限」分成兩層驗收。

