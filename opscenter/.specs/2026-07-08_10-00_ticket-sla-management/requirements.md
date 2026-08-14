# Ticket SLA 管理與 SLA Checker 需求

## 文件定位

本 spec 接續 `../2026-06-01_10-22_oncall-ticket-system` 的需求 11：SLA 管理。

原始設計稿為初始基準，本文件不修改原始 spec。此文件只處理 SLA 管理、Ticket SLA 狀態追蹤與 SLA checker。

## 目前狀態

- Ticket 事件類型已具備 `ticket_types.applies_sla` 欄位，可作為是否套用 SLA 的基礎。
- Ticket 已有 priority、status、created_at、resolved_at、closed_at 等欄位，可支援 SLA 起算與停止。
- 前端已有 `SLACountdown` 元件與部分 i18n 文案，但目前不是完整 SLA 契約。
- 儀表板 SLA 違反數目前仍標示為 `not_implemented`。
- 通知機制已由 `../2026-07-01_10-51_Ticket_notifications` 接走，包含站內通知、Webhook、dispatcher 與 worker。
- SLA 本 spec 不重新設計通知基礎建設，只新增 SLA breach 事件與既有通知模組的整合契約。

## 範圍

本次範圍包含：

- 專案層級 SLA 設定。
- SLA 預設值與專案覆寫。
- Ticket 建立時建立 SLA 快照。
- Ticket 狀態變更時更新回應時間與解決時間。
- SLA checker 定期檢查回應逾時與解決逾時。
- SLA breach 寫入 Ticket activity。
- SLA breach 產生既有 `notification_events`，由既有站內通知與 Webhook 流程處理。
- Ticket 列表與詳情顯示 SLA 狀態與倒數。
- 多 Pod 執行時的分散式鎖與 idempotency。

本次範圍不包含：

- 重新實作通知 dispatcher、站內通知 API 或 Webhook worker。
- Email 通知。
- Slack / Telegram 專用格式。
- SLA escalation 多層升級流程。
- SLA 達成率報表大改。
- 儀表板 SLA 指標完整接線。
- 工作時間日曆、休假日排除、營業時間 SLA。

## 需求 1：SLA 設定

專案管理者需要依專案與優先級設定 SLA 回應與解決時間。

### 驗收條件

- [ ] 1.1 SLA 設定需為 project scope，不得做成全域唯一設定。
- [ ] 1.2 每個 project + priority 可設定 response minutes 與 resolve minutes。
- [ ] 1.3 預設值需支援 P1=15min / 2hr、P2=1hr / 8hr、P3=4hr / 24hr、P4=24hr / 72hr。
- [ ] 1.4 專案未設定 SLA 時，後端需使用系統預設值產生 Ticket SLA 快照。
- [ ] 1.5 response minutes 與 resolve minutes 必須為正整數。
- [ ] 1.6 resolve minutes 不得小於 response minutes。
- [ ] 1.7 停用 SLA 設定不得影響既有 Ticket SLA 快照。
- [ ] 1.8 修改 SLA 設定只影響後續新 Ticket，預設不得回寫歷史 Ticket。

## 需求 2：SLA 適用範圍

系統需要清楚判斷哪些 Ticket 要套用 SLA。

### 驗收條件

- [ ] 2.1 只有 `ticket_types.applies_sla = true` 的 Ticket 需要建立 SLA 狀態。
- [ ] 2.2 `ticket_types.applies_sla = false` 的 Ticket 不得建立 SLA breach。
- [ ] 2.3 已刪除 Ticket 不得被 SLA checker 處理。
- [ ] 2.4 cancelled Ticket 應標記為 excluded，不列入 breach。
- [ ] 2.5 Ticket 建立後若事件類型改為不套用 SLA，需標記為 excluded 並保留 audit。
- [ ] 2.6 Ticket 建立後若事件類型改為套用 SLA，需依目前 Ticket priority 與建立時間補建 SLA 快照並保留 audit。

## 需求 3：SLA 快照

系統需要保存每張 Ticket 建立當下的 SLA 判定，避免後續設定變動污染歷史資料。

### 驗收條件

- [ ] 3.1 Ticket 建立成功後需建立 `ticket_sla_states`。
- [ ] 3.2 SLA 快照需包含 project、ticket、priority、response_due_at、resolve_due_at 與 applies_sla。
- [ ] 3.3 SLA due time 以 Ticket created_at 起算。
- [ ] 3.4 SLA 快照需記錄來源為 project config 或 default config。
- [ ] 3.5 SLA 設定更新後不得自動改寫既有 Ticket 的 due time。
- [ ] 3.6 Ticket priority 變更時需重新計算該 Ticket SLA due time，並寫入 Ticket activity。
- [ ] 3.7 Ticket priority 變更後若已經 breach，既有 breach 紀錄不得被刪除。

## 需求 4：回應 SLA

系統需要判斷 Ticket 是否在 SLA 回應時間內被處理。

### 驗收條件

- [ ] 4.1 第一版 response SLA 以 Ticket 第一次進入 `in_progress` 作為 responded_at。
- [ ] 4.2 若 Ticket 直接 resolved 或 closed，且尚未 responded_at，需同時補 responded_at。
- [ ] 4.3 若 responded_at <= response_due_at，response SLA 狀態為 met。
- [ ] 4.4 若 now > response_due_at 且 responded_at 為 NULL，SLA checker 需標記 response breach。
- [ ] 4.5 response breach 只能標記一次。
- [ ] 4.6 response breach 後才進入 in_progress，不得清除 response breach。

