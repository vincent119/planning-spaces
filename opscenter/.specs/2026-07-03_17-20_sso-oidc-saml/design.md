# SSO / OIDC / SAML 設計

## 文件定位

本文件描述 SSO / OIDC / SAML 的後端、前端、資料模型與安全設計。需求來源為本 spec 的 `requirements.md`，規劃順序來源為 `planning.md`。

本文件不修改 `../2026-06-01_10-22_oncall-ticket-system`。

## 設計目標

- OIDC 與 SAML 都轉換成同一套內部登入結果。
- 外部身分識別與本地使用者分離保存。
- 外部群組只透過 mapping 轉換，不直接授權。
- Provider secret 不進入公開 API。
- SSO 成功後沿用既有 token 與 MFA policy。
- 後續支援 Provider 由 DB 與管理 UI 維護，但 secret 加密 key 仍由部署設定提供。

## 整體架構

```text
登入頁
  -> GET /api/v1/auth/config
  -> 顯示可用 SSO Provider
  -> GET /api/v1/auth/sso/{provider_key}/login
       -> OIDC authorization endpoint
       -> 或 SAML IdP SSO URL

callback
  -> 驗證 OIDC ID Token 或 SAML Assertion
  -> 解析 external identity
  -> 連結或建立本地 user
  -> 套用 group mapping
  -> 套用本地 MFA policy
  -> 簽發 tokens 或回 MFA flow
```

## Provider 設定模型

第一版建議 Provider 設定由 `config.yaml` / ENV 提供，避免敏感設定先進 DB。

```yaml
sso:
  providers:
    - key: "company_oidc"
      protocol: "oidc"
      display_name: "公司 OIDC"
      enabled: true
      issuer_url: "https://idp.example.com"
      client_id: ""
      client_secret: ""
      redirect_url: "https://opscenter.example.com/api/v1/auth/oidc/callback"
      scopes: ["openid", "profile", "email"]
      groups_claim: "groups"
      auto_provision: true
      auto_link_email: false
    - key: "company_saml"
      protocol: "saml"
      display_name: "公司 SAML"
      enabled: false
      issuer_url: "https://idp.example.com"
      entity_id: "https://opscenter.example.com/saml/metadata"
      acs_url: "https://opscenter.example.com/api/v1/auth/saml/company_saml/acs"
      idp_metadata_url: "https://idp.example.com/metadata"
      idp_sso_url: "https://idp.example.com/sso"
      sp_certificate: "" # ENV: SSO_COMPANY_SAML_SP_CERTIFICATE
      private_key: "" # ENV: SSO_COMPANY_SAML_PRIVATE_KEY
      name_id_format: "persistent"
      groups_attribute: "groups"
```

敏感欄位：

- OIDC `client_secret`
- SAML SP private key
- SAML SP certificate 為公開憑證，但仍只在後端設定使用，不透過公開 Auth Config 回傳
- SAML IdP certificate 若由檔案提供，需限制檔案權限

若後續做 DB Provider 管理，需新增加密欄位，不可明文保存 secret。

## Provider DB 管理模型

後續管理 UI 不應把整段 YAML 放進 `system_settings`。建議拆成：

- `system_settings`：保存全域小設定，例如 `sso.state_ttl_seconds`、`sso.provider_config_source`。
- `sso_providers`：保存 Provider 非敏感設定與加密後 secret。
- `sso_provider_test_results`：保存最近測試結果與錯誤摘要。

### `sso_providers`

