# SSO / OIDC / SAML 規劃

## 規劃原則

- 先修補 OIDC 已有骨架，不一次混入 SAML 實作。
- 外部 IdP 只負責身分驗證，系統授權仍使用內部 `users.global_role`、`groups`、`group_members` 與表單權限。
- 敏感設定優先走 `config.yaml` / ENV / Secret Manager；若必須進 DB，必須加密。
- SSO 登入成功後仍套用本地 MFA policy。
- 每個階段都需可回滾，不影響既有帳密登入。

## 階段 0：現況收斂

目標：

- 明確標示目前 OIDC 按鈕只是入口骨架。
- 避免 `oidc.enabled = true` 時導向不存在的 endpoint。
- 決定第一版 Provider 來源：`config.yaml` / ENV。

產出：

- Auth config response 契約更新。
- OIDC route 是否開啟需與後端 handler 實作狀態一致。
- 若 handler 尚未完成，公開 config 不應宣告該 Provider 可登入。

風險：

- 目前若直接啟用 OIDC，前端按鈕會導向尚未實作的 `/auth/oidc/login`。

## 階段 1：OIDC 基礎登入

目標：

- 完成 OIDC authorization code flow。
- 完成 state、nonce、ID Token 驗證。
- 登入成功後簽發既有 access token / refresh token。

主要工作：

- OIDC discovery client。
- JWKS 快取與簽章驗證。
- state / nonce 暫存。
- callback 錯誤回登入頁。
- 既有使用者綁定規則。

不在本階段：

- 群組 mapping。
- SAML。
- 管理 UI。

## 階段 2：OIDC 群組 Mapping

目標：

- 把外部群組轉換成內部 `global_role` 與 permission group。
- 不讓外部群組直接參與授權判斷。

主要工作：

- 建立 mapping 資料表。
- seed 預設 mapping。
- callback 同步 global role。
- callback 同步 SSO 來源的 group membership。
- 安全稽核摘要。

重要限制：

- 不得修改 `roles.code` 來配合外部群組。
- 不得刪除手動群組成員關係。

## 階段 3：SAML 基礎登入

目標：

- 支援 SAML SP initiated login。
- 支援 metadata 與 ACS。
- 完成 assertion 驗證與本地使用者綁定。

主要工作：

- SP metadata endpoint。
- AuthnRequest 產生。
- ACS endpoint。
- SAMLResponse 簽章、issuer、audience、recipient、時間窗驗證。
- SAML attribute mapping。

不在本階段：

- IdP initiated login。
- SAML 單一登出。
- 多 IdP UI 管理。

## 階段 4：管理與維運

目標：

- 管理者可以檢視 SSO Provider 狀態、mapping 狀態與最後同步摘要。
- 維運人員可以定位 callback 失敗原因。

主要工作：

- Provider 狀態 API。
- Mapping 管理 API。
- 登入與安全稽核查詢欄位補強。
- 手動測試 Provider 連線。

交付物：

- `GET /api/v1/admin/sso/providers/status`：顯示 Provider 是否啟用、設定是否完整、是否出現在登入頁、最近測試狀態與去敏摘要。
- `/api/v1/admin/sso/mappings`：支援查詢、新增、修改、停用、啟用 mapping，並驗證 global role / permission group target。
- Provider 測試結果模型：保存 connection / login 測試狀態、檢查項目、操作者、開始與完成時間。
- 系統管理 UI：新增 SSO Provider 入口，包含 Provider 列表、設定表單、secret 更新、測試結果、mapping 頁籤與維運總覽。
- 稽核：Provider 與 mapping 異動、測試啟動與測試結果需寫入 security audit。

## 建議執行順序

1. 補 OIDC login / callback handler，先不開放群組 mapping。
2. 補外部身分綁定表，避免 email 與 subject 混淆。
3. 補 OIDC group mapping。
4. 補 SAML 基礎登入。
5. 補管理 API 與前端管理頁。

## 驗收策略

- 每一階段都保留帳密登入可用。
- 每一階段都需有失敗回復路徑。
- 每一階段都需有安全稽核。
- 完成 OIDC 後再開啟公開 OIDC 按鈕。
- 完成 SAML 後再把 SAML Provider 加入公開 auth config。
