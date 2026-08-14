# Ticket 通知機制設計

## 文件定位

本文件描述 Ticket 站內通知與 Webhook 通知的後端、前端與資料模型設計。需求來源為原始 spec `../2026-06-01_10-22_oncall-ticket-system` 的需求 10，並依目前系統狀態收斂為第一版可實作範圍。

## 設計目標

- Ticket 操作完成後可產生可追蹤的通知事件。
- 站內通知可以送達登入帳號，並顯示在 Header 通知鈴鐺與通知面板。
- Webhook 以專案層級設定，非同步投遞，具備簽章、重試與投遞紀錄。
- 通知投遞失敗不得影響 Ticket 主流程。
- 所有外部 URL 與 secret 處理必須符合最小權限與敏感資料保護原則。

## 範圍邊界

本次實作包含：

- Ticket event outbox
- 站內通知資料模型與 API
- Webhook 設定 API
- Webhook delivery worker
- Webhook 簽章與重試
- 前端通知面板
- 專案層級 Webhook 管理頁

本次實作不包含：

- Email 通知
- Slack / Telegram 專用 adapter
- SLA checker 主動升級
- 行動推播
- 外部系統 inbound callback

## 事件流程

```text
Ticket command
  -> 寫入 Ticket / activity
  -> 同一 transaction 寫入 notification_events
  -> Ticket API 回應成功
  -> notification dispatcher 解析收件人
  -> 寫入 user_notifications
  -> 建立 webhook_deliveries
  -> webhook worker 非同步投遞
```

設計規則：

- `notification_events` 是通知系統的 outbox。
- Ticket transaction 成功才會留下 event。
- dispatcher 與 worker 可重跑，需依 idempotent key 避免重複。
- Webhook 只保證至少一次投遞，外部系統需依 event id 做去重。

## 事件類型

第一版支援下列 event type：

| event type | 來源操作 | 說明 |
| --- | --- | --- |
| `ticket.created` | 建立 Ticket | 新 Ticket 建立 |
| `ticket.status_changed` | 狀態流轉 | Ticket 狀態變更 |
| `ticket.assigned` | 指派變更 | 指派人新增、變更或清空 |
| `ticket.collaborator_added` | 協作者新增 | 新協作者被加入 |
| `ticket.comment_added` | 留言新增 | Ticket 留言新增 |
| `ticket.mentioned` | mention 解析 | 留言中 mention 有效專案成員 |

`ticket.mentioned` 可與 `ticket.comment_added` 來自同一留言，但收件人與標題不同，需使用不同 idempotent key。

## 收件人解析

### 帳號模型

目前資料模型中：

- `tickets.created_by` 指向登入帳號 `users.id`。
- `tickets.assignee_id` 指向運維人員 `ops_user.id`。
- `ticket_collaborators.user_id` 指向運維人員 `ops_user.id`。

站內通知只能送給可登入系統的 `users`。因此需新增 `ops_user.linked_user_id`：

```sql
ALTER TABLE ops_user
  ADD COLUMN linked_user_id CHAR(26) REFERENCES users(id);
```

遷移規則：

- 先以 `LOWER(ops_user.user_name) = LOWER(users.username)` 回填 `linked_user_id`。
- 無法回填者仍保留 `ops_user`，但站內通知解析時標記為 skipped。
- 後續專案成員維護頁可補上或調整連結帳號。

### 收件人規則

| event type | 收件人 |
| --- | --- |
| `ticket.created` | 被指派人、協作者、專案管理者 |
| `ticket.created` 且 P1 Issue | 被指派人、協作者、專案管理者，後續納入值班組長 |
| `ticket.status_changed` | 建立者、被指派人、協作者、專案管理者 |
| `ticket.assigned` | 新被指派人、建立者、協作者 |
| `ticket.collaborator_added` | 新增的協作者 |
| `ticket.comment_added` | 建立者、被指派人、協作者 |
| `ticket.mentioned` | 被 mention 的有效專案成員 |

