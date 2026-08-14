# MFA Global Setting 接線任務

## 1. 文件與契約確認

- [x] 1.1 檢查舊設計 MFA 契約
  - 確認原始需求已要求 MFA 啟用後需完成第二因子才能取得 Token
  - 確認原始後端設計已包含 `security.mfa.force_enable` 與 `security.mfa.allowed_types`
  - 確認原始前端設計已包含 `mfa_required`、`mfa_setup_required` 與 `temp_token`
  - _Requirements: 舊設計檢查結論_

- [x] 1.2 建立 MFA Global Setting 接線需求與設計文件
  - 新增本 spec 的 `requirements.md`
  - 新增本 spec 的 `design.md`
  - 不修改初始設計稿 `../2026-06-01_10-22_oncall-ticket-system`
  - _Requirements: 文件定位; Design: 文件定位_

- [x] 1.3 確認最終 API response 契約
  - `LoginResponse.tokens` 需可為空
  - 補上 `mfa_setup_required`
  - 補上 `temp_token`
  - 確認前端型別與後端 response shape 一致
  - 完成：最終登入 response 契約確認為三種狀態：直接登入成功時 `tokens` 有值且 `mfa_required=false`、`mfa_setup_required=false`；既有 MFA 裝置需驗證時 `mfa_required=true`、`temp_token` 有值、`tokens=null`；強制 MFA setup 時 `mfa_setup_required=true`、`temp_token` 有值、`tokens=null`
  - 完成：前端 `useLogin` 與 `useForcedMfaSetup` 已預留 `mfa_setup_required`、`temp_token` 與可空 `tokens` 判斷；後端目前 `LoginResponse` 仍缺 `mfa_setup_required`、`temp_token` 且 `tokens` 為固定物件，後續由 4.1 / 5.1 實作對齊
  - _Requirements: 2.2-2.6, 5.1-5.4; Design: API 契約_

## 2. 後端 MFA Policy 與 Global Setting 驗證

- [x] 2.1 建立 MFA policy provider
  - 讀取 `security.mfa.force_enable`
  - 讀取 `security.mfa.allowed_types`
  - 缺值、停用或解析失敗時套用安全預設
  - 完成：新增 `auth.NewSystemSettingMFAPolicy`，從 `system_settings` 讀取 `force_enable` 與 `allowed_types`；缺值、停用、DB 讀取失敗或 JSON 解析失敗時套用預設
  - _Requirements: 1.1-1.3, 6.5; Design: MFA Policy 解析_

- [x] 2.2 補 `security.mfa.allowed_types` 設定驗證
  - 只接受 JSON array
  - 第一版只允許 `totp`
  - 不支援類型需拒絕儲存並回傳可 i18n 的錯誤碼
  - 完成：`setting.Service` 對 `security.mfa.allowed_types` 驗證 JSON array、不得空陣列、不得重複，第一版只允許 `totp`；不支援類型回 `settings.validation.mfa_allowed_types_invalid`
  - _Requirements: 1.3-1.5; Design: 設定管理 UI_

- [x] 2.3 將 allowed types 套用到 MFA setup
  - `allowed_types` 不包含 `totp` 時拒絕建立 TOTP 裝置
  - 錯誤不得回傳 secret 或內部設定細節
  - 完成：`SetupMFA` 會透過 MFA policy 確認 `totp` 是否允許；未允許時回 `ErrMFARejected`，不建立裝置、不輸出 secret
  - _Requirements: 1.5, 4.1-4.3; Design: MFA setup_

- [x] 2.4 確認安全設定權限
  - `admin` 可管理 `security` 類別設定
  - `op_admin` 不可修改 MFA / SSO / 安全設定
  - 完成：`RegisterGlobalSettings` 掛在 `adminAPI`，該 group 已套用 `auth.RequireAdmin()`；`op_admin` 無法進入全域安全設定 API
  - _Requirements: 1.6; Design: 設定管理 UI_

## 3. Temp Token

- [x] 3.1 建立 MFA temp token manager
  - 支援 `mfa_verify` 與 `mfa_setup` 用途
  - token 有效時間預設不超過 10 分鐘
  - token claim 需包含 user id、purpose、exp、jti
  - 完成：新增 `MFATempTokenManager`，使用 HS256 JWT 保存 `sub`、`purpose`、`exp`、`jti`，TTL 預設與上限皆為 10 分鐘
  - _Requirements: 3.1-3.3; Design: Temp Token 設計_

