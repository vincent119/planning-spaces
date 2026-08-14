# SDD Draft Index

Status: Draft

## 文件定位

本文件把 `.kiro/specs` 既有舊設計重新映射為 `sdd-skill` 的 Draft 規劃。這不是原始 spec 的狀態盤點，而是「如果今天重新用 SDD 開始規劃」的建議 Draft 清單。

所有 Draft 都放在 `.kiro/specs/temp/drafts`，避免影響現有需求、設計與 task 文件。

## Draft 命名規則

正式採用時應移到：

```text
.kiro/specs/drafts/{YYYY-MM-DD-HH-mm}_Draft-{kebab-case-name}/
```

目前在 temp 中先用穩定名稱：

```text
.kiro/specs/temp/drafts/Draft-{kebab-case-name}/brief.md
```

## Draft 清單

| Draft | 來源舊 spec | 功能定位 | Promote 類型建議 |
| --- | --- | --- | --- |
| `Draft-oncall-ticket-system-core` | `2026-06-01_10-22_oncall-ticket-system` | Oncall Ticket System 核心總規劃、共用資料模型、使用者角色與 MVP 邊界 | `Feature` |
| `Draft-ticket-workflow` | `2026-06-15_15-35_Ticket` | Ticket 建立、查詢、編輯、狀態流轉、詳情頁與前後端流程 | `Feature` |
| `Draft-dashboard` | `2026-06-17_1619_Dashbard` | 專案 Dashboard、統計卡、圖表資料與讀取權限 | `Feature` |
| `Draft-menu-navigation` | `2026-06-18_17-45_Menu` | 前端選單、導覽、權限可見性與 UX 一致性 | `Feature` |
| `Draft-reporting` | `2026-06-22_11-06_Report` | 報表查詢、匯出、視覺化與權限控制 | `Feature` |
| `Draft-ticket-import-tools` | `2026-06-22_15-24_Import_ticket_tools` | Ticket 批次匯入工具、欄位驗證、錯誤回報與 dry-run | `Feature` |
| `Draft-jira-report-integration` | `2026-06-22_1832_jira_report` | Jira 報表資料接入、映射、統計與展示 | `Feature` |
| `Draft-report-bi-templates` | `2026-06-24_17-45_Report_BI_templates` | BI 報表模板、合約、遷移與模板管理 | `Feature` |
| `Draft-fixed-shift-leave-management` | `2026-06-25_11-04_fixed-shift-schedule-leave-management` | 固定班表、假勤、排班規則與前後端管理流程 | `Feature` |
| `Draft-ticket-notifications` | `2026-07-01_10-51_Ticket_notifications` | Ticket 通知事件、訂閱對象、站內通知與發送策略 | `Feature` |
| `Draft-ticket-partition-scheduler` | `2026-07-01_12-12_ticket-partition-scheduler` | Ticket 分區、保留策略、排程維護與查詢效能 | `Chore` |
| `Draft-mfa-global-settings` | `2026-07-02_10-00_mfa-global-settings` | MFA 全域設定、登入流程整合與管理頁 | `Feature` |
| `Draft-public-site-settings` | `2026-07-03_10-00_public-site-settings` | 公開站台設定、公開資訊讀取與後台管理 | `Feature` |
| `Draft-ticket-activity-log-retention` | `2026-07-03_11-30_ticket-activity-log-retention` | Ticket 活動紀錄保留、清理排程與驗證紀錄 | `Chore` |
| `Draft-sso-oidc-saml` | `2026-07-03_17-20_sso-oidc-saml` | SSO、OIDC、SAML、登入入口與身分同步 | `Feature` |
| `Draft-metrics-observability` | `2026-07-07_10-07_metrics-observability` | 技術 metrics、Prometheus、health、日誌與追蹤 | `Feature` |
| `Draft-ticket-sla-management` | `2026-07-08_10-00_ticket-sla-management` | SLA 規則、計算、Checker、違規狀態與專案 SLA 讀取 | `Feature` |
| `Draft-business-metrics-observability` | `2026-07-09_business-metrics-observability` | 業務指標、統計口徑、Dashboard/Report 指標一致性 | `Feature` |
| `Draft-k8s-deployment-plan` | `2026-07-14_k8s-deployment-plan` | Kubernetes 部署、設定、secret、健康檢查與 rollout | `Chore` |
| `Draft-frontend-button-consistency` | `2026-07-22_12-00_frontend-button-consistency` | 前端按鈕樣式、狀態、元件使用規範與回歸檢查 | `Refactor` |
| `Draft-op-admin-cross-project-read-access` | `2026-07-22_15-35_op-admin-cross-project-read-access` | op_admin 跨專案讀取規則、例外與不可升權邊界 | `BugFix` |
| `Draft-datefield-year-selector` | `2026-07-23_10-00_datefield-year-selector` | DateField 年份選擇 UX、日期輸入契約與相容性 | `Feature` |
| `Draft-group-based-operation-permission` | `2026-07-23_11-00_group-based-operation-permission` | 群組表單 read 權限、操作頁讀取規則與 middleware contract | `BugFix` |
| `Draft-native-datefield-calendar` | `2026-07-23_11-59_native-datefield-calendar` | 原生與自製日曆切換、DateField 可用性與瀏覽器行為 | `Feature` |
| `Draft-session-idle-timeout` | 近期新增文件需求與實作 | 閒置時間自動登出、session policy API、前端 idle bridge 與驗收測試 | `Feature` |
| `Draft-structured-access-logging` | 近期 debug fix | Gin 內建 access log 與 zlogger 格式一致性 | `BugFix` |

## Promote 原則

1. 有明確待修 bug 的 Draft 優先 promote 成 `BugFix`。
2. 已完成但缺新式文件的功能，只 promote 成 `Docs` 或保留 Draft trace。
3. 大型未完成功能先保留 Draft，拆清楚 bounded context、合約與驗收後再 promote。
4. 跨功能共用契約優先寫成獨立 design section，不把權限、日誌、session 行為散落在各功能 task。