共通規則：

- 排除操作者本人，除非後續新增個人偏好允許自我通知。
- 排除停用登入帳號。
- 排除停用 `ops_user`。
- 排除非本專案有效成員。
- 同一 event 解析出的收件人需去重。
- 無法解析為 `users.id` 的 `ops_user` 不建立 `user_notifications`，但需記錄 skipped reason。

## 資料模型

### `notification_events`

通知事件 outbox。

```sql
CREATE TABLE notification_events (
  id CHAR(26) PRIMARY KEY DEFAULT generate_ulid(),
  event_type VARCHAR(64) NOT NULL,
  project_id CHAR(26) NOT NULL REFERENCES projects(id),
  ticket_id CHAR(26) NOT NULL,
  ticket_created_at TIMESTAMPTZ NOT NULL,
  activity_id CHAR(26),
  actor_user_id CHAR(26) NOT NULL REFERENCES users(id),
  idempotency_key VARCHAR(160) NOT NULL UNIQUE,
  payload JSONB NOT NULL,
  occurred_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  processed_at TIMESTAMPTZ,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

索引：

```sql
CREATE INDEX idx_notification_events_pending
  ON notification_events (created_at)
  WHERE processed_at IS NULL;

CREATE INDEX idx_notification_events_project_created
  ON notification_events (project_id, created_at DESC);
```

### `user_notifications`

站內通知。

```sql
CREATE TABLE user_notifications (
  id CHAR(26) PRIMARY KEY DEFAULT generate_ulid(),
  event_id CHAR(26) NOT NULL REFERENCES notification_events(id) ON DELETE CASCADE,
  recipient_user_id CHAR(26) NOT NULL REFERENCES users(id),
  project_id CHAR(26) NOT NULL REFERENCES projects(id),
  ticket_id CHAR(26) NOT NULL,
  title VARCHAR(160) NOT NULL,
  body TEXT NOT NULL,
  link_path VARCHAR(255) NOT NULL,
  event_type VARCHAR(64) NOT NULL,
  read_at TIMESTAMPTZ,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  deleted_at TIMESTAMPTZ,
  UNIQUE (event_id, recipient_user_id)
);
```

索引：

```sql
CREATE INDEX idx_user_notifications_recipient_unread
  ON user_notifications (recipient_user_id, created_at DESC)
  WHERE read_at IS NULL AND deleted_at IS NULL;

CREATE INDEX idx_user_notifications_recipient_created
  ON user_notifications (recipient_user_id, created_at DESC)
  WHERE deleted_at IS NULL;
