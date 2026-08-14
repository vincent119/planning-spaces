# Frontend Task Plan

## Scope

本文件依據 `design-frontend.md` 拆解 OnCall Ticket System 的前端任務。MVP 目標是完成登入 / MFA、主版面、儀表板、Ticket 基礎操作入口、個人資料、附件元件、Admin 管理頁、全域設定、排程管理與 i18n / theme 基礎能力。

Phase 2 補強 Jira、進階日誌、更多管理操作與排程互動。Phase 3 實作報表中心、報表設計器與完整圖表匯出。

---

## 1. Frontend 專案骨架與基礎設施

- [x] 1.1 建立 React 19 + TypeScript + Vite 專案骨架
  - 確認 `react`、`react-dom`、`@types/react`、`@types/react-dom` 使用 19.x 相容版本
  - 設定 `tsconfig`、路徑 alias、Vite build
  - _Design: 技術選型_

- [x] 1.2 安裝並設定核心套件
  - React Router v6
  - TanStack Query
  - React Hook Form + Zod
  - Material UI v9、`@mui/icons-material`
  - MUI X Data Grid Community
  - i18next / react-i18next
  - ECharts / `echarts-for-react`
  - _Design: 技術選型, Material UI / MUI X 使用規範_

- [x] 1.3 建立前端目錄結構
  - `components`
  - `features/{auth,ticket,project,report,jira,dashboard,admin}`
  - `hooks`
  - `locales/{zh-TW,zh-CN,en}`
  - `theme`
  - `utils`
  - `types`
  - _Design: 元件架構_

- [x] 1.4 建立 API client 與統一錯誤處理
  - 自動帶入 JWT
  - 統一解析 `code`、`message`、`data`、`trace_id`
  - 401 / refresh token / logout 流程
  - _Design: 狀態管理_

- [x] 1.5 建立 TanStack Query Provider
  - 預設 stale time
  - 錯誤 toast
  - dashboard query 支援 30 秒刷新
  - _Design: Server State_

- [x] 1.6 建立 App Context
  - `currentUser`
  - `currentProject`
  - `locale`
  - `themeMode`
  - localStorage 持久化
  - _Design: Client State_

- [x] 1.7 建立全域 Toast 通知機制
  - MUI `Snackbar` + `Alert`
  - success / error / warning 樣式
  - 所有新增、修改、刪除、批量操作、立即執行與狀態變更成功 / 失敗都必須提示
  - 錯誤訊息優先使用後端 `message`
  - i18n 文案
  - _Design: 全域 Toast 提示規範_

---

## 2. i18n 與主題系統

- [x] 2.1 建立 i18next 初始化
  - 預設 `zh-TW`
  - 支援 `zh-TW`、`zh-CN`、`en`
  - localStorage / navigator 語系偵測
  - _Design: 國際化_

- [x] 2.2 建立翻譯檔 namespace
  - `common.json`
  - `auth.json`
  - `ticket.json`
  - `project.json`
  - `report.json`
  - `admin.json`
  - _Design: 目錄結構, 翻譯 key 命名規則_

- [x] 2.3 建立四套 MUI theme
  - `light`
  - `light-glass`
  - `dark`
  - `dark-glass`
  - 初始值跟隨系統，預設對應 `dark-glass`
  - _Design: 主題設計_

- [x] 2.4 建立毛玻璃樣式工具
  - `glassEffect`
  - 卡片 / AppBar / Menu 共用樣式
  - 僅透過 `sx` 或 `styled()` 套用
  - _Design: 毛玻璃效果規範_

- [x] 2.5 整合 ThemeProvider
  - theme mode 存入 localStorage
  - Avatar 選單內切換主題
  - 即時套用不需儲存
  - _Design: ThemeProvider 整合, 切換按鈕_

- [x] 2.6 建立 Table / Data Grid i18n 共用 localeText
  - 建立 MUI X Data Grid `localeText` 共用 helper
  - 覆蓋排序、篩選、欄位管理、隱藏欄位、分頁、空資料、載入、錯誤與工具列文字
  - 所有使用 Data Grid 的頁面都需套用，語系切換後即時更新
  - 自行組合的 Table / Pagination / Menu 也不得保留英文內建文字
  - _Design: Table / Grid 內建文案_

- [x] 2.7 建立可控日期欄位語系
  - 目前語系需同步到 `<html lang>`
  - 日期欄位需使用共用 `DateField`，月曆選單文案、月份與星期需跟隨 i18n
  - _Design: 國際化_

---

## 3. Routing、Layout 與權限導覽

- [x] 3.1 建立 React Router 路由表
  - `/login`
  - `/dashboard`
  - `/projects`
  - `/projects/:id`
  - `/projects/:id/tickets`
  - `/projects/:id/tickets/new`
  - `/tickets/:id`
  - `/projects/:id/reports`
  - `/projects/:id/jira`
  - `/admin/*`
  - `/settings/profile`
  - `/settings/mfa`
  - `/auth/mfa/setup`
  - _Design: 頁面結構_

- [x] 3.2 建立 Route Guard
  - 未登入導向 `/login`
  - 強制 MFA setup 導向 `/auth/mfa/setup`
  - Admin 頁面檢查角色
  - 保留 `redirectTo`
  - _Design: 登入流程狀態機, 強制 MFA 裝置新增頁_

- [x] 3.3 建立主 Layout
  - AppBar
  - Sidebar
  - Content 區域
  - responsive 行為
  - _Design: 主頁面佈局設計_

- [x] 3.4 實作 AppBar
  - Sidebar 收合按鈕
  - Logo / 當前專案名稱
  - 通知 Badge
  - Avatar + 使用者選單
  - _Design: AppBar_

- [x] 3.5 實作 Avatar 下拉選單
  - 個人資料入口
  - 設定子面板
  - 主題選擇
  - 語系選擇
  - 登出
  - _Design: AppBar 設定子面板_

- [x] 3.6 實作 Sidebar
  - 展開寬度 `240px`
  - 收合寬度 `64px`
  - Admin 群組展開 / 收合
  - active item highlight
  - 收合狀態 Tooltip
  - _Design: Sidebar_

---

## 4. Auth、登入與 MFA

- [x] 4.1 建立登入頁背景與毛玻璃卡片
  - `public/bg/login.jpg`
  - 背景遮罩
  - 中央登入卡片
  - 右下角語系切換
  - _Design: 登入頁視覺規格_

- [x] 4.2 實作帳密登入表單
  - 帳號或電子郵件
  - 密碼顯示切換
  - 記住我
  - 忘記密碼連結
  - React Hook Form + Zod
  - _Design: 表單欄位_

- [x] 4.3 實作 OIDC 入口
  - 呼叫 `GET /api/v1/auth/config`
  - 僅 `oidc.enabled = true` 顯示
  - outlined button 樣式
  - _Design: OIDC 登入按鈕_

- [x] 4.4 實作登入狀態機
  - `idle`
  - `submitting`
  - `mfa_required`
  - `mfa_setup_required`
  - `success`
  - `error`
  - _Design: 登入流程狀態機_