```sql
CREATE TABLE sso_providers (
  id CHAR(26) PRIMARY KEY DEFAULT generate_ulid(),
  provider_key VARCHAR(64) NOT NULL UNIQUE,
  protocol VARCHAR(16) NOT NULL CHECK (protocol IN ('oidc', 'saml')),
  display_name VARCHAR(128) NOT NULL,
  is_enabled BOOLEAN NOT NULL DEFAULT FALSE,
  sort_order INT NOT NULL DEFAULT 100,
  config JSONB NOT NULL DEFAULT '{}'::jsonb,
  secret_ciphertext TEXT,
  secret_version INT NOT NULL DEFAULT 1,
  last_test_status VARCHAR(32),
  last_tested_at TIMESTAMPTZ,
  last_tested_by CHAR(26) REFERENCES users(id),
  created_by CHAR(26) NOT NULL REFERENCES users(id),
  updated_by CHAR(26) REFERENCES users(id),
  deleted_at TIMESTAMPTZ,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  CHECK (jsonb_typeof(config) = 'object')
);
```

`config` 保存非敏感欄位：

OIDC：

```json
{
  "issuer_url": "https://idp.example.com",
  "client_id": "opscenter",
  "redirect_url": "https://opscenter.example.com/api/v1/auth/oidc/callback",
  "scopes": ["openid", "profile", "email"],
  "groups_claim": "groups",
  "auto_provision": true,
  "auto_link_email": false
}
```

SAML：

```json
{
  "issuer_url": "https://idp.example.com",
  "entity_id": "https://opscenter.example.com/saml/metadata",
  "acs_url": "https://opscenter.example.com/api/v1/auth/saml/company_saml/acs",
  "idp_metadata_url": "https://idp.example.com/metadata",
  "idp_sso_url": "https://idp.example.com/sso",
  "sp_certificate": "PEM 公開憑證",
  "name_id_format": "persistent",
  "groups_attribute": "groups"
}
```

`secret_ciphertext` 加密保存：

- OIDC `client_secret`
- SAML SP private key
- SAML SP certificate
- 若 IdP certificate 視為敏感資料，也放入加密 payload；若只是公開憑證，可放在 `config`

當 `idp_metadata_url` 缺省且改用 `idp_certificate` 時，Provider 必須同時設定 `idp_sso_url`，讓系統可以產生 AuthnRequest redirect URL。若 IdP assertion issuer 與 SSO URL 不同，需同時設定 SAML provider 的 `issuer_url` 作為 IdP entity ID。若使用 `idp_metadata_url`，系統可從 metadata 解析 IdP issuer、SSO URL 與 signing certificate。

加密 key：

- 使用 `SSO_SECRET_ENCRYPTION_KEY` 或同等部署 Secret。
- 不存 DB。
- 啟動時若 DB Provider 有 encrypted secret，但缺少加密 key，Provider resolver 必須回設定不完整。

API 回傳時只提供：

- `has_client_secret`
- `has_private_key`
- `has_idp_certificate`
- `secret_updated_at`

不得回傳 secret 明文或 ciphertext。

### `sso_provider_test_results`

```sql
CREATE TABLE sso_provider_test_results (
  id CHAR(26) PRIMARY KEY DEFAULT generate_ulid(),
  provider_id CHAR(26) NOT NULL REFERENCES sso_providers(id) ON DELETE CASCADE,
  test_type VARCHAR(32) NOT NULL CHECK (test_type IN ('oidc_discovery', 'oidc_login', 'saml_metadata', 'saml_login')),
  status VARCHAR(32) NOT NULL CHECK (status IN ('success', 'failed')),
  summary TEXT NOT NULL,
  detail JSONB NOT NULL DEFAULT '{}'::jsonb,
  tested_by CHAR(26) NOT NULL REFERENCES users(id),
  tested_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  CHECK (jsonb_typeof(detail) = 'object')
);
```

`detail` 可保存 endpoint 狀態、HTTP status、metadata 欄位是否存在，但不得保存 token、authorization code、完整 claim、完整 assertion、secret。

## 設定來源優先序

Provider resolver 建議支援來源切換：

1. `system_settings.sso.provider_config_source = db` 時，以 DB Provider 為主。
2. DB 無任何啟用 Provider 時，可依設定 fallback 到 config / ENV bootstrap Provider。
3. `system_settings.sso.provider_config_source = config` 時，只讀 config / ENV。
4. 公開 auth config 只列出 resolver 判定「啟用且設定完整」的 Provider。

