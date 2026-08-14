# SSO / OIDC / SAML 需求

## 文件定位

本 spec 接續 `../2026-06-01_10-22_oncall-ticket-system` 的 D8 Phase 3：SSO / OIDC / SAML。

原始設計稿為初始基準，本文件不修改原始 spec。此文件只定義需求與驗收範圍，不代表功能已實作。

## 目前狀態

- 前端已完成 OIDC 入口按鈕，會讀取 `GET /api/v1/auth/config`。
- 後端已提供公開 auth config，僅回傳 OIDC 是否啟用與 login URL。
- 後端已有 `config.yaml` / ENV 的 OIDC 設定骨架。
- 後端尚未註冊 `/api/v1/auth/oidc/login`。
- 後端尚未註冊 `/api/v1/auth/oidc/callback`。
- 尚未實作 OIDC token 驗證、使用者自動建立、群組 mapping 與登入同步。
- 尚未實作 SAML 設定、metadata、ACS、簽章驗證或登入流程。

## 範圍

本次規劃範圍包含：

- OIDC 登入啟動與 callback。
- OIDC Discovery 與 ID Token 驗證。
- OIDC 外部身分與本地使用者綁定。
- OIDC 群組 mapping 到 `users.global_role` 與權限群組。
- SAML SP initiated login、metadata 與 ACS callback。
- SAML assertion 驗證與外部身分綁定。
- SSO Provider 後台 DB 設定、secret 加密保存與管理 UI。
- OIDC / SAML Provider 連線測試與登入流程測試。
- SSO 登入後沿用現有 access token / refresh token / MFA policy。
- 安全稽核、登入日誌與錯誤處理。

本次規劃不包含：

- SCIM 使用者同步。
- IdP initiated SAML 第一版強制支援。
- OIDC / SAML 登入第一版不要求完成管理 UI；管理 UI 以後續階段補上。
- 多租戶 IdP 自助設定。
- Email domain 自動核准。
- 直接信任外部群組取代內部權限模型。

## 需求 1：SSO Provider 設定

管理者需要能設定 OIDC 或 SAML Provider，且敏感資訊不得暴露到前端。

### 驗收條件

- [ ] 1.1 系統需支援至少一個啟用中的 OIDC Provider。
- [ ] 1.2 系統需支援至少一個啟用中的 SAML Provider。
- [ ] 1.3 Provider 設定需包含 provider key、顯示名稱、啟用狀態、protocol、issuer 與 callback URL。
- [ ] 1.4 OIDC client secret、SAML private key 與其他敏感設定不得透過任何前端 API 明文回傳。
- [ ] 1.5 第一版可由 `config.yaml` / ENV 管理敏感設定；若放入 DB，必須加密保存，且加密 key 仍由 config / ENV 提供。
- [ ] 1.6 公開 auth config 只能回傳可顯示資訊，例如 provider key、名稱、protocol、login URL 與 enabled。

## 需求 2：OIDC 登入

使用者需要透過 OIDC Provider 登入 Opscenter。

### 驗收條件

- [ ] 2.1 `GET /api/v1/auth/oidc/login` 需產生 state、nonce，並導向 OIDC authorization endpoint。
- [ ] 2.2 state 與 nonce 需短效保存，callback 時必須驗證。
- [ ] 2.3 `GET /api/v1/auth/oidc/callback` 需驗證 code、state、nonce 與 ID Token。
- [ ] 2.4 ID Token 需驗證 issuer、audience、exp、iat、nonce 與簽章。
- [ ] 2.5 OIDC 使用者唯一識別需以 provider + subject 為主，不得只依 email 判斷。
- [ ] 2.6 若本地已存在相同 provider + subject 綁定，需登入該本地使用者。
- [ ] 2.7 若沒有綁定但 email 命中既有使用者，需依安全規則決定是否允許自動連結。
- [ ] 2.8 若允許自動建立使用者，新使用者預設 `global_role = member`，不得自動加入專案。
- [ ] 2.9 OIDC 登入成功後需沿用既有 token 簽發與 refresh token 流程。

## 需求 3：SAML 登入

使用者需要透過 SAML IdP 登入 Opscenter。

### 驗收條件

- [ ] 3.1 系統需提供 SAML metadata endpoint，供 IdP 設定 SP。
- [ ] 3.2 系統需提供 SAML login endpoint，產生 AuthnRequest 並導向 IdP。
- [ ] 3.3 系統需提供 ACS endpoint 接收 SAMLResponse。
- [ ] 3.4 SAMLResponse 必須驗證簽章、issuer、audience、NotBefore、NotOnOrAfter 與 recipient。
- [ ] 3.5 SAML NameID 或指定 attribute 需轉為 provider + subject。
- [ ] 3.6 SAML 登入成功後需沿用既有 token 簽發與 refresh token 流程。
- [ ] 3.7 SAML 第一版不強制支援 IdP initiated login；若開放，必須具備 RelayState 驗證與 redirect allowlist。

## 需求 4：外部身分綁定