- [x] 4.5 實作 MFA 驗證畫面
  - in-place 切換，不跳頁
  - TOTP 6 位數輸入
  - 備用碼模式
  - 返回帳密畫面清除 temp token
  - _Design: MFA 驗證步驟_

- [x] 4.6 實作帳號遮罩工具
  - email 顯示 `ad***@example.com`
  - username 顯示前 2 字元 + `***`
  - _Design: 帳號遮罩規則_

- [x] 4.7 實作強制 MFA 裝置新增頁
  - `/auth/mfa/setup`
  - 不允許略過
  - 返回登入頁清除 temp token
  - 使用登入頁相同背景
  - _Design: 強制 MFA 裝置新增頁_

- [x] 4.8 實作強制 MFA Stepper
  - 步驟 1：掃描 QR Code
  - 步驟 2：驗證設定
  - 步驟 3：保存備用碼
  - _Design: 步驟設計_

- [x] 4.9 實作 MFA setup API 流程
  - `POST /api/v1/auth/mfa/setup`
  - `POST /api/v1/auth/mfa/verify`
  - 成功後儲存正式 token
  - 導回 `redirectTo`
  - _Design: API 流程_

- [x] 4.10 實作登入頁
  - 右上角顯示成功 / 錯誤 Toast
  - 語系切換成功提示
  - 登入失敗 / 密碼錯誤提示
  - MFA 驗證失敗提示
  - MFA setup 初始化 / 驗證失敗提示
  - OIDC 啟動 / callback 失敗提示
  - 自動關閉、手動關閉與底部進度條
  - 手機版維持上方顯示並避免遮擋表單
  - _Design: 登入頁 Toast 提示_

---

## 5. Dashboard 與統計卡片

- [x] 5.1 建立 `StatCard` 共用元件
  - 漸層背景
  - 圓形 icon 背景
  - 大數字
  - trend 顯示
  - _Design: 儀表板統計卡片設計_

- [x] 5.2 實作 Dashboard 四張卡片
  - Open Tickets
  - 今日新建
  - 待處理 P1 / P2
  - SLA 違反
  - _Design: OnCall Ticket System 卡片定義_

- [x] 5.3 實作 Dashboard 圖表區
  - Ticket 趨勢折線圖
  - 優先級分佈 Donut Chart
  - 最近 10 筆 Ticket
  - _Design: 儀表板頁面佈局_

- [x] 5.4 設定 Dashboard 自動刷新
  - `refetchInterval: 30_000`
  - loading / error state
  - _Design: Server State, 儀表板_

- [x] 5.5 實作 Dashboard 自適應縮放與版面挪移
  - 維持全域與專案儀表板既有響應式換欄，不使用整頁 CSS `scale`
  - 新增明確的版面編輯模式、拖曳把手、預設尺寸切換、取消與重設版面
  - 桌面版支援面板順序與允許尺寸調整；手機版維持自動單欄或既有響應式排列
  - 提供鍵盤移動替代操作、`aria-label`、tooltip 與可感知的移動狀態
  - 以含 schema version 的 `localStorage` 保存，依使用者、全域／專案範圍與專案 ID 隔離
  - 驗證 ECharts resize、Data Grid 欄寬、載入／錯誤狀態及 30 秒刷新不受影響
  - 實作前評估拖曳套件的 React 19 相容性、鍵盤支援與 bundle 影響；不得假設新套件已安裝
  - 假設：第一階段不支援跨瀏覽器或跨裝置同步；若需同步，另立後端 API 與資料契約任務
  - _Requirements: 13.5、13.6、13.7_
  - _Design: 儀表板自適應與版面編輯_
  - 完成：全域與專案儀表板已共用 60 欄桌面版面，支援原生拖曳、前移／後移、方向鍵移動、允許尺寸循環、取消、儲存與確認重設；版面依使用者與專案範圍以 schema version 1 儲存於 `localStorage`
  - 驗證：`npm run typecheck`、`npm run build`、三語系 JSON 解析與 `git diff --check` 通過；目前專案未提供前端 unit test 或 lint script

---

## 6. Ticket 共用元件與附件

- [x] 6.1 建立 Ticket 基礎元件
  - `TicketCard`
  - `StatusBadge`
  - `PriorityBadge`
  - `SLACountdown`
  - _Design: 元件架構_

- [x] 6.2 建立 MarkdownEditor
  - 留言輸入
  - Markdown 預覽
  - 錯誤提示
  - 完成：`components/MarkdownEditor.tsx` 已提供編輯 / 預覽切換、錯誤提示、禁用狀態與安全預覽 renderer，並用於建立 Ticket 描述與詳情頁留言
  - 來源：`2026-06-15_15-35_Ticket/task-frontend.md` 的 `5.2`、`7.3`、`7.4`
  - 驗證：來源 task 已通過 `npm run typecheck`
  - _Design: 元件架構_

- [x] 6.3 實作 AttachmentUpload
  - 點擊選擇檔案
  - 拖曳上傳
  - 貼上截圖
  - content-type 白名單
  - 單檔小於等於 10MB
  - 數量小於等於 20
  - 完成：附件上傳能力已整合於建立 Ticket 頁、Ticket 詳情附件區與留言圖片區，支援點擊、拖曳、貼上、白名單、10MB、20 個限制
  - 備註：目前是頁面內整合，未抽成獨立 `AttachmentUpload` 元件；若需要共用元件化，另開 refactor task
  - 來源：`2026-06-15_15-35_Ticket/task-frontend.md` 的 `8.2`、`8.3`、`8.6`，以及本文件 `6.6`
  - 驗證：來源 task 已通過 `npm run typecheck`；本文件 `6.6` 已通過 `npm run typecheck`、`npm run build`
  - _Design: AttachmentUpload_

- [x] 6.4 實作附件 metadata 列表
  - 不使用 `storage_key`
  - 顯示檔名、格式、大小
  - loading / error state
  - 完成：Ticket 詳情頁已串 `listTicketAttachments` 與 detail response attachments，顯示檔名、content type、大小、建立時間與 loading / error state，前端型別不暴露 `storage_key`
  - 來源：`2026-06-15_15-35_Ticket/task-frontend.md` 的 `6.3`、`8.1`、`8.2`、`8.3`
  - 驗證：來源 task 已通過 `npm run typecheck`
  - _Design: 附件顯示_

- [x] 6.5 實作附件內容預覽與下載
  - `GET /api/v1/attachments/:id/content`
  - authenticated blob fetch
  - object URL lifecycle 清理
  - S3 不由 Browser 直連
  - 完成：附件內容透過 `downloadTicketAttachmentContent` 以 authenticated blob fetch 載入，縮圖與預覽 dialog 均使用 object URL 並清理；不直接連 S3 / Local 路徑
  - 來源：`2026-06-15_15-35_Ticket/task-frontend.md` 的 `8.1`、`8.2`、`8.3`、`8.5`
  - 驗證：來源 task 已通過 `npm run typecheck`
  - _Design: 附件顯示_

