# Ticket SLA 管理與 SLA Checker 設計

## 設計目標

- 專案可獨立設定 P1-P4 的回應與解決 SLA。
- Ticket 建立時保存 SLA 快照，避免 SLA 設定變更影響歷史判定。
- SLA checker 可在多 Pod 部署下安全執行，不重複產生 breach。
- SLA breach 可追蹤、可通知、可被 Webhook 投遞，但不重新實作通知系統。
- Ticket 列表與詳情可直接取得 SLA 狀態，不讓前端自行推論核心規則。

## 範圍邊界

本設計只負責 SLA 管理與 checker。

通知模組已由 `../2026-07-01_10-51_Ticket_notifications` 完成核心能力。本設計只新增 SLA event type、payload 與呼叫既有 notification publisher 的契約。

第一版不處理：

- 營業時間 SLA。
- 假日或排班日曆扣除。
- 多層 escalation。
- SLA 達成率報表大改。
- 儀表板 SLA 指標完整接線。

## SLA 規則

### 適用判定

Ticket 建立時讀取 `ticket_types.applies_sla`。

- `true`：建立 `ticket_sla_states`。
- `false`：不建立 SLA state，API 回傳 `applies_sla=false`。

### 預設 SLA

| priority | response minutes | resolve minutes |
| --- | ---: | ---: |
| P1 | 15 | 120 |
| P2 | 60 | 480 |
| P3 | 240 | 1440 |
| P4 | 1440 | 4320 |

專案 SLA 設定存在且啟用時優先使用專案設定。沒有專案設定時使用預設值。

### 起算與停止

- 起算時間：`tickets.created_at`。
- response SLA 停止：第一次進入 `in_progress`，或直接 `resolved` / `closed`。
- resolve SLA 停止：`resolved_at` 或 `closed_at`。
- cancelled：標記 `excluded`，不再產生 breach。

### 歷史保護

SLA 設定更新只影響新 Ticket。既有 Ticket 使用 `ticket_sla_states` 內的快照。

Ticket priority 變更是 Ticket 本身條件變更，需重新計算該 Ticket 未完成 SLA due time，並寫入 activity。已產生的 breach 不清除。

## 資料模型

### `sla_configs`

專案層級 SLA 設定。

現況確認：`opscenter-server/sql/0026_create_ticket_schema.sql` 已存在簡版 `sla_configs`，欄位包含 `project_id`、`priority`、`response_minutes`、`resolve_minutes`。後續 migration 不得假設資料表全新建立，需以相容方式補齊 `is_active`、audit 欄位、時間驗證 constraint 與軟刪除欄位。

```sql
CREATE TABLE sla_configs (
  id CHAR(26) PRIMARY KEY DEFAULT generate_ulid(),
  project_id CHAR(26) NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
  priority VARCHAR(4) NOT NULL,
  response_minutes INTEGER NOT NULL,
  resolve_minutes INTEGER NOT NULL,
  is_active BOOLEAN NOT NULL DEFAULT TRUE,
  created_by CHAR(26) NOT NULL REFERENCES users(id),
  updated_by CHAR(26) REFERENCES users(id),
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  deleted_at TIMESTAMPTZ,
  CONSTRAINT chk_sla_config_priority CHECK (priority IN ('P1', 'P2', 'P3', 'P4')),
  CONSTRAINT chk_sla_config_response_positive CHECK (response_minutes > 0),
  CONSTRAINT chk_sla_config_resolve_positive CHECK (resolve_minutes > 0),
  CONSTRAINT chk_sla_config_resolve_gte_response CHECK (resolve_minutes >= response_minutes)
);

CREATE UNIQUE INDEX uq_sla_configs_project_priority_active
  ON sla_configs (project_id, priority)
  WHERE deleted_at IS NULL;
```

### `ticket_sla_states`

每張 Ticket 的 SLA 快照與執行狀態。

