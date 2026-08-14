# 全域儀表板未結案 Ticket 明細需求

## 文件定位

本規格新增全域儀表板下方的「未結案 Ticket 明細」面板，接續既有 Dashboard snapshot 與最近 10 筆 Ticket 功能，但不修改 `.kiro/specs/2026-06-17_1619_Dashbard` 已完成工作。本規格是獨立新功能，不屬於 Ticket 列表 URL 狀態需求。

## 背景

目前 `/dashboard` 下方只有「最近 10 筆 Ticket」面板，依 `created_at DESC` 顯示使用者可存取範圍內最近建立的 10 筆 Ticket，不限制 Ticket 狀態。使用者還需要在同一區域直接查看尚未結案的 Ticket 明細，不必先進入各專案 Ticket 列表篩選。

## 目標

1. 在全域儀表板「最近 10 筆 Ticket」右側新增「未結案 Ticket 明細」。
2. 未結案明細沿用全域儀表板既有專案可見範圍與權限。
3. 未結案口徑與既有 `open_tickets` 指標一致。
4. 桌面版左右並排，窄螢幕自動改為上下排列。
5. Dashboard 每 30 秒刷新時，兩份明細使用同一份 snapshot 一起更新。

## 非目標

- 不新增或修改上方統計卡片。
- 不修改現有「最近 10 筆 Ticket」的資料口徑。
- 不修改專案儀表板 `/projects/:id/dashboard`。
- 不提供明細分頁、搜尋、排序控制或批次操作。
- 不修改 Ticket 狀態定義與權限模型。

## 已定義口徑

未結案 Ticket 條件：

```sql
t.status NOT IN ('resolved', 'closed', 'cancelled')
```

包含 `open`、`pending`、`in_progress`、`escalated`。資料依 `created_at DESC, id DESC` 排序，最多回傳 10 筆。

## 假設

「未結案 Ticket 明細」先以最近建立的 10 筆為上限，與左側最近 Ticket 面板的資料量一致。若後續需要顯示全部未結案 Ticket，應另行設計分頁或導向 Ticket 列表，不在本次範圍。

## 使用者故事

身為全域儀表板使用者，我希望在最近 Ticket 右側看到最近 10 筆未結案 Ticket，快速確認仍需處理的工作，且只能看到我原本有權存取的專案資料。

## 驗收條件

- [x] 1.1 全域 Dashboard snapshot 新增 `open_ticket_details` 陣列，最多 10 筆。
- [x] 1.2 `open_ticket_details` 只包含狀態不為 `resolved`、`closed`、`cancelled` 的 Ticket。
- [x] 1.3 明細依 `created_at DESC, id DESC` 穩定排序。
- [x] 1.4 明細欄位至少包含 Ticket ID、標題、優先級、狀態、專案、子專案及建立時間。
- [x] 1.5 全域 Admin／OP Admin 與一般使用者的資料範圍沿用既有 Dashboard scope，不得暴露無權存取的專案。
- [x] 1.6 現有 `recent_tickets` 仍不限狀態並維持最近 10 筆，不得被未結案條件影響。
- [x] 1.7 桌面寬度下，「最近 10 筆 Ticket」位於左側，「未結案 Ticket 明細」位於右側，兩者各占同列一半寬度。
- [x] 1.8 平板與手機寬度下，兩個面板改為上下排列，不產生非必要的整頁水平捲動。
- [x] 1.9 未結案明細具備載入、空資料及 API 錯誤狀態，並沿用 Dashboard 30 秒自動刷新。
- [x] 1.10 三語系新增必要標題與空資料文案。
- [x] 1.11 上方所有 Dashboard 卡片維持原狀。

## 驗收情境

### 場景 A：同時顯示兩份明細

假設使用者開啟全域儀表板，且可存取範圍內同時存在已結案與未結案 Ticket。

當 Dashboard snapshot 載入完成。

那麼左側顯示不限狀態的最近 10 筆 Ticket，右側只顯示最近 10 筆未結案 Ticket。

### 場景 B：未結案資料篩選

假設最近 Ticket 中包含 `resolved`、`closed` 或 `cancelled` 狀態。

當後端建立 `open_ticket_details`。

那麼這些 Ticket 不得出現在未結案明細，但仍可出現在左側最近 Ticket。

### 場景 C：一般使用者權限

假設一般使用者只屬於部分專案。

當使用者讀取全域 Dashboard。

那麼未結案明細只能包含該使用者可存取專案的 Ticket。

### 場景 D：響應式版面

假設使用者在桌面、平板與手機開啟全域儀表板。

當可用寬度改變。

那麼桌面顯示左右兩欄，平板與手機顯示上下排列，表格內容可在自身容器內處理寬度。

## 驗證需求

- Dashboard repository 未結案條件、排序、limit 與權限 scope 測試。
- Dashboard service／delivery snapshot contract 測試。
- 前端 TypeScript contract 與全域 Dashboard 面板測試或可重現驗收。
- `go test ./internal/dashboard/...`。
- 前端 `npm test`、`npm run typecheck`、`npm run build`。
- `git diff --check`。
