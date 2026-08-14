# Draft: SSO OIDC SAML

Status: Draft

## Source

- `.kiro/specs/2026-07-03_17-20_sso-oidc-saml`

## Intent

重新規劃 SSO、OIDC、SAML、登入入口、使用者同步、設定管理與安全驗收。

## Bounded Context

包含：

- OIDC login flow
- SAML login flow
- Provider settings
- User linking 與 provisioning

不包含：

- MFA policy 完整設計
- 企業目錄同步平台

## Promotion Notes

Promote 前需先決定 IdP 設定是否支援多 provider，以及 local login 與 SSO login 的優先順序。