```sql
CREATE TABLE ticket_sla_states (
  ticket_id CHAR(26) PRIMARY KEY,
  ticket_created_at TIMESTAMPTZ NOT NULL,
  project_id CHAR(26) NOT NULL REFERENCES projects(id),
  priority VARCHAR(4) NOT NULL,
  applies_sla BOOLEAN NOT NULL,
  config_source VARCHAR(16) NOT NULL,
  response_minutes INTEGER,
  resolve_minutes INTEGER,
  response_due_at TIMESTAMPTZ,
  resolve_due_at TIMESTAMPTZ,
  responded_at TIMESTAMPTZ,
  resolved_at TIMESTAMPTZ,
  response_breached_at TIMESTAMPTZ,
  resolve_breached_at TIMESTAMPTZ,
  status VARCHAR(16) NOT NULL DEFAULT 'tracking',
  excluded_reason VARCHAR(64),
  last_checked_at TIMESTAMPTZ,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  CONSTRAINT chk_ticket_sla_priority CHECK (priority IN ('P1', 'P2', 'P3', 'P4')),
  CONSTRAINT chk_ticket_sla_config_source CHECK (config_source IN ('project', 'default', 'none')),
  CONSTRAINT chk_ticket_sla_status CHECK (status IN ('tracking', 'met', 'breached', 'excluded'))
);

CREATE INDEX idx_ticket_sla_response_due
  ON ticket_sla_states (response_due_at)
  WHERE applies_sla = TRUE
    AND responded_at IS NULL
    AND response_breached_at IS NULL
    AND status IN ('tracking', 'breached');

CREATE INDEX idx_ticket_sla_resolve_due
  ON ticket_sla_states (resolve_due_at)
  WHERE applies_sla = TRUE
    AND resolved_at IS NULL
    AND resolve_breached_at IS NULL
    AND status IN ('tracking', 'breached');
```

`tickets` 目前是月份分區表，主鍵為 `(id, created_at)`。`ticket_sla_states` 不直接建立 Ticket FK，需保存 `ticket_created_at` 作為應用層定位分區與驗證用欄位。

## 後端模組

建議新增 bounded context：

```text
internal/sla/
  domain.go
  repository.go
  service.go
  checker.go
  delivery.go
```

### Service 職責

- 讀取與更新 project SLA config。
- Ticket 建立後建立 SLA state。
- Ticket 狀態變更後更新 responded_at / resolved_at / excluded。
- Ticket priority 或 ticket type 變更後調整 SLA state。
- 產生 SLA API response。

### Checker 職責

- 由 scheduler task `sla_checker` 觸發。
- 取得分散式鎖後查詢到期 SLA state。
- 使用條件更新標記 breach。
- 寫入 Ticket activity。
- 建立 notification event。
- 寫入 scheduler log。

## Idempotency

SLA checker 必須允許重跑。

response breach 更新範例：

```sql
UPDATE ticket_sla_states
SET
  response_breached_at = NOW(),
  status = 'breached',
  last_checked_at = NOW(),
  updated_at = NOW()
WHERE ticket_id = $1
  AND applies_sla = TRUE
  AND responded_at IS NULL
  AND response_breached_at IS NULL
  AND response_due_at < NOW()
  AND status = 'tracking'
RETURNING *;
```

只有 `RETURNING` 有資料時才寫入 activity 與 notification event。

Ticket activity 可用 idempotency key 避免重複：

```text
ticket_sla:<ticket_id>:response
ticket_sla:<ticket_id>:resolve
```

notification event idempotency key：

```text
ticket_sla:<ticket_id>:response:notification
ticket_sla:<ticket_id>:resolve:notification
```

## Scheduler

新增 task：

| name | task key | cron | 說明 |
| --- | --- | --- | --- |
| SLA 檢查 | `sla_checker` | `*/5 * * * *` | 檢查 Ticket SLA 回應與解決逾時 |

執行規則：

- 使用既有 scheduler distributed lock。
- lock busy 時寫入 skipped log。
- 每批處理數量需有限制，第一版由 `system_settings.sla.checker_batch_limit` 控制，預設 200，上限 1000。
- 單筆處理失敗不應中止整批，需統計 failed count。

Scheduler log detail 建議格式：

```text
scanned=120 response_breached=3 resolve_breached=1 failed=0 skipped=false lock_result=acquired
```

Ticket activity 的系統自動活動不得假冒任一 admin。第一版允許 `ticket_activities.actor_id` 為 `NULL`，Ticket 詳情以「系統」顯示；一般人工操作仍帶入實際使用者。

## Notification 整合

既有通知系統已具備：

- `notification_events`
- dispatcher
- `user_notifications`
- `project_webhooks`
- `webhook_deliveries`
- Webhook worker

SLA 只新增事件型態：

| event type | 說明 |
| --- | --- |
| `ticket.sla_response_breached` | Ticket 未在 response SLA 前進入處理 |
| `ticket.sla_resolve_breached` | Ticket 未在 resolve SLA 前完成 |

