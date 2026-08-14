# MFA Global Setting 接線需求

## 文件定位

本 spec 接續 `../2026-06-01_10-22_oncall-ticket-system` 的 MFA 原始設計，將已存在但尚未完整接線的 Global Setting 納入登入與 MFA setup 流程。

原始設計稿為初始設計基準，本文件不修改原始 spec。

## 舊設計檢查結論

- 原始需求 1.9 已定義 MFA 啟用後，登入時必須完成第二因子驗證才能取得 Token。
- 原始後端設計已 seed `security.mfa.force_enable`、`security.mfa.allowed_types`、`security.mfa.pending_device_retention_minutes`。
- 原始前端設計已定義登入回應 `mfa_required`、`mfa_setup_required` 與 `temp_token`，並規劃 `/auth/mfa/setup` 強制 MFA setup 頁。
- 目前後端登入流程在帳密驗證成功後直接簽發 access token 與 refresh token，尚未依 MFA 設定分流。
- 目前 MFA 裝置管理與未驗證裝置清理已存在，但 `force_enable` 與 `allowed_types` 尚未作為登入決策來源。

## 範圍

本次範圍包含：

- 後端讀取 `security.mfa.force_enable` 與 `security.mfa.allowed_types`。
- 登入流程依 MFA 狀態回傳 `mfa_required` 或 `mfa_setup_required`。
- 建立登入後、正式 token 前使用的短效 `temp_token`。
- 支援強制 MFA setup 完成後才簽發正式 token。
- 前端沿用既有登入狀態機與強制 MFA setup 頁契約。
- 補齊驗證、稽核與錯誤處理規則。

本次範圍不包含：

- Email MFA、SMS MFA、WebAuthn。
- 備用碼登入。
- 使用者個別豁免 MFA。
- Admin 自身豁免 MFA。第一版 `security.mfa.force_enable = true` 時適用所有啟用帳號，包含 admin。
- OIDC / SAML MFA 串接。

## 需求 1：MFA 政策設定

管理者需要透過 Global Setting 控制 MFA 是否強制啟用，以及允許的 MFA 類型。

### 驗收條件

- [ ] 1.1 後端需讀取 `security.mfa.force_enable`；缺值、停用或解析失敗時預設為 `false`。
- [ ] 1.2 `security.mfa.force_enable = true` 時，所有啟用帳號在取得正式 token 前都必須有已驗證且啟用中的 MFA 裝置。
- [ ] 1.3 後端需讀取 `security.mfa.allowed_types`；缺值、停用或解析失敗時預設為 `["totp"]`。
- [ ] 1.4 第一版只支援 `totp`。Global Setting 更新時若包含不支援類型，後端需拒絕儲存或回傳明確驗證錯誤。
- [ ] 1.5 若 `allowed_types` 不包含 `totp`，`POST /api/v1/auth/mfa/setup` 不得建立 TOTP 裝置，需回傳設定不可用錯誤。
- [ ] 1.6 `security` 類別設定維持只有 `admin` 可管理；`op_admin` 不可修改 MFA / SSO / 安全設定。

## 需求 2：登入 MFA 分流

使用者完成帳密驗證後，系統需依 MFA 狀態決定是否簽發正式 token。

### 驗收條件

- [ ] 2.1 帳號不存在、帳號停用或密碼錯誤時，登入行為維持既有 401，不揭露帳號狀態。
- [ ] 2.2 使用者已有已驗證且啟用中的 MFA 裝置時，帳密驗證成功後需回傳 `mfa_required = true`、`temp_token`，且不得回傳 access token 或 refresh token。
- [ ] 2.3 使用者尚無已驗證且啟用中的 MFA 裝置，且 `security.mfa.force_enable = true` 時，帳密驗證成功後需回傳 `mfa_setup_required = true`、`temp_token`，且不得回傳 access token 或 refresh token。
- [ ] 2.4 使用者尚無已驗證 MFA 裝置，且 `security.mfa.force_enable = false` 時，登入流程維持既有行為，直接回傳 access token 與 refresh token。
- [ ] 2.5 `password_verified = true` 只代表帳密已驗證，不代表使用者已取得完整登入狀態；前端不得在 tokens 缺失時寫入登入狀態。
- [ ] 2.6 MFA 驗證或強制 setup 完成後，後端才可簽發 access token 與 refresh token。

## 需求 3：暫時 Token

系統需要短效憑證承接「帳密已通過但尚未完成 MFA」的流程。

### 驗收條件

