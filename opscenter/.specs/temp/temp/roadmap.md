# OnCall Ticket System Core Roadmap

Status: Draft

## 文件定位

本文件記錄從舊 `2026-06-01_10-22_oncall-ticket-system` 拆出、不再放在核心 MVP spec 的功能。

## 拆出原則

- 已有獨立 spec 的功能，後續以獨立 spec 為準。
- 尚未有獨立 spec 的 deferred 功能，等需求明確後再建立新版 SDD Draft。
- 核心 spec 只保留共用契約與必要引用。

## 已有獨立 spec 或應轉向的功能

| 功能 | 舊來源 | 建議去向 | Promote 建議 |
| --- | --- | --- | --- |
| Ticket 前後端完整流程 | 需求 3、6、7、8、9 | `.kiro/specs/2026-06-15_15-35_Ticket` | `Feature-ticket-workflow` |
| Dashboard 強化 | 需求 13 | `.kiro/specs/2026-06-17_1619_Dashbard` | `Feature-project-dashboard` |
| Menu / Navigation | 需求 2 前端入口 | `.kiro/specs/2026-06-18_17-45_Menu` | `Feature-menu-navigation` |
| Report | 需求 14 | `.kiro/specs/2026-06-22_11-06_Report` | `Feature-reporting` |
| Jira CSV / Report | 需求 15 | `.kiro/specs/2026-06-22_1832_jira_report` | `Feature-jira-report-integration` |
| Report BI Templates | 需求 14 | `.kiro/specs/2026-06-24_17-45_Report_BI_templates` | `Feature-report-bi-templates` |
| Fixed Shift / Leave | 需求 5 | `.kiro/specs/2026-06-25_11-04_fixed-shift-schedule-leave-management` | `Feature-fixed-shift-leave-management` |
| Ticket Notifications | 需求 10 | `.kiro/specs/2026-07-01_10-51_Ticket_notifications` | `Feature-ticket-notifications` |
| Ticket Partition Scheduler | backend deferred | `.kiro/specs/2026-07-01_12-12_ticket-partition-scheduler` | `Chore-ticket-partition-scheduler` |
| MFA Global Settings | 需求 1.9 | `.kiro/specs/2026-07-02_10-00_mfa-global-settings` | `Feature-mfa-global-settings` |
| SSO / OIDC / SAML | 需求 1.11、1.17 | `.kiro/specs/2026-07-03_17-20_sso-oidc-saml` | `Feature-sso-oidc-saml` |
| Ticket Activity Retention | 需求 8 延伸 | `.kiro/specs/2026-07-03_11-30_ticket-activity-log-retention` | `Chore-ticket-activity-log-retention` |
| Metrics Observability | 非功能需求 | `.kiro/specs/2026-07-07_10-07_metrics-observability` | `Feature-metrics-observability` |
| Ticket SLA Management | 需求 11 | `.kiro/specs/2026-07-08_10-00_ticket-sla-management` | `Feature-ticket-sla-management` |
| Business Metrics | Dashboard / Report 指標 | `.kiro/specs/2026-07-09_business-metrics-observability` | `Feature-business-metrics-observability` |
| K8s Deployment | 部署設計 | `.kiro/specs/2026-07-14_k8s-deployment-plan` | `Chore-k8s-deployment-plan` |

## 核心 spec 仍需保留的引用

- SLA：Ticket type 可保留 `applies_sla` 類型設定，但 SLA rule、checker、倒數與報表不在核心。
- Notification：Ticket activity 可作為事件來源，但通知投遞不在核心。
- Report：Ticket 查詢與活動資料是報表資料來源，但報表 API 與 UI 不在核心。
- Schedule：Ticket 不保存班別快照；班別統計由排班資料推導。
- SSO：內部 `global_role` 與權限群組模型保留，不讓外部群組直接取代。

## Deferred

下列功能若要重新啟動，應先建立 Draft：

- `Draft-api-key-authentication`
- `Draft-webhook-notifications`
- `Draft-async-import-export-job`
- `Draft-local-development-compose`

## Promote 順序建議

1. 先確認 `Feature-oncall-ticket-system-core`。
2. 再確認 `BugFix-group-based-read-permission`，作為 SLA、Dashboard、Ticket metadata read 的共同依據。
3. 接著處理 `Feature-ticket-workflow`。
4. 再依交付需要排 SLA、Dashboard、Report、Notification。