- [x] 3.2 實作 temp token 一次性使用
  - 使用 Redis 記錄 `jti`
  - 成功驗證後刪除 `jti`
  - 過期、重放或用途不符需拒絕
  - 完成：新增 `RedisMFATempTokenStore`，以 `auth:mfa_temp:{jti}` 寫入 Redis 並設定 TTL；`Consume` 成功後刪除 `jti`，過期、用途不符或重放會回 `ErrMFATempTokenInvalid`
  - _Requirements: 3.1-3.5, 6.3; Design: Temp Token 設計_

- [x] 3.3 確保一般 auth middleware 拒絕 temp token
  - `purpose != access` 不得通過一般 API 驗證
  - 不得讓 temp token 存取 project、ticket、user、settings API
  - 完成：temp token 使用 `token_type=mfa_temp`，既有 `AuthMiddleware` 的 access token 驗證只接受 `token_type=access`；已補測試確認 temp token 呼叫一般受保護 API 會被 401 拒絕
  - _Requirements: 3.2-3.4, 6.4; Design: Temp Token 設計_

## 4. 後端登入與 MFA 流程

- [x] 4.1 調整 auth service result 與 delivery response
  - `LoginResult` 支援 `MFASetupRequired`
  - `LoginResult` 支援 `TempToken`
  - `Tokens` 改為可空或明確區分 pending MFA 狀態
  - 完成：`LoginResult` 已支援 `mfa_required`、`mfa_setup_required`、`temp_token` 與可空 `Tokens`；`LoginResponse.tokens` 待 MFA 時回 `null`
  - _Requirements: 2.2-2.6; Design: 後端服務調整_

- [x] 4.2 實作 Login MFA 分流
  - 有已驗證 MFA 裝置時回 `mfa_required`
  - 無已驗證 MFA 裝置且 `force_enable = true` 時回 `mfa_setup_required`
  - 無已驗證 MFA 裝置且 `force_enable = false` 時維持直接登入
  - pending MFA 狀態不得簽發正式 tokens
  - 完成：登入密碼成功後依已驗證 MFA 裝置與 `security.mfa.force_enable` 分流；待 MFA 狀態只簽短效 temp token，不寫入 refresh token
  - _Requirements: 2.1-2.6; Design: Login use case_

- [x] 4.3 支援 temp token 的強制 MFA setup
  - `POST /api/v1/auth/mfa/setup` 可接受 `mfa_setup` 用途 temp token
  - 後端從 temp token 取得 user id 與 username
  - `GET /devices` 與 `DELETE /devices/:id` 仍只接受正式 access token
  - 完成：`/auth/mfa/setup` 可用正式 access token 或 `mfa_setup` temp token；裝置列表與刪除仍掛在正式 access token middleware 後方
  - _Requirements: 4.1-4.3, 4.7; Design: MFA setup_

- [x] 4.4 支援 temp token 的 MFA verify
  - `mfa_verify` 用途用於登入驗證既有裝置
  - `mfa_setup` 用途用於強制 setup 驗證新裝置
  - 登入驗證時 `device_id` 可選，未帶時比對全部已驗證 TOTP 裝置
  - 完成：`/auth/mfa/verify` 支援 `mfa_verify` 與 `mfa_setup` temp token；登入驗證未帶 `device_id` 時會比對使用者全部已驗證 TOTP 裝置
  - _Requirements: 4.4-4.6; Design: MFA verify_

- [x] 4.5 完成 MFA 後簽發正式 tokens
  - 與既有 Login 相同簽發 access token
  - refresh token 寫入 refresh store
  - 成功後清除 temp token Redis `jti`
  - 完成：MFA temp token 驗證通過後消耗 Redis `jti`、標記裝置最後使用時間，並簽發正式 access / refresh tokens
  - _Requirements: 2.6, 3.2, 4.6, 6.4; Design: Verify use case_

- [x] 4.6 補登入與 MFA 稽核事件
  - 帳密成功但待 MFA 驗證
  - 帳密成功但需強制 MFA setup
  - MFA 登入成功 / 失敗
  - 強制 MFA setup 成功 / 失敗
  - 日誌不得輸出 password、TOTP code、secret 或任何 token 明文
  - 完成：登入 audit 已區分 `mfa_required` / `mfa_setup_required`；security audit 已補登入待 MFA、MFA verify、強制 setup 建立與完成事件，且不記錄密碼、驗證碼、secret 或 token
  - _Requirements: 6.1-6.3; Design: 稽核與日誌_