- [x] 6.6 補齊建立 Ticket 頁附件暫存與建立後上傳
  - 建立 Ticket 前可點擊選圖、拖曳圖片、貼上截圖
  - 貼上截圖需在表單頁全域生效，不能只限附件區取得焦點
  - 建立成功後使用新 Ticket ID 逐一上傳 pending attachments
  - 上傳失敗時保留已建立 Ticket，顯示附件上傳失敗提示並導向詳情頁
  - 驗證：`npm run typecheck`、`npm run build` 通過
  - _Requirements: 3.10; Design: AttachmentUpload_

- [x] 6.7 修正 Ticket 列表編輯操作
  - 列表操作欄的編輯按鈕需可進入既有 Ticket 編輯流程
  - 點擊列表編輯按鈕後導向 Ticket 詳情頁並自動開啟編輯對話框
  - 完成：列表編輯按鈕導向 `/tickets/:id?edit=1`；詳情頁讀取 `edit=1` 後自動開啟既有編輯對話框並移除 query
  - 驗證：`npm run typecheck`、`npm run build` 通過
  - _Requirements: 1.6, 3.6; Design: Ticket 列表_

- [x] 6.8 Ticket 列表複製列資訊操作
  - 操作欄在查看按鈕前新增 `ContentCopyIcon`
  - 點擊後複製該列溝通用文字欄位
  - 複製內容不得包含 Ticket ID、附件 / 貼圖、storage key 或詳情連結
  - 複製欄位包含：標題、狀態、優先級、事件類型、子專案、資訊來源、班別、指派人員、更新時間
  - 成功與失敗需顯示 toast，並補 `ticket.json` 三語系
  - 完成：Ticket 列表操作欄已新增複製按鈕，使用 Clipboard API 與舊式 textarea fallback，並排除 ID、附件 / 貼圖、storage key 與詳情連結
  - 驗證：`npm run typecheck`、`npm run build` 通過
  - _Requirements: 3.16; Design: Ticket 列表_

- [x] 6.9 修正 Ticket 列表下一頁跳回第一頁
  - Status: Complete
  - Depends: Ticket 列表既有伺服器分頁
  - Context: 分頁 query key 改變後，新頁資料回傳前 `ticketsQuery.data` 暫時為空，導致 Data Grid 的 `rowCount` 變成 `0` 並將受控頁碼重設為第一頁
  - Boundary: Allowed Changes 為 Ticket 列表查詢設定、需求與前端設計及本任務紀錄；Forbidden 為 Ticket API 契約、後端分頁、其他列表頁與 Data Grid 共用元件
  - 使用 TanStack Query `keepPreviousData` 保留上一筆成功頁面資料與總筆數
  - 連續切換第 2、3、4 頁時，API `page` 需依序為 `2`、`3`、`4`，不得被載入中狀態重設為 `1`
  - 篩選、搜尋及日期條件變更仍需重設為第一頁
  - Verify: `npm run typecheck`、`npm run build`、`git diff --check`
  - Implementation Notes: Ticket 列表 query 已加入 `placeholderData: keepPreviousData`，切頁載入期間沿用上一筆成功資料的 `rows` 與 `total`，避免 Data Grid 因 `rowCount` 暫時歸零而重設頁碼。`npm run typecheck` 與 `npm run build` 均通過。
  - _Requirements: 3.18; Design: Ticket 列表_

- [x] 6.10 Ticket 列表新增開單人員欄位
  - Status: Complete
  - 在「班別」與「指派人員」之間顯示「開單人員」
  - 沿用 Ticket 列表既有 `creator_full_name`、`creator_username` 與 `created_by`，不變更後端契約
  - 顯示優先順序為全名、帳號、建立者 ID，長文字提供 tooltip
  - Verify: `npm run typecheck`、`npm run build`、`git diff --check`
  - _Requirements: 3.19; Design: Ticket 列表_

- [x] 6.11 Ticket 列表新增開單人員篩選
  - Status: Complete
  - 搜尋區新增「開單人員」下拉選單，候選資料使用 Ticket metadata 的實際開單人去重清單
  - 前端傳送 `creator_id`，後端依 `tickets.created_by` 執行伺服器端篩選
  - 條件變更時重設第一頁，並納入 query key 與有效篩選判斷
  - 補後端 repository 與 delivery 參數測試，並更新 OpenAPI
  - Verify: `go test ./internal/ticket`、`npm run typecheck`、`npm run build`、`git diff --check`
  - _Requirements: 3.20; Design: Ticket 列表_

- [x] 6.12 Ticket 列表新增外部單號欄位
  - Status: Complete
  - 在「指派人員」與「更新時間」之間顯示「外部單號」
  - 沿用 Ticket 列表 API 既有 `external_ref`，不變更後端契約與資料庫
  - 空值使用列表既有空值符號，長文字單行省略並提供 tooltip
  - 不在本任務新增外部單號搜尋、排序或複製列內容
  - Verify: `npm run typecheck`、`npm run build`、`git diff --check`
  - Implementation Notes: 已在指派人員與更新時間之間加入 `external_ref` 欄位，沿用既有三語系 `field.external_ref`、空值顯示與 tooltip；欄位停用 Data Grid 排序及篩選，未修改後端契約與資料庫
  - _Requirements: 3.21; Design: Ticket 列表_

---

## 7. 個人資料與 MFA 管理

- [x] 7.1 建立 `/settings/profile` 頁面
  - 頁面標題
  - Tabs：基本資料 / 修改密碼 / MFA 設定
  - hash 或 state 控制 tab
  - _Design: 個人資料頁設計_

- [x] 7.2 實作 Avatar 區塊
  - 中文取前兩字
  - 英文取前兩字大寫
  - 顯示 username 與角色名稱
  - _Design: Avatar 區塊_

- [x] 7.3 實作基本資料表單
  - `full_name`
  - `email`
  - `phone`
  - API request / response 必須包含 `phone`，對應後端 `users.phone`
  - 儲存時呼叫個人資料更新 API，不得依賴 Admin 使用者更新 API
  - username / role 僅顯示不編輯
  - Zod 驗證
  - _Design: 基本資料表單_

- [x] 7.4 實作修改密碼 Tab
  - 目前密碼
  - 新密碼
  - 確認新密碼
  - 密碼顯示切換
  - 呼叫 `GET /api/v1/auth/password-policy` 取得 `min_length`，不得直接讀取 Admin Global Settings API
  - 新密碼最小長度依 `min_length` 顯示與驗證；policy API 失敗、缺值或無效時預設 4
  - 呼叫 `PUT /api/v1/auth/password`
  - 後端必須驗證目前密碼正確後才更新，不得沿用 Admin 使用者更新 API
  - 密碼更新成功顯示成功 Toast，並清空三個密碼欄位
  - 目前密碼錯誤、驗證失敗或 API 失敗顯示錯誤 Toast
  - 不處理任何前端可解密的混淆或加密設定 payload；前端只使用最小揭露的 `min_length`
  - _Design: 修改密碼 Tab_

