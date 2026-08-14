# Ticket Frontend Design

## 文件定位

本文件只設計 Ticket 工作區前端頁面。共用 Layout、登入、Admin、儀表板、報表與排程設計仍在原 spec。

## 設計原則

- Ticket 工作區是操作型介面，採密集但清楚的資訊布局。
- 不做行銷式 hero 或裝飾性卡片堆疊。
- 列表與矩陣使用 MUI X Data Grid Community。
- 按鈕需使用 lucide 或現有 MUI icon。
- Dialog、Select、Menu 不得被毛玻璃效果影響。
- 所有 table / grid 內建文字需支援 i18n。
- API 失敗不得先顯示成功。

## 路由

```text
/projects/:id/tickets             Ticket 列表
/projects/:id/tickets/new         建立 Ticket
/tickets/:id                      Ticket 詳情
/projects/:id/sub-projects        子專案管理
/projects/:id/members             專案成員管理
/projects/:id/sources             資訊來源管理
/projects/:id/ticket-types        事件類型管理
```

側欄 `Tickets` 入口若尚未有 current project，只能作為 fallback route 使用：

- 讀取 `GET /api/v1/projects`。
- 找到可存取專案時，直接導向第一個專案的 `/projects/:id/tickets`。
- 不建立獨立專案列表頁，不顯示 placeholder shell。
- API 失敗或沒有可存取專案時顯示錯誤 / 空狀態，不得顯示假專案。
- 側欄 `Tickets` 為父選單，底下包含：
  - `子專案`：`/projects/:id/sub-projects`
  - `專案成員`：`/projects/:id/members`
  - `資訊來源`：`/projects/:id/sources`
  - `事件類型`：`/projects/:id/ticket-types`
  - `Ticket 列表`：`/projects/:id/tickets`
- 側欄子選單預設 seed 排序為：子專案、專案成員、資訊來源、事件類型、Ticket 列表；最終顯示順序由 Menu spec 的 `form_nodes.sort_order` 決定。
- 側欄可見性與排序需依 `.kiro/specs/2026-06-18_17-45_Menu`；Ticket spec 不再以硬編陣列作為最終排序來源。
- `子專案`、`專案成員`、`資訊來源`、`事件類型` 不在頁首以 tabs 呈現。
- `資訊來源` 與 `事件類型` 目前只建立側欄入口與待實作頁；完整管理 UI 需另立 task。

## 專案工作區骨架

```text
專案 / OPS 平台維運 / Tickets

┌──────────────────────────────────────────────────────────────┐
│ OPS 平台維運                                                  │
│ Key: OPS   Private   Active   20px   主專案 [OPS 平台維運 ▼] │
└──────────────────────────────────────────────────────────────┘

頁面內容依側欄子表單 route 切換。
```

頁首需顯示：

- 主專案下拉選單
- 專案名稱
- Project key
- 專案 visibility / status
- 主專案下拉選單需放在 status chip 後方，桌面版與 status chip 間距 20px
- 頁首不顯示 Tickets / 子專案 / 成員 tabs

主專案下拉選單規則：

- 資料來源使用 `GET /api/v1/projects`，只顯示目前使用者可存取的專案。
- 選項顯示專案名稱，輔助資訊顯示 project key 與 status。
- 目前路由的 `:id` 必須與選取的主專案一致。
- 切換主專案後導向相同側欄子表單的新 project route，例如從 `/projects/A/tickets` 切到 `/projects/B/tickets`。
- 若目前在建立 Ticket 頁且表單已 dirty，切換前需二次確認。
- API 失敗時顯示錯誤狀態，不得顯示假專案。

## Ticket 列表頁

路由：`/projects/:id/tickets`

```text
專案 / OPS 平台維運 / Tickets

[搜尋標題 / 描述 / 外部單號] [日期起] [日期迄] [狀態 ▼] [優先級 ▼] [事件類型 ▼] [子專案 ▼] [資訊來源 ▼] [指派人 ▼] [刷新] [+ 新增 Ticket]

┌──────────────────────────────────────────────────────────────────────────────────────────────┐
│ ID        標題              狀態        優先級  類型   子專案  資訊來源  指派人  更新時間 操作 │
│ 01K...    支付域名切換       Open        P2      Issue  QIEZ    Zabbix   alice   15:20    👁 ✏ │
│ 01K...    每日巡檢           Resolved    P4      Daily  OPS     Manual   -       14:10    👁   │
└──────────────────────────────────────────────────────────────────────────────────────────────┘
```

篩選列排版規則：