Payload 範例：

```json
{
  "breach_type": "response",
  "ticket_id": "01K...",
  "project_id": "01K...",
  "priority": "P1",
  "due_at": "2026-07-08T10:15:00+08:00",
  "breached_at": "2026-07-08T10:20:00+08:00",
  "config_source": "project",
  "response_minutes": 15,
  "resolve_minutes": 120
}
```

收件人第一版沿用 Ticket status changed 規則：

- 建立者
- 被指派人
- 協作者
- 專案管理者

由於 SLA checker 沒有一般操作者，需新增 system actor 契約：

- 優先使用系統保留帳號。
- 若現有 notification event 必填 actor，需補支援 system actor。
- 不得使用隨機 admin 作為 actor。

Webhook 管理 UI 的事件類型選項需新增 SLA breach 事件。既有 Webhook delivery worker 不需重寫。

## API 設計

### 查詢專案 SLA 設定

```text
GET /api/v1/projects/:id/sla
```

Response：

```json
{
  "items": [
    {
      "priority": "P1",
      "response_minutes": 15,
      "resolve_minutes": 120,
      "source": "project",
      "is_active": true
    }
  ],
  "defaults": [
    {
      "priority": "P1",
      "response_minutes": 15,
      "resolve_minutes": 120
    }
  ]
}
```

### 更新專案 SLA 設定

```text
PUT /api/v1/projects/:id/sla
```

Request：

```json
{
  "items": [
    {
      "priority": "P1",
      "response_minutes": 15,
      "resolve_minutes": 120,
      "is_active": true
    }
  ]
}
```

更新採整包覆蓋 P1-P4，後端需驗證 priority 完整性與時間合法性。

### Ticket SLA 摘要

Ticket list item 新增：

```json
{
  "sla": {
    "applies_sla": true,
    "status": "tracking",
    "response_due_at": "2026-07-08T10:15:00+08:00",
    "resolve_due_at": "2026-07-08T12:00:00+08:00",
    "responded_at": null,
    "resolved_at": null,
    "response_breached": false,
    "resolve_breached": false,
    "next_due_at": "2026-07-08T10:15:00+08:00"
  }
}
```

不套用 SLA：

```json
{
  "sla": {
    "applies_sla": false,
    "status": "excluded",
    "excluded_reason": "ticket_type_not_applies_sla"
  }
}
```

## 前端設計

### 專案 SLA 設定頁

入口建議放在 Ticket 專案工作區：

```text
Tickets / 專案 / SLA 設定
```

畫面內容：

- P1-P4 設定表。
- response minutes。
- resolve minutes。
- 是否啟用。
- 顯示預設值與目前來源。
- 儲存前驗證 resolve >= response。

### Ticket 列表

操作重點：

- 加入 SLA chip 欄位。
- 狀態包含無 SLA、正常、即將逾時、已逾時、已完成、排除。
- overdue 使用 error 色系；due soon 使用 warning 色系；正常使用 neutral / info 色系。
- 文案全部走 i18n。

### Ticket 詳情

顯示：

- Response SLA 倒數。
- Resolve SLA 倒數。
- 設定來源。
- 已 breach 時顯示 breach time。
- 不套用時顯示「無 SLA」。

## 權限

| 操作 | 權限 |
| --- | --- |
| 查詢 SLA 設定 | 專案成員以上 |
| 更新 SLA 設定 | `admin`、`op_admin`、`project_manager` |
| 查看 Ticket SLA 狀態 | 可查看該 Ticket 者 |
| 執行 SLA checker | scheduler 內部 task |

## 測試策略

後端：

- SLA config 驗證。
- default fallback。
- Ticket 建立時 SLA state。
- status transition 更新 responded_at / resolved_at。
- priority 變更重新計算。
- response breach idempotency。
- resolve breach idempotency。
- cancelled excluded。
- scheduler lock busy。
- notification event 建立。

前端：

- SLA 設定表單驗證。
- Ticket list SLA chip。
- Ticket detail SLA panel。
- i18n key 完整性。
- typecheck。

手動驗收：

- 建立 applies_sla Ticket 後產生 SLA state。
- 超過 response due 後 checker 產生 response breach。
- 超過 resolve due 後 checker 產生 resolve breach。
- breach 只產生一次 activity。
- breach 會建立 notification event。
- Webhook 可選擇 SLA breach event type。