- [x] 7.5 實作 MFA 設定 Tab
  - 2FA 狀態卡片
  - 已綁定裝置列表
  - 已綁定裝置列表只顯示 `is_verified = true` 的裝置，未完成 setup 的待驗證裝置不得顯示
  - 主要裝置 Chip
  - 刪除確認 Dialog
  - _Design: MFA 設定 Tab_

- [x] 7.6 實作新增 MFA 裝置 Dialog
  - 使用者可自行輸入 MFA 裝置名稱
  - QR Code 步驟
  - 手動密鑰複製
  - TOTP 驗證
  - 成功後刷新列表
  - 使用者關閉 Dialog 或未完成驗證時，不得在前端把待驗證裝置視為已綁定；逾時待驗證裝置由後端自動清理
  - _Design: 新增裝置 Dialog_

---

## 8. Admin 使用者與角色管理

- [x] 8.1 建立 `/admin/users` 用戶管理頁
  - Breadcrumbs
  - 搜尋列
  - 新增按鈕
  - Data Grid
  - 分頁
  - _Design: 用戶管理頁面結構_

- [x] 8.2 實作用戶列表欄位
  - username
  - full_name
  - email
  - phone，對應 `users.phone`
  - is_active Chip
  - MFA 狀態 Chip：`has_mfa = true` 顯示已啟用，否則顯示未啟用
  - remark
  - 操作按鈕
  - _Design: 用戶管理欄位定義_

- [x] 8.3 實作用戶新增 / 編輯 Dialog
  - username
  - full_name
  - email
  - phone，對應 `users.phone`
  - password
  - global_role
  - is_active
  - remark
  - _Design: 用戶新增 / 編輯 Dialog_

- [x] 8.4 實作用戶刪除確認
  - 自身帳號不可刪
  - 最後一個 admin 不可刪
  - 禁用按鈕 Tooltip
  - _Design: 刪除確認 Dialog_

- [x] 8.5 建立 `/admin/roles` 角色管理頁
  - Breadcrumbs
  - 搜尋列
  - 新增按鈕
  - Data Grid
  - 分頁
  - _Design: 角色管理頁面結構_

- [x] 8.6 實作角色列表欄位
  - name
  - code
  - is_active Chip
  - description
  - 操作按鈕
  - 內建角色不可刪
  - _Design: 角色欄位定義_

- [x] 8.7 實作角色新增 / 編輯 Dialog
  - name
  - code
  - description
  - is_active
  - code 建立後唯讀
  - Zod 驗證英文小寫與底線
  - _Design: 角色新增 / 編輯 Dialog_

---

## 8A. Admin 選單管理

- [x] 8A.1 建立 `/admin/menus` 選單管理頁
  - Breadcrumbs：系統管理 / 選單管理
  - 上方篩選列
  - 左側選單樹區
  - 右側節點詳情區
  - 下方子項 Data Grid
  - 空狀態與 403 狀態
  - _Design: 選單管理頁設計, 頁面結構, 權限與空狀態_

- [x] 8A.2 實作選單管理 API client
  - 串接 `GET /api/v1/admin/forms`
  - 串接 `GET /api/v1/admin/forms/:id`
  - 串接 `POST /api/v1/admin/forms`
  - 串接 `PUT /api/v1/admin/forms/:id`
  - 串接 `DELETE /api/v1/admin/forms/:id`
  - Query key 使用 `['admin', 'forms']`
  - 失敗時顯示後端錯誤訊息，不得使用假資料或偽造成功
  - _Design: API 契約_

- [x] 8A.3 實作選單樹列表
  - 使用現有 MUI `List` + `Collapse`，不新增 `@mui/x-tree-view`
  - 支援展開 / 收合
  - 支援選取節點
  - 節點列固定高度 `44px`
  - 選中狀態使用 Admin 區青藍色半透明背景
  - 顯示節點名稱、類型與啟用 / 停用狀態
  - _Design: 版面規格_

- [x] 8A.4 實作篩選列
  - 搜尋 `name`、`form_key`、`full_path`
  - 節點類型 Select：全部 / root / category / form
  - 狀態 Select：全部 / 啟用 / 停用
  - Enter 或 debounce 300ms 套用
  - 若後端未提供搜尋參數，僅對已載入樹資料做本地過濾
  - _Design: 篩選列_

- [x] 8A.5 實作節點詳情區
  - 顯示 `id`
  - 顯示 `parent_id`
  - 顯示 `node_type`
  - 顯示 `form_key`
  - 顯示 `full_path`
  - 顯示 `name`
  - 顯示 `description`
  - 顯示 `sort_order`
  - 顯示 `is_active`
  - 顯示 `created_at`、`updated_at`
  - 不使用巢狀卡片
  - _Design: 選單樹節點欄位, 版面規格_

- [x] 8A.6 實作子項 Data Grid
  - 顯示目前選取節點的直接子節點
  - 欄位：名稱、路徑、類型、排序、狀態、更新時間、操作
  - 使用共用 `getDataGridLocaleText`
  - 欄位選單、排序、篩選、分頁、空資料與載入文字跟隨 i18n
  - 操作欄提供編輯與刪除
  - _Design: 頁面結構, i18n 規範_

- [x] 8A.7 實作新增 / 編輯節點 Dialog
  - 新增根節點
  - 新增子節點
  - 編輯節點
  - 欄位：parent_id、node_type、form_key、name、description、sort_order、is_active
  - `form_key` 使用 Zod 驗證 `/^[a-z][a-z0-9_-]*$/`
  - 顯示「完整路徑將由後端依父節點計算」
  - Dialog / Select 選單不得被毛玻璃背景影響
  - _Design: 新增 / 編輯 Dialog_

- [x] 8A.8 實作節點啟用 / 停用流程
  - 在節點詳情區提供啟用 / 停用按鈕
  - 使用 `PUT /api/v1/admin/forms/:id` 更新 `is_active`
  - 成功後 invalidate `['admin', 'forms']`
  - 成功 / 失敗都顯示 Toast
  - API 失敗不得先行改 UI 成功狀態
  - _Design: 節點操作, 全域 Toast 提示規範_

- [x] 8A.9 實作刪除節點流程
  - 刪除確認 Dialog
  - 無子節點才啟用刪除按鈕
  - API 回傳 409 時顯示後端訊息
  - 成功後 invalidate `['admin', 'forms']`
  - 成功 / 失敗都顯示 Toast
  - _Design: 刪除確認 Dialog, 節點操作_

- [x] 8A.10 補齊選單管理 i18n
  - namespace 使用 `admin`
  - key 前綴使用 `menus.*`
  - 補齊繁中 / 簡中 / 英文
  - 欄位標題、節點類型、狀態、Dialog、Toast、錯誤、刪除確認、空狀態都需翻譯
  - Data Grid 內建文字需跟隨語系切換
  - _Design: i18n 規範, Table / Grid 內建文案_