- 篩選列需自適應容器寬度，不得把「狀態」等下拉欄位擠壓到文字重疊。
- 搜尋欄可佔較大寬度，下拉欄位需保留可讀最小寬度。
- 日期區間使用兩個日期欄位，顯示於搜尋欄後方；桌面版可同列，寬度不足時可換行。
- 寬度不足時篩選欄位與操作按鈕可換行，操作按鈕不得遮住或壓縮下拉選單。

篩選行為：

- 進入 Ticket 列表頁時，建立日期區間預設為使用者目前時區的當天。
- 預設日期區間需實際帶入 API query，不得只改 UI 顯示；列表初次載入即查詢當天資料。
- 查詢參數使用 `created_from` 與 `created_to`，日期欄位以 `YYYY-MM-DD` 傳給後端。
- 台灣使用者預設以 `Asia/Taipei` 日界線計算當天日期；後端會以同一時區將日期轉成起訖時間查詢 `tickets.created_at`。
- 使用者清空日期區間時，才查詢不限制建立日期的列表。
- 日期起點不可晚於日期迄點；錯誤時停留在前端表單驗證，不送出 API。
- 點擊搜尋、按 Enter 或變更篩選後送出查詢時，pagination 需重設為第一頁。

欄位：

| 欄位 | 來源 | 備註 |
| --- | --- | --- |
| ID | `id` | ULID，縮短顯示但 tooltip 顯示完整值 |
| 標題 | `title` | 點擊可進詳情 |
| 狀態 | `status` | Chip |
| 優先級 | `priority` | P1/P2 使用警示色 |
| 類型 | `ticket_type_name` 或 contract 欄位 | 不得自行假造 |
| 子專案 | `sub_project_name` 或 contract 欄位 | 不得自行假造 |
| 資訊來源 | `ticket_resource_name` 或 `ticket_resource_code` | 不得自行假造 |
| 指派人 | `assignee_full_name` 或 `assignee_username` | 無資料顯示 `-` |
| 更新時間 | `updated_at` | 依目前語系格式化 |
| 操作 | row actions | 複製建立、查看、編輯、刪除 |

列操作「複製建立」：

- 使用複製圖示按鈕放在查看按鈕前方。
- 點擊後導向 `/projects/:id/tickets/new`，並透過路由 state 帶入建立頁草稿資料。
- 不直接呼叫 create API，也不建立草稿資料列；操作員需在建立頁確認後送出。
- 建立頁可預填 `title`、`ticket_type_id`、`priority`、`sub_project_id`、`ticket_resource_id`、`assignee_id`。
- `description` 只在原 Ticket 有描述內容時才帶入原描述；不得把列資訊摘要寫入描述欄。
- 不複製 Ticket ID、附件 / 貼圖、storage key、詳情連結。

列操作「刪除」：

- 刪除圖示按鈕放在操作欄最後。
- 只有目前使用者為該 Ticket 建立者，且該 Ticket 狀態為 `open` 時顯示。
- 點擊後開啟二次確認 Dialog，確認文案需包含 Ticket 標題。
- 確認後呼叫 `DELETE /api/v1/tickets/:id`；成功後重新讀取目前列表。
- 刪除失敗需顯示錯誤 toast，不得從列表假移除。

空狀態：

- 無資料：顯示「目前沒有 Ticket」。
- 有建立權限：顯示「新增 Ticket」按鈕。
- 無建立權限：不顯示新增按鈕。

主專案連動規則：

- Ticket 列表資料來源必須使用目前主專案下拉選單選取的 project id。
- 路由為 `/projects/:id/tickets` 時，列表 API 使用 `GET /api/v1/projects/{id}/tickets`。
- 主專案下拉選單切換後，需更新 route、清空目前列表選取列、重設 pagination 至第一頁，並重新讀取列表資料。
- 子專案、成員、事件類型、資訊來源等篩選選項若與 project scope 有關，也必須重新讀取，不得沿用上一個主專案的選項。
- 切換期間顯示 loading 狀態；新專案資料回來前不得顯示上一個主專案的 Ticket 為目前資料。

## 建立 Ticket 頁

路由：`/projects/:id/tickets/new`

```text
專案 / OPS 平台維運 / 新增 Ticket

┌─ 基本資料 ───────────────────────────────┐
│ 標題 *                                    │
│ 描述 *                                    │
│ 事件類型 *     優先級 *                  │
│ 子專案 *       資訊來源 *                │
│ 外部單號                                  │
└──────────────────────────────────────────┘

┌─ 附件 ───────────────────────────────────┐
│ 拖曳圖片到這裡，或 [上傳圖片]             │
│ 支援貼上截圖、JPG / PNG / GIF / WebP      │
└──────────────────────────────────────────┘

[取消] [建立 Ticket]
```

