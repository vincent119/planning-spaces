# Ticket 列表 URL 查詢狀態設計

## 文件定位

本設計實作 `requirements.md` 定義的 Ticket 列表 URL 查詢狀態。修改限於前端 Ticket feature 與導覽測試，不變更後端 API 或資料庫。

## 現有行為

`ProjectTicketsPage` 以多個 `useState` 保存日期、搜尋、篩選與分頁。元件重新掛載時，日期使用 `todayInTaipei()`，其他狀態使用空值或第 1 頁，因此詳情返回後無法恢復原列表。

詳情頁返回按鈕目前使用 `navigate(-1)`；只要歷史中的列表 URL 帶有完整 query，返回後即可透過 URL 還原。直接開啟詳情頁時仍需要明確 fallback。

## Bounded Context

### 包含

- `ProjectTicketsPage` 的 URL query parse、serialize 與狀態更新。
- 列表控制項、Data Grid 與 TanStack Query key 同步。
- 查看／編輯導覽與詳情返回 fallback。
- URL helper 與導覽行為測試。

### 不包含

- 後端 Ticket API、資料庫與權限。
- 其他列表頁。
- Ticket 查詢條件本身的業務定義。
- 全域狀態或瀏覽器 storage。

## URL Query Contract

| URL query | 前端／API 狀態 | 預設值與驗證 |
|---|---|---|
| `keyword` | 已提交搜尋關鍵字 | 空字串 |
| `created_from` | 建立日期起 | 台北今日；`YYYY-MM-DD` |
| `created_to` | 建立日期迄 | 台北今日；`YYYY-MM-DD` |
| `status` | 狀態 | 既有狀態白名單 |
| `priority` | 優先級 | 既有優先級白名單 |
| `ticket_type_id` | 事件類型 | 空值代表全部 |
| `sub_project_id` | 子專案 | 空值代表全部 |
| `ticket_resource_id` | 資訊來源 | 空值代表全部 |
| `duty_shift_id` | 班別 | 無班別讀取權限時忽略 |
| `creator_id` | 開單人員 | 空值代表全部 |
| `assignee_id` | 指派人員 | 空值代表全部 |
| `page` | API 頁碼 | URL 從 1 起算；預設 1 |
| `page_size` | 每頁筆數 | 支援值 `10`、`25`、`50`；預設 10 |

空字串與預設值可以省略以保持 URL 精簡。非預設日期與有效篩選必須寫入 URL，確保可分享與還原。

## 狀態模型

新增純函式 helper，集中處理：

```text
URLSearchParams
  -> parseTicketListURLState(params, today)
  -> TicketListURLState
  -> list API params / UI controls / Data Grid model

TicketListURLState
  -> serializeTicketListURLState(state, defaults)
  -> URLSearchParams
```

建議位置：

```text
opscenter-frontend/src/features/ticket/utils/ticketListURLState.ts
```

helper 必須是無副作用純函式，方便測試日期、enum 與數字正規化。不得在多個事件 handler 重複手寫 query key。

## 同步策略

URL 是已提交查詢狀態來源；頁面以 `useSearchParams` 取得目前 query，解析後直接提供 API query key、API params 與控制項值。

使用者操作時建立下一份完整狀態並呼叫 `setSearchParams`：

- 搜尋送出：更新 `keyword`，`page = 1`。
- 日期或篩選變更：更新對應欄位，`page = 1`。
- 清除日期：移除日期 query；解析後套用既有預設規則。
- Data Grid 換頁：更新 `page`。
- page size 改變：更新 `page_size` 並將 `page = 1`。

`keywordDraft` 是唯一允許保留的輸入中本機 state。URL `keyword` 變更時，包括上一頁／下一頁，草稿需同步為已提交關鍵字，避免輸入框與實際查詢不一致。

不使用「URL 改 state、state effect 再改 URL」的雙向 effect，避免同步迴圈。已提交值直接由解析後的 URL state 衍生；事件 handler 是唯一寫入 URL 的入口。

## 歷史與導覽

一般篩選與分頁操作使用 push 或符合現有 UX 的歷史策略，但必須確保瀏覽器上一頁／下一頁可以還原查詢。短時間輸入草稿不寫入歷史。

從列表點擊查看或編輯時，不取代目前列表 history entry，因此原本含 query 的 URL 留在上一筆歷史中。

詳情頁返回：

1. 若由列表導覽進入，使用歷史返回，恢復完整 query。
2. 若直接開啟詳情頁，沒有可信任列表來源，導向 `/projects/{ticket.project_id}/tickets`。
3. 編輯成功把 `/tickets/{id}?edit=1` replace 成 `/tickets/{id}`，只取代詳情 entry，不影響前一筆列表 entry。

必要時列表導覽可在 `location.state` 放置非查詢用途的來源標記，例如 `fromTicketList: true`；查詢條件本身不得存入 state，仍以 URL 為準。

## 驗證與正規化

- 日期使用嚴格 `YYYY-MM-DD` 驗證，不接受 JavaScript Date 自動校正出的其他日期。
- `created_from > created_to` 時保留畫面錯誤狀態並停用 API，符合現有行為。
- 狀態與優先級使用既有型別白名單。
- ID 類 query 去除前後空白；空值視為未篩選。
- `page` 只接受正整數。
- `page_size` 只接受 `10`、`25`、`50`。
- 無班別讀取權限時，不將 `duty_shift_id` 傳給 API；可同步清除 URL 中該值，避免畫面與查詢不一致。

## 測試設計

### Helper 單元測試

- 完整 query parse 與 serialize round trip。
- 缺少 query 的預設值。
- 日期格式及實際日曆日期驗證。
- enum 白名單。
- page 與 page size 邊界。
- 空白 ID 與 keyword 正規化。

### 頁面與導覽測試

- 日期 7/1 進詳情返回後仍為 7/1。
- 多篩選與第 3 頁返回後完整恢復。
- URL 變更時控制項、Data Grid 與 API params 同步。
- 重新整理與直接開啟 query URL。
- 詳情直接開啟時的返回 fallback。
- 編輯成功 replace 不破壞列表 history。

## 風險與處理

### URL 與本機 state 雙重來源

已提交查詢不得同時保留獨立 `useState`。以 URL 衍生狀態，避免 effect 競態與返回後被預設值覆蓋。

### 歷史紀錄過多

只在已提交搜尋、選取篩選或分頁時更新 URL；文字輸入過程不寫入。若個別快速操作需要 replace，必須保留上一頁／下一頁可理解的查詢節點。

### 無效分享 URL

所有外部輸入先由純函式正規化，再建立 API params；不得直接把未驗證 query 傳給後端。

## 驗證指令

- `npm test` 或專案既有前端單元測試指令。
- `npm run typecheck`。
- `npm run build`。
- `git diff --check`。
