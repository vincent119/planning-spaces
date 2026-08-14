# Dashboard Requirements

## 文件定位

本 spec 獨立追蹤 Opscenter Dashboard。舊 spec 已有 Dashboard 初版設計與歷史 task，但目前前端仍顯示「Dashboard API 尚未提供，目前顯示空資料狀態」，代表現有前端資料來源仍是 fallback，不可視為端到端完成。

## 需求 1：Dashboard API 真實資料

**使用者故事**：身為值班組長，我希望儀表板顯示真實 Ticket 統計，而不是空資料或假資料。

### 驗收條件

- [ ] 1.1 `GET /api/v1/dashboard` 回傳目前使用者可存取專案的跨專案彙總資料。
- [ ] 1.2 `GET /api/v1/projects/{id}/dashboard` 回傳指定專案的彙總資料。
- [ ] 1.3 API 回傳統一 Dashboard snapshot contract，包含 metrics、trend、priority_distribution、recent_tickets、scope 與 generated_at。
- [ ] 1.4 前端不得再使用 `is_api_connected=false` 的假空資料作為正常狀態；API 失敗需顯示錯誤，空資料需顯示真實空狀態。

## 需求 2：指標與圖表

**使用者故事**：身為值班組長，我希望快速掌握目前待處理量、今日變化、優先級分佈與近期 Ticket。

### 驗收條件

- [ ] 2.1 指標卡片顯示：進行中 Ticket、今日新增、今日解決、待處理 P1/P2。
- [ ] 2.2 SLA 違反為 Phase 2 指標；MVP contract 可保留 `sla_breaches`，值為 `0` 且標記 `source=not_implemented`，不得假造 SLA 統計。
- [ ] 2.3 Ticket 趨勢圖顯示最近 7 天每日新增與解決數。
- [ ] 2.4 優先級分佈顯示目前未結束 Ticket 的 P1/P2/P3/P4 數量。
- [ ] 2.5 最近 Ticket 顯示最近 10 筆可存取 Ticket，欄位包含 id、title、priority、status、project、sub_project、created_at。

## 需求 3：範圍與權限

**使用者故事**：身為不同權限的使用者，我只能看到自己有權存取的 Dashboard 資料。

### 驗收條件

- [ ] 3.1 全域 Dashboard：`admin` / `op_admin` 可看全部專案；一般使用者只看自己是專案成員的專案。
- [ ] 3.2 專案 Dashboard：需通過 project access viewer 權限檢查。
- [ ] 3.3 私有專案不可因全域 Dashboard 暴露給非成員；公開專案的可見性規則依 project access 設計執行。
- [ ] 3.4 API 不回傳使用者無權存取的 Ticket 或專案名稱。

## 需求 4：刷新與狀態

**使用者故事**：身為值班人員，我希望 Dashboard 自動刷新，但能清楚知道資料是否載入失敗。

### 驗收條件

- [ ] 4.1 前端每 30 秒自動刷新 Dashboard。
- [ ] 4.2 首次載入顯示 skeleton 或 progress。
- [ ] 4.3 API 錯誤顯示錯誤狀態與重試操作。
- [ ] 4.4 API 回傳真實空資料時顯示空圖表與空列表，不顯示「API 尚未提供」。