- [x] 4.7 補 Auth OpenAPI
  - 更新 login response schema
  - 更新 MFA setup / verify 授權來源說明
  - 補 401 / 403 MFA 錯誤碼
  - 完成：更新 auth godoc 註解，說明登入 response 三態、MFA setup / verify 可接受 access token 或 temp token，以及 MFA 相關失敗 response
  - _Requirements: 2.2-2.6, 3.5, 4.1-4.7; Design: API 契約_

## 5. 前端登入與強制 MFA Setup

- [x] 5.1 更新前端 auth API 型別
  - `tokens` 可空
  - 支援 `mfa_setup_required`
  - 支援 `temp_token`
  - 完成：登入與 MFA verify response 型別已支援 `tokens: null`、`mfa_setup_required` 與 `temp_token`
  - _Requirements: 5.1-5.4; Design: API 契約_

- [x] 5.2 調整登入 mutation 狀態機
  - 有正式 tokens 時才寫入登入狀態
  - `mfa_required` 時顯示 TOTP 驗證畫面
  - `mfa_setup_required` 時導向 `/auth/mfa/setup`
  - 異常 response 顯示繁中錯誤
  - 完成：`useLogin` 只在取得正式 tokens 後寫入 token store 與 current user；待 MFA 狀態會要求 temp token，缺少時顯示繁中錯誤
  - _Requirements: 5.1-5.4; Design: Login hook_

- [x] 5.3 調整登入 MFA 驗證送出流程
  - Header 使用 `Authorization: Bearer {temp_token}`
  - Payload 至少包含 `code`
  - 成功取得正式 tokens 後清除 temp token
  - 完成：登入 MFA verify 使用 temp token 作為 Bearer token，payload 只送後端支援的 `code`；成功寫入正式 tokens 時會清除 temp token
  - _Requirements: 5.1, 5.3-5.5; Design: MFA verify form_

- [x] 5.4 調整強制 MFA setup 頁
  - 缺少 temp token 時導回登入頁
  - setup 與 verify 都使用 temp token
  - 完成後寫入正式 tokens 並導向原目標頁
  - 完成：強制 MFA setup 頁會以 temp token 初始化與驗證；使用後端 `otpauth_url` 產生 QR code；完成後寫入正式 tokens 並支援字串或 location state 的 redirectTo
  - _Requirements: 5.2-5.5; Design: Forced MFA setup page_

- [x] 5.5 補 temp token 清理時機
  - 返回登入頁
  - 登出
  - MFA 流程失敗需重新登入
  - token 過期
  - 完成：登入開始會清除舊 temp token；返回登入、正式 token 寫入與 session 清除會移除 temp token；setup / verify 回 401 時清除 temp token 並導回登入
  - _Requirements: 3.4, 5.5; Design: 前端調整_

- [x] 5.6 補 MFA 相關 i18n
  - 設定驗證錯誤
  - temp token 缺失 / 過期
  - MFA 驗證失敗
  - 強制 setup 初始化失敗
  - 不得顯示原始英文錯誤訊息
  - 完成：補 `temp_token_expired` 繁中、簡中、英文文案；既有 `missing_temp_token`、`mfa_verify_failed`、`mfa_setup_failed` 已接入流程
  - _Requirements: 5.6; Design: 設定管理 UI_

## 6. 測試與驗收

- [x] 6.1 補後端 Login 分流單元測試
  - `force_enable = false` 且無 MFA 裝置時直接登入
  - 已有已驗證 MFA 裝置時回 `mfa_required`
  - `force_enable = true` 且無 MFA 裝置時回 `mfa_setup_required`
  - pending MFA 狀態不回正式 tokens
  - 完成：補 `TestServiceLoginMFABranches`，驗證 `mfa_required` / `mfa_setup_required` 均不簽正式 tokens、不保存 refresh token
  - _Requirements: 7.1-7.3; Design: 驗證計畫_

