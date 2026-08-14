# Ticket 通知機制需求

## 文件定位

此 spec 接續 `../2026-06-01_10-22_oncall-ticket-system` 的需求 10，將原本保留設計的通知機制拆成可執行範圍。

本次範圍只包含：

- 站內通知
- Webhook 通知

本次範圍不包含：

- Email 通知
- Slack / Telegram 專用格式轉換
- SLA 自動升級通知
- 行動推播

## 目前已知契約狀態

- 原始需求 10 定義站內通知、Email 通知、Webhook 通知與 Ticket 相關事件。
- 目前前端 Header 有通知鈴鐺與未讀數視覺入口，但未讀數仍為靜態資料，尚未串 API。
- 目前後端沒有通知資料表、通知 API、Webhook 設定 API 或 Webhook 投遞紀錄。
- Ticket 建立者使用 `users.id`，但 Ticket 指派人與協作者使用 `ops_user.id`；站內通知需送給登入帳號，因此必須補上 `ops_user` 與 `users` 的對應規則。
- Webhook 為專案層級設定，不能作為全域設定套用到所有專案。

## 需求 1：通知事件

使用者需要在 Ticket 有重要更新時收到通知，外部系統也需要能透過 Webhook 接收事件。

### 驗收條件

- [ ] 1.1 系統需建立通知事件模型，事件來源至少包含 Ticket 建立、狀態變更、指派變更、協作者變更、留言新增與被 mention。
- [ ] 1.2 通知事件需保存事件類型、專案、Ticket、操作者、發生時間、事件摘要與必要 payload。
- [ ] 1.3 通知事件需具備 idempotent key，避免同一業務操作重複產生同一事件。
- [ ] 1.4 通知事件不得保存附件二進位內容、外部 Webhook secret、登入 token 或其他敏感資料。
- [ ] 1.5 Ticket 業務操作成功後才可產生通知事件；業務 transaction rollback 時不得留下通知事件。
- [ ] 1.6 通知事件產生不得阻塞 Ticket API 回應；Webhook 投遞失敗不得讓 Ticket 建立或更新失敗。

## 需求 2：站內通知

使用者需要在系統內看到自己需要處理或關注的 Ticket 事件。

### 驗收條件

- [ ] 2.1 系統需提供目前登入者的站內通知列表 API。
- [ ] 2.2 系統需提供目前登入者的未讀通知數 API。
- [ ] 2.3 使用者可將單筆通知標記為已讀。
- [ ] 2.4 使用者可將目前可見的通知批次標記為已讀。
- [ ] 2.5 通知列表需支援 unread / all 篩選與游標分頁。
- [ ] 2.6 通知內容需包含標題、摘要、事件類型、專案、Ticket 連結、發生時間與讀取狀態。
- [ ] 2.7 Header 通知鈴鐺未讀數必須來自 API，不得硬寫固定數字。
- [ ] 2.8 使用者點擊通知後需進入對應 Ticket 詳情頁，並可自動標記為已讀。
- [ ] 2.9 API 只允許讀取與修改目前登入者自己的通知，不得讀取其他使用者通知。
- [ ] 2.10 無法解析到登入帳號的 `ops_user` 不產生站內通知，但需保留可追蹤的解析略過原因。

## 需求 3：站內通知收件人規則

使用者需要只收到與自己相關的通知，避免通知噪音。

### 驗收條件

- [ ] 3.1 Ticket 建立時，需通知被指派人、協作者與專案管理者；操作者本人預設不收同一操作產生的通知。
- [ ] 3.2 P1 且事件類型為 Issue 的 Ticket 建立時，需通知專案管理者、被指派人與協作者；若後續補上值班組長模型，需納入值班組長。
- [ ] 3.3 Ticket 狀態變更時，需通知建立者、被指派人、協作者與專案管理者。
- [ ] 3.4 Ticket 指派變更時，需通知新的被指派人、Ticket 建立者與協作者。
- [ ] 3.5 Ticket 協作者新增時，需通知新增的協作者。
- [ ] 3.6 留言新增時，需通知 Ticket 建立者、被指派人、協作者與被 mention 的使用者。
- [ ] 3.7 mention 收件人必須限制在同專案有效成員內，不得通知非專案成員。
- [ ] 3.8 同一事件解析出的重複收件人需去重。
- [ ] 3.9 停用使用者、停用專案成員與已刪除 Ticket 不得產生新站內通知。

## 需求 4：Webhook 設定

專案管理者需要在專案層級設定 Webhook，讓外部系統接收 Ticket 事件。

### 驗收條件

- [ ] 4.1 Webhook 設定需掛在 project scope，不得做成全域設定。
- [ ] 4.2 Webhook 設定欄位至少包含名稱、URL、啟用狀態、事件類型、secret 狀態、最後投遞狀態與更新時間。
- [ ] 4.3 Webhook secret 需由後端產生或更新；建立後只顯示一次，後續 API 不得回傳明文 secret。
- [ ] 4.4 Webhook 需支援事件類型篩選，第一版至少支援 Ticket 建立、狀態變更、指派變更、留言新增與 mention。
- [ ] 4.5 Webhook 需支援測試投遞，測試事件不得建立真實 Ticket。
- [ ] 4.6 Webhook 設定新增、修改、停用與刪除需寫入審計紀錄。
- [ ] 4.7 停用的 Webhook 不得再建立新的投遞任務。
- [ ] 4.8 Webhook 需支援可選的 outbound endpoint 認證方式：none、Basic Auth、JWT Auth；認證方式由每個 Webhook 設定獨立指定。
- [ ] 4.9 Basic Auth 的 username 與 password 需加密保存，列表與詳情 API 不得回傳 password 明文。
- [ ] 4.10 JWT Auth 需由 Opscenter 於每次投遞時產生短效 Bearer JWT，外部系統可用 Webhook auth secret 驗證，不得保存長效明文 bearer token。
- [ ] 4.11 JWT Auth 的 `jwt_ttl_seconds` 若未由單一 Webhook 明確指定，後端需讀取 `system_settings.webhook.jwt_ttl_seconds` 作為全域預設；全域設定無效或未啟用時回退為 300 秒，且建立或更新後需將實際 TTL 快照保存於該 Webhook 的加密 `auth_config`。

