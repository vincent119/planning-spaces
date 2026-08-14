# Draft Promotion Roadmap

Status: Draft

## 文件定位

本文件描述 `.kiro/specs/temp/drafts` 的升級順序。Draft 在確認前只代表規劃草案；正式實作前需 promote 到 `.kiro/specs/{YYYY-MM-DD-HH-mm}_{Type}-{name}`，並補齊 `requirements.md`、`design.md`、`tasks.md`。

## 第一批：近期缺口與 bugfix trace

### `Draft-group-based-operation-permission`

理由：近期遇到 op_member 無法讀取事件類型、SLA、Dashboard 等問題，本 Draft 應先釐清「讀取類操作是否一律走群組設定的表單 read 權限」與例外邊界。

Promote 後至少要補：

- `requirements.md`：列出每個讀取 API 的權限來源。
- `design.md`：定義 form read middleware、project membership fallback、admin override 的契約。
- `tasks.md`：每個 endpoint 的允許修改範圍與回歸測試。

### `Draft-structured-access-logging`

理由：近期確認 Gin access log 與 zlogger 混用會造成 log 格式不一致，應保留為 bugfix trace。

Promote 後至少要補：

- `requirements.md`：統一結構化日誌格式與不可再輸出 Gin 預設格式。
- `design.md`：列出 server middleware 與 logger 初始化契約。
- `tasks.md`：驗證啟動後 request log 只出現 zlogger JSON。

### `Draft-session-idle-timeout`

理由：閒置時間自動登出已實作，但前端 fake timer 單元測試仍是文件化缺口，應用新 SDD 補齊驗收。

Promote 後至少要補：

- `requirements.md`：session policy、使用者互動重置、跨分頁行為與登出流程。
- `design.md`：後端設定來源、API contract、前端 idle bridge。
- `tasks.md`：後端測試、前端 build、前端 timer 測試缺口。

## 第二批：共用契約先行

### `Draft-oncall-ticket-system-core`

定位：抽出所有功能共用的角色、專案、表單、Ticket、權限、設定與基礎 API envelope。

目的：避免後續 Ticket、SLA、Dashboard、Report 各自定義相同名詞。

### `Draft-ticket-workflow`

定位：Ticket 主流程與共用查詢條件，作為 SLA、Notification、Activity Log、Report 的上游契約。

目的：先固定 Ticket 狀態、欄位、查詢、排序與權限。

### `Draft-ticket-sla-management`

定位：SLA 讀取與計算要依賴 Ticket、Project、Form Permission 的共同契約。

目的：修正過去 SLA 讀取被 project membership 擋住的文件缺口。

## 第三批：使用者介面與報表能力

- `Draft-dashboard`
- `Draft-reporting`
- `Draft-jira-report-integration`
- `Draft-report-bi-templates`
- `Draft-business-metrics-observability`

升級時要先統一：

- 指標口徑
- timezone
- 權限
- 匯出與查詢效能
- 空資料與錯誤狀態

## 第四批：安全、身分與營運

- `Draft-mfa-global-settings`
- `Draft-sso-oidc-saml`
- `Draft-metrics-observability`
- `Draft-k8s-deployment-plan`

升級時要先固定：

- Secret 與設定來源
- 登入與 session contract
- Health check 與 metrics endpoint
- 部署環境差異

## 第五批：前端一致性與表單元件

- `Draft-menu-navigation`
- `Draft-frontend-button-consistency`
- `Draft-datefield-year-selector`
- `Draft-native-datefield-calendar`

升級時要先固定：

- Design token 與 Button 元件來源
- DateField 的輸入輸出型別
- 瀏覽器原生行為與自製元件切換條件
- 可及性與鍵盤操作驗收

## 整合驗收

每個 Draft promote 後，正式 `tasks.md` 必須包含：

- `git diff --check`
- 受影響後端 package 的 `go test`
- 受影響前端 app 的 build 或測試
- 權限變更需列出正向與反向案例
- UI 變更需列出桌面與手機驗收條件