```

### `notification_recipient_skips`

收件人解析略過紀錄。

```sql
CREATE TABLE notification_recipient_skips (
  id CHAR(26) PRIMARY KEY DEFAULT generate_ulid(),
  event_id CHAR(26) NOT NULL REFERENCES notification_events(id) ON DELETE CASCADE,
  source_type VARCHAR(32) NOT NULL,
  source_id CHAR(26) NOT NULL,
  reason VARCHAR(64) NOT NULL,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

常見 reason：

- `ops_user_not_linked`
- `user_inactive`
- `project_member_inactive`
- `actor_self_notification`
- `duplicate_recipient`

### `project_webhooks`

專案 Webhook 設定。

```sql
CREATE TABLE project_webhooks (
  id CHAR(26) PRIMARY KEY DEFAULT generate_ulid(),
  project_id CHAR(26) NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
  name VARCHAR(128) NOT NULL,
  url TEXT NOT NULL,
  secret_ciphertext TEXT NOT NULL,
  auth_type VARCHAR(16) NOT NULL DEFAULT 'none',
  auth_config_ciphertext TEXT,
  event_types JSONB NOT NULL,
  is_active BOOLEAN NOT NULL DEFAULT TRUE,
  last_delivery_status VARCHAR(32),
  last_delivered_at TIMESTAMPTZ,
  created_by CHAR(26) NOT NULL REFERENCES users(id),
  updated_by CHAR(26) REFERENCES users(id),
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  deleted_at TIMESTAMPTZ
);
```

設計規則：

- `secret_ciphertext` 使用後端設定的加密金鑰加密保存。
- 建立後只回傳一次明文 secret。
- 列表與詳情只回傳 `has_secret=true`。
- `event_types` 為字串陣列，例如 `["ticket.created","ticket.status_changed"]`。
- `auth_type` 可用值為 `none`、`basic`、`jwt`。
- `auth_config_ciphertext` 保存 outbound endpoint 認證設定，需加密保存且不得在列表或詳情回傳明文。

### `webhook_deliveries`

Webhook 投遞紀錄。

```sql
CREATE TABLE webhook_deliveries (
  id CHAR(26) PRIMARY KEY DEFAULT generate_ulid(),
  webhook_id CHAR(26) NOT NULL REFERENCES project_webhooks(id) ON DELETE CASCADE,
  event_id CHAR(26) NOT NULL REFERENCES notification_events(id) ON DELETE CASCADE,
  status VARCHAR(32) NOT NULL DEFAULT 'pending',
  attempt_count INTEGER NOT NULL DEFAULT 0,
  next_attempt_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  request_id VARCHAR(64) NOT NULL,
  http_status INTEGER,
  response_excerpt TEXT,
  error_message TEXT,
  delivered_at TIMESTAMPTZ,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  UNIQUE (webhook_id, event_id)
);
```

狀態值：

- `pending`
- `delivering`
- `delivered`
- `retrying`
- `failed`
- `cancelled`

索引：

```sql
CREATE INDEX idx_webhook_deliveries_due
  ON webhook_deliveries (next_attempt_at)
  WHERE status IN ('pending', 'retrying');

CREATE INDEX idx_webhook_deliveries_webhook_created
  ON webhook_deliveries (webhook_id, created_at DESC);
```

## Webhook payload

```json
{
  "event_id": "01K...",
  "event_type": "ticket.status_changed",
  "occurred_at": "2026-07-01T10:51:00+08:00",
  "project": {
    "id": "01K...",
    "key": "MS",
    "name": "ms"
  },
  "ticket": {
    "id": "01K...",
    "title": "zabbix 告警",
    "status": "open",
    "priority": "P1",
    "ticket_type": "Issue",
    "sub_project": "公共支持",
    "resource": "Zabbix Alert"
  },
  "actor": {
    "id": "01K...",
    "username": "admin",
    "full_name": "系統管理員"
  },
  "changes": {
    "status": {
      "before": "open",
      "after": "in_progress"
    }
  },
  "url": "/tickets/01K..."
}
```

Payload 規則：

- 不包含附件內容。
- 不包含 attachment storage key。
- 不包含登入 token。
- 不包含 Webhook secret。
- `url` 使用前端內部路徑，外部系統如需完整 URL 由接收端自行組合或後續新增 `app.public_base_url` 設定。

## Webhook 簽章

Request headers：

```text
Content-Type: application/json
X-Opscenter-Event: ticket.status_changed
X-Opscenter-Delivery: 01K...
X-Opscenter-Timestamp: 1782899460
X-Opscenter-Signature: sha256=<hex>
```

簽章規則：

```text
signed_payload = timestamp + "." + raw_body
signature = HMAC-SHA256(secret, signed_payload)
```

驗證建議：

- 接收端需檢查 timestamp 與目前時間差距，建議允許 5 分鐘。
- 接收端需使用 constant-time compare 比對簽章。
- 接收端需依 `X-Opscenter-Delivery` 或 payload `event_id` 去重。

## Webhook outbound 認證

Webhook 簽章用於 payload 完整性驗證；outbound 認證用於外部 endpoint 存取控制。兩者可同時存在，所有 Webhook request 都仍會保留 `X-Opscenter-Signature`。

支援 `auth_type`：

| auth type | Header | 說明 |
| --- | --- | --- |
| `none` | 無額外 header | 只使用 Webhook 簽章 |
| `basic` | `Authorization: Basic <base64>` | username / password 由管理者設定，加密保存 |
| `jwt` | `Authorization: Bearer <jwt>` | Opscenter 每次投遞產生短效 JWT |

### Basic Auth

`auth_config` 明文只允許在建立或更新 request 中出現：

```json
{
  "username": "opscenter",
  "password": "secret"
}
```

保存規則：

- username 與 password 一起加密後保存於 `auth_config_ciphertext`。
- 列表、詳情、投遞紀錄與審計紀錄不得回傳 password。
- 前端只顯示 `auth_type=basic` 與 `has_auth_secret=true`。

### JWT Auth

JWT Auth 使用短效 token，不保存長效 bearer token。每次投遞由 Opscenter 產生 JWT：

Header：

```json
{
  "alg": "HS256",
  "typ": "JWT"
}
```

Claims：

```json
{
  "iss": "opscenter",
  "aud": "webhook:<webhook_id>",
  "sub": "project:<project_id>",
  "jti": "<delivery_id>",
  "event_id": "<event_id>",
  "event_type": "ticket.created",
  "iat": 1782899460,
  "exp": 1782899760
}
```

JWT 規則：

- 預設有效時間讀取 `system_settings.webhook.jwt_ttl_seconds`，預設 seed 為 300 秒。
- 若單一 Webhook 建立或更新 request 明確帶入 `auth_config.jwt_ttl_seconds`，以單一 Webhook 設定為準。
- 若 request 未帶入 TTL，後端需讀取全域設定並將解析後的實際秒數快照保存於 `auth_config_ciphertext`，避免後續調整全域設定時改變既有 Webhook 的投遞契約。
- 若全域設定不存在、停用、非正整數或超出允許範圍，回退為 300 秒。
- `jti` 使用 delivery id，接收端可用於防重放。
- 簽署 secret 可使用 Webhook auth secret，與 payload signature secret 分開保存較佳。
- rotate secret 後，新投遞使用新 secret；舊 delivery 重試需使用當下有效 secret。
- JWT 不放入 payload，也不寫入投遞紀錄。

## Webhook 重試策略

第一版策略：

| attempt | 延遲 |
| --- | --- |
| 1 | 立即 |
| 2 | 1 分鐘 |
| 3 | 5 分鐘 |
| 4 | 15 分鐘 |
| 5 | 1 小時 |

規則：

- 2xx 標記為 `delivered`。
- 408、429、5xx、timeout 與暫時性網路錯誤進入 `retrying`。
- 429 若有 `Retry-After`，優先使用接收端指定時間，但不得超過系統最大延遲。
- 400、401、403、404、410、422 標記為 `failed`。
- 達最大嘗試次數後標記為 `failed`。
- 管理者手動重試會新增一次嘗試，不重建 event。

## SSRF 防護

Webhook URL 驗證需在建立、更新與投遞前都執行。

規則：

- 正式環境只允許 `https://`。
- 拒絕 `localhost`、`127.0.0.0/8`、`::1`、`10.0.0.0/8`、`172.16.0.0/12`、`192.168.0.0/16`、`169.254.0.0/16`。
- 拒絕雲端 metadata 位址。
- DNS 解析後 IP 仍需檢查。
- 不跟隨跨 host redirect。
- 不帶任何使用者 cookie。

本機開發若需要測試 private IP，需新增明確設定，例如：

```text
webhook.allow_private_network_in_dev = false
```

預設仍為 `false`。

## API 設計

### 站內通知

```text
GET   /api/v1/notifications
GET   /api/v1/notifications/unread-count
PATCH /api/v1/notifications/{id}/read
POST  /api/v1/notifications/mark-read
```

`GET /api/v1/notifications` query：

| 參數 | 型別 | 說明 |
| --- | --- | --- |
| `status` | string | `unread` 或 `all`，預設 `all` |
| `limit` | int | 預設 20，上限 50 |
| `cursor` | string | 游標分頁 |

Response `items[]`：

```json
{
  "id": "01K...",
  "event_type": "ticket.created",
  "title": "新的 P1 Issue",
  "body": "ms 專案建立了 zabbix 告警",
  "project_id": "01K...",
  "project_name": "ms",
  "ticket_id": "01K...",
  "ticket_title": "zabbix 告警",
  "link_path": "/tickets/01K...",
  "read_at": null,
  "created_at": "2026-07-01T10:51:00+08:00"
}
```

`POST /api/v1/notifications/mark-read` request：

```json
{
  "ids": ["01K..."],
  "scope": "selected"
}
```

`scope` 可用值：

- `selected`
- `all_visible`
- `all_unread`

### Webhook 設定

```text
GET    /api/v1/projects/{id}/webhooks
POST   /api/v1/projects/{id}/webhooks
GET    /api/v1/projects/{id}/webhooks/{webhook_id}
PUT    /api/v1/projects/{id}/webhooks/{webhook_id}
DELETE /api/v1/projects/{id}/webhooks/{webhook_id}
POST   /api/v1/projects/{id}/webhooks/{webhook_id}/rotate-secret
POST   /api/v1/projects/{id}/webhooks/{webhook_id}/test
GET    /api/v1/projects/{id}/webhooks/{webhook_id}/deliveries
POST   /api/v1/projects/{id}/webhook-deliveries/{delivery_id}/retry
```

Create request：

```json
{
  "name": "N8N webhook",
  "url": "https://example.com/opscenter/webhook",
  "auth_type": "jwt",
  "auth_config": {
    "jwt_ttl_seconds": 300
  },
  "event_types": ["ticket.created", "ticket.status_changed"],
  "is_active": true
}
```

Create response 只在建立時回傳一次 `secret`：

```json
{
  "id": "01K...",
  "name": "N8N webhook",
  "url": "https://example.com/opscenter/webhook",
  "auth_type": "jwt",
  "has_auth_secret": true,
  "event_types": ["ticket.created", "ticket.status_changed"],
  "is_active": true,
  "has_secret": true,
  "secret": "whsec_...",
  "created_at": "2026-07-01T10:51:00+08:00"
}
```

列表與詳情不得回傳 `secret`。

`auth_config` response 規則：

- `auth_type=none`：回傳 `has_auth_secret=false`。
- `auth_type=basic`：回傳 username 可遮罩顯示，password 不回傳。
- `auth_type=jwt`：只回傳 `has_auth_secret=true` 與 JWT TTL，不回傳 secret。

## 權限設計

站內通知：

- 登入即可讀取自己的通知。
- 只能 mark read 自己的通知。
- `admin` 也不得透過一般 API 讀取其他人的站內通知。

Webhook：

- `admin` 可管理所有專案 Webhook。
- `op_admin` 可管理營運專案 Webhook。
- 專案 `project_manager` 可管理自己專案 Webhook。
- `engineer`、`viewer` 不可管理 Webhook。

表單權限節點建議：

```text
tickets/webhooks
tickets/webhooks/create
tickets/webhooks/update
tickets/webhooks/delete
tickets/webhooks/test
tickets/webhooks/deliveries
```

## 後端模組設計

建議新增 bounded context：

```text
internal/notification
  domain.go
  repository.go
  service.go
  delivery.go
  dispatcher.go
  webhook_worker.go
  signer.go
  url_validator.go
```

主要介面：

```go
type EventPublisher interface {
    PublishTicketEvent(ctx context.Context, tx *gorm.DB, input TicketEventInput) error
}

type RecipientResolver interface {
    Resolve(ctx context.Context, event NotificationEvent) ([]Recipient, []RecipientSkip, error)
}

type WebhookDispatcher interface {
    DispatchDue(ctx context.Context, now time.Time, limit int) error
}
```

Ticket service 在 transaction 內呼叫 `EventPublisher`，不得直接呼叫 Webhook。

## 前端設計

### Header 通知面板

調整 `AppHeader`：

- `notificationCount` 從 API 取得，不再由 `MainLayout` 傳固定值。
- 點擊鈴鐺開啟 Popover 或 Drawer。
- 面板顯示未讀通知、全部通知切換。
- 點擊通知導向 `link_path`。
- 已讀與全部已讀操作需 invalidate unread count。

React Query keys：

```text
notifications.unreadCount
notifications.list(status, cursor)
```

### Webhook 管理頁

路由：

```text
/projects/:id/webhooks
```

入口：

- Ticket 專案工作區側邊選單新增「Webhook」。
- 僅具備管理權限時顯示。

頁面區塊：

- Webhook 設定 Data Grid
- 新增 / 編輯 Dialog
- rotate secret Dialog
- 測試投遞 Dialog
- 投遞紀錄 Data Grid

欄位：

- 名稱
- URL host
- 事件類型
- 啟用狀態
- 最後投遞狀態
- 最後投遞時間
- 更新時間
- 操作

## i18n

新增 namespace 建議：

```text
notification.json
webhook.json
```

必要文案：

- 通知
- 未讀通知
- 全部通知
- 全部標記已讀
- 尚無通知
- Webhook
- 新增 Webhook
- 測試投遞
- 重新產生 secret
- 投遞成功
- 投遞失敗
- URL 不允許使用內部網路位址

不得在中文語系中直接顯示 `schedule conflict` 這類未翻譯錯誤。

## 設定項

新增 global settings：

| key | value_type | 預設 | 說明 |
| --- | --- | --- | --- |
| `notification.retention_days` | number | `90` | 站內通知保留天數 |
| `webhook.delivery_retention_days` | number | `30` | Webhook 投遞紀錄保留天數 |
| `webhook.max_attempts` | number | `5` | 最大投遞嘗試次數 |
| `webhook.timeout_seconds` | number | `5` | HTTP timeout |
| `webhook.require_https` | boolean | `true` | 是否強制 HTTPS |
| `webhook.allow_private_network_in_dev` | boolean | `false` | 本機開發是否允許 private network |
| `webhook.jwt_ttl_seconds` | number | `300` | Webhook JWT Auth 預設有效秒數 |

`webhook.jwt_ttl_seconds` 只作為建立或更新 JWT Auth Webhook 時的預設值來源。已保存於單一 Webhook `auth_config_ciphertext` 的 TTL 是該 Webhook 的契約快照，不會因全域設定後續變更而自動改寫。

敏感設定：

```text
WEBHOOK_SECRET_ENCRYPTION_KEY
```

此值不得寫入資料庫或日誌。

## 測試策略

後端測試：

- notification event idempotency
- recipient resolver 去重與 skipped reason
- in-app notification list / unread count / mark read
- webhook CRUD 權限
- webhook URL validator SSRF 防護
- webhook signature
- webhook retry 狀態轉換
- Ticket 操作 transaction rollback 不產生 event

前端測試：

- Header 未讀數來自 API
- 通知面板空狀態與已讀操作
- 通知點擊導向 Ticket 詳情
- Webhook 新增 / 編輯表單驗證
- delivery Data Grid 狀態顯示
- i18n 文案不得漏 key

## 遷移與相容性

- 新資料表使用新 migration，不修改舊 migration。
- `ops_user.linked_user_id` 新增為 nullable，避免破壞既有資料。
- 無法對應登入帳號的 `ops_user` 不阻止 Ticket 流程。
- Header 目前硬寫未讀數需改成 API 載入；API 尚未完成前應顯示 `0` 或 loading，不得再顯示固定 `3`。
- Webhook 設定不存在時不影響 Ticket 操作。