不建議混用同一 provider key 的 DB 與 config 設定。若重複，以主要來源為準並寫入啟動警告。

## Provider 管理 API

管理 API 僅 `admin` 可呼叫。

```text
GET    /api/v1/admin/sso/providers
GET    /api/v1/admin/sso/providers/status
POST   /api/v1/admin/sso/providers
GET    /api/v1/admin/sso/providers/:id
PUT    /api/v1/admin/sso/providers/:id
PATCH  /api/v1/admin/sso/providers/:id/enabled
POST   /api/v1/admin/sso/providers/:id/secrets
DELETE /api/v1/admin/sso/providers/:id
```

設計規則：

- `POST` / `PUT` 接受非敏感設定與可選 secret。
- `GET` 不回傳 secret，只回傳 `has_*` 旗標。
- `PATCH enabled=true` 前需檢查必要欄位與最近測試狀態。
- `DELETE` 優先軟刪除；若已有 external identity 或 mapping，需阻擋硬刪除。
- secret 更新獨立 endpoint，避免一般設定更新時意外清空 secret。
- 每次異動寫入 security audit。

### Provider 狀態 API

`GET /api/v1/admin/sso/providers/status` 用於維運總覽，回傳每個 provider 的可用狀態與最近測試摘要。

Response 欄位：

| 欄位 | 說明 |
| --- | --- |
| `id` | DB provider id；config / ENV bootstrap provider 可為空 |
| `key` | provider key |
| `protocol` | `oidc` 或 `saml` |
| `display_name` | 顯示名稱 |
| `enabled` | 是否啟用 |
| `config_source` | `db` 或 `config` |
| `config_complete` | 必要欄位與 secret 是否完整 |
| `login_ready` | 是否會出現在公開 auth config |
| `last_test_status` | `never_tested`、`success`、`failed`、`running` |
| `last_tested_at` | 最近測試時間 |
| `last_tested_by` | 最近測試操作者顯示名稱 |
| `last_test_summary` | 去敏後錯誤或成功摘要 |

安全規則：

- 不回傳 issuer secret、client secret、private key、token、authorization code、完整 claim 或完整 assertion。
- `last_test_summary` 需去敏與截斷，避免把 IdP 回傳原文直接送到 UI。
- config / ENV bootstrap provider 可顯示唯讀狀態，但不可由 DB 管理 API 修改。

錯誤碼建議：

- `sso_provider_not_found`
- `sso_provider_key_exists`
- `sso_provider_invalid_config`
- `sso_provider_secret_required`
- `sso_provider_test_required`
- `sso_provider_in_use`

## Mapping 管理 API

管理 API 僅 `admin` 可呼叫，所有異動都需寫入 security audit。

```text
GET    /api/v1/admin/sso/mappings
POST   /api/v1/admin/sso/mappings
GET    /api/v1/admin/sso/mappings/:id
PUT    /api/v1/admin/sso/mappings/:id
PATCH  /api/v1/admin/sso/mappings/:id/enabled
DELETE /api/v1/admin/sso/mappings/:id
```

查詢參數：

| 參數 | 說明 |
| --- | --- |
| `provider_key` | 依 provider key 過濾 |
| `external_group` | 依外部群組關鍵字搜尋 |
| `target_type` | `global_role` 或 `permission_group` |
| `is_active` | 是否啟用 |
| `limit` / `offset` | 分頁 |

Request 欄位：

| 欄位 | 說明 |
| --- | --- |
| `provider_key` | provider key，允許 `default` 作為共用 mapping |
| `external_group` | 外部群組名稱，需 trim 後非空 |
| `target_type` | `global_role` 或 `permission_group` |
| `target_code` | global role 或 permission group code |
| `priority` | 同類 mapping 優先序 |
| `is_active` | 是否啟用 |