表單規則：

- `title` 必填，長度依後端契約。
- `description` 必填。
- `ticket_type_id` 必填。
- `priority` 必填，選項需依 metadata options 回傳的 `priorities` 與目前事件類型的 `allowed_priorities`。
- `project_id` 由路由帶入，不讓使用者在此頁任意改成其他 project。
- `sub_project_id` 必填。
- `ticket_resource_id` 必填，選項必須來自目前 project scope 的 `ticket_resources`。
- `external_ref` 選填。
- 若從列表「複製建立」進入，頁面需顯示提示，告知已套用複製資料且仍需操作員確認送出。

若 Ticket create API 尚未出現在 OpenAPI，不得實作送出流程。

主專案連動規則：

- 建立 Ticket 頁的 project id 必須來自主專案下拉選單目前選取值與 route `:id`，不得在表單內另放主專案欄位。
- 表單中的子專案與成員選項，必須依目前主專案 scope 讀取。
- 事件類型與 priority 的基礎選項來自 metadata options；資訊來源必須使用目前主專案 scope 的 `ticket_resources`，不得使用已廢棄的 `ticket_sources`。
- 建立 API body 的 `project_id` 必須等於目前主專案 id。
- 主專案下拉選單切換時，若表單已 dirty，需二次確認；確認後導向新主專案的 `/projects/:id/tickets/new` 並重設表單。
- 切換期間顯示 loading 狀態；新主專案參照資料回來前不得允許送出。

## Ticket 詳情頁

路由：`/tickets/:id`

```text
Ticket / 01K...

┌─ 標題與狀態列 ─────────────────────────────────────────────────────┐
│ 支付域名切換                         [Open ▼] [指派] [編輯]       │
│ P2 Issue / QIEZ / Zabbix / 建立於 2026/06/15 15:20                 │
└───────────────────────────────────────────────────────────────────┘

┌─ 主內容 ───────────────────────┐ ┌─ 側欄 ────────────────────────┐
│ 描述                            │ │ 指派人                         │
│ 活動紀錄 / 留言 Tabs             │ │ 協作者                         │
│ 附件                            │ │ 事件類型 / 子專案 / 來源        │
└─────────────────────────────────┘ └──────────────────────────────┘
```

主要區塊：

- 描述
- 活動紀錄 timeline
- 留言輸入框
- 附件列表與預覽

側欄：

- 狀態
- 優先級
- 事件類型
- 子專案
- 資訊來源
- 指派人
- 協作者
- 建立人
- 建立 / 更新時間

## 活動紀錄

```text
15:20  alice 建立 Ticket
15:25  bob 將狀態從 Open 改為 In Progress
15:30  alice 留言：已確認影響範圍
15:35  bob 上傳附件 image.webp
```

規則：

- 依 `created_at` 由新到舊或舊到新需一致。
- 系統事件與留言需視覺區分。
- Markdown 留言顯示需避免 XSS，使用安全 renderer 或純文字策略。

## MarkdownEditor

MarkdownEditor 用於 Ticket 描述與留言輸入。建立 Ticket 的描述欄位與詳情頁留言欄位都使用同一個元件，不各自重新實作。

UI 行為：

```text
┌─ 描述 / 留言 ─────────────────────────────┐
│ [編輯] [預覽]                              │
│ ┌──────────────────────────────────────┐ │
│ │ Markdown 內容                         │ │
│ └──────────────────────────────────────┘ │
│ 錯誤訊息 / 字數提示 / 送出狀態             │
└──────────────────────────────────────────┘
```

規則：

- 編輯模式保留 Markdown 原文。
- 預覽模式使用安全 renderer 或純文字 fallback，禁止直接注入未消毒 HTML。
- 空內容、超過後端長度限制、送出失敗時需顯示欄位錯誤。
- 送出中禁用重複提交。

## 附件

附件 UI：

```text
┌─ 附件 ────────────────────────────────┐
│ 拖曳圖片到此處，或貼上截圖 / 選擇檔案  │
│ [縮圖] image.webp  320KB  alice  刪除 │
│ [縮圖] screenshot.webp  420KB bob     │
└──────────────────────────────────────┘
```

規則：

- 支援點擊選擇檔案、拖曳上傳、貼上截圖。
- 拖曳區需有 hover / active / invalid 狀態。
- 貼上截圖只接受 `ClipboardEvent` 中的 image item。
- 點擊、拖曳、貼上三種入口都必須共用同一套檔案驗證。
- 驗證包含 content-type 白名單、單檔小於等於 10MB、每張 Ticket 附件數量小於等於 20。
- 圖片內容透過 `GET /api/v1/attachments/:id/content` 取得。
- 不顯示 storage key。
- 已關閉 Ticket 不顯示上傳 / 刪除按鈕。