系統需要保存外部 SSO 身分與本地使用者的關係，避免不同 IdP 或不同 subject 混淆。

### 驗收條件

- [ ] 4.1 系統需建立外部身分綁定資料模型。
- [ ] 4.2 綁定唯一鍵需至少包含 provider key、protocol 與 external subject。
- [ ] 4.3 同一外部身分不得綁定多個本地使用者。
- [ ] 4.4 同一本地使用者可綁定多個外部身分，但需保留 provider 區分。
- [ ] 4.5 自動連結既有 email 使用者必須可設定，預設建議關閉。
- [ ] 4.6 外部身分資料不得保存 ID Token、Access Token、SAML Assertion 原文或 refresh token。

## 需求 5：群組 Mapping 與權限同步

系統需要把外部群組轉換為內部授權結果，但不得直接使用外部群組作為權限判斷來源。

### 驗收條件

- [ ] 5.1 系統需提供 `sso_group_mappings` 或相容資料模型。
- [ ] 5.2 mapping 來源需包含 provider key、external group、target type、target code、priority 與啟用狀態。
- [ ] 5.3 `target_type = global_role` 時，只允許 `admin`、`op_admin`、`member`。
- [ ] 5.4 多個 global role mapping 命中時，需取最高權限序：`admin > op_admin > member`。
- [ ] 5.5 `target_type = permission_group` 時，只允許同步到已存在且啟用中的 `groups.code`。
- [ ] 5.6 未命中 mapping 的新 SSO 使用者預設 `global_role = member`。
- [ ] 5.7 未命中 mapping 時不得自動加入任何權限群組。
- [ ] 5.8 權限群組同步不得刪除手動加入的群組成員關係。
- [ ] 5.9 同步結果需寫入安全稽核摘要，但不得記錄原始 token 或完整 assertion。

## 需求 6：MFA 與登入狀態

SSO 成功只代表外部身分已驗證，系統仍需遵守 Opscenter 內部 MFA policy。

### 驗收條件

- [ ] 6.1 SSO 登入成功後需套用現有 `security.mfa.force_enable` 規則。
- [ ] 6.2 若本地 MFA policy 要求驗證，SSO callback 不得直接簽發正式 tokens，需回到既有 `mfa_required` 或 `mfa_setup_required` 流程。
- [ ] 6.3 第一版不信任 IdP MFA claim 作為本地 MFA 通過依據。
- [ ] 6.4 未來若支援信任 IdP MFA，需新增明確設定與稽核，不得預設啟用。

## 需求 7：前端體驗

使用者需要在登入頁看到可用的 SSO Provider，並在失敗時取得清楚提示。

### 驗收條件

- [ ] 7.1 登入頁需從 `GET /api/v1/auth/config` 取得可用 SSO Provider 清單。
- [ ] 7.2 OIDC 與 SAML Provider 可各自顯示登入按鈕。
- [ ] 7.3 Provider 未啟用時不得顯示入口。
- [ ] 7.4 SSO callback 失敗時需回到登入頁並顯示 i18n 錯誤。
- [ ] 7.5 SSO 成功但需要 MFA 時，前端需進入既有 MFA 驗證或強制 setup 流程。
- [ ] 7.6 前端不得顯示 provider_url、client_secret、SAML certificate private key 或其他敏感設定。

## 需求 8：安全與稽核

SSO 流程涉及外部信任邊界，需可追蹤且不得洩漏敏感資料。

### 驗收條件

- [ ] 8.1 OIDC state、nonce 與 SAML RelayState 必須短效保存。
- [ ] 8.2 callback redirect 只能導向系統允許的內部路徑或固定頁面。
- [ ] 8.3 登入成功、登入失敗、mapping 命中、mapping 未命中、使用者建立、使用者綁定需寫入稽核。
- [ ] 8.4 日誌與稽核不得記錄 authorization code、ID Token、Access Token、SAML Assertion、client secret 或 private key。
- [ ] 8.5 OIDC Discovery、JWKS 取得或 SAML metadata 解析失敗時需回明確錯誤，不得 fallback 到不驗證簽章。
- [ ] 8.6 Provider 停用後不得再啟動新的 SSO 登入流程。

## 需求 9：驗證

功能需具備可重複驗證的測試與手動驗收步驟。

### 驗收條件

- [ ] 9.1 後端測試需覆蓋 OIDC state / nonce 驗證。
- [ ] 9.2 後端測試需覆蓋 ID Token issuer / audience / signature 驗證失敗。
- [ ] 9.3 後端測試需覆蓋 OIDC 群組 mapping 最高權限選擇。
- [ ] 9.4 後端測試需覆蓋 permission group 不存在時不得建立假群組。
- [ ] 9.5 後端測試需覆蓋 SAML assertion 時間窗與 audience 驗證。
- [ ] 9.6 前端測試需覆蓋 Provider 清單顯示、按鈕導向與錯誤顯示。

## 需求 10：SSO Provider DB 與 UI 管理