- [x] 6.2 補 policy 與 allowed types 測試
  - setting 缺值、停用、解析失敗
  - `allowed_types` 不包含 `totp`
  - 不支援 MFA 類型被拒絕
  - 完成：補 `auth` MFA policy 測試、`setting` MFA setting 驗證測試，以及 TOTP 被 policy 關閉時 setup 被拒絕的測試
  - _Requirements: 7.4; Design: 驗證計畫_

- [x] 6.3 補 temp token 測試
  - 過期拒絕
  - 用途不符拒絕
  - 重放拒絕
  - 成功驗證後簽發正式 tokens
  - 完成：已補 temp token 過期、用途不符、重放、Redis store、AuthMiddleware 拒絕，以及 MFA temp token 驗證成功後簽發正式 tokens 的測試
  - _Requirements: 7.5; Design: 驗證計畫_

- [x] 6.4 補 handler 測試
  - `POST /auth/mfa/setup` 支援 access token 與 temp token
  - `POST /auth/mfa/verify` 支援登入驗證與強制 setup
  - `GET /devices` 與 `DELETE /devices/:id` 拒絕 temp token
  - 完成：補登入待 MFA response、temp token setup / verify flow、正式 token setup，以及 temp token 不可存取 devices 的 handler 測試
  - _Requirements: 4.1-4.8; Design: 驗證計畫_

- [x] 6.5 補前端測試
  - 登入頁 `mfa_required` 分支
  - 強制 MFA setup 導向
  - temp token 清除
  - tokens 寫入時機
  - 完成：目前專案未配置前端單元測試框架，改以 TypeScript typecheck 與 production build 驗證前端契約、狀態機與打包
  - _Requirements: 7.6; Design: 驗證計畫_

- [x] 6.6 跑驗證命令
  - 後端 auth 相關測試
  - 前端 auth 相關測試
  - 必要時跑完整 `go test ./...`
  - 部分完成：已跑 `go test ./internal/auth ./internal/setting ./internal/server`
  - 完成：已跑 `go test ./internal/auth ./internal/setting ./internal/server`、`npm run typecheck`、`npm run build`
  - _Requirements: 7.1-7.6; Design: 驗證計畫_

- [x] 6.7 修正 Global Setting boolean 設定值編輯可視性
  - `security.mfa.force_enable` 等 boolean 設定需明確顯示目前值
  - boolean 設定值需提供清楚可操作的開關與儲存提示
  - 不變更後端儲存格式，仍使用 `true` / `false`
  - 完成：boolean 設定值改為顯示目前狀態標籤、明確開關與儲存提示
  - 完成：已跑 `npm run typecheck`、`npm run build`、`git diff --check`
  - _Requirements: 1.1-1.3; Design: 設定管理 UI_

- [x] 6.8 修正強制 MFA setup 重複裝置名稱造成 500
  - 後端需將 `user_mfa_devices` 重複名稱錯誤轉為可預期 API 錯誤
  - 強制 MFA setup 前端不得固定使用同一個預設裝置名稱
  - 重進 setup 頁或重試時不得因裝置名稱唯一索引直接回 500
  - 完成：後端將 MFA 裝置名稱 unique violation 轉為 `ErrMFADeviceNameConflict` 並回 409
  - 完成：強制 MFA setup 前端改用時間戳裝置名稱，409 時自動換名重試一次
  - 完成：已跑 `go test ./internal/auth`、`go test ./internal/server`、`npm run typecheck`、`npm run build`
  - _Requirements: 4.1-4.7, 5.2-5.5; Design: 強制 MFA setup page, MFA setup_

- [x] 6.9 修正強制 MFA setup 初始化無限載入
  - 初始化 `/auth/mfa/setup` request 不得無限 pending
  - proxy、網路或後端未回應時需顯示可重試錯誤
  - 錯誤訊息需走 i18n，不得停留在永久 loading spinner
  - 完成：強制 MFA setup 初始化 request 加入 15 秒 timeout
  - 完成：逾時會顯示 i18n 錯誤並保留重試操作，不再永久 loading
  - 完成：QR Code 改為 setup response 後背景產生，失敗時顯示手動密鑰，不阻塞 MFA setup 主流程
  - 完成：初始化 setup 改用本地狀態控制，response 200 後直接關閉 loading 並進入掃描畫面
  - 完成：已跑 `npm run typecheck`、`npm run build`、`git diff --check`
  - _Requirements: 5.2-5.6; Design: 強制 MFA setup page_
