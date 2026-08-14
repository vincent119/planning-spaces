# MFA Global Setting 接線設計

## 文件定位

本文件描述 `security.mfa.force_enable` 與 `security.mfa.allowed_types` 如何接入既有 Auth / MFA 流程。需求來源為本 spec 的 `requirements.md`，並對齊原始 spec `../2026-06-01_10-22_oncall-ticket-system` 的 MFA 設計。

## 設計目標

- MFA policy 由 `system_settings` 管理，不在程式碼硬寫。
- 帳密驗證成功不等於完整登入成功；完成必要 MFA 後才簽發正式 tokens。
- `temp_token` 只服務 MFA 驗證與強制 setup，不可存取一般 API。
- 現有個人 MFA 裝置管理、未驗證裝置清理與使用者 MFA 狀態顯示維持相容。
- 前端沿用已規劃的 `mfa_required`、`mfa_setup_required` 與 `/auth/mfa/setup` 流程。

## 舊設計對齊

原始設計中已存在下列契約：

- `system_settings`
  - `security.mfa.force_enable`
  - `security.mfa.allowed_types`
  - `security.mfa.pending_device_retention_minutes`
- 登入序列
  - 帳密成功後若 MFA 已啟用，回 `mfa_required: true` 與 `temp_token`
  - `POST /auth/mfa/verify` 驗證 TOTP 後再核發正式 token
- 前端登入狀態機
  - `mfa_required`：登入卡片內切 TOTP 輸入
  - `mfa_setup_required`：導向 `/auth/mfa/setup`
  - `temp_token`：只在 MFA 流程暫存，完成後清除

目前缺口是後端 `Login()` 尚未使用上述設定與狀態機，導致 MFA policy 沒有生效。

## 整體流程

```text
POST /auth/login
  -> 驗證帳號、狀態、密碼
  -> 讀取 MFA policy
  -> 查詢使用者是否有已驗證且啟用中的 MFA 裝置
  -> 依狀態分流
       有已驗證 MFA 裝置
         -> 回 mfa_required + temp_token，不簽正式 tokens
       無已驗證 MFA 裝置且 force_enable = true
         -> 回 mfa_setup_required + temp_token，不簽正式 tokens
       無已驗證 MFA 裝置且 force_enable = false
         -> 直接簽正式 tokens
```

完成 MFA 後：

```text
POST /auth/mfa/verify
  -> 驗證 temp_token 用途
  -> 驗證 TOTP
  -> 必要時標記 MFA device verified
  -> 簽發 access token + refresh token
  -> 回正式登入結果
```

## MFA Policy 解析

新增或擴充 auth 服務可注入的 policy provider：

```go
type MFAPolicyProvider interface {
	ForceEnabled(ctx context.Context) bool
	AllowedTypes(ctx context.Context) []string
}
```

解析規則：

| key | 預設值 | 解析規則 |
| --- | --- | --- |
| `security.mfa.force_enable` | `false` | 只接受 boolean |
| `security.mfa.allowed_types` | `["totp"]` | 只接受 JSON array |

第一版支援類型：

| type | 狀態 | 說明 |
| --- | --- | --- |
| `totp` | 支援 | Google Authenticator / Authy 相容 TOTP |
| `email` | 不支援 | 不在本次範圍 |
| `sms` | 不支援 | 不在本次範圍 |
| `backup_code` | 不支援 | 不在本次範圍 |

Global Setting 管理端需在更新 `security.mfa.allowed_types` 時驗證內容。後端 auth service 仍需防禦性檢查，避免 DB 既有髒資料導致 MFA setup 流程不一致。

## 登入狀態機

| 條件 | Response | 正式 tokens |
| --- | --- | --- |
| 帳密失敗或停用 | 401 | 無 |
| 帳密成功，有已驗證 MFA 裝置 | `mfa_required = true`、`temp_token` | 無 |
| 帳密成功，無已驗證 MFA 裝置，`force_enable = true` | `mfa_setup_required = true`、`temp_token` | 無 |
| 帳密成功，無已驗證 MFA 裝置，`force_enable = false` | `password_verified = true`、`tokens` | 有 |

