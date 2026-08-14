# Replan Method

Status: Draft

## 文件定位

本文件描述舊 `.kiro/specs` 如果改用新版 `sdd-skill`，重新規劃時應採用的流程。

## 舊設計的問題

目前舊 spec 的主要問題不是內容不足，而是規劃層級混在一起：

- 初始大型 spec 同時包含 MVP、Phase 2、Phase 3、後續 bugfix 與驗收紀錄。
- 多數目錄使用 `task.md`，新版 SDD 預期是 `tasks.md`。
- frontend/backend 常被拆成不同文件，但使用者感知功能其實是同一個交付。
- 權限、日誌、session、timezone、指標口徑等共用契約散落在不同功能。
- 已完成、未完成、待驗收與 debug trace 沒有清楚分層。

## 新版 SDD 的重新規劃流程

### 1. 先建立 Draft

大型或不確定需求不直接建立正式 spec。每個 Draft 至少要回答：

- 來源舊 spec 是哪個。
- 這個功能要解決什麼使用者可感知問題。
- 包含與不包含的邊界。
- 需要先確認的契約。
- promote 成正式 spec 時建議的類型。

### 2. 抽出共用契約

以下內容應先從舊設計中抽出，成為其他功能引用的基礎：

- 使用者、群組、角色、專案、表單與 Ticket 的資料關係。
- 一般讀取 API 的權限來源。
- API envelope、錯誤格式、request id、trace id。
- timezone、日期格式與報表統計口徑。
- session、登入與安全設定。

### 3. 再依功能 promote

Draft 確認後，才建立正式 spec：

```text
.kiro/specs/{YYYY-MM-DD-HH-mm}_{Type}-{kebab-case-name}/
```

正式 spec 必須包含：

- `requirements.md`
- `design.md`
- `tasks.md`

若只是補舊功能文件，使用 `Docs`。若是近期 bugfix trace，使用 `BugFix`。若會修改功能行為，使用 `Feature` 或 `Refactor`。

## 拆分原則

### 應該合併在同一 Draft

- 同一個頁面流程的 frontend/backend。
- 同一個使用者操作所需的 API、UI、驗收。
- DateField 年份選擇與日曆策略，若最後落在同一元件。
- Dashboard 指標與它直接依賴的讀取權限契約。

### 應該拆成不同 Draft

- Ticket 主流程與 SLA checker。
- Report 查詢與 BI template 管理。
- 技術 observability 與 business metrics。
- SSO 身分協議與 MFA policy。
- K8s 部署與應用程式功能。

## 本次 review 的輸出限制

本次只建立 Draft 規劃審閱稿，不建立正式 spec，不修改舊 spec，不執行程式碼實作。