驗證規則：

- `target_type = global_role` 時，`target_code` 僅允許 `admin`、`op_admin`、`member`。
- `target_type = permission_group` 時，`target_code` 必須對應已存在且啟用的權限群組。
- 同一 `provider_key + external_group + target_type + target_code` 不得有重複有效資料。
- 停用 mapping 不會立刻移除既有 SSO 來源群組成員；下一次使用者 SSO 登入同步時依新 mapping 收斂。
- `DELETE` 第一版採軟刪除或停用，不做硬刪除。

錯誤碼建議：

- `sso_mapping_not_found`
- `sso_mapping_key_exists`
- `sso_mapping_invalid_target`
- `sso_mapping_provider_not_found`
- `sso_mapping_in_use`

## 公開 Auth Config API

既有 `GET /api/v1/auth/config` 擴充為可列出 Provider。

Response 範例：

```json
{
  "oidc": {
    "enabled": true,
    "login_url": "/api/v1/auth/oidc/login"
  },
  "sso": {
    "providers": [
      {
        "key": "company_oidc",
        "protocol": "oidc",
        "display_name": "公司 OIDC",
        "login_url": "/api/v1/auth/sso/company_oidc/login"
      }
    ]
  }
}
```

相容策略：

- 保留既有 `oidc.enabled` 與 `oidc.login_url`，避免前端立即破壞。
- 新前端優先讀 `sso.providers`。
- Provider 未完成 handler 前不得出現在 `sso.providers`。

禁止回傳：

- provider_url / issuer_url 若不需要顯示，預設不回。
- client secret。
- SAML private key。
- SAML assertion 或 token。

## API 設計

### 公開登入入口

```text
GET /api/v1/auth/sso/:provider_key/login
```

行為：

- 驗證 provider 存在且啟用。
- 依 protocol 產生 OIDC authorization request 或 SAML AuthnRequest。
- 建立短效 state 記錄。
- 302 redirect 到 IdP。

錯誤：

- `404`：provider 不存在。
- `409`：provider 停用。
- `500`：provider 設定不完整。

### OIDC 相容入口

```text
GET /api/v1/auth/oidc/login
GET /api/v1/auth/oidc/callback
```

用途：

- 保留舊前端與舊設計契約。
- 內部可導向 default OIDC provider 的通用 SSO flow。

### SAML endpoint

```text
GET  /api/v1/auth/saml/:provider_key/metadata
GET  /api/v1/auth/saml/:provider_key/login
POST /api/v1/auth/saml/:provider_key/acs
```

第一版只要求 SP initiated login。

## Provider 測試 API

Provider 測試分為連線測試與登入流程測試。

```text
POST /api/v1/admin/sso/providers/:id/test-connection
POST /api/v1/admin/sso/providers/:id/test-login
GET  /api/v1/admin/sso/providers/:id/test-results
```

測試結果欄位：

| 欄位 | 說明 |
| --- | --- |
| `id` | 測試結果 id |
| `provider_id` | provider id |
| `test_type` | `connection` 或 `login` |
| `status` | `success`、`failed`、`running` |
| `summary` | 去敏後摘要 |
| `checked_items` | 已檢查項目，例如 discovery、jwks、metadata、signature |
| `started_at` | 測試開始時間 |
| `finished_at` | 測試完成時間 |
| `tested_by` | 操作者 |

維運規則：

- 同一 provider 同一時間只允許一個測試登入流程，避免 RelayState / callback 結果互相覆蓋。
- 測試登入使用獨立 `purpose = provider_test` state，不可與正式登入共用。
- 測試 callback 結果只保存去敏摘要，不保存 token、assertion、authorization code 或完整 claim。
- 連線測試可同步更新 Provider 狀態 API 的 `last_test_status` 與 `last_test_summary`。

### OIDC 連線測試

流程：