## 需求 5：Webhook 投遞

外部系統需要收到格式穩定、可驗證且可重試的事件。

### 驗收條件

- [ ] 5.1 Webhook 投遞需非同步執行，不得阻塞 Ticket API。
- [ ] 5.2 每次投遞需保存 delivery record，包含 Webhook、事件、狀態、嘗試次數、HTTP 狀態碼、錯誤摘要、下次重試時間與完成時間。
- [ ] 5.3 投遞 payload 需包含 event id、event type、occurred at、project、ticket、actor、changes 與 url。
- [ ] 5.4 Webhook request 需帶簽章 header，外部系統可用 secret 驗證 payload 未被竄改。
- [ ] 5.5 投遞 timeout 預設不得超過 5 秒。
- [ ] 5.6 5xx、timeout 與暫時性網路錯誤需重試；2xx 視為成功；4xx 除 429 外預設不重試。
- [ ] 5.7 429 需依 `Retry-After` 或系統退避策略重試。
- [ ] 5.8 重試達上限後需標記為 failed，不得無限重試。
- [ ] 5.9 管理者可在 Webhook 投遞紀錄頁手動重試 failed delivery。
- [ ] 5.10 Webhook delivery response body 只保留截斷摘要，不得完整保存外部系統可能回傳的敏感內容。

## 需求 6：Webhook 安全

Webhook URL 屬於外部輸入，系統必須避免 SSRF 與 secret 洩漏。

### 驗收條件

- [ ] 6.1 Webhook URL 必須通過格式驗證，正式環境預設只允許 HTTPS。
- [ ] 6.2 Webhook 投遞需拒絕 localhost、loopback、link-local、private network 與 metadata service 位址；本機開發環境需透過明確設定才可放寬。
- [ ] 6.3 Webhook 不得跟隨跨 host redirect。
- [ ] 6.4 Webhook request 不得夾帶使用者 cookie、登入 token 或內部服務憑證。
- [ ] 6.5 Webhook secret 明文不得出現在日誌、審計紀錄或列表 API response。
- [ ] 6.6 Webhook secret 需可 rotate，rotate 後新投遞使用新 secret。
- [ ] 6.7 Webhook payload 中不得包含 Ticket 附件內容、附件 storage key、內部檔案路徑或 S3 URL。
- [ ] 6.8 Webhook outbound endpoint 認證資料需與簽章 secret 一樣加密保存，且不得出現在日誌、審計紀錄、投遞紀錄 response excerpt 或前端列表。

## 需求 7：權限

通知功能需符合既有 global role、project role 與表單權限模型。

### 驗收條件

- [ ] 7.1 站內通知列表與未讀數只需要登入即可讀取自己的資料。
- [ ] 7.2 Webhook 設定管理需限制為 `admin`、`op_admin` 或該專案 `project_manager`。
- [ ] 7.3 Webhook delivery 紀錄查詢需限制為 `admin`、`op_admin` 或該專案 `project_manager`。
- [ ] 7.4 一般專案成員可因 Ticket 事件收到站內通知，但不可管理 Webhook 設定。
- [ ] 7.5 權限不足時後端需回 403，前端不得顯示假成功。

## 需求 8：前端體驗

使用者需要能清楚查看通知與管理專案 Webhook。

### 驗收條件

- [ ] 8.1 Header 通知鈴鐺點擊後需顯示通知面板。
- [ ] 8.2 通知面板需顯示未讀優先、事件類型、專案、Ticket 標題與相對時間。
- [ ] 8.3 通知面板需提供全部標記已讀操作。
- [ ] 8.4 通知面板空狀態需清楚呈現，不得顯示假通知。
- [ ] 8.5 Webhook 管理入口放在 Ticket 專案工作區，不放在系統管理全域頁。
- [ ] 8.6 Webhook 管理頁需使用 Data Grid 顯示設定清單與投遞紀錄。
- [ ] 8.7 Webhook 新增與編輯需提供事件類型多選、URL 驗證、secret rotate 與測試投遞入口。
- [ ] 8.8 所有通知與 Webhook 文案需支援 i18n，不得出現未翻譯的英文錯誤訊息。

## 需求 9：可觀測與維運

維運人員需要追蹤通知產生與 Webhook 投遞狀態。

### 驗收條件

- [ ] 9.1 通知事件產生、站內通知展開、Webhook 投遞成功與失敗需有結構化日誌。
- [ ] 9.2 Webhook 投遞需具備 trace id 或 request id，便於與外部系統對帳。
- [ ] 9.3 系統需提供 pending、delivered、failed delivery 數量查詢或管理頁統計。
- [ ] 9.4 Webhook worker 停止後重啟，不得遺失 pending delivery。
- [ ] 9.5 資料保留策略需可設定，預設站內通知保留 90 天，Webhook delivery 保留 30 天。
