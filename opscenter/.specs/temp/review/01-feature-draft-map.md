# Feature Draft Map

Status: Draft

## 文件定位

本文件把每個舊功能改寫成新版 SDD 的 Draft 規劃。這裡的 Draft 是審閱用藍圖，不是正式 spec。

## Draft 分組總覽

### Core / Permission / Auth

| Draft | 來源 | 建議類型 | 新版 SDD 規劃方式 |
| --- | --- | --- | --- |
| `Draft-oncall-ticket-core` | `2026-06-01_10-22_oncall-ticket-system` | `Feature` | 抽出核心資料模型、角色、專案、表單、Ticket、API envelope 與共用權限名詞，作為其他 spec 的上游契約。 |
| `Draft-menu-navigation` | `2026-06-18_17-45_Menu` | `Feature` | 把選單可見性、route guard、API 權限錯誤處理視為同一個使用者導覽功能。 |
| `Draft-mfa-global-settings` | `2026-07-02_10-00_mfa-global-settings` | `Feature` | 固定 MFA policy、登入流程接點、管理 API 與安全驗收，不混入 SSO protocol。 |
| `Draft-sso-oidc-saml` | `2026-07-03_17-20_sso-oidc-saml` | `Feature` | 先確認 IdP 數量、OIDC/SAML contract、使用者同步與 local login 關係，再拆 tasks。 |
| `Draft-op-admin-cross-project-read-access` | `2026-07-22_15-35_op-admin-cross-project-read-access` | `BugFix` | 以 bugfix spec 補齊 op_admin 跨專案 read 的允許與禁止行為。 |
| `Draft-group-based-operation-permission` | `2026-07-23_11-00_group-based-operation-permission` | `BugFix` | 把一般讀取 API 的權限來源集中定義，避免 SLA、Dashboard、事件類型各自走不同判斷。 |

### Ticket / SLA / Notification

| Draft | 來源 | 建議類型 | 新版 SDD 規劃方式 |
| --- | --- | --- | --- |
| `Draft-ticket-workflow` | `2026-06-15_15-35_Ticket` | `Feature` | Ticket 建立、查詢、編輯、狀態流轉、詳情頁與前後端資料載入放同一 Draft。 |
| `Draft-ticket-sla-management` | `2026-07-08_10-00_ticket-sla-management` | `Feature` | SLA rule、project SLA read、checker、違規狀態與權限驗收獨立規劃。 |
| `Draft-ticket-notifications` | `2026-07-01_10-51_Ticket_notifications` | `Feature` | Ticket event、通知對象、站內通知、未讀數與發送失敗策略一起定義。 |
| `Draft-ticket-activity-log-retention` | `2026-07-03_11-30_ticket-activity-log-retention` | `Chore` | 保留策略、清理 job、驗證報告與查詢效能保護獨立成維護型 spec。 |
| `Draft-ticket-partition-scheduler` | `2026-07-01_12-12_ticket-partition-scheduler` | `Chore` | Ticket 分區、排程、migration、rollback 與大量資料驗證要獨立規劃。 |

### Report / BI / Import

| Draft | 來源 | 建議類型 | 新版 SDD 規劃方式 |
| --- | --- | --- | --- |
| `Draft-reporting` | `2026-06-22_11-06_Report` | `Feature` | 報表查詢、篩選、匯出、視覺化、權限與 timezone 統一規劃。 |
| `Draft-ticket-import-tools` | `2026-06-22_15-24_Import_ticket_tools` | `Feature` | 匯入檔格式、dry-run、錯誤回報與寫入驗收一起定義。 |
| `Draft-jira-report-integration` | `2026-06-22_1832_jira_report` | `Feature` | Jira 資料來源、欄位映射、同步方式與報表呈現先確認再 promote。 |
| `Draft-report-bi-templates` | `2026-06-24_17-45_Report_BI_templates` | `Feature` | Template contract、migration、CRUD、套用流程與既有報表整合分清楚。 |

### Schedule / Leave

| Draft | 來源 | 建議類型 | 新版 SDD 規劃方式 |
| --- | --- | --- | --- |
| `Draft-fixed-shift-leave-management` | `2026-06-25_11-04_fixed-shift-schedule-leave-management` | `Feature` | 三班固定制、假勤、當前班別查詢、專案成員依賴與前端頁面流程一起規劃。 |

### Observability / Deployment

| Draft | 來源 | 建議類型 | 新版 SDD 規劃方式 |
| --- | --- | --- | --- |
| `Draft-metrics-observability` | `2026-07-07_10-07_metrics-observability` | `Feature` | HTTP/DB metrics、health、structured logging、trace 欄位與驗收集中定義。 |
| `Draft-business-metrics-observability` | `2026-07-09_business-metrics-observability` | `Feature` | 業務指標、資料來源、timezone、Dashboard/Report 一致性與驗證口徑集中定義。 |
| `Draft-k8s-deployment-plan` | `2026-07-14_k8s-deployment-plan` | `Chore` | Deployment、ConfigMap、Secret、Service、Ingress、rollout、rollback 與 dry-run 驗收。 |
| `Draft-structured-access-logging` | 近期 debug fix | `BugFix` | 明確定義 request log 只走 zlogger JSON，禁止 Gin 預設 `[GIN]` 格式混入。 |

### Frontend UX / Components

| Draft | 來源 | 建議類型 | 新版 SDD 規劃方式 |
| --- | --- | --- | --- |
| `Draft-frontend-button-consistency` | `2026-07-22_12-00_frontend-button-consistency` | `Refactor` | Button variant、size、loading、disabled、icon 與頁面遷移範圍要寫清楚。 |
| `Draft-datefield` | `2026-07-23_10-00_datefield-year-selector`、`2026-07-23_11-59_native-datefield-calendar` | `Feature` | 年份選擇與原生/自製日曆其實是同一元件契約，建議合成一個 Draft。 |
| `Draft-public-site-settings` | `2026-07-03_10-00_public-site-settings` | `Feature` | 公開可讀欄位、後台管理、cache 與禁止外洩資料要先定義。 |

### Session / Security Runtime

| Draft | 來源 | 建議類型 | 新版 SDD 規劃方式 |
| --- | --- | --- | --- |
| `Draft-session-idle-timeout` | 近期補文件與實作 | `Feature` | session policy API、security setting、前端 idle bridge、跨分頁與 timer 測試驗收。 |

## 重要調整

這裡刻意把 `datefield-year-selector` 與 `native-datefield-calendar` 合併成 `Draft-datefield`。前一版若一對一機械轉換，會讓同一個元件被兩個 Draft 拉扯，這就是看起來怪的主因之一。

`group-based-operation-permission` 與 `op-admin-cross-project-read-access` 不合併。前者是一般群組 read 權限規則，後者是 op_admin 的跨專案例外，兩者驗收方向不同。