1. 讀取 issuer 的 discovery metadata。
2. 確認 authorization endpoint、token endpoint、JWKS URI 存在。
3. 讀取 JWKS。
4. 檢查 JWKS 至少有可用 key。
5. 保存測試結果。

此測試不需要使用者登入，也不得要求 authorization code。

### OIDC 登入流程測試

流程：

1. 管理員在 Provider 管理頁點測試登入。
2. 系統建立 `purpose = provider_test` 的短效 state。
3. 導向 IdP。
4. callback 回系統後完成 code exchange 與 ID Token 驗證。
5. 只顯示 subject、email 是否存在、groups claim 是否存在等摘要。
6. 不建立正式 session。
7. 不建立 user。
8. 不同步 group mapping。
9. 不寫入一般登入成功紀錄。

測試成功只代表 Provider 設定可登入，不代表該使用者已取得系統權限。

### SAML 連線測試

流程：

1. 解析 IdP metadata URL 或 certificate。
2. 確認 IdP issuer、SSO URL 與 signing certificate。
3. 檢查 ACS URL 與 entity ID 設定完整。
4. 保存測試結果。

### SAML 登入流程測試

流程：

1. 管理員在 Provider 管理頁點測試登入。
2. 系統建立 `purpose = provider_test` 的 RelayState。
3. 導向 IdP。
4. ACS 收到 SAMLResponse 後驗證簽章、issuer、audience、recipient 與時間窗。
5. 只顯示 NameID、email attribute、groups attribute 是否存在等摘要。
6. 不建立正式 session。
7. 不建立 user。
8. 不同步 group mapping。

測試 callback 只能回固定管理端結果頁，例如 `/admin/sso/providers/:id/test-result`，不得接受外部 redirect。

## Provider 管理 UI

入口放在系統管理：

```text
系統管理 / SSO Provider
```

主要畫面：

- Provider 列表：名稱、protocol、狀態、最近測試狀態、更新時間、操作。
- 新增 / 編輯 Provider 表單。
- Secret 更新區塊。
- 測試結果抽屜或詳情頁。
- Mapping 頁籤。
- 維運狀態總覽。

表單行為：

- OIDC 與 SAML 用 tab 或 protocol selector 切換欄位。
- secret 欄位只顯示「已設定 / 未設定」，修改時以重新輸入覆蓋。
- 啟用按鈕在必要欄位缺失或測試未通過時 disabled。
- 可保存為停用狀態，待測試通過後再啟用。
- 測試中需顯示 loading，避免重複送出。

Mapping 頁籤：

- 依 provider 過濾 mapping。
- 顯示 external group、target type、target code、priority、啟用狀態。
- target type 使用固定選項，不允許任意輸入。
- permission group target code 使用已啟用權限群組下拉選單。
- global role target code 使用固定角色下拉選單。

維運狀態總覽：

- 顯示 provider 是否會出現在登入頁。
- 顯示設定來源為 DB 或 config / ENV。
- 顯示設定完整性與最近測試摘要。
- 顯示最近 callback 失敗摘要，但不得顯示敏感原文。
- config / ENV bootstrap provider 顯示唯讀標記。

權限：

- 只有 `admin` 可新增、修改、刪除、測試與啟用 Provider。
- `op_admin` 可在後續視需求增加只讀權限，但第一版不開放管理。

## 資料模型

### `sso_external_identities`

保存外部身分與本地使用者關係。

```sql
CREATE TABLE sso_external_identities (
  id CHAR(26) PRIMARY KEY,
  user_id CHAR(26) NOT NULL REFERENCES users(id),
  provider_key VARCHAR(64) NOT NULL,
  protocol VARCHAR(16) NOT NULL CHECK (protocol IN ('oidc', 'saml')),
  external_subject VARCHAR(255) NOT NULL,
  email_snapshot VARCHAR(255),
  display_name_snapshot VARCHAR(255),
  last_login_at TIMESTAMPTZ,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  UNIQUE (provider_key, protocol, external_subject)
);
```

