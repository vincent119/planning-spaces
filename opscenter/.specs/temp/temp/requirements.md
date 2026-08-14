# OnCall Ticket System Core Requirements

Status: Draft

## 文件定位

本文件收斂自 `.kiro/specs/2026-06-01_10-22_oncall-ticket-system`。

新版定位是「核心平台與 Ticket MVP」需求，不再承載所有 Phase 2 / Phase 3 功能。被拆出的功能記錄於 `roadmap.md`，後續應各自 promote 成獨立 spec。

## 背景

系統提供工程師使用的運維平台，用於記錄、追蹤與管理運維事件、日常巡檢與問題處理。舊 spec 同時包含核心 MVP、後續功能、前端細節、後端資料模型與近期 bugfix，導致 scope 過大。

本收斂稿將 MVP 固定為：

- 使用者、認證、個人資料與 MFA 基礎。
- 表單樹、權限群組與 Casbin 授權。
- 專案、子專案與專案成員。
- Ticket CRUD、狀態流轉、指派、協作、留言、活動紀錄、搜尋與附件。
- Global Setting、排程基礎、操作日誌與結構化 request log。
- 基礎 Dashboard 統計。

## 目標

- 建立可追溯、可驗收的核心 MVP spec。
- 將「核心契約」與「後續功能」分離。
- 明確定義一般工作區讀取 API 的表單 read 權限規則。
- 讓 frontend/backend 共用同一份需求與設計邊界。

## 非目標

本 spec 不實作下列功能，只保留外部相依或 roadmap：

- SLA 管理與 SLA checker。
- 站內通知、Email、Webhook。
- 自訂報表、BI 範本、Jira 報表。
- 完整排班與假勤管理。
- SSO / OIDC / SAML。
- API Key 認證。
- Kubernetes 部署規劃。

## 現有行為

- 舊 spec 已記錄大量需求與實作 task，但範圍跨越 MVP、Phase 2、Phase 3。
- backend task 多數核心項目已勾選完成，但文件本身註明 Ticket OpenAPI 與前端端到端仍不能以舊勾選視為完整完成。
- frontend task 包含大量已完成頁面，也保留報表、Jira、無障礙與測試基礎等未完成項目。
- 近期修正已補入舊 spec，例如一般工作區讀取 API 應先走表單 read 權限、access log 統一 zlogger、session idle timeout。

## 新行為

- 核心 MVP 只追蹤本文件列出的核心需求。
- 後續功能不得追加到本 spec 的已完成 task 內。
- 一般工作區 read API 應以對應 `form_nodes.full_path` 的有效 `read` 權限作為第一層授權。
- 若 read API 涉及特定專案資料，表單 read 通過後只做必要的專案存在性與資料範圍檢查；不得無條件以 project membership 取代表單 read 權限。
- 寫入、管理、刪除仍需依對應 action 與必要 project role 檢查。

## 使用情境

### 使用者登入並進入工作區

使用者以帳密登入，必要時完成 MFA。前端取得目前使用者、session policy、表單樹與有效選單，僅顯示具備 read 權限的入口。

### 工程師建立 Ticket

工程師具備 `tickets/list:create` 或對應建單節點 create 權限時，可選擇專案、子專案、事件類型、資訊來源與優先級建立 Ticket。建立成功時同交易寫入第一筆 `ticket_activities`。

### 工程師查詢與處理 Ticket

工程師具備 Ticket read 權限時可查詢列表與詳情；具備 update 或對應專案角色時可更新狀態、指派、留言與附件。

### 管理員維護權限

Admin 管理表單樹、權限群組、群組成員與群組表單權限。權限異動需同步 Casbin policy 並寫入審計。

## 驗收情境

### 場景：一般工作區讀取 API 使用表單 read 權限

測試：`go test ./internal/auth ./internal/form ./internal/project ./internal/ticket`

假設：使用者屬於權限群組，對 `tickets/sla`、`tickets/ticket-types` 或 `dashboard` 具備 `read`。

當：使用者呼叫對應的 read API。

那麼：API 應通過表單 read 檢查；不得因缺少 project membership 直接回 403，除非該 API 的正式需求明確要求 project role。

### 場景：無 read 權限時拒絕工作區資料

測試：`go test ./internal/auth ./internal/form`

假設：使用者不具備目標表單節點 read 權限。

當：使用者直接呼叫受保護 read API。

那麼：API 回傳 403，前端選單也不顯示該入口。

### 場景：Ticket 建立同交易寫入活動紀錄

測試：`go test ./internal/ticket`

假設：使用者具備建單權限，且 payload 通過驗證。

當：使用者建立 Ticket。

那麼：`tickets.created_by` 與第一筆 `ticket_activities.action_type = created` 必須在同一交易完成；任一失敗時 rollback。

### 場景：附件不暴露儲存後端位置

測試：`go test ./internal/attachment ./internal/ticket`

假設：Ticket 有圖片附件。

當：使用者查詢附件 metadata 或讀取附件內容。

那麼：metadata 不包含 `storage_key`，內容僅能透過 API 串流取得，不使用 S3 pre-signed URL 或直接暴露 Local path。

### 場景：閒置時間自動登出政策最小揭露

測試：`go test ./internal/auth ./internal/server`

假設：使用者已登入。

當：前端呼叫 `GET /api/v1/auth/session-policy`。

那麼：response 只包含 `idle_timeout_seconds`，不包含 system setting key、category、description、is_secret 或設定來源。

## 驗收條件

- 核心需求能以 `requirements.md`、`design.md`、`tasks.md` 追溯。
- MVP 與 Phase 2 / Phase 3 邊界清楚。
- 權限、API、資料模型、前端入口與驗證方式在設計文件中一致。
- 任何新增功能不得寫入已完成 task 並維持完成狀態。

## 驗證需求

- 後端：執行受影響 package 的 `go test`。
- 前端：執行既有 build 或測試指令。
- 文件：執行 `git diff --check`。
- 權限變更：至少包含 allow 與 deny 案例。
- UI 變更：至少包含入口可見性與直接開路由/API 的拒絕案例。