- [x] 8A.11 修正新增節點後立即刪除的目標解析
  - 刪除 Dialog 只保存節點 ID，不保存可能過期的樹節點物件
  - 確認刪除前需從最新 `['admin', 'forms']` query data 重新解析節點
  - 新增節點 response 若缺少有效 id，不得選取空 id；需重新整理選單樹
  - 刪除前需擋下空白 id，避免送出 `DELETE /api/v1/admin/forms/`
  - 若最新資料找不到節點，不得送出刪除 API；需重新整理選單樹並顯示錯誤
  - 避免新增節點後 query invalidate 空窗期送出錯誤 ID，造成後端回 `form node not found`
  - _Design: 刪除確認 Dialog, API 失敗不得先行改 UI 成功狀態_

---

## 8B. Admin 權限群組與選單權限設定

> 此功能放在角色管理區下，但資料來源不是 `roles` 表。前端需串接後端 `groups`、`group_members`、`group_form_permissions` 與 `forms` API；不得自行假造 `/api/v1/admin/roles/:id/menus`，也不得用 `role.code` 硬編碼選單權限。

- [x] 8B.1 建立角色管理 Tabs 與權限分頁入口
  - 在 `/admin/roles` 增加 Tabs：角色清單 / 選單權限
  - 支援 `/admin/roles?tab=permissions` 直接開啟選單權限分頁
  - Breadcrumbs：系統管理 / 角色管理 / 選單權限
  - 保留既有角色清單功能，不改已完成角色 CRUD 行為
  - 非 `global_role = admin` 顯示無權限狀態，不顯示假資料
  - _Design: 權限群組與選單權限設定, 頁面結構, 權限與空狀態_

- [x] 8B.2 實作權限群組 API client
  - 串接 `GET /api/v1/admin/groups`
  - 串接 `POST /api/v1/admin/groups`
  - 串接 `GET /api/v1/admin/groups/:id`
  - 串接 `PUT /api/v1/admin/groups/:id`
  - 串接 `DELETE /api/v1/admin/groups/:id`
  - Query key 使用 `['admin', 'groups']`
  - 失敗時顯示後端錯誤訊息，不得使用假資料或偽造成功
  - _Design: API 契約, 資訊架構_

- [x] 8B.3 實作權限群組列表與篩選列
  - 搜尋 `name`、`code`
  - 狀態 Select：全部 / 啟用 / 停用
  - 顯示群組名稱、代碼、狀態、成員數、權限數、更新時間
  - 支援選取目前群組
  - 空狀態顯示「尚未建立權限群組」並提供新增按鈕
  - _Design: 頁面結構, 權限與空狀態_

- [x] 8B.4 實作權限群組詳情區
  - 顯示 `id`
  - 顯示 `code`
  - 顯示 `name`
  - 顯示 `description`
  - 顯示 `is_active`
  - 顯示 `member_count`
  - 顯示 `permission_count`
  - 顯示 `created_at`、`updated_at`
  - 提供編輯群組、管理成員、啟用 / 停用、刪除操作
  - _Design: 群組詳情, 頁面結構_

- [x] 8B.5 實作新增 / 編輯權限群組 Dialog
  - 欄位：name、code、description、is_active
  - `code` 使用 Zod 驗證小寫英文、數字與底線
  - code 建立後不可修改
  - Dialog / Select 選單不得被毛玻璃背景影響
  - 成功後 invalidate `['admin', 'groups']`
  - 成功 / 失敗都顯示 Toast，且不得先行顯示成功
  - _Design: 新增 / 編輯權限群組 Dialog, Toast 提示規範_

- [x] 8B.6 實作權限群組啟用 / 停用與刪除流程
  - 啟用 / 停用需呼叫 `PUT /api/v1/admin/groups/:id`
  - 刪除前顯示確認 Dialog
  - 後端拒絕刪除時顯示後端錯誤訊息
  - 成功後 invalidate `['admin', 'groups']`
  - API 失敗不得先行改 UI 成功狀態
  - _Design: 群組詳情, 刪除確認 Dialog_

- [x] 8B.7 實作群組成員 API client 與管理 Dialog
  - 串接 `GET /api/v1/admin/groups/:id/members`
  - 串接 `POST /api/v1/admin/groups/:id/members`
  - 串接 `DELETE /api/v1/admin/groups/:id/members/:uid`
  - 成員搜尋需串真實使用者 API，不得使用本地假清單
  - 新增成員成功後 invalidate `['admin', 'groups']` 與 `['admin', 'groups', id, 'members']`
  - 移除成員需二次確認
  - _Design: 管理成員 Dialog, API 契約_

- [x] 8B.8 實作群組選單 / 表單權限 API client
  - 串接 `GET /api/v1/admin/groups/:id/permissions`
  - 串接 `POST /api/v1/admin/groups/:id/permissions`
  - 串接 `DELETE /api/v1/admin/groups/:id/permissions/:pid`
  - 串接 `GET /api/v1/admin/forms` 作為選單樹節點來源
  - 型別需包含 `source`：direct / inherited / override
  - 若後端未回傳有效權限來源，前端不得自行推測繼承結果
  - _Design: API 契約, 權限矩陣欄位_

- [x] 8B.9 實作選單 / 表單權限矩陣
  - 顯示目前群組對選單樹節點的權限
  - 欄位：節點名稱、路徑、來源、讀取、新增、修改、刪除、子節點、覆寫、操作
  - `source` 顯示直接授權 / 繼承授權 / 覆寫授權
  - `inherited` 權限列只可顯示來源，不可直接刪除
  - 使用 Data Grid 時需套用共用 `getDataGridLocaleText`
  - _Design: 權限矩陣欄位, Table / Grid 內建文案_

- [x] 8B.10 實作編輯節點權限 Dialog
  - 顯示群組、節點、路徑、目前來源
  - 權限 Checkbox：read、create、update、delete
  - 設定 `inherit_children`
  - 設定 `override_parent`
  - 至少需勾選一個操作層級；完全取消權限時使用刪除直接授權流程
  - 儲存前顯示異動摘要
  - 儲存成功後 invalidate `['admin', 'groups', id, 'permissions']`
  - _Design: 編輯節點權限 Dialog_

- [x] 8B.11 實作刪除直接授權流程
  - 只允許刪除 `source = direct` 或 `source = override` 的直接授權
  - `source = inherited` 的列刪除按鈕需禁用並顯示 Tooltip 說明
  - 刪除前顯示確認 Dialog
  - 成功後重新讀取群組權限與有效來源
  - API 失敗不得先行改 UI 成功狀態
  - _Design: 權限矩陣欄位, 刪除確認 Dialog_

- [x] 8B.12 補齊權限群組與選單權限 i18n
  - namespace 使用 `admin`
  - key 前綴使用 `permission_groups.*`
  - 補齊繁中 / 簡中 / 英文
  - 群組欄位、成員 Dialog、權限矩陣欄位、來源狀態、Dialog、Toast、錯誤、刪除確認、空狀態都需翻譯
  - Data Grid 內建文字需跟隨語系切換
  - _Design: i18n 規範, Table / Grid 內建文案_