## 需求 5：解決 SLA

系統需要判斷 Ticket 是否在 SLA 解決時間內完成。

### 驗收條件

- [ ] 5.1 resolve SLA 以 Ticket resolved_at 或 closed_at 作為 resolved_at。
- [ ] 5.2 若 Ticket 已 closed 但 resolved_at 為 NULL，需以 closed_at 作為 SLA 解決時間。
- [ ] 5.3 若 resolved_at <= resolve_due_at，resolve SLA 狀態為 met。
- [ ] 5.4 若 now > resolve_due_at 且 resolved_at 為 NULL，SLA checker 需標記 resolve breach。
- [ ] 5.5 resolve breach 只能標記一次。
- [ ] 5.6 resolve breach 後才 resolved，不得清除 resolve breach。

## 需求 6：SLA Checker

系統需要定期檢查 SLA 逾時並保證多 Pod 不重複執行。

### 驗收條件

- [ ] 6.1 需新增 scheduler task key：`sla_checker`。
- [ ] 6.2 預設 cron 為每 5 分鐘執行一次。
- [ ] 6.3 SLA checker 必須使用既有 scheduler 分散式鎖。
- [ ] 6.4 SLA checker 只能處理 active 且 applies_sla 的 Ticket SLA state。
- [ ] 6.5 SLA checker 查詢需避免全表掃描，需具備 due time 索引。
- [ ] 6.6 SLA checker 更新 breach 欄位需使用 idempotent 條件，避免多 Pod 或重跑造成重複寫入。
- [ ] 6.7 SLA checker 每次執行需寫入 scheduler log，包含 scanned、response_breached、resolve_breached、skipped 與 lock result。
- [ ] 6.8 SLA checker 失敗不得中斷 server 主流程，但需寫入錯誤 log。

## 需求 7：Ticket Activity 與通知整合

SLA breach 需要可追蹤，也需要沿用已完成的通知機制通知相關人員。

### 驗收條件

- [ ] 7.1 response breach 需寫入 Ticket activity，action type 為 `sla_response_breached`。
- [ ] 7.2 resolve breach 需寫入 Ticket activity，action type 為 `sla_resolve_breached`。
- [ ] 7.3 activity payload 需包含 breach type、due_at、checked_at、priority 與 SLA config source。
- [ ] 7.4 SLA breach 需建立既有 `notification_events` outbox event。
- [ ] 7.5 SLA breach event type 需使用 `ticket.sla_response_breached` 與 `ticket.sla_resolve_breached`。
- [ ] 7.6 SLA breach 通知收件人第一版沿用 Ticket status changed 規則：建立者、被指派人、協作者與專案管理者。
- [ ] 7.7 操作者為 scheduler 時，站內通知不得因 actor 缺失而失敗，需使用 system actor 或 nullable actor 契約。
- [ ] 7.8 Webhook event type 清單需允許管理者選擇 SLA breach 事件。
- [ ] 7.9 SLA spec 不重新實作通知 dispatcher、站內通知 API 或 Webhook worker。

## 需求 8：API

前後端需要穩定 API 管理 SLA 設定與顯示 Ticket SLA 狀態。

### 驗收條件

- [ ] 8.1 提供 `GET /api/v1/projects/:id/sla` 查詢專案 SLA 設定。
- [ ] 8.2 提供 `PUT /api/v1/projects/:id/sla` 更新專案 SLA 設定。
- [ ] 8.3 SLA 管理 API 需限制為 `admin`、`op_admin` 或該專案 `project_manager`。
- [ ] 8.4 Ticket 列表 response 需包含 SLA 摘要。
- [ ] 8.5 Ticket 詳情 response 需包含完整 SLA 狀態。
- [ ] 8.6 SLA response 不得讓前端自行推論核心 breach 狀態。
- [ ] 8.7 API response 需支援 i18n 錯誤碼，不得直接回未翻譯英文訊息給前端顯示。

## 需求 9：前端體驗

使用者需要能管理 SLA 並在 Ticket 操作中看到 SLA 狀態。

### 驗收條件

- [ ] 9.1 專案工作區需提供 SLA 設定入口。
- [ ] 9.2 SLA 設定頁需使用 Data Grid 或結構化表單呈現 P1-P4 設定。
- [ ] 9.3 Ticket 列表需顯示 SLA 狀態 chip。
- [ ] 9.4 Ticket 詳情需顯示 response SLA 與 resolve SLA 倒數。
- [ ] 9.5 SLA overdue 需使用明確但不突兀的主題色。
- [ ] 9.6 不套用 SLA 的 Ticket 需顯示「無 SLA」，不得顯示錯誤狀態。
- [ ] 9.7 所有 SLA 文案需支援繁中、簡中、英文 i18n。

## 需求 10：驗證

SLA 判定涉及時間與排程，必須具備可重複驗證。

### 驗收條件

- [ ] 10.1 後端測試需覆蓋 SLA due time 計算。
- [ ] 10.2 後端測試需覆蓋 project config 與 default config fallback。
- [ ] 10.3 後端測試需覆蓋 applies_sla true / false。
- [ ] 10.4 後端測試需覆蓋 response breach idempotency。
- [ ] 10.5 後端測試需覆蓋 resolve breach idempotency。
- [ ] 10.6 後端測試需覆蓋 cancelled excluded。
- [ ] 10.7 後端測試需覆蓋 scheduler lock busy 不重複執行。
- [ ] 10.8 前端需通過 typecheck。
- [ ] 10.9 需提供手動驗收 SQL，確認 SLA state、activity、notification event 與 scheduler log。