- [ ] 3.1 `temp_token` 必須短效，預設有效時間不得超過 10 分鐘。
- [ ] 3.2 `temp_token` 需包含用途範圍，例如 `mfa_verify` 或 `mfa_setup`，不得被一般 auth middleware 視為 access token。
- [ ] 3.3 `temp_token` 不得授權讀取一般 API、專案資料、Ticket、使用者列表或系統設定。
- [ ] 3.4 `temp_token` 不得寫入前端一般 access token store。
- [ ] 3.5 `temp_token` 驗證失敗、過期或用途不符時，後端需回 401 或 403，並回傳 i18n 可對應的錯誤碼。

## 需求 4：MFA setup 與驗證流程

使用者需能在登入流程中完成強制 MFA setup，也需保留已登入後的個人 MFA 管理。

### 驗收條件

- [ ] 4.1 `POST /api/v1/auth/mfa/setup` 需支援已登入 access token 的個人 MFA 新增流程。
- [ ] 4.2 `POST /api/v1/auth/mfa/setup` 需支援 `temp_token` 的強制 MFA setup 流程，且只能在 `mfa_setup` 用途下使用。
- [ ] 4.3 強制 MFA setup 建立的裝置仍需先標記為未驗證，TOTP 驗證成功後才可視為已綁定。
- [ ] 4.4 `POST /api/v1/auth/mfa/verify` 需支援 `temp_token` 的登入 MFA 驗證流程。
- [ ] 4.5 登入 MFA 驗證時，若 request 未指定 `device_id`，後端可對使用者所有已驗證且啟用中的 TOTP 裝置逐一驗證。
- [ ] 4.6 強制 MFA setup 驗證成功後，後端需同一次 response 回傳正式 access token 與 refresh token。
- [ ] 4.7 `GET /api/v1/auth/mfa/devices` 與 `DELETE /api/v1/auth/mfa/devices/:id` 維持只接受正式 access token，不接受 `temp_token`。
- [ ] 4.8 未驗證 MFA 裝置清理維持由 `security.mfa.pending_device_retention_minutes` 控制。

## 需求 5：前端登入體驗

前端需依後端 MFA 分流結果呈現正確畫面，避免使用者在未完成 MFA 前進入系統。

### 驗收條件

- [ ] 5.1 登入 API 回傳 `mfa_required = true` 時，登入卡片需切換為 TOTP 驗證畫面，不跳頁。
- [ ] 5.2 登入 API 回傳 `mfa_setup_required = true` 時，前端需保存 `temp_token` 並導向 `/auth/mfa/setup`。
- [ ] 5.3 MFA 驗證或 setup 完成前，前端不得呼叫需正式登入的 API。
- [ ] 5.4 MFA 驗證成功且取得正式 tokens 後，前端需清除 `temp_token` 並寫入正式登入狀態。
- [ ] 5.5 使用者返回登入頁、登出或流程逾時時，前端需清除 `temp_token`。
- [ ] 5.6 相關錯誤訊息需補齊繁體中文 i18n，不得顯示原始英文錯誤。

## 需求 6：稽核與安全

MFA 流程涉及登入安全，所有狀態轉換需可追蹤但不得洩漏敏感資料。

### 驗收條件

- [ ] 6.1 帳密成功但待 MFA 驗證、MFA 驗證成功、MFA 驗證失敗、強制 MFA setup 成功與失敗需寫入安全稽核或登入稽核。
- [ ] 6.2 日誌與稽核不得記錄 password、TOTP secret、TOTP code、access token、refresh token 或 `temp_token` 明文。
- [ ] 6.3 `temp_token` 過期、用途不符與 MFA 驗證失敗需有可觀測日誌，並包含 request id / trace id。
- [ ] 6.4 Refresh token 只能在 MFA 完成後簽發；refresh endpoint 不得成為繞過 MFA 的路徑。
- [ ] 6.5 MFA policy 讀取失敗時，後端需採安全預設並記錄錯誤；不得因讀取失敗直接放行強制 MFA。

## 需求 7：驗證

本功能需以測試覆蓋主要登入分支與設定分支。

### 驗收條件

- [ ] 7.1 後端單元測試需覆蓋 `force_enable = false` 且無 MFA 裝置時直接登入成功。
- [ ] 7.2 後端單元測試需覆蓋已有已驗證 MFA 裝置時回傳 `mfa_required` 且不回正式 tokens。
- [ ] 7.3 後端單元測試需覆蓋 `force_enable = true` 且無 MFA 裝置時回傳 `mfa_setup_required` 且不回正式 tokens。
- [ ] 7.4 後端單元測試需覆蓋 `allowed_types` 不包含 `totp` 時 setup 被拒絕。
- [ ] 7.5 後端單元測試需覆蓋 `temp_token` 過期、用途不符與驗證成功簽發正式 tokens。
- [ ] 7.6 前端測試需覆蓋登入 MFA 驗證畫面、強制 MFA setup 導向與 tokens 寫入時機。