規則：

- `external_subject` 來自 OIDC `sub` 或 SAML NameID / subject attribute。
- email 只做輔助顯示與首次連結判斷，不作唯一身分來源。

### `sso_group_mappings`

保存外部群組到內部授權結果的 mapping。

```sql
CREATE TABLE sso_group_mappings (
  id CHAR(26) PRIMARY KEY,
  provider_key VARCHAR(64) NOT NULL,
  external_group VARCHAR(255) NOT NULL,
  target_type VARCHAR(32) NOT NULL CHECK (target_type IN ('global_role', 'permission_group')),
  target_code VARCHAR(64) NOT NULL,
  priority INT NOT NULL DEFAULT 100,
  is_active BOOLEAN NOT NULL DEFAULT TRUE,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  UNIQUE (provider_key, external_group, target_type, target_code)
);
```

seed 建議：

| external group | target type | target code |
| --- | --- | --- |
| `ops_admin` | `global_role` | `admin` |
| `ops_op_admin` | `global_role` | `op_admin` |
| `ops_member` | `global_role` | `member` |
| `ops_admin` | `permission_group` | `ops_admin` |
| `ops_op_admin` | `permission_group` | `ops_op_admin` |
| `ops_member` | `permission_group` | `ops_member` |
| `ops_op_member` | `permission_group` | `ops_op_member` |

## 群組同步策略

輸入：

- OIDC groups claim，預設 `groups`。
- SAML groups attribute，預設 `groups`。

處理：

1. 去重外部群組。
2. 查詢啟用 mapping。
3. `global_role` 命中多筆時取最高權限序。
4. `permission_group` 僅同步到已存在且啟用的 group。
5. 未命中 mapping 時，新使用者預設 `member`。
6. 手動群組成員關係不得被 SSO 同步刪除。

建議為 group membership 加上來源欄位：

```sql
ALTER TABLE group_members
  ADD COLUMN source VARCHAR(32) NOT NULL DEFAULT 'manual',
  ADD COLUMN source_provider VARCHAR(64);
```

同步時只新增、更新或移除 `source = 'sso'` 且同 provider 的資料。

## OIDC 流程設計

### Login

```text
GET /api/v1/auth/sso/company_oidc/login
  -> 讀 provider
  -> 取得 discovery metadata
  -> 建立 state、nonce
  -> 302 redirect
```

state 保存：

- Redis key：`auth:sso:state:{state}`
- TTL：5 到 10 分鐘
- value：provider key、nonce、redirect target、created at

### Callback

```text
GET /api/v1/auth/oidc/callback?code=...&state=...
  -> 驗證 state
  -> token exchange
  -> 驗證 ID Token
  -> 解析 subject、email、name、groups
  -> resolve local user
  -> sync mapping
  -> apply MFA policy
  -> redirect frontend
```

ID Token 驗證：

- issuer 等於 provider issuer。
- audience 包含 client id。
- exp 未過期。
- iat 合理。
- nonce 等於 state 記錄。
- 簽章符合 JWKS。

## SAML 流程設計

### Metadata

```text
GET /api/v1/auth/saml/company_saml/metadata
```

需輸出：

- EntityID。
- ACS URL。
- NameID format。
- SP certificate。

若未設定 SP certificate，metadata 仍可輸出 EntityID 與 ACS URL，但不會包含 signing / encryption key descriptor；生產環境若 IdP 要求簽署 AuthnRequest，必須提供 `sp_certificate` 與 `private_key`。

### Login

```text
GET /api/v1/auth/saml/company_saml/login
  -> 建立 AuthnRequest
  -> 建立 RelayState
  -> 302 redirect IdP SSO URL
```

### ACS

```text
POST /api/v1/auth/saml/company_saml/acs
  -> 驗證 RelayState
  -> 解析 SAMLResponse
  -> 驗證簽章與 assertion 條件
  -> 解析 subject、email、display name、groups
  -> resolve local user
  -> sync mapping
  -> apply MFA policy
  -> redirect frontend
```