`password_verified` 的語意限定為帳密階段成功。前端必須以 `tokens` 是否存在作為完整登入狀態依據。

## API 契約

### Login response

`LoginResponse` 需擴充為 tokens 可空，並新增 `mfa_setup_required` 與 `temp_token`。

```json
{
  "user": {
    "id": "01...",
    "username": "terry",
    "full_name": "Terry",
    "global_role": "admin"
  },
  "password_verified": true,
  "mfa_required": true,
  "mfa_setup_required": false,
  "temp_token": "short-lived-token",
  "tokens": null
}
```

直接登入成功時：

```json
{
  "user": {
    "id": "01...",
    "username": "terry",
    "full_name": "Terry",
    "global_role": "member"
  },
  "password_verified": true,
  "mfa_required": false,
  "mfa_setup_required": false,
  "tokens": {
    "access_token": "...",
    "refresh_token": "...",
    "token_type": "Bearer"
  }
}
```

### MFA setup

`POST /api/v1/auth/mfa/setup`

支援兩種授權來源：

| 授權來源 | 用途 |
| --- | --- |
| 正式 access token | 已登入使用者在個人資料頁新增 MFA 裝置 |
| `temp_token` 且用途為 `mfa_setup` | 強制 MFA setup |

強制 MFA setup 時，後端由 `temp_token` 取得 user id 與 username，不接受 client 指定使用者。

### MFA verify

`POST /api/v1/auth/mfa/verify`

支援三種情境：

| 情境 | 授權來源 | 成功結果 |
| --- | --- | --- |
| 個人資料頁驗證新裝置 | 正式 access token | 標記 device verified |
| 登入時驗證既有 MFA | `temp_token`，用途 `mfa_verify` | 簽發正式 tokens |
| 強制 setup 驗證新裝置 | `temp_token`，用途 `mfa_setup` | 標記 device verified 並簽發正式 tokens |

登入時驗證既有 MFA 可支援 `device_id` 可選：

- 有帶 `device_id`：只驗證該裝置。
- 未帶 `device_id`：逐一比對使用者所有已驗證且啟用中的 TOTP 裝置。

## Temp Token 設計

`temp_token` 不使用一般 access token 權限。可採兩種實作：

| 方案 | 優點 | 限制 |
| --- | --- | --- |
| 簽章 JWT，加入 `purpose`、`sub`、`exp`、`jti` | 不需新增資料表，容易接到現有 JWT 工具 | 若要強制一次性需搭配 Redis |
| Redis opaque token | 可一次性使用與撤銷 | 需要新增存取介面 |

第一版建議採簽章 JWT 搭配 Redis jti 記錄：

- JWT claim：
  - `sub`：user id
  - `username`：顯示與 setup label 用途
  - `purpose`：`mfa_verify` 或 `mfa_setup`
  - `exp`：預設 10 分鐘
  - `jti`：一次性識別
- Redis key：
  - `auth:mfa_temp:{jti}`
  - TTL 與 token exp 一致
- 成功驗證後刪除 Redis key，避免重放。

一般 auth middleware 必須拒絕 `purpose` 非 access 的 token。

## 後端服務調整

### Service dependencies

`auth.Service` 增加：

- `mfaPolicy MFAPolicyProvider`
- `mfaTempTokens MFATempTokenManager`

`LoginResult` 增加：

- `MFASetupRequired bool`
- `TempToken string`
- `Tokens *TokenPair`

### Login use case

```text
Login()
  -> FindByLogin
  -> bcrypt password check
  -> list verified active MFA devices
  -> resolve MFA policy
  -> if has verified device
       issue temp token purpose=mfa_verify
       return pending MFA result
  -> if force enabled
       ensure allowed_types contains totp
       issue temp token purpose=mfa_setup
       return forced setup result
  -> issue formal tokens
```

### Verify use case

新增或調整 MFA verify use case，讓 delivery 可辨識授權來源：