留言圖片附件：

- 留言編輯區需支援拖曳圖片與貼上截圖。
- 待送出的留言圖片需顯示在 MarkdownEditor 下方，包含縮圖、檔名、大小與移除按鈕。
- 點擊「新增留言」後，前端需先建立留言 activity，再將待上傳圖片逐張送到 `POST /api/v1/tickets/{id}/attachments`，並帶入該留言的 `activity_id`。
- 送出過程中留言內容、mentions、圖片佇列與送出按鈕需禁用，避免重複提交。
- 成功後需重新讀取 Detail；留言列表需在每則留言下方顯示其 `attachments` 縮圖。
- 如果留言建立成功但部分圖片上傳失敗，前端需顯示錯誤並重新讀取 Detail，不得顯示假成功圖片。

## 子專案管理頁

路由：`/projects/:id/sub-projects`

```text
專案 / OPS 平台維運 / 子專案

[搜尋 key / 名稱] [狀態 ▼] [+ 新增子專案]

┌───────────────────────────────────────────────┐
│ Key   名稱       狀態    更新時間       操作   │
│ QIEZ  支付平台   啟用    2026/...       ✏ 🗑   │
└───────────────────────────────────────────────┘
```

API：

```text
GET    /api/v1/projects/:id/sub-projects
POST   /api/v1/projects/:id/sub-projects
GET    /api/v1/sub-projects/:sid
PUT    /api/v1/sub-projects/:sid
DELETE /api/v1/sub-projects/:sid
```

主專案連動規則：

- 子專案列表資料來源必須使用目前主專案下拉選單選取的 project id。
- 路由為 `/projects/:id/sub-projects` 時，列表 API 使用 `GET /api/v1/projects/{id}/sub-projects`。
- 新增子專案 API 使用目前主專案 id 呼叫 `POST /api/v1/projects/{id}/sub-projects`，不得讓表單自行指定其他主專案。
- 編輯與刪除子專案時，需確認該子專案屬於目前主專案；切換主專案後需關閉正在編輯或刪除確認中的 Dialog。
- 主專案下拉選單切換後，需更新 route、清空列表選取列、重設 pagination 至第一頁，並重新讀取子專案列表。
- 切換期間顯示 loading 狀態；新專案資料回來前不得顯示上一個主專案的子專案為目前資料。

## 專案成員管理頁

路由：`/projects/:id/members`

資料模型規則：

- 專案成員以原 spec `design-backend.md` 的 `project_members` 表為目標契約，不是 Admin 用戶管理表的前端投影。
- 專案成員畫面使用 `project_members -> ops_user` 作為顯示來源；`ops_user` 不具備角色欄位，因此前端不得提供角色篩選、角色欄位或角色下拉選單。
- `ops_user` 目前沒有真實姓名與 Email 欄位，專案成員列表不得顯示真實姓名或 Email 欄。
- 前端不得顯示 Admin 用戶管理欄位，例如 `global_role`、MFA、phone、remark 或系統帳號狀態。
- 列表只顯示目前可由 `ops_user` / `project_members` 支撐的欄位：使用者名稱、加入時間與操作。
- 後端 `0.6` 已移除專案成員與候選人 response 的 `global_role`、`is_active`；前端不得重新加入這些欄位。

```text
專案 / OPS 平台維運 / 成員

[搜尋使用者名稱] [+ 新增維運人員]

┌──────────────────────────────────────────────────────────┐
│ 使用者名稱                         加入時間          操作 │
│ alice                              2026/...          刪除 │
└──────────────────────────────────────────────────────────┘
```

API：

```text
GET    /api/v1/projects/:id/members
POST   /api/v1/projects/:id/members
DELETE /api/v1/projects/:id/members/:uid
```

新增專案維運人員：

- 新增成員不是從既有候選人挑選，而是在目前主專案下新增一筆運維人員資料。
- 後端需在同一個 transaction 內建立：
  - `ops_user`：新增運維人員主檔。
  - `project_members`：將剛建立的 `ops_user.id` 綁定到目前主專案。
- 前端送出欄位只對應 `ops_user` 目前資料表欄位：
  - `user_name`：必填，1 到 128 字。