驗證項：

- Response 或 Assertion 簽章。
- issuer。
- audience。
- recipient。
- destination。
- NotBefore / NotOnOrAfter。
- InResponseTo 對應 RelayState。

## 使用者解析策略

順序：

1. 以 provider key + protocol + external subject 查 `sso_external_identities`。
2. 若存在，使用該本地 user。
3. 若不存在且 `auto_link_email = true`，可用已驗證 email 查既有 user 並建立綁定。
4. 若不存在且 `auto_provision = true`，建立新 user。
5. 其他情況拒絕登入，回可讀錯誤。

新 user 預設：

- `username` 由 email local part 或 provider subject 產生，需處理重複。
- `email` 來自 verified email。
- `full_name` 來自 name claim，缺值使用 username。
- `global_role = member`，除非 group mapping 命中更高角色。
- `is_active = true`。

## MFA 接線

SSO callback 完成外部驗證與本地 user resolution 後，呼叫既有 auth service 的登入完成流程。

若需 MFA：

- 產生 `temp_token`。
- redirect 前端 `/login?sso=mfa_required` 或回傳中介頁。
- 前端進入既有 MFA 驗證畫面。

若不需 MFA：

- 簽發 access token / refresh token。
- 建立登入 session。
- redirect 前端指定成功頁。

第一版不信任 IdP MFA claim。若未來要信任，需新增 `sso.trust_idp_mfa` 並記錄稽核。

## 前端設計

登入頁：

- `GET /api/v1/auth/config` 取得 `sso.providers`。
- 對每個 provider 顯示按鈕。
- 點擊後 `window.location.href = login_url`。

Callback 結果：

- 後端 callback 不把 access token、refresh token 或 temp token 放在 URL。
- 後端把 SSO 登入結果存入短效一次性 `result_token`，再 redirect 前端 `/auth/sso/callback?result_token=...`。
- 前端以 `result_token` 呼叫 `GET /api/v1/auth/sso/result/:result_token` 交換結果。
- 成功：前端寫入正式 token 與 current user，進入系統首頁或原始 redirect target。
- 需要 MFA：前端保存 temp token 後導入既有 MFA flow，完成後才寫入正式登入狀態。
- 失敗：回登入頁顯示 i18n 錯誤，並清除暫存登入狀態。

錯誤 key 建議：

| key | 說明 |
| --- | --- |
| `auth.sso_failed` | SSO 登入失敗 |
| `auth.sso_provider_disabled` | Provider 未啟用 |
| `auth.sso_account_not_allowed` | 帳號不允許自動建立或連結 |
| `auth.sso_mapping_failed` | 群組同步失敗 |

## 稽核與日誌

登入日誌需包含：

- provider key。
- protocol。
- result。
- local user id，若已解析。
- error code。

安全稽核需包含：

- 外部身分首次綁定。
- 自動建立 user。
- global role 變更。
- permission group 同步摘要。

不得記錄：

- authorization code。
- ID Token。
- Access Token。
- SAML Assertion。
- client secret。
- private key。

## 測試策略

後端：

- OIDC state / nonce。
- OIDC token exchange mock。
- JWKS signature 驗證。
- user resolution。
- group mapping。
- SAML assertion 驗證。
- MFA 分流。
- 稽核敏感資料遮罩。

前端：

- SSO provider 清單顯示。
- provider disabled 不顯示。
- 點擊導向 login URL。
- callback 失敗訊息。
- SSO 後 MFA 分流。

## 未決問題

- 第一版是否只允許單一 OIDC Provider。
- `auto_link_email` 是否允許在生產環境啟用。
- SAML 是否需要 IdP initiated login。
- SSO 同步是否要在每次登入覆蓋 `full_name` 與 `email`。
- 是否需要管理 UI 維護 `sso_group_mappings`，或第一版只由 migration seed。
