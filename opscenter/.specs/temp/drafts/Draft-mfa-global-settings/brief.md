# Draft: MFA Global Settings

Status: Draft

## Source

- `.kiro/specs/2026-07-02_10-00_mfa-global-settings`

## Intent

重新規劃 MFA 全域設定、使用者登入流程、管理頁、例外政策與驗證。

## Bounded Context

包含：

- MFA policy setting
- Login flow integration
- Admin settings API
- 使用者狀態與錯誤提示

不包含：

- SSO provider protocol
- 外部 IdP 管理

## Promotion Notes

Promote 時需和 SSO Draft 區分：MFA 是登入後安全政策，SSO 是身分來源與協議。

