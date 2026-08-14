# SSO / OIDC / SAML Task

## 1. 現況收斂與入口保護

- [x] 1.1 確認公開 Auth Config 契約
  - 保留既有 `oidc.enabled` / `oidc.login_url` 相容欄位
  - 新增 `sso.providers` 規劃契約
  - 不回傳 issuer URL、client secret、SAML private key 或其他敏感設定
  - _Requirements: 1.4, 1.6, 7.1, 7.6；Design: 公開 Auth Config API_

- [x] 1.2 補 OIDC 未完成時的入口保護
  - 後端尚未提供 login / callback handler 前，不得讓公開 config 宣告 OIDC 可登入
  - 避免前端按鈕導向尚未實作的 `/api/v1/auth/oidc/login`
  - _Requirements: 7.3, 8.6；Planning: 階段 0_

- [x] 1.3 補 SSO 錯誤碼與前端 i18n 規劃
  - `sso_failed`
  - `sso_provider_disabled`
  - `sso_account_not_allowed`
  - `sso_mapping_failed`
  - _Requirements: 7.4, 8.3；Design: 前端設計_

## 2. 資料模型與 Migration

- [x] 2.1 建立外部身分綁定資料模型
  - 新增 `sso_external_identities`
  - 唯一鍵使用 provider key、protocol、external subject
  - 不保存 token、assertion 或外部 refresh token
  - _Requirements: 4.1-4.6；Design: sso_external_identities_

- [x] 2.2 建立 SSO 群組 mapping 資料模型
  - 新增 `sso_group_mappings`
  - 支援 `global_role` 與 `permission_group`
  - seed 預設 `ops_admin`、`ops_op_admin`、`ops_member`、`ops_op_member`
  - _Requirements: 5.1-5.7；Design: sso_group_mappings_

- [x] 2.3 補群組成員來源欄位
  - `group_members.source`
  - `group_members.source_provider`
  - SSO 同步只管理 `source = sso` 的關係
  - 不刪除手動加入的群組成員
  - _Requirements: 5.8；Design: 群組同步策略_

## 3. Provider 設定與安全設定層

- [x] 3.1 擴充 SSO Provider config
  - 支援 `protocol = oidc`
  - 支援 `protocol = saml`
  - 第一版敏感設定走 config / ENV
  - _Requirements: 1.1-1.5；Design: Provider 設定模型_

- [x] 3.2 實作 Provider resolver
  - 依 provider key 查詢已啟用 provider
  - 停用 provider 不得啟動登入
  - 設定不完整時回明確錯誤
  - _Requirements: 1.3, 8.6；Design: API 設計_

- [x] 3.3 補 SSO state store
  - 支援 OIDC state / nonce
  - 支援 SAML RelayState
  - TTL 預設 5 到 10 分鐘
  - 成功或失敗後需清理
  - _Requirements: 2.1, 2.2, 8.1；Design: OIDC 流程設計, SAML 流程設計_

## 4. OIDC 基礎登入

- [x] 4.1 實作 OIDC login endpoint
  - `GET /api/v1/auth/sso/:provider_key/login`
  - 相容 `GET /api/v1/auth/oidc/login`
  - 產生 state、nonce 並導向 authorization endpoint
  - _Requirements: 2.1, 2.2；Planning: 階段 1_

- [x] 4.2 實作 OIDC discovery 與 JWKS 驗證
  - 讀取 discovery metadata
  - 快取 JWKS
  - 驗證 ID Token 簽章
  - _Requirements: 2.3, 2.4, 8.5；Design: OIDC 流程設計_

- [x] 4.3 實作 OIDC callback
  - `GET /api/v1/auth/oidc/callback`
  - 驗證 code、state、nonce
  - 驗證 issuer、audience、exp、iat
  - _Requirements: 2.3, 2.4, 8.1, 8.5_

- [x] 4.4 實作外部身分解析與本地 user resolution
  - 先查 provider + subject 綁定
  - 視設定決定是否 email 自動連結
  - 視設定決定是否 auto provision
  - 新使用者預設 `global_role = member`
  - _Requirements: 2.5-2.8, 4.1-4.6；Design: 使用者解析策略_

- [x] 4.5 實作 OIDC 登入後 token / MFA 接線
  - 不需要 MFA 時簽發正式 tokens
  - 需要 MFA 時進入既有 `mfa_required` 或 `mfa_setup_required` 流程
  - 第一版不信任 IdP MFA claim
  - _Requirements: 2.9, 6.1-6.4；Design: MFA 接線_

- [x] 4.6 補 OIDC 登入稽核
  - 登入成功
  - 登入失敗
  - 外部身分首次綁定
  - 自動建立 user
  - 不記錄 authorization code、token、secret
  - _Requirements: 8.3, 8.4；Design: 稽核與日誌_

## 5. OIDC 群組 Mapping

- [x] 5.1 實作 group mapping resolver
  - 從 OIDC groups claim 讀取外部群組
  - 查詢啟用 mapping
  - 去重外部群組
  - _Requirements: 5.1, 5.2；Planning: 階段 2_

- [x] 5.2 實作 global role mapping
  - 只允許 `admin`、`op_admin`、`member`
  - 多個命中取最高權限序
  - 未命中時新使用者維持 `member`
  - _Requirements: 5.3, 5.4, 5.6_

