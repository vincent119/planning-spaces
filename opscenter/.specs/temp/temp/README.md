# OnCall Ticket System Core 收斂稿

Status: Draft

## 文件定位

本目錄是 `.kiro/specs/2026-06-01_10-22_oncall-ticket-system` 依新版 `sdd-skill` 收斂後的審閱稿。

本目錄不取代原始 spec，也不代表已進入實作。若確認採用，應 promote 成正式 spec：

```text
.kiro/specs/{YYYY-MM-DD-HH-mm}_Feature-oncall-ticket-system-core/
```

## 收斂原則

- 保留核心平台與 Ticket MVP。
- 將 SLA、通知、報表、Jira、完整排班、SSO、API Key、K8s 等後續功能移出核心 spec。
- frontend/backend 不再分散成兩份設計文件，改由同一個 `design.md` 描述跨層 contract。
- task 改為新版 SDD 要求的 `tasks.md`，每個任務包含邊界與驗證。

## 文件清單

- `requirements.md`：收斂後的核心需求。
- `design.md`：核心資料、API、權限、前端與驗證設計。
- `tasks.md`：新版 SDD 任務與品質檢查。
- `roadmap.md`：從舊大 spec 拆出去的功能與後續 promote 建議。