---

## 8C. Admin 選單管理資訊架構調整

> 新需求：選單 / 表單權限設定需移到選單管理之下。8B 已完成的是舊入口 `/admin/roles?tab=permissions`，不得回頭修改已完成 task 狀態；此搬移作為新的未完成工作排程。
> 後續對齊：`.kiro/specs/2026-06-18_17-45_Menu` 已將目標資訊架構調整為 `/admin/groups` 群組管理；本 8C 區塊保留歷史完成紀錄，不代表最新目標架構。

- [x] 8C.1 將選單權限分頁從角色管理搬移到選單管理
  - `/admin/roles` 回復為角色清單 / 角色 CRUD，不再顯示選單權限分頁
  - `/admin/menus` 增加 Tabs：選單節點 / 選單權限
  - 支援 `/admin/menus?tab=permissions` 直接開啟選單權限分頁
  - Breadcrumbs：系統管理 / 選單管理 / 選單權限
  - 搬移既有 8B 權限群組、成員管理、權限矩陣、編輯權限與刪除直接授權流程
  - Query key、API client 與 i18n 沿用 `groups`、`group_members`、`group_form_permissions`、`permission_groups.*`
  - 移除 `/admin/roles?tab=permissions` 舊入口，避免角色管理與選單權限資訊架構混淆
  - _Design: 權限群組與選單權限設定, 選單管理頁設計, 資訊架構調整_

- [x] 8C.2 修正選單權限群組列表 response adapter
  - 後端 `GET /api/v1/admin/groups` 目前回傳陣列，前端需正規化為 `{ items, total }`
  - 後端 `GET /api/v1/admin/groups/:id/members` 目前回傳陣列，前端需正規化為 `{ items, total }`
  - 不得因 response shape 不一致顯示「尚未建立權限群組」
  - _Design: API 契約, 權限群組與選單權限設定_

- [x] 8C.3 修正新增權限群組 payload 與權限來源顯示
  - 新增權限群組只送 `code`、`name`、`description`，不得把 `is_active` 送入建立 API
  - 編輯權限群組只送 `name`、`description`；啟用 / 停用才送 `is_active`
  - `source` 欄位需直接使用後端回傳值顯示「直接授權 / 覆寫授權 / 繼承授權」
  - 後端已回傳 `source` 時，權限矩陣不得再顯示「後端未回傳」
  - _Design: API 契約, 權限群組與選單權限設定, 權限矩陣欄位_

- [x] 8C.4 修正權限群組刪除後 permissions query 空資料錯誤
  - `GET /api/v1/admin/groups/:id/permissions` 若後端因空陣列省略 `data`，前端需正規化為 `[]`
  - 刪除權限群組成功後需取消並移除該群組的 permissions query cache
  - 刪除目前選取群組後需清空 selected group，避免繼續讀取已刪除群組的權限矩陣
  - 不得顯示 React Query `data is undefined` 內部錯誤 Toast
  - _Design: API 契約, 權限群組與選單權限設定_

---

## 9. Admin 排程管理

- [x] 9.1 建立 `/admin/schedulers` 頁面
  - Breadcrumbs
  - Tabs：任務列表 / 執行記錄
  - _Design: 排程管理頁設計_

- [x] 9.2 實作任務列表 Tab
  - 搜尋任務名稱
  - 新增任務
  - 任務名稱
  - task key
  - cron
  - 啟用狀態
  - 詳情 / 編輯 / 立即執行 / 刪除
  - _Design: Tab 1 任務列表_

- [x] 9.3 實作任務新增 / 編輯 Dialog
  - 任務名稱
  - Cron 表達式
  - 描述
  - Cron Zod 驗證
  - _Design: 新增 / 編輯 Dialog_

- [x] 9.4 實作任務詳情 Dialog
  - key-value 列表
  - 日期格式化
  - 狀態顯示
  - _Design: 任務詳情 Dialog_

- [x] 9.5 實作立即執行與刪除流程
  - 成功 / 失敗 Snackbar
  - 刪除二次確認
  - 執行後刷新記錄
  - _Design: 任務列表欄位_

- [x] 9.6 實作執行記錄 Tab
  - 搜尋任務名稱
  - 搜尋 task key
  - 刷新
  - 批量刪除
  - 清空
  - Data Grid 多選
  - _Design: Tab 2 執行記錄_

- [x] 9.7 實作執行記錄詳情 Dialog
  - 任務名稱
  - task key
  - 狀態
  - 耗時
  - 執行時間
  - pre 格式化訊息
  - _Design: 執行記錄詳情 Dialog_

- [x] 9.8 實作排程任務新增 / 編輯 / 啟停的真實後端串接
  - 新增與編輯任務需呼叫後端 API，成功回應後才顯示成功 Toast
  - 操作欄提供啟用 / 停止按鈕，需呼叫後端狀態 API
  - 成功 / 失敗都必須 Toast，且不得依賴缺失的 response body 造成前端例外
  - 若後端 route 未載入、未登入、或 API 失敗，不得偽造成功狀態
  - _Design: 任務列表欄位, 狀態變更 Toast_

---

## 10. Admin 日誌查詢

- [x] 10.1 建立 `/admin/logs` 頁面
  - Breadcrumbs
  - Tabs：Ticket 活動 / 登入日誌
  - 匯出按鈕
  - _Design: 日誌查詢頁面結構_

- [x] 10.2 實作搜尋列
  - username
  - module
  - ip
  - method
  - status_code
  - result
  - Enter 觸發搜尋
  - _Design: 搜尋列_

- [x] 10.3 實作操作日誌 Data Grid
  - username
  - module
  - action
  - method
  - path
  - ip
  - HTTP status Chip
  - result Chip
  - duration
  - created_at
  - 詳情
  - _Design: 操作日誌欄位定義_

- [x] 10.4 實作日誌詳情 Dialog
  - Request Body
  - Response Body
  - JSON pretty print
  - 可捲動 pre 區塊
  - _Design: 詳情 Dialog_

- [x] 10.5 實作登入日誌 Data Grid
  - username
  - ip
  - user agent 解析
  - result Chip
  - reason
  - created_at
  - _Design: 登入日誌欄位定義_

- [ ] 10.6 實作 CSV 匯出
  - 依目前篩選條件匯出
  - `type=activity|login`
  - blob download
  - _Design: 匯出按鈕_

- [x] 10.7 配合後端 Activity Logs 欄位契約補強前端查詢與欄位來源
  - 後端完成 server task 9.6 後，前端 `method`、`path`、`status_code`、`result`、`duration_ms` 欄位需改讀 response 一等欄位
  - `ip`、`method`、`status_code`、`result` 搜尋條件需送到 `GET /api/v1/admin/logs/activity`
  - 移除僅從 `detail` 讀列表欄位的 fallback 邏輯，保留 `detail` 給詳情 Dialog
  - 不得在後端缺欄位時推測或偽造成功狀態、HTTP 狀態碼或耗時
  - _Depends on: server task 9.6；Design: 操作日誌欄位定義_