- 前端不得讓使用者手動輸入 `ops_user.id`；ID 由後端產生。
- 前端不得讓表單自行指定 `project_id`；project id 固定使用目前主專案路由或主專案下拉選單。
- 目前後端若仍要求 `role`，需由後端 contract 調整；前端 UI 不顯示角色，也不應要求使用者理解或選擇角色。
- API 需要目前使用者具備該專案 `project_manager` 權限；403 時不得顯示假候選人。
- 不得使用 `/api/v1/admin/users` 作為 Ticket 工作區的候選人來源。
- Ticket 指派、mention、協作者候選人使用 `GET /api/v1/projects/{id}/members`，不得顯示非專案成員。
- `GET /api/v1/projects/{id}/user-options` 不應作為新增專案成員的主要流程；若後端仍保留此 API，只能供其他搜尋場景使用，不得讓新增成員 UI 退回「挑候選人」模式。
- 移除專案成員後，後端會將 `ops_user.is_active` 改為 `false`；列表重新載入後不得顯示停用成員，前端不得用本地假刪除取代後端結果。

新增成員 Dialog 設計：

- Dialog 類型：Form Dialog，寬度 `sm`，第一個可互動元素 focus 在 `使用者名稱` 欄位。
- 標題：`新增專案維運人員`
- 說明文字：顯示目前主專案名稱與代碼，讓使用者確認新增目標專案。
- 表單欄位：
  - `使用者名稱`：必填，對應 `ops_user.user_name`。
- 建立摘要：
  - 顯示即將新增到哪個主專案。
  - 顯示新增後會建立 `ops_user` 與 `project_members` 關聯。
  - 不顯示 ID 預覽，因為 ID 由後端建立後才存在。
- 操作：
  - 左側次要按鈕：`取消`
  - 右側主要按鈕：`新增維運人員`
  - 送出中按鈕文字：`新增中...`
  - 送出 payload 只包含 `user_name`；project id 由路由帶入。
- 驗證與錯誤：
  - `user_name` 空白時顯示欄位錯誤。
  - `user_name` 超過 128 字時顯示欄位錯誤。
  - 後端回 409 時顯示該專案已有相同運維人員或名稱衝突，實際訊息以後端 response 為準。
  - API 失敗時顯示 error alert，不顯示成功，不從前端假造新增結果。
- 鍵盤與無障礙：
  - Enter 在單行欄位送出表單；Escape 關閉 Dialog。
  - 必填欄位需有 `aria-required` 與錯誤描述。
  - 送出失敗時 focus 回到第一個錯誤欄位或錯誤 alert。

```text
┌──────────────────────────────────────────────┐
│ 新增專案維運人員                              │
│ 專案：OPS 平台維運（OPS）                     │
│                                              │
│ 使用者名稱 *                                  │
│ [oncall01                                  ] │
│                                              │
│ 建立內容                                     │
│ 將建立 ops_user，並加入目前主專案。           │
│                                              │
│                          [取消] [新增維運人員]│
└──────────────────────────────────────────────┘
```

主專案連動規則：

- 專案成員列表資料來源必須使用目前主專案下拉選單選取的 project id。
- 路由為 `/projects/:id/members` 時，列表 API 使用 `GET /api/v1/projects/{id}/members`。
- 新增成員 API 使用目前主專案 id 呼叫 `POST /api/v1/projects/{id}/members`，不得讓表單自行指定其他主專案。
- 移除成員 API 使用目前主專案 id 與目標 user id。
- 主專案下拉選單切換後，需更新 route、清空列表選取列、重設 pagination 至第一頁，並重新讀取成員列表。
- 切換主專案後需關閉新增成員與移除確認 Dialog，避免對上一個主專案送出操作。
- 切換期間顯示 loading 狀態；新專案資料回來前不得顯示上一個主專案的成員為目前資料。

## 資訊來源管理頁

路由：`/projects/:id/sources`

資料模型規則：

- 依使用者指定，本頁 UI 先對應 `ticket_resources`。
- `ticket_resources` 是 project scoped 資料，列表必須依目前主專案 route `:id` 讀取。
- `ticket_resources.sub_project_id` 為選填欄位；表單以下拉選單選取目前主專案下的子專案，亦可不指定子專案。
- `ticket_resources` 不使用 `details JSONB`；UI 以一般欄位管理 `code`、`name`、`resource_type`、`description`、`sub_project_id`、`is_active`、`sort_order`。
- `ticket_resources` 需新增 `is_active BOOLEAN NOT NULL DEFAULT TRUE`；新增資訊來源預設啟用。
- 刪除資訊來源採 soft delete，將 `is_active` 改為 `false`；列表、Ticket 建立頁與 Ticket 列表篩選不得顯示 `is_active=false` 的資訊來源。
- `ticket_sources` 已廢棄；前端不得使用 `ticket_sources` 作為 Ticket 建立、列表篩選或資訊來源管理資料。
- OpenAPI 已提供 `ticket_resources` 管理 API，前端需依 `design.md` 的 Ticket Resource Management Contract 串接。

