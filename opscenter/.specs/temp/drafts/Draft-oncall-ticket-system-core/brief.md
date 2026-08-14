# Draft: Oncall Ticket System Core

Status: Draft

## Source

- `.kiro/specs/2026-06-01_10-22_oncall-ticket-system`

## Intent

以新 SDD 重新規劃時，這個 Draft 只保留 Oncall Ticket System 的核心總契約：角色、專案、表單、Ticket、權限、設定、審計與基礎 API 行為。

## Bounded Context

包含：

- 核心資料模型與名詞定義
- API envelope 與錯誤格式
- 使用者、群組、專案、表單的關係
- 其他功能 spec 需要引用的共用 contract

不包含：

- Dashboard 細節
- SLA 計算
- Report 模板
- SSO、MFA、K8s、Observability 的落地細節

## Promotion Notes

Promote 時應建立正式 `Feature-oncall-ticket-system-core`，並把舊大型 spec 拆成可被其他 spec 引用的共用契約，不再把新需求追加到 legacy task。

