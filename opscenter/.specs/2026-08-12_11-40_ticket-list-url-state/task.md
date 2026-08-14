# Ticket 列表 URL 查詢狀態 Tasks

Status: Complete

## 文件定位

本文件追蹤 `requirements.md` 與 `design.md` 的實作工作。這是獨立新需求，不修改既有 oncall spec 已完成 task 的狀態。

## Execution Context

### 意圖

讓 Ticket 列表的日期、搜尋、篩選與分頁可由 URL 還原，解決詳情或編輯返回後日期變成今日及其他條件遺失的問題。

### 非目標

- 不修改後端 Ticket API、資料庫或查詢口徑。
- 不修改其他列表頁。
- 不引入全域 Context 或瀏覽器 storage 保存查詢。

### 已定決策

- URL query 是已提交列表查詢狀態的可還原來源。
- 搜尋草稿保留本機 state，送出後才寫入 URL。
- URL `page` 使用 1-based，Data Grid 使用 0-based。
- 無效 query 先正規化，不直接傳給 API。
- 詳情返回優先使用保留 query 的歷史；直接開啟詳情使用專案列表 fallback。

### 邊界

Allowed Changes：Ticket 列表 URL helper、`ProjectTicketsPage`、必要的詳情導覽／返回邏輯、前端測試與本規格文件。

Forbidden：後端 API、資料庫、其他列表頁、Ticket 統計口徑與權限模型。

### 關鍵檔案

- `opscenter-frontend/src/features/ticket/pages/ProjectTicketsPage.tsx`
- `opscenter-frontend/src/features/ticket/pages/ProjectTicketDetailPage.tsx`
- `opscenter-frontend/src/features/ticket/utils/ticketListURLState.ts`
- 對應前端測試檔案

### 完成條件

- Requirements 1.1 至 1.11 具測試或可重現驗證證據。
- 返回、重新整理、分享與歷史導覽均可恢復列表。
- `npm run typecheck`、`npm run build`、測試與 `git diff --check` 通過。

## Protected Behavior

- 預設日期仍為台北今日。
- 列表 API query parameter 名稱與伺服器端分頁維持不變。
- `keepPreviousData` 的換頁穩定行為保留。
- 日期區間錯誤時不送 API 的既有行為保留。
- 班別篩選仍受 `schedule/shifts:read` 權限限制。

## 1. URL 狀態模型

- [x] 1.1 建立 Ticket 列表 URL 狀態型別與解析 helper
  - Boundary: 僅新增 ticket feature URL helper 與單元測試。
  - Depends: 無。
  - Context: 定義完整 query contract、預設值、日期／enum／數字正規化。
  - Verify: parse 測試涵蓋完整、缺值與無效 query。

- [x] 1.2 建立序列化與狀態更新 helper
  - Boundary: URL helper 與單元測試。
  - Depends: 1.1。
  - Context: 省略空值／預設值，保留非預設日期與篩選；資料集合變更可重設 page。
  - Verify: serialize 與 round-trip 測試。

## 2. Ticket 列表整合

- [x] 2.1 將已提交列表狀態改由 URL 衍生
  - Boundary: `ProjectTicketsPage.tsx` 與必要 helper。
  - Depends: 1.1、1.2。
  - Context: 控制項、API params、TanStack Query key 與 Data Grid model 共用解析後狀態，避免雙重來源。
  - Verify: 初始 URL、非預設 URL 與無效 URL 的畫面／API 行為。

- [x] 2.2 串接搜尋、篩選與分頁 URL 更新
  - Boundary: Ticket 列表事件 handler。
  - Depends: 2.1。
  - Context: 搜尋送出才更新 keyword；資料集合變更重設 page；換頁更新 page／page_size。
  - Verify: 控制項操作後 URL 與 API query 一致。

- [x] 2.3 支援瀏覽器歷史與重新整理還原
  - Boundary: Ticket 列表 URL 同步與頁面測試。
  - Depends: 2.2。
  - Context: popstate／React Router search params 變更後，控制項、Data Grid 與 query key 自動跟隨。
  - Verify: 上一頁、下一頁、重新整理與分享 URL 情境。

## 3. 詳情導覽與返回

- [x] 3.1 保留列表 URL history entry
  - Boundary: Ticket 列表查看／編輯導覽。
  - Depends: 2.2。
  - Context: 進入詳情不得 replace 含 query 的列表 entry。
  - Verify: 7/1 列表進詳情返回後仍為 7/1。

- [x] 3.2 實作直接開啟詳情的返回 fallback
  - Boundary: `ProjectTicketDetailPage.tsx` 返回導覽與必要來源標記。
  - Depends: 3.1。
  - Context: 有列表來源時返回 history；直接開啟時導向 Ticket 所屬專案預設列表。
  - Verify: 列表來源與直接開啟兩種情境。

- [x] 3.3 保護編輯成功的歷史行為
  - Boundary: 詳情頁移除 `edit=1` 的 replace 行為及測試。
  - Depends: 3.2。
  - Context: replace 只更新詳情 entry，不影響前一筆列表 URL。
  - Verify: 編輯成功後返回仍恢復完整列表 query。

## 4. 驗證與文件

- [x] 4.1 執行自動化驗證
  - Boundary: 僅修正本需求造成的測試、型別或 build 問題。
  - Depends: 1.1 至 3.3。
  - Verify:
    - 前端單元／導覽測試
    - `npm run typecheck`
    - `npm run build`
    - `git diff --check`

- [x] 4.2 完成瀏覽器驗收與文件紀錄
  - Boundary: 人工驗收與本 task Implementation Notes。
  - Depends: 4.1。
  - Verify:
    - 日期 7/1 編輯後返回仍為 7/1
    - 多篩選與第 3 頁返回
    - 重新整理與上一頁／下一頁
    - 直接開啟詳情返回 fallback

## 品質檢查清單

- [x] Requirements、design、task 與實作可追溯。
- [x] URL 是已提交列表查詢的單一可還原來源。
- [x] 無效 query 不會送出不合法 API request。
- [x] 返回、重新整理、分享與歷史導覽均可恢復。
- [x] 後端 API、資料庫與其他列表頁未修改。
- [x] 自動化驗證與人工瀏覽器驗收通過。

## Implementation Notes

- 2026-08-12：建立獨立新規格，從既有 oncall spec 移出尚未實作的 URL 狀態需求；目前只完成 requirements、design 與 Pending tasks，尚未修改程式。
- 2026-08-12：使用者指示執行，狀態調整為 InProgress；實作範圍維持前端 URL 狀態、詳情返回與對應測試。
- 2026-08-12：完成 URL parse／serialize helper、列表 URL 單一狀態來源、搜尋／篩選／分頁同步、詳情來源標記與直接開啟 fallback。`npm test` 共 14 項通過，`npm run typecheck`、`npm run build` 與 `git diff --check` 通過；人工瀏覽器驗收尚待執行，因此狀態維持 InProgress。
- 2026-08-12：使用者確認可定義完成，人工驗收與文件回填完成，規格狀態更新為 Complete。