---

## 11. Admin 全域設定

- [x] 11.1 建立 `/admin/system` 列表頁
  - Breadcrumbs
  - 分類 Select
  - 搜尋列
  - 新增按鈕
  - Data Grid
  - _Design: 全域設定頁設計_

- [x] 11.2 實作設定列表欄位
  - key
  - value
  - category
  - value_type Chip
  - description
  - 操作按鈕
  - _Design: 列表頁結構_

- [x] 11.3 實作設定值顯示規則
  - boolean 顯示圓點
  - json 截斷 + Tooltip
  - 空值顯示 `-`
  - _Design: 設定值顯示規則_

- [x] 11.4 建立 `/admin/system/:key/edit` 編輯頁
  - key 唯讀
  - value
  - category
  - value_type
  - description
  - sort_order
  - is_active
  - 儲存 / 取消
  - _Design: 編輯頁_

- [x] 11.5 建立 `/admin/system/new` 新增頁
  - key 可編輯
  - 與編輯頁共用表單元件
  - _Design: 新增頁_

- [x] 11.6 實作依 value type 動態輸入
  - string：TextField
  - number：number input
  - boolean：Switch
  - json：multiline + `JSON.parse` 驗證
  - _Design: 編輯頁_

---

## 11A. Admin 專案管理

- OpenAPI 來源：`opscenter-server/Docs/openapi.json`
- API 範圍：`projects` tag 的 `/api/v1/projects`、`/api/v1/projects/{id}`
- 注意：不得使用不存在的 `/api/v1/admin/projects`；不得在前端偽造 `owner`、`is_active`、`sub_project_count` 等 server 未回傳欄位。

- [x] 11A.1 建立 `/admin/projects` 專案管理列表頁
  - Breadcrumbs
  - 搜尋列
  - 新增專案按鈕
  - Data Grid
  - 使用既有 `/api/v1/projects`，不得串不存在的 `/api/v1/admin/projects`
  - _Design: 專案管理頁設計_

- [x] 11A.2 實作專案列表 API client
  - `GET /api/v1/projects?keyword=&limit=&offset=`
  - `POST /api/v1/projects`
  - `GET /api/v1/projects/{id}`
  - `PUT /api/v1/projects/{id}`
  - `DELETE /api/v1/projects/{id}`
  - 依據 `Docs/openapi.json` 處理 `400`、`401`、`403`、`404`、`409`、`500` 錯誤狀態
  - API 型別需對齊 server response：`id`、`key`、`name`、`description`、`visibility`、`status`、`created_by`、`created_at`、`updated_at`
  - _Design: 專案管理頁設計；Dependency: server project API_

- [x] 11A.3 實作專案列表欄位
  - ID
  - 專案名稱
  - 專案代碼 `key`
  - 可見性 `visibility` Chip
  - 狀態 `status` Chip
  - 建立者 `created_by`
  - 備註
  - 操作按鈕
  - 不顯示 `owner`、`is_active`、`sub_project_count` 等 server 未回傳欄位
  - _Design: 專案管理頁設計_

- [x] 11A.4 實作專案搜尋與分頁
  - keyword 對 `key` / `name` 模糊搜尋
  - Enter 或搜尋按鈕觸發查詢
  - 搜尋時重設頁碼
  - server 目前不支援 status 篩選，前端不得加入假篩選
  - Data Grid 內建文字需接 `getDataGridLocaleText`
  - _Design: 專案管理頁設計；Design: Table / Grid 內建文案_

- [x] 11A.5 實作新增 / 編輯專案 Dialog
  - name
  - key：新增可編輯，編輯唯讀；後端會正規化為大寫
  - description
  - visibility：`public` / `private`
  - status：`active` / `inactive` / `archived`
  - React Hook Form + Zod
  - 儲存成功 / 失敗 Toast
  - _Design: 專案管理頁設計_

- [x] 11A.6 實作刪除專案流程
  - 刪除確認 Dialog
  - 刪除按鈕使用 `color="error"`
  - 成功後重新整理列表
  - 失敗時顯示後端錯誤 Toast，不得先行從列表移除資料
  - 對 `401`、`403`、`404`、`500` 顯示後端 `message`
  - _Design: 專案管理頁設計_

- [x] 11A.7 明確排除子專案與專案成員管理
  - `/admin/projects` 不顯示 `sub_project_count`
  - `/admin/projects` 不提供子專案 CRUD 入口
  - `/admin/projects` 不顯示成員數
  - `/admin/projects` 不提供成員新增、編輯或移除入口
  - 子專案與專案成員管理放在 Ticket / 專案工作區脈絡
  - _Design: 子專案顯示邊界；Design: 專案成員管理邊界_

---

## 12. 報表中心與圖表

- [x] 12.1 建立 `/projects/:id/reports` 報表中心頁
  - 範本列表
  - 執行範本
  - 報表設計器
  - 內建月報
  - _Phase: Phase 3_
  - _Design: 報表中心頁面_

- [x] 12.2 實作報表範本列表
  - 專案共用範本
  - PM+ 可新增 / 編輯 / 刪除
  - _Phase: Phase 3_
  - _Design: TemplateList_

- [ ] 12.3 實作範本執行
  - 本週 / 上週 / 本月 / 上月 / 自訂
  - 預覽
  - CSV 匯出
  - _Phase: Phase 3_
  - _Design: TemplateExecute_

- [ ] 12.4 實作週報 Bar Chart
  - 橫軸日期
  - 縱軸 Ticket 數量
  - 分組班別或個人
  - _Phase: Phase 2_
  - _Design: 週報_

- [ ] 12.5 實作月報模式 A
  - 指標導向
  - 週區間 + 總計
  - 個人 / 班別
  - 明細表
  - _Phase: Phase 3_
  - _Design: 月報模式 A_

- [ ] 12.6 實作月報模式 B
  - 任務導向
  - Ticket 標題 × 人員
  - Stacked Bar
  - 交叉明細表
  - _Phase: Phase 3_
  - _Design: 月報模式 B_

- [ ] 12.7 實作月報模式 C
  - 人員 × 子專案堆疊
  - OP 月報格式
  - created_by / actor 統計口徑
  - _Phase: Phase 3_
  - _Design: 月報模式 C_

- [ ] 12.8 實作月報模式 D 報表設計器
  - 橫軸
  - 縱軸分組
  - 統計指標
  - 圖表類型
  - 即時預覽
  - 日期預覽與匯出 payload 固定使用 `YYYY-MM-DD`，避免瀏覽器 locale 造成後端 `invalid_date_config`
  - 日期粒度僅在橫軸或系列使用日期維度時送出，避免非日期維度預覽被後端拒絕
  - 儲存範本
  - _Phase: Phase 3_
  - _Design: 月報模式 D_

