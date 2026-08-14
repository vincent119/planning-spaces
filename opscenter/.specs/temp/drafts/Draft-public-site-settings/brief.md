# Draft: Public Site Settings

Status: Draft

## Source

- `.kiro/specs/2026-07-03_10-00_public-site-settings`

## Intent

重新規劃公開站台設定、公開資訊讀取、後台管理、快取與公開頁安全邊界。

## Bounded Context

包含：

- Public site settings API
- Admin settings UI
- Public read contract
- Cache 與公開資料限制

不包含：

- 完整 CMS
- 多租戶站台建置平台

## Promotion Notes

Promote 時需列出哪些欄位允許公開讀取，禁止讓內部設定或 secret 透過公開 API 外洩。