```text
專案 / OPS 平台維運 / 資訊來源

[搜尋名稱 / 類型] [子專案 ▼] [類型 ▼] [+ 新增資訊來源]

┌────────────────────────────────────────────────────────────────────────┐
│ 代碼        名稱              類型       子專案      狀態  排序  更新時間 操作 │
│ tg_alert    TG Alert          alert      支付平台    啟用  1     2026/... ✏ 🗑 │
│ signal      Signal 項目群     message    API         啟用  3     2026/... ✏ 🗑 │
└────────────────────────────────────────────────────────────────────────┘
```

欄位：

| 欄位 | 來源 | 備註 |
| --- | --- | --- |
| 代碼 | `ticket_resources.code` | 必填，同一 project 內唯一，例如 `tg_alert` |
| 名稱 | `ticket_resources.name` | 必填，128 字以內 |
| 類型 | `ticket_resources.resource_type` | 例如 `alert`、`message`、`business_operation` |
| 子專案 | `sub_project_id` 關聯名稱 | 選填；未指定時顯示 `-`，不得顯示其他主專案子專案 |
| 狀態 | `is_active` | 列表預設只顯示啟用資料；若後續提供狀態篩選，停用資料也不得作為建立 Ticket 選項 |
| 說明 | `description` | 選填文字 |
| 排序 | `sort_order` | 數字越小越前面 |
| 更新時間 | `updated_at` | 依目前語系格式化 |
| 操作 | row actions | 編輯、刪除 |

新增 / 編輯 Dialog：

- Dialog 類型：Form Dialog，寬度 `md`。
- 標題：新增時 `新增資訊來源`，編輯時 `編輯資訊來源`。
- 欄位：
  - `名稱`：必填，對應 `name`。
  - `代碼`：必填，對應 `code`，同一 project 內唯一。
  - `類型`：必填，對應 `resource_type`；第一版使用下拉選單，選項包含 `alert`、`message`、`business_operation`、`server`、`database`、`network`。
  - `說明`：選填，對應 `description`。
  - `子專案`：選填，對應 `sub_project_id`，選項來自目前主專案子專案清單；可選「不指定子專案」。
  - `啟用`：Switch，對應 `is_active`；新增預設 `true`。
  - `排序`：數字輸入，對應 `sort_order`。
- 操作：
  - 新增：`儲存`
  - 編輯：`儲存`
  - 取消：關閉 Dialog，不送 API。
- 驗證：
  - 名稱不可空白。
  - 代碼不可空白。
  - 類型不可空白。
- 刪除：
  - 使用確認 Dialog。
  - 刪除送出後端 soft delete，後端將 `ticket_resources.is_active` 改為 `false`。
  - 成功後重新讀取列表；因列表預設只顯示 `is_active=true`，被停用資料不再顯示。
  - 若後端拒絕刪除，顯示後端錯誤，不從前端移除該列。

API contract：

```text
GET    /api/v1/projects/:id/ticket-resources
POST   /api/v1/projects/:id/ticket-resources
GET    /api/v1/ticket-resources/:rid
PUT    /api/v1/ticket-resources/:rid
DELETE /api/v1/ticket-resources/:rid
```

API contract 規則：

- List API 預設只回傳 `is_active=true`。
- Create API 若未提供 `is_active`，後端預設為 `true`。
- Delete API 不硬刪資料，只將 `is_active=false` 並更新 `updated_at`。
- Ticket 建立頁與 metadata options 若改用 `ticket_resources`，只能回傳啟用資料。
- `GET /api/v1/projects/:id/ticket-resources` 需具備專案 Viewer 權限。
- 新增、編輯、刪除需具備所屬專案 Manager 權限。
- request body 只送 `code`、`name`、`resource_type`、`description`、`sub_project_id`、`is_active`、`sort_order`，不得送 `project_id` 覆寫 route scope；`sub_project_id` 可為空值。

## 事件類型管理頁

路由：`/projects/:id/ticket-types`

資料模型規則：

- 本頁對應 `ticket_types`。
- `ticket_types` 目前是系統層級參照資料，不依 project scope 過濾；但入口放在 Ticket 工作區內，仍顯示目前主專案頁首脈絡。
- `is_system = TRUE` 的事件類型不可刪除。
- `is_system = TRUE` 的事件類型不可修改 `code`、`supports_escalation`、`applies_sla`、`allowed_priorities`、`default_priority`、`assignee_required` 等核心流程欄位。
- `is_system = TRUE` 的事件類型仍可修改 `name`、`description`、`is_active`、`sort_order`。
- 目前 `GET /api/v1/ticket-metadata/options` 只回傳啟用中的事件類型選項，不是完整 CRUD 管理 API；前端不得用它假裝新增、編輯或刪除成功。
- OpenAPI 已提供 `ticket_types` 管理 API，前端需依 `design.md` 的 Ticket Type Management Contract 串接。