- [x] 12.9 值班統計指定報表新增總計欄
  - 告警通知、域名更換及支付域名更換在第一個日期前顯示總計
  - 依目前查詢日期欄位加總每列數值，缺值以 `0` 計算
  - 指標與人員列使用相同計算方式
  - 隱藏早班、中班、晚班的班別彙總列，保留班別-人員明細列
  - Jira 通知不新增總計欄，後端 API 契約維持不變
  - 補齊繁體中文、簡體中文與英文欄位名稱
  - _Requirement: 14.12_
  - _Design: 值班統計矩陣顯示規則_

- [x] 12.10 值班統計顯示名稱調整
  - 三張指定報表移除標題中的「每日」
  - 報表模式與空資料提示統一使用「值班統計」
  - 不變更 `daily_shift_execution`、API 路徑或資料庫
  - _Requirement: 14.13_
  - _Design: 值班統計矩陣顯示規則_

- [x] 12.11 值班統計項目移至數值選單
  - 報表模式只保留單一「值班統計」選項
  - 選到值班統計時，數值提供告警通知、域名更換及支付域名更換
  - 沿用既有 `dailyShiftMetricGroup`、`metric_groups` 與範本 `indicators`
  - 不修改後端契約與資料庫
  - _Requirement: 14.14_
  - _Design: 值班統計矩陣顯示規則_

---

## 13. Jira 與專案頁面入口

- [ ] 13.1 建立 `/projects` 專案列表入口
  - 專案列表
  - 切換 currentProject
  - 進入專案儀表板
  - _Design: 頁面結構_

- [ ] 13.2 建立 `/projects/:id` 專案儀表板
  - 專案概況
  - Ticket 摘要
  - 導向 Ticket / Reports / Jira
  - _Design: 頁面結構_

- [ ] 13.3 建立 `/projects/:id/jira` Jira 匯入與統計頁
  - CSV 匯入入口
  - 匯入結果
  - 統計圖表
  - _Phase: Phase 2_
  - _Design: 頁面結構, 報表設計_

- [ ] 13.4 建立專案工作區子專案管理入口
  - 入口放在 `/projects/:id` 專案工作區，不放在 `/admin/projects`
  - 顯示子專案列表、狀態、建立時間、更新時間與操作
  - 使用與既有管理頁一致的 Data Grid 樣式、Toast、Dialog、i18n 與權限提示
  - _Design: 子專案顯示邊界；Dependency: `Docs/openapi.json` sub-projects API_

- [ ] 13.5 實作子專案 API client
  - `GET /api/v1/projects/{id}/sub-projects`
  - `POST /api/v1/projects/{id}/sub-projects`
  - `GET /api/v1/sub-projects/{sid}`
  - `PUT /api/v1/sub-projects/{sid}`
  - `DELETE /api/v1/sub-projects/{sid}`
  - API 型別需對齊 server response：`id`、`project_id`、`key`、`name`、`description`、`is_active`、`created_at`、`updated_at`
  - 依據 `Docs/openapi.json` 處理 `400`、`401`、`403`、`404`、`409`、`500` 錯誤狀態
  - _Design: 子專案顯示邊界；Dependency: server sub-project API_

- [ ] 13.6 實作子專案新增 / 編輯 / 刪除流程
  - 欄位：`key`、`name`、`description`、`is_active`
  - `key` 新增可編輯，編輯時依後端允許行為處理
  - 刪除需確認 Dialog
  - 成功後重新整理列表
  - 失敗時顯示後端錯誤 Toast，不得偽造成功狀態
  - _Design: 子專案顯示邊界_

- [ ] 13.7 建立專案成員管理入口
  - 入口放在 `/projects/:id` 專案工作區，不放在 `/admin/projects`
  - 顯示專案成員列表、全域角色、專案角色、啟用狀態與加入時間
  - 專案角色支援 `project_manager`、`engineer`、`viewer`
  - 使用與既有管理頁一致的 Data Grid 樣式、Toast、Dialog、i18n 與權限提示
  - _Design: 專案成員管理邊界；Dependency: `Docs/openapi.json` project-members API_

- [ ] 13.8 實作專案成員 API client 與操作流程
  - `GET /api/v1/projects/{id}/members`
  - `POST /api/v1/projects/{id}/members`
  - `PUT /api/v1/projects/{id}/members/{uid}`
  - `DELETE /api/v1/projects/{id}/members/{uid}`
  - API 型別需對齊 server response：`project_id`、`user_id`、`username`、`email`、`full_name`、`global_role`、`is_active`、`role`、`joined_at`
  - 新增成員使用 `user_id` 與 `role`
  - 更新成員使用 path `uid` 與 request body `role`
  - 刪除需確認 Dialog
  - 依據 `Docs/openapi.json` 處理 `400`、`401`、`403`、`404`、`500` 錯誤狀態
  - _Design: 專案成員管理邊界；Dependency: server project-members API_

---

## 14. 無障礙、響應式與品質驗證

- [ ] 14.1 落實語意化互動元素
  - button
  - input
  - select
  - label 關聯
  - _Design: 無障礙_

- [ ] 14.2 補齊 aria 標籤
  - Badge
  - IconButton
  - Data Grid 操作按鈕
  - Dialog close
  - _Design: 無障礙_

- [ ] 14.3 確認鍵盤操作
  - Tab 順序
  - Enter 觸發
  - Dialog focus trap
  - Menu / Popover 可鍵盤操作
  - _Design: 無障礙_

- [ ] 14.4 確認色彩對比
  - WCAG 2.1 AA
  - Chip / Badge / 漸層按鈕可讀性
  - dark / glass theme 皆需檢查
  - _Design: 無障礙, 主題設計_

- [ ] 14.5 建立前端測試基礎
  - unit test
  - component test
  - route guard 測試
  - form validation 測試
  - _Design: 全域_

- [ ] 14.6 建立 Playwright 視覺與流程驗證
  - login
  - MFA
  - dashboard
  - admin list
  - mobile viewport
  - desktop viewport
  - _Design: 全域_

- [ ] 14.7 建立 build / lint / typecheck 指令
  - `npm run lint`
  - `npm run typecheck`
  - `npm run test`
  - `npm run build`
  - _Design: 技術選型_

---

## 15. 後端依賴確認

- [ ] 15.1 確認 Auth API 契約
  - `POST /api/v1/auth/login`
  - `POST /api/v1/auth/mfa/setup`
  - `POST /api/v1/auth/mfa/verify`
  - `GET /api/v1/auth/config`
  - _Dependency: Backend_

- [ ] 15.2 確認 Admin API 契約
  - users
  - roles
  - logs
  - schedulers
  - global settings
  - _Dependency: Backend_

- [ ] 15.3 確認附件 API 契約
  - metadata list
  - upload
  - delete
  - content stream
  - _Dependency: Backend_

- [ ] 15.4 確認 Dashboard / Report API 契約
  - stat cards
  - chart payload
  - report preview
  - CSV export
  - _Dependency: Backend_