管理者需要在系統管理中新增、編輯、停用與測試 OIDC / SAML Provider，不必直接修改 `config.yaml`。

### 驗收條件

- [ ] 10.1 系統需提供 SSO Provider 專用資料表，不得把完整 Provider YAML 直接塞入 `system_settings`。
- [ ] 10.2 `system_settings` 只保存全域小型設定，例如 `sso.state_ttl_seconds`、`sso.provider_config_source`。
- [ ] 10.3 Provider 資料需包含 `key`、`protocol`、`display_name`、啟用狀態、排序、建立者、更新者與時間戳。
- [ ] 10.4 OIDC Provider 需可設定 issuer URL、client id、redirect URL、scopes、groups claim、auto provision 與 auto link email。
- [ ] 10.5 SAML Provider 需可設定 entity ID、ACS URL、IdP metadata URL 或 IdP certificate、NameID format、groups attribute。
- [ ] 10.6 OIDC client secret、SAML private key 等敏感欄位必須加密保存，不得明文存放。
- [ ] 10.7 加密 key 不得存放於 DB，必須由 config / ENV 或部署 Secret 注入。
- [ ] 10.8 查詢與列表 API 不得回傳 secret 明文，只能回傳 `has_client_secret`、`has_private_key` 等狀態旗標。
- [ ] 10.9 修改 secret 時需以「重新輸入」覆蓋，前端不得顯示舊 secret。
- [ ] 10.10 停用 Provider 後不得出現在公開 auth config，也不得啟動新的登入流程。
- [ ] 10.11 刪除 Provider 預設採軟刪除或停用；若已有外部身分綁定或群組 mapping，需阻擋硬刪除。
- [ ] 10.12 Provider 設定異動需寫入安全稽核，包含建立、修改、停用、啟用、secret 更新與刪除。
- [ ] 10.13 管理 UI 入口需放在系統管理，且僅 `admin` 可管理。
- [ ] 10.14 第一版需保留 `config.yaml` / ENV bootstrap 或 fallback 能力，避免 DB 設定失誤造成所有 SSO 入口中斷。

## 需求 11：SSO Provider 測試功能

管理者需要在儲存 Provider 後執行連線測試與登入流程測試，確認設定可用後再啟用。

### 驗收條件

- [ ] 11.1 OIDC 連線測試需讀取 discovery metadata，確認 authorization endpoint、token endpoint 與 JWKS URI。
- [ ] 11.2 OIDC 連線測試需讀取 JWKS 並檢查 key 格式，不得略過簽章驗證要求。
- [ ] 11.3 OIDC 登入流程測試需導向 IdP 並完成 callback，但不得建立正式 session、不得建立 user、不得同步群組。
- [ ] 11.4 SAML 連線測試需解析 IdP metadata URL 或 certificate，確認 SSO URL、issuer 與簽章憑證可用。
- [ ] 11.5 SAML 登入流程測試需完成 RelayState / ACS 測試，但不得建立正式 session、不得建立 user、不得同步群組。
- [ ] 11.6 測試結果需保存最近一次狀態、時間、操作者與錯誤摘要。
- [ ] 11.7 測試錯誤訊息不得包含 client secret、private key、authorization code、token、完整 claim 或完整 assertion。
- [ ] 11.8 未通過必要測試的 Provider 不得直接啟用；若允許強制啟用，需 admin 權限與稽核原因。
- [ ] 11.9 前端需顯示測試狀態，例如未測試、成功、失敗、測試中。
- [ ] 11.10 測試 callback redirect 只能導回管理端固定測試結果頁，不得接受任意外部 redirect。

## 需求 12：SSO Mapping 管理與維運狀態

管理者需要在系統管理中檢視 Provider 狀態、最近測試結果與群組 mapping，讓維運可以快速判斷登入失敗是 Provider、測試、mapping 或帳號連結問題。

### 驗收條件

- [ ] 12.1 Provider 狀態 API 需回傳 provider key、protocol、display name、enabled、config source、設定完整性與最後測試摘要。
- [ ] 12.2 Provider 狀態 API 不得回傳 issuer secret、client secret、private key、token、完整 claim 或完整 assertion。
- [ ] 12.3 Mapping 管理 API 需支援依 provider key 查詢、新增、修改、停用與重新啟用 mapping。
- [ ] 12.4 Mapping target type 僅允許 `global_role` 與 `permission_group`；target code 需驗證是否為合法角色或已存在且啟用的權限群組。
- [ ] 12.5 同一 provider key、external group、target type、target code 不得重複建立有效 mapping。
- [ ] 12.6 Mapping 異動需寫入 security audit，包含建立、修改、停用、啟用與刪除。
- [ ] 12.7 Provider 狀態與 mapping 管理 API 僅 `admin` 可操作。
- [ ] 12.8 管理 UI 需顯示 mapping 命中會造成的 global role 與 permission group 結果，並標示未通過測試或設定不完整的 Provider。
- [ ] 12.9 維運查詢需能看到最近 callback 失敗摘要與最後測試摘要，但不得顯示敏感原文。