```text
專案 / OPS 平台維運 / 事件類型

[搜尋代碼 / 名稱] [狀態 ▼] [系統內建 ▼] [+ 新增事件類型]

┌──────────────────────────────────────────────────────────────────────────────────────────────┐
│ 代碼   名稱   狀態  系統內建  升級  SLA  預設優先級  允許優先級      指派必填  排序  操作 │
│ daily  Daily  啟用  是        否    否   P4          P3, P4          否        1     查看 │
│ issue  Issue  啟用  是        是    是   P3          P1, P2, P3, P4  否        2     查看 │
└──────────────────────────────────────────────────────────────────────────────────────────────┘
```

欄位：

| 欄位 | 來源 | 備註 |
| --- | --- | --- |
| 代碼 | `ticket_types.code` | 唯一代碼，自訂類型可編輯；系統內建不可編輯 |
| 名稱 | `ticket_types.name` | 顯示名稱 |
| 狀態 | `is_active` | Chip |
| 系統內建 | `is_system` | Chip；系統內建限制刪除與核心欄位編輯 |
| 升級 | `supports_escalation` | 是否支援 Escalated 狀態 |
| SLA | `applies_sla` | 是否套用 SLA |
| 預設優先級 | `default_priority` | 必須在 allowed priorities 內 |
| 允許優先級 | `allowed_priorities` | 多選 P1 到 P4 |
| 指派必填 | `assignee_required` | 建立 Ticket 時是否必填指派人 |
| 排序 | `sort_order` | 數字越小越前面 |
| 操作 | row actions | 查看、編輯、停用、刪除 |

新增 / 編輯 Dialog：

- Dialog 類型：Form Dialog，寬度 `md`。
- 標題：新增時 `新增事件類型`，編輯時 `編輯事件類型`。
- 欄位：
  - `代碼`：必填，英文小寫、數字、底線，32 字以內；系統內建不可編輯。
  - `名稱`：必填，64 字以內。
  - `描述`：選填。
  - `啟用`：Switch。
  - `支援升級`：Switch。
  - `套用 SLA`：Switch。
  - `允許優先級`：多選 checkbox，P1、P2、P3、P4。
  - `預設優先級`：下拉選單，只能選已勾選的允許優先級。
  - `指派必填`：Switch。
  - `排序`：數字輸入。
- 驗證：
  - 代碼不可空白且格式正確。
  - 名稱不可空白。
  - 至少選一個允許優先級。
  - 預設優先級必須包含在允許優先級中。
  - 系統內建列的核心欄位 disabled，並顯示 tooltip 說明限制。
- 刪除 / 停用：
  - 系統內建事件類型不顯示刪除按鈕。
  - 非系統內建事件類型刪除前需確認。
  - 若已有 Ticket 引用，後端可拒絕刪除；前端顯示錯誤，不假刪除。
  - 停用後不得再出現在建立 Ticket 與列表篩選選項。

API contract：

```text
GET    /api/v1/ticket-types
POST   /api/v1/ticket-types
GET    /api/v1/ticket-types/:tid
PUT    /api/v1/ticket-types/:tid
DELETE /api/v1/ticket-types/:tid
```

API contract 規則：

- List API 回傳全部事件類型，包含啟用、停用與系統內建狀態。
- 事件類型管理 API 需登入，且只有 `admin` / `op_admin` 可操作。
- 新增自訂事件類型時，`is_system` 由後端固定為 `false`，前端不得送出。
- Create / Update 若未提供 `is_active`，後端預設為 `true`。
- `code` 必填，允許小寫英文字母、數字、底線與短橫，長度上限 64。
- `name` 必填，長度上限 64。
- `allowed_priorities` 至少一個值，且只能包含 `P1`、`P2`、`P3`、`P4`。
- `default_priority` 必須包含在 `allowed_priorities`。
- Delete API 採 soft delete，成功後將 `is_active=false`。
- 系統內建事件類型刪除或核心欄位更新會回 409，前端需顯示後端錯誤，不得假成功。

## i18n

- namespace 使用 `ticket`。
- 共用按鈕與狀態盡量使用 `common`。
- Data Grid 內建文案需使用 `getDataGridLocaleText(commonT, locale)`。

## API 缺口