- 正式 access token：沿用既有 `UserID`。
- `temp_token`：先驗證 token，再取得 `UserID` 與 `purpose`。

成功簽發正式 tokens 時，需與目前 `Login()` 相同：

- 呼叫 `TokenManager.Issue(user)`。
- refresh token 寫入 refresh store。
- 回 `TokenResponse`。

## 前端調整

### Login hook

登入 mutation 收到 response 後：

```text
if tokens exists:
  store tokens
  redirect
else if mfa_required:
  store temp_token in temp token store
  show MFA verify form
else if mfa_setup_required:
  store temp_token in temp token store
  navigate /auth/mfa/setup
else:
  show invalid login response
```

### Forced MFA setup page

進入頁面時：

- 從 temp token store 取 `temp_token`。
- 缺少 `temp_token` 時導回 `/login` 並顯示繁中錯誤。
- 呼叫 `POST /auth/mfa/setup`，Header 使用 `Authorization: Bearer {temp_token}`。
- 驗證成功回正式 tokens 後，清除 temp token，寫入正式 tokens，導向原目標頁。

### MFA verify form

登入頁 MFA 驗證：

- Header 使用 `Authorization: Bearer {temp_token}`。
- Payload 至少包含 `code`。
- 若後續前端提供裝置選擇，再加上 `device_id`。

## 設定管理 UI

Global Settings 頁維持既有通用表格，但更新時需依 key 做型別與內容驗證：

- `security.mfa.force_enable`：boolean。
- `security.mfa.allowed_types`：JSON array，第一版只能包含 `totp`。

錯誤訊息需使用 i18n key，例如：

- `settings.validation.mfa_allowed_types_invalid`
- `settings.validation.mfa_type_unsupported`

## 稽核與日誌

新增事件類型建議：

| event | 說明 |
| --- | --- |
| `login_password_verified_mfa_required` | 帳密成功，需 MFA 驗證 |
| `login_password_verified_mfa_setup_required` | 帳密成功，需強制 MFA setup |
| `mfa_login_verified` | 登入 MFA 驗證成功 |
| `mfa_login_failed` | 登入 MFA 驗證失敗 |
| `mfa_forced_setup_completed` | 強制 MFA setup 完成 |
| `mfa_forced_setup_failed` | 強制 MFA setup 失敗 |

日誌欄位只保留：

- user id
- username
- request id / trace id
- MFA branch
- failure reason code

不得記錄：

- password
- TOTP secret
- TOTP code
- access token
- refresh token
- `temp_token`

## 相容性

- 既有未啟用 MFA 的使用者，在 `force_enable = false` 時登入行為不變。
- 已有 MFA 裝置的使用者，不論 `force_enable` 是否為 true，都需完成 MFA 驗證後才能取得正式 tokens。
- `security.mfa.pending_device_retention_minutes` 的排程清理不變。
- 使用者列表 `has_mfa` 判斷仍以 `is_active = TRUE AND is_verified = TRUE` 為準。

## 驗證計畫

後端：

- Auth service 單元測試覆蓋登入四個主要分支。
- MFA setup / verify handler 測試覆蓋 access token 與 `temp_token` 兩種授權來源。
- Policy provider 測試覆蓋 setting 缺值、停用、解析失敗與不支援類型。
- Temp token 測試覆蓋過期、用途不符、重放與成功簽發正式 tokens。

前端：

- Login hook 測試覆蓋 `mfa_required`、`mfa_setup_required`、直接 tokens 三種回應。
- Forced MFA setup page 測試覆蓋缺少 temp token、setup 初始化失敗、verify 成功。
- Token store 測試覆蓋 temp token 與正式 tokens 分離。

手動驗收：

- `force_enable = false`、無 MFA 裝置：可直接登入。
- `force_enable = true`、無 MFA 裝置：導向強制 setup，完成後才進入系統。
- 已有 MFA 裝置：每次登入需輸入 TOTP。
- 使用過或過期的 `temp_token` 無法再次取得正式 tokens。