- [x] 5.3 實作 permission group 同步
  - 只同步已存在且啟用中的 group
  - 不建立假群組
  - 不刪手動群組成員關係
  - _Requirements: 5.5, 5.7, 5.8_

- [x] 5.4 補 mapping 稽核摘要
  - 命中 mapping
  - 未命中 mapping
  - global role 變更
  - permission group 同步摘要
  - 不記錄完整 claim 或 token
  - _Requirements: 5.9, 8.3, 8.4_

## 6. SAML 基礎登入

- [x] 6.1 實作 SAML Provider config
  - IdP issuer / entity ID
  - entity ID
  - ACS URL
  - IdP metadata URL 或 certificate
  - 使用 IdP certificate 時需有 IdP SSO URL
  - SP certificate / private key 供 metadata 與簽署 AuthnRequest 使用
  - groups attribute
  - _Requirements: 1.2, 1.3；Planning: 階段 3_

- [x] 6.2 實作 SAML metadata endpoint
  - `GET /api/v1/auth/saml/:provider_key/metadata`
  - 輸出 SP entity ID、ACS URL、certificate
  - _Requirements: 3.1；Design: SAML 流程設計_

- [x] 6.3 實作 SAML login endpoint
  - `GET /api/v1/auth/saml/:provider_key/login`
  - 產生 AuthnRequest
  - 建立 RelayState
  - 導向 IdP SSO URL
  - _Requirements: 3.2, 8.1_

- [x] 6.4 實作 SAML ACS endpoint
  - `POST /api/v1/auth/saml/:provider_key/acs`
  - 驗證 RelayState
  - 解析 SAMLResponse
  - _Requirements: 3.3, 8.1_

- [x] 6.5 實作 SAML assertion 驗證
  - 簽章
  - issuer
  - audience
  - recipient
  - NotBefore / NotOnOrAfter
  - InResponseTo
  - _Requirements: 3.4, 8.5_

- [x] 6.6 實作 SAML 使用者解析與登入完成
  - NameID 或指定 subject attribute 轉 external subject
  - 套用外部身分綁定
  - 套用 group mapping
  - 套用 MFA policy
  - _Requirements: 3.5, 3.6, 4.1-4.6, 5.1-5.9, 6.1-6.4_

## 7. 前端 SSO 體驗

- [x] 7.1 擴充登入頁 SSO Provider 清單
  - 優先讀 `sso.providers`
  - 保留既有 `oidc` 欄位相容
  - Provider 未啟用不顯示
  - _Requirements: 7.1-7.3；Design: 前端設計_

- [x] 7.2 實作 OIDC / SAML 登入按鈕
  - 依 protocol 顯示 Provider 名稱
  - 點擊導向 `login_url`
  - 不顯示敏感設定
  - _Requirements: 7.2, 7.6_

- [x] 7.3 實作 SSO callback 錯誤呈現
  - 回到登入頁
  - 顯示 i18n 錯誤
  - 清除暫存登入狀態
  - _Requirements: 7.4, 8.2_

- [x] 7.4 接上 SSO 後 MFA 分流
  - SSO 成功但需要 MFA 時進入既有 MFA flow
  - MFA 完成後才寫入正式登入狀態
  - _Requirements: 7.5, 6.1-6.4_

## 8. 管理與維運規劃

- [x] 8.1 規劃 Provider 狀態 API
  - 顯示 provider key、protocol、enabled、最後測試結果
  - 不回傳 secret
  - _Requirements: 1.4, 1.6, 12.1, 12.2, 12.9；Planning: 階段 4；Design: Provider 狀態 API_

- [x] 8.2 規劃 mapping 管理 API
  - 查詢 mapping
  - 新增 / 修改 / 停用 mapping
  - 僅 admin 可管理
  - _Requirements: 5.1-5.9, 8.3, 12.3-12.8；Design: Mapping 管理 API_

- [x] 8.3 規劃 Provider 測試功能
  - OIDC discovery 測試
  - JWKS 讀取測試
  - SAML metadata 解析測試
  - _Requirements: 8.5, 8.6, 11.1-11.10；Design: Provider 測試 API_

## 9. 驗證

- [x] 9.1 補 OIDC 後端測試
  - state / nonce 驗證
  - issuer / audience / signature 驗證失敗
  - user resolution
  - MFA 分流
  - _Requirements: 9.1, 9.2_

- [x] 9.2 補 OIDC group mapping 測試
  - `ops_admin`
  - `ops_op_admin`
  - `ops_member`
  - 多群組最高權限
  - permission group 不存在時不得建立假群組
  - _Requirements: 9.3, 9.4_

- [ ] 9.3 補 SAML 後端測試
  - assertion 時間窗
  - audience
  - recipient
  - signature 驗證失敗
  - _Requirements: 9.5_

- [ ] 9.4 補前端測試
  - Provider 清單顯示
  - disabled provider 不顯示
  - 按鈕導向
  - callback 失敗訊息
  - SSO 後 MFA 分流
  - _Requirements: 9.6_

- [ ] 9.5 手動驗收
  - OIDC 成功登入
  - OIDC callback 錯誤回登入頁
  - OIDC group mapping 正確同步
  - SAML SP initiated login 成功
  - 停用 provider 不可登入
  - SSO 成功後仍遵守本地 MFA policy
  - _Requirements: 1.1-9.6_