已讀取 `opscenter-server/Docs/openapi.json`。目前 Ticket 工作區可見的 API paths 如下：

```text
GET    /api/v1/projects
GET    /api/v1/projects/{id}
GET    /api/v1/projects/{id}/sub-projects
POST   /api/v1/projects/{id}/sub-projects
GET    /api/v1/sub-projects/{sid}
PUT    /api/v1/sub-projects/{sid}
DELETE /api/v1/sub-projects/{sid}
GET    /api/v1/projects/{id}/members
GET    /api/v1/projects/{id}/user-options
POST   /api/v1/projects/{id}/members
PUT    /api/v1/projects/{id}/members/{uid}
DELETE /api/v1/projects/{id}/members/{uid}
GET    /api/v1/projects/{id}/ticket-resources
POST   /api/v1/projects/{id}/ticket-resources
GET    /api/v1/ticket-resources/{rid}
PUT    /api/v1/ticket-resources/{rid}
DELETE /api/v1/ticket-resources/{rid}

GET    /api/v1/tickets
POST   /api/v1/tickets
GET    /api/v1/projects/{id}/tickets
GET    /api/v1/tickets/{id}
PUT    /api/v1/tickets/{id}
DELETE /api/v1/tickets/{id}
POST   /api/v1/tickets/{id}/status
POST   /api/v1/tickets/{id}/assign
POST   /api/v1/tickets/{id}/collaborators
DELETE /api/v1/tickets/{id}/collaborators/{uid}
POST   /api/v1/tickets/{id}/activities
GET    /api/v1/tickets/{id}/attachments
POST   /api/v1/tickets/{id}/attachments
DELETE /api/v1/tickets/{id}/attachments/{aid}
GET    /api/v1/attachments/{id}/content
GET    /api/v1/ticket-metadata/options
GET    /api/v1/ticket-types
POST   /api/v1/ticket-types
GET    /api/v1/ticket-types/{tid}
PUT    /api/v1/ticket-types/{tid}
DELETE /api/v1/ticket-types/{tid}
```

目前可直接依 OpenAPI 建立 client 的部分：

- Ticket list path 與 query parameters 已存在，但資訊來源 filter 需由 `ticket_source_id` 收斂為 `ticket_resource_id`；其餘包含 `keyword`、`status`、`priority`、`sub_project_id`、`assignee_id`、`ticket_type_id`、`created_from`、`created_to`、`sort_by`、`sort_direction`、`page`、`page_size`、`limit`、`offset`。
- Project scoped list 使用 `GET /api/v1/projects/{id}/tickets`，可支援主專案下拉選單連動。
- Ticket 列表日期區間使用既有 `created_from` / `created_to` query parameters；前端需預設帶入當天起訖時間。
- Attachment upload 已有 `multipart/form-data` request body，`file` 為 binary required。
- Attachment content 已有 `application/octet-stream` binary response。
- Project / Sub Project / Project Member paths 已存在，可先建立前端 API client。
- Project Member 新增流程需改為建立目前專案的 `ops_user` 並同步建立 `project_members` 關聯；`GET /api/v1/projects/{id}/user-options` 不作為新增成員主流程。
- Ticket metadata options 需收斂為可取得 `ticket_types`、project scoped `ticket_resources` 與 `priorities`；建立 Ticket 頁與列表篩選不得硬編事件類型、資訊來源或 priority 選項。
- Ticket Resource 管理 paths 已存在，前端資訊來源管理頁需使用 `ticket_resources` CRUD API，不得使用 metadata options 假成功。
- Ticket Type 管理 paths 已存在，前端事件類型管理頁需使用 `ticket_types` CRUD API；停用後需重新讀取 metadata options。

仍需前端實作時注意的缺口：

- OpenAPI 已輸出 JSON requestBody 與 response DTO schema；前端型別仍需對照 `design.md` 的 Ticket API Contract，不得自行增加欄位。
- OpenAPI 未描述可用 `sort_by` enum 與狀態 enum；前端只可使用 `design.md` 已列狀態與後端已支援排序欄位，並需顯示後端錯誤。
- Attachment content 的錯誤 response 在 OpenAPI 目前繼承 `application/octet-stream` content，前端仍需依 HTTP status 處理錯誤，不可把非 2xx blob 當作圖片成功顯示。

結論：

- Ticket CRUD / Activity / Attachment / Ticket Resource / Ticket Type paths 已補齊，不再因為 paths 缺失阻擋 API client task。
- JSON schema 已補強，metadata options contract 已補齊；前端 API client 可以實作，但必須以 OpenAPI schema、`design.md` contract 和 server DTO 為準，並在發現欄位落差時新增未完成 task。
