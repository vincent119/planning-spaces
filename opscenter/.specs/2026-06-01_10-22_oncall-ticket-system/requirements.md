但是# Requirements Document

## Introduction

建立一套供工程師使用的運維平台提供排班、排定假期、Ticket System，用於記錄、追蹤與管理運維事件（Incident）、日常巡檢（Daily）與問題處理（Issue），以及（周/月/年）報表產生。系統需支援多專案（含子專案）以及多人協作、狀態流轉、優先級管理與通知機制。

組織架構為 SRE 經理 → 值班組長 → 早/中/晚三班工程師。人員班制與排班 / 假勤資料由獨立 bounded context 管理；Ticket 本身不保存班別快照，僅記錄開單人與操作人，班別統計需依人員與時間查詢排班資料。

班別類型為 固定班（正常班）、三班制（早/中/晚輪班）、四班二輪（日/夜輪班）、On-call 輪值

假期安排為 正常工時、二週變形工時、四週變形工時、八週變形工時、責任制 / 勞基法 84-1

---

## Delivery Phases

### MVP

使用者與權限管理、專案 / 子專案管理、Ticket CRUD、Ticket 狀態流轉、指派與協作、留言與活動記錄、基本搜尋與篩選、附件管理、Global Setting。MVP 目標是先打通 Ticket 主流程；Ticket 不直接記錄歸屬班別，僅記錄開單人與每次操作人員。

MVP 明確不包含：SLA、通知、完整排班 / 假勤、Jira CSV 匯入、報表設計器、API Key、SSO。

### Phase 2

SLA 管理、通知機制、儀表板強化、自訂報表基礎查詢、Jira 簡中 CSV 匯入與統計報表、非同步匯入 / 匯出 Job。

### Phase 3

完整排班 / 假勤管理、完整勞基法驗證、排班週期確認與鎖定、排班 Excel / CSV 匯出、SSO 整合、報表設計器與範本、Webhook 通知、自動國定假日 API、人事系統匯入整合、API Key 認證。

---

## Requirements

### 需求 1：使用者與權限管理

**使用者故事**：身為系統管理員，我希望能管理使用者帳號與權限，確保資料安全。

#### 驗收條件

- [ ] 1.1 支援全域角色：`Admin`、`op_admin`、`Member`
- [ ] 1.2 支援專案層級角色：`Project Manager`（值班組長）、`Engineer`、`Viewer`
- [ ] 1.3 使用者帳號包含屬性：姓名、帳號、Email（選填，唯一）、聯絡電話（選填）、全域角色；班制歸屬由人員班制指派記錄（`user_shift_assignments`）獨立管理，不儲存於帳號欄位
- [ ] 1.4 `Admin` 可管理所有專案、Ticket 與使用者帳號
- [ ] 1.5 `Project Manager` 可管理所屬專案的 Ticket、成員與設定，可關閉 Ticket
- [ ] 1.6 `Engineer` 可在所屬專案建立、編輯 Ticket，可新增留言
- [ ] 1.7 `Viewer` 僅可查看所屬專案的 Ticket（不可留言或修改）
- [ ] 1.8 支援 JWT 認證；Access Token 有效期 8 小時，支援 Refresh Token
- [ ] 1.8a 支援閒置時間自動登出；後端需提供已登入可讀的 session policy API，僅回傳 `idle_timeout_seconds`，來源為 `system_settings.security.session.timeout`。前端需在登入狀態下監聽使用者活動，超過設定秒數無操作時清除 Access Token、Refresh Token、目前使用者狀態與 React Query cache，顯示閒置逾時 Toast 並導回登入頁。`security.session.timeout` 預設為 28800 秒；設定值小於等於 0 時停用前端閒置自動登出；設定缺值、停用或格式無效時使用預設值。此機制不取代後端 JWT / Refresh Token 到期驗證。
- [ ] 1.9 支援 MFA（Multi-Factor Authentication）：
  - 一個帳號可綁定多個 MFA 裝置（例如多支手機）
  - 支援 TOTP（Time-based One-Time Password，相容 Google Authenticator / Authy）
  - MFA 啟用後，登入時需完成第二因子驗證才能取得 Token
  - 可由 Admin 強制要求用戶啟用 MFA；是否強制 Admin 自身啟用 MFA 由系統設定控制
  - 支援 MFA 裝置管理：新增、命名、移除裝置
  - MFA setup 建立但尚未完成 TOTP 驗證的裝置不得視為已綁定裝置；列表 API 與前端已綁定裝置列表只能呈現已驗證裝置
  - 後端必須自動清理逾時未驗證 MFA 裝置，避免未完成 setup 長期保留或造成同一使用者裝置名稱唯一鍵衝突；清理採軟刪除並需可由系統設定調整保留時間
- [ ] 1.10 登入頁需以右上角 Toast 顯示登入流程與語系切換結果；成功事件使用成功樣式，登入失敗、密碼錯誤、MFA 驗證失敗與系統錯誤使用錯誤樣式，Toast 不得遮擋登入表單與語系切換控制
- [ ] 1.11 支援 SSO 整合（OIDC / SAML，可選，屬 Phase 3，MVP 階段暫不實作）
- [ ] 1.12 權限合併規則：
  - `Admin` 擁有系統最高權限，可管理所有專案、使用者、系統設定、安全設定、表單樹與群組權限，所有操作直接放行
  - `op_admin` 擁有運維管理權限，可管理運維資料與跨專案運維視圖；不可管理系統安全設定、全域使用者、MFA / SSO 設定，除非同時具備 `Admin`
  - `Member` 為一般登入使用者，需依專案角色與表單群組權限共同判斷
  - 專案內操作需同時滿足「專案角色允許」與「表單操作層級允許」；`Admin` 例外直接放行
  - OIDC 新使用者預設全域角色為 `Member`，未加入專案前不具私有專案存取權；公開專案依公開可見規則處理
  - 範例：peter = `op_admin` + `Project Manager`，可管理所屬專案運維資料；Kevin = `Member` + `Engineer`，可在所屬專案開單 / 編輯 / 留言；john = `Member` + `Viewer`，僅可查看；OIDC 新使用者 = `Member` 且無專案角色，預設僅可登入
- [ ] 1.13 使用者可於個人資料頁維護自己的基本資料：姓名、Email、聯絡電話；`username` 與 `global_role` 僅顯示不可自行修改；後端個人資料 API 與使用者管理 API 都必須讀寫 `users.phone`
- [ ] 1.13a Admin 用戶管理列表需顯示 MFA 狀態；若使用者存在 `is_active = TRUE AND is_verified = TRUE` 的 MFA 裝置則顯示「已啟用」，否則顯示「未啟用」。未完成驗證或已停用的 MFA 裝置不得使列表顯示為已啟用
- [ ] 1.14 使用者修改密碼時必須提供目前密碼、新密碼與確認新密碼；前端需透過已登入的密碼政策 API 取得 `security.login.password_length` 對應的最小長度並驗證確認一致，不得直接讀取 Admin Global Settings API；後端必須以目前登入使用者的 `password_hash` 驗證目前密碼正確後才可更新，並以 `security.login.password_length` 作為新密碼最小長度的最終驗證來源；設定缺值或無效時預設為 4，不得只依登入狀態或 Admin 使用者更新 API 直接改密碼
- [ ] 1.15 個人資料儲存、修改密碼、MFA 裝置新增 / 刪除等所有新增、修改、刪除與狀態變更操作，成功與失敗皆必須顯示 Toast；成功使用成功樣式，驗證失敗、權限不足、後端錯誤與系統錯誤使用錯誤樣式，訊息優先使用後端回傳內容
- [ ] 1.16 前端可讀取的安全政策 API 必須採最小揭露原則，例如密碼政策僅回傳 `min_length`；不得以「前端可解密」的混淆或加密 response 作為保護手段。JWT secret、OIDC client secret、S3 secret、DB / Redis 密碼等機密設定不得透過任何前端 API 明文回傳，若需顯示狀態只能回傳遮罩值或 `configured: true`
- [ ] 1.17 OIDC 群組對應需保留內部權限模型，不得直接把外部群組名稱寫入 `users.global_role` 或要求修改 `roles.code`。OIDC callback 取得外部群組後，需透過明確 mapping 轉換為內部結果：
  - `ops_admin` 對應 `global_role = admin`
  - `ops_op_admin` 對應 `global_role = op_admin`
  - `ops_member` 對應 `global_role = member`
  - `ops_op_member` 對應 `global_role = member`
  - 若 mapping 目標包含權限群組，登入同步流程才可新增 / 移除 `group_members`；選單 / 表單操作權限仍以 `groups`、`group_members`、`group_form_permissions` 與 Casbin policy 判斷，不得以 OIDC 群組名稱或 `roles.code` 硬編碼放行
  - 多個 OIDC 群組同時命中時，`global_role` 取最高權限序：`admin` > `op_admin` > `member`；權限群組採聯集同步
  - 未命中任何 mapping 的 OIDC 新使用者預設 `global_role = member`，不得自動加入任何權限群組

---

### 需求 2：表單樹與 RBAC 權限

**使用者故事**：身為系統管理員，我希望能自訂表單樹的節點識別，並透過群組與 Casbin RBAC 權限控制使用者對表單的讀取、新增、修改、刪除操作層級。

#### 驗收條件

- [ ] 2.1 表單以樹狀結構組織；每個節點具唯一識別碼（`form_key`，路徑式，例如 `ticket/create`），同一父節點下 `form_key` 不可重複
- [ ] 2.2 節點類型包含：根分類、中繼分類、表單（葉節點）；`Admin` 可自訂節點名稱、識別碼、排序、說明與父子關係
- [ ] 2.3 `Admin` 可對表單樹執行 CRUD：**讀取**（查看樹狀結構與節點詳情）、**新增**（建立節點）、**修改**（更新節點屬性與層級）、**刪除**（移除節點）
- [ ] 2.4 刪除節點前須確認無子節點；若該表單已被業務資料引用則拒絕刪除並回傳錯誤
- [ ] 2.5 支援群組（`groups`）管理使用者；使用者可隸屬一個或多個群組
- [ ] 2.6 `group_form_permissions` 作為業務授權來源表，群組可被授予表單操作層級，類型包含：`read`（讀取）、`create`（新增）、`update`（修改）、`delete`（刪除）；權限可綁定至特定 `form_key` 節點
- [ ] 2.7 子節點預設繼承父節點的群組操作層級；子節點可另設覆寫規則；使用者實際權限為其所屬群組權限的聯集，並由 application layer 同步為 Casbin policy
- [ ] 2.8 API 授權檢查必須集中於 authorization middleware / service，使用 Casbin `Enforce(userID, formPath, action)` 判斷；handler 不得自行查表判斷表單權限
- [ ] 2.9 未具備對應操作層級時，API 與 UI 皆拒絕該操作（回傳 403）；僅 `Admin` 可管理表單樹與群組權限設定
- [ ] 2.10 表單樹、群組與群組權限異動需同步更新 Casbin policy，並記錄審計日誌（操作人員、時間、異動前後內容）；提供依 `form_key` 查詢子樹並回傳當前使用者有效操作層級的 API
- [ ] 2.11 前端選單管理區需提供「選單權限 / 表單權限」設定流程；此流程管理的是權限群組（`groups`）與表單節點（`form_nodes`）的授權關係，不得把 `roles` 表或 `users.global_role` 誤用為表單權限來源。角色管理區只保留角色 CRUD，不承載選單 / 表單權限設定入口
- [ ] 2.12 `Admin` 可在權限設定流程中建立 / 編輯 / 停用 / 刪除權限群組，並管理群組成員；同一使用者可加入多個權限群組，停用群組後不得再產生有效 Casbin policy
- [ ] 2.13 `Admin` 可針對單一權限群組在表單樹上設定 `read`、`create`、`update`、`delete` 操作層級，並設定是否套用至子節點（`inherit_children`）與是否覆寫父節點繼承（`override_parent`）
- [ ] 2.14 權限設定 UI 需顯示「直接授權」、「繼承授權」、「覆寫授權」三種狀態，避免管理員誤判子節點權限來源；儲存前需顯示異動摘要，儲存失敗不得先行顯示成功
- [ ] 2.15 選單可見性由當前使用者對 `form_nodes.full_path` 的有效 `read` 權限決定；未具備 `read` 權限的節點不得出現在一般使用者側邊欄與可操作入口，直接開啟路由時仍需後端 403 驗證
- [ ] 2.16 權限設定流程的所有新增、修改、刪除、成員異動與同步 Casbin policy 結果皆需寫入 `system_audit_logs` 或 `form_audit_logs`；審計內容需包含群組、表單節點、操作層級、異動前後摘要與操作人員
- [ ] 2.17 後端需提供預設選單樹與預設權限群組 seed，供 OIDC 群組對應與初始權限使用：
  - 預設選單樹需包含 `system` 根節點「系統管理」，以及子節點 `system/users`、`system/roles`、`system/menus`、`system/projects`、`system/logs`、`system/schedulers`、`system/settings`
  - 預設權限群組需包含 `ops_admin` 與 `ops_op_admin`；`ops_admin` 對 `system` 子樹具備 `read/create/update/delete`，`ops_op_admin` 僅對 `system/logs` 具備 `read`
  - 預設權限必須寫入業務來源表 `group_form_permissions`，再由後端同步或 seed 對應 Casbin policy；不得只寫入 `casbin_rule`
  - `casbin_rule` 是後端授權 runtime table，由 Go 後端或 migration 同步控制；前端不得提供直接管理 `casbin_rule` 的介面
- [ ] 2.18 所有對應一般側邊欄或工作區表單節點的後端讀取 API，必須以該節點 `form_nodes.full_path` 的有效 `read` 權限作為第一層授權來源；不得只依 `global_role`、任一工作區讀取權限或 project member 身分放行。若 API 同時涉及特定專案資料，可在表單 `read` 通過後再檢查專案存在性或必要的 project role。真正不屬於表單樹入口的 API 才可例外，例如個人 profile、個人通知、公開設定、health、metrics、登入 / MFA 流程與僅限 Admin 的安全管理 API。

---

### 需求 3：Ticket 建立與管理

**使用者故事**：身為值班工程師，我希望能快速建立 Ticket 並填寫必要資訊，以便追蹤問題處理進度。

#### 驗收條件

- [ ] 3.1 使用者須具備對應 Ticket 表單的 `create` 操作層級（需求 2）方可建立 Ticket；必填欄位包含：標題、描述、事件類型（來自 `ticket_types`，預設 Daily / Issue）、優先級（P1 / P2 / P3 / P4）、所屬專案、所屬子專案、資訊來源
- [ ] 3.2 Ticket 建立後自動產生 ULID 格式唯一編號（26 字元 Crockford Base32，例如 `01HYX7Y9Z4Q9V7A3QF9M6B2C8D`），全系統唯一且可依建立時間排序；與外部 Jira 單號完全獨立，Jira 單號為另一個選填欄位
- [ ] 3.3 Ticket 本身不保存班別快照，也不直接記錄歸屬班別；班別僅作為人員排班 / 假勤資料的一部分。開單人由 `created_by` 記錄，每次建立、更新、處理、留言皆記錄於 `ticket_activities.actor_id`；若報表需依班別統計，需依 Ticket 建立人 / 活動操作人與對應時間查詢人員班制或排班資料
- [ ] 3.4 資訊來源選項來自 `ticket_resources` 資料表；Ticket 建立時儲存對應的資訊來源引用，不直接儲存顯示文字。系統預設提供：TG Alert、Mail、Signal、Signal 項目群、WhatsApp、WhatsApp 項目群、Zabbix Alert、支付or業務項目域名更換
- [ ] 3.5 外部單號（例如 Jira 單號）為選填欄位，格式為自由文字，一張 Ticket 對應一個外部單號
- [ ] 3.6 使用者可編輯 Ticket 的所有欄位（已關閉的 Ticket 除外）
- [ ] 3.7 系統不支援 Ticket 草稿狀態；Ticket 建立後狀態即為 `Open`。使用者僅可刪除自己建立且仍為 `Open` 狀態的 Ticket；Ticket 刪除採軟刪除（soft delete），需記錄刪除人員、刪除時間與 `ticket_activities`，不進行實體刪除
- [ ] 3.8 Ticket 支援附加標籤（Tags）以便分類與搜尋
- [ ] 3.9 Ticket 可關聯受影響的服務（Affected Services）
- [ ] 3.10 Ticket 建立與編輯時可上傳截圖或圖片附件（原始格式支援 JPG、PNG、GIF、WebP），單檔上限 10MB，每張 Ticket 最多 20 個附件；後端必須驗證實際圖片內容，不可只信任副檔名或 request `Content-Type`
- [ ] 3.11 圖片附件寫入儲存後端前，必須使用 `govips`（libvips）統一轉換為 Global Setting `storage.image.output_format` 指定的 AVIF 或 WebP；轉檔需套用正確方向並移除 EXIF、GPS 等非必要 metadata。附件儲存後端可設定為本機檔案系統（Local）或 AWS S3 Private Bucket，透過設定檔切換，不需修改程式碼；S3 使用 AWS SDK for Go v2，支援 IRSA、Instance Role、static credentials 與 MinIO custom endpoint。Local 相對路徑與 S3 object key 統一使用 `yyyy/mm/dd/{attachment_ulid}.{ext}`（日期依 `Asia/Taipei` 上傳日，`ext` 僅允許 `avif` 或 `webp`）
- [ ] 3.12 附件一律透過系統 API 存取（需認證），不直接暴露 Local 路徑、S3 路徑或 Bucket URL；前端以 `GET /api/v1/attachments/:id/content` 取得內容，Go 後端自 S3（或 Local）讀取後，以轉檔後的 `Content-Type`（`image/avif` 或 `image/webp`）串流回傳 Browser，**不使用 Pre-signed URL**
- [ ] 3.13 使用者可刪除自己上傳的附件（已關閉的 Ticket 除外），刪除時同步清除儲存後端（Local 檔案或 S3 object）的實體檔案
- [ ] 3.14 Ticket 事件類型由 `ticket_types` 資料表管理；Ticket 建立時儲存對應的 `ticket_type_id`，不直接儲存類型顯示文字。系統預設提供：
  - Daily：用於日常巡檢與例行作業，可使用 `Open` / `In Progress` / `Resolved` / `Closed` / `Cancelled` 狀態，不支援 `Escalated`；預設不套用 SLA；優先級預設為 `P4`，可手動調整為 `P3` / `P4`；可選擇性指派 Assignee
  - Issue：用於異常事件與問題處理，可使用 `Open` / `In Progress` / `Escalated` / `Resolved` / `Closed` / `Cancelled` 狀態；套用 SLA；支援 `P1` / `P2` / `P3` / `P4`；建議指派 Assignee
  - Daily / Issue 為系統內建事件類型，`ticket_types.is_system = TRUE`，不可刪除，且不可修改會破壞系統流程的核心設定
  - 後續若新增事件類型，需於 `ticket_types` 設定是否支援 `Escalated`、是否套用 SLA、允許優先級與預設優先級；自訂事件類型 `is_system = FALSE`
- [ ] 3.15 建立 Ticket 時需在同一交易中同時寫入 `tickets.created_by = login user` 與第一筆 `ticket_activities`；該活動紀錄的 `action_type = created`、`actor_id = login user`，確保 Ticket 完整歷程從建立事件開始
- [ ] 3.16 Ticket 列表操作欄需提供「複製列資訊」功能，複製內容只包含可供溝通使用的文字欄位，不包含 Ticket ID、附件 / 貼圖、內部 storage key 或任何帶 ID 的詳情連結；複製成功與失敗需顯示對應提示。
- [ ] 3.17 Ticket 附件權限需與 Ticket 詳情頁表單權限一致：列出附件與讀取內容使用 `tickets/list:read`，上傳與刪除使用 `tickets/list:update`，不得再要求使用者存在於 `project_members`。刪除仍僅限原上傳者，且已關閉 Ticket 不可上傳或刪除附件。
- [ ] 3.18 Ticket 列表使用伺服器端分頁時，切換頁面期間必須保留上一筆成功查詢的總筆數，避免載入新頁資料時因 `rowCount` 暫時歸零而將目前頁碼重設為第一頁；使用者連續點擊下一頁時，頁碼與 API `page` 參數必須依序遞增。
- [ ] 3.19 Ticket 列表需在「班別」與「指派人員」之間顯示「開單人員」欄位；內容優先顯示開單人全名，其次顯示帳號，最後以建立者 ID 作為 fallback，不得將指派人員誤作開單人員。
- [ ] 3.20 Ticket 列表搜尋區需提供「開單人員」篩選，選項只能來自目前專案未刪除 Ticket 實際出現過的開單人去重清單，不得沿用指派人員的專案成員清單；選取後將開單人 ID 傳至後端做完整資料集的伺服器端篩選，變更條件時需回到第一頁，不得只篩選目前頁面資料。
- [x] 3.21 Ticket 列表需在「指派人員」與「更新時間」之間顯示「外部單號」欄位；內容使用列表 API 既有的 `external_ref`，未填寫時顯示既有空值符號，長文字需以省略方式顯示並提供完整內容 tooltip，不得把 Ticket ID 當作外部單號。

---

### 需求 4：多專案與子專案管理

**使用者故事**：身為系統管理員，我希望能建立多個獨立專案與子專案，讓不同團隊在各自的空間內管理 Ticket，互不干擾。

#### 驗收條件

- [ ] 4.1 系統支援建立多個專案（Project），每個專案有獨立的名稱、描述與識別碼（Project Key）
- [ ] 4.2 每個專案下可建立多個子專案（Sub Project），子專案有獨立的名稱與識別碼（Sub Project Key，例如 `QIEZ`、`QT`、`DSP`、`OPS01`）
- [ ] 4.3 Sub Project Key 為大寫英文字母或數字，長度 2~8 字元；同一 Project 內不可重複，唯一鍵為 `(project_id, sub_project_key)`
- [ ] 4.4 Ticket 編號為全系統唯一 ULID，不含專案前綴；子專案僅作為 Ticket 的分類屬性
- [ ] 4.5 每個專案可獨立設定成員與角色（專案層級權限）
- [ ] 4.6 使用者可同時參與多個專案，並在各專案中擁有不同角色
- [ ] 4.7 專案可設定為「公開」（組織內所有人可查看）或「私有」（僅成員可查看）
- [ ] 4.8 Admin 可建立、封存（Archive）、刪除專案與子專案；刪除前需確認無進行中 Ticket；專案與子專案刪除採軟刪除（soft delete），需記錄刪除人員、刪除時間與審計日誌，不進行實體刪除
- [ ] 4.9 儀表板支援跨專案彙總視圖，也可切換至單一專案或子專案視圖

---

### 需求 5：班別與排班 / 假勤管理

> **注意：本需求為獨立的排班 / 假勤 bounded context，主要用於依四週 / 八週變形工時展開行事曆，讓值班人員安排工作日、例假、休假、國定假與請假，並產出 Excel / CSV 排班表給 HR。完整排班 / 假勤管理屬 Phase 3，不作為 Ticket MVP 的前置依賴；Ticket 本身不直接記錄歸屬班別，班別統計需依開單人 / 操作人與人員班制或排班資料查詢。**

**使用者故事**：身為值班組長，我希望能依班制週期展開行事曆，安排組員工作與休假時間，檢查排班衝突與法規條件，並匯出排班表提供 HR 使用。

#### 驗收條件

- [ ] 5.1 系統支援多種班制類型：固定班（正常班）、三班制（早/中/晚輪班）、四班二輪（日/夜輪班）、On-call 輪值；每個班制包含一或多個具體班別（含起訖時間）
- [ ] 5.2 每位使用者透過人員班制歸屬（`user_shift_assignments`）關聯班制，歸屬含生效日期與結束日期，支援調動歷史追溯；使用者帳號本身不直接儲存班別
- [ ] 5.3 系統支援每日排班記錄（`user_schedules`）：正常排班每人每天唯一，代班支援同一天多段（例如早上代 A、下午代 B）
- [ ] 5.4 排班 / 假勤系統需能依人員與日期查詢當日有效班制、工作日、例假、休假與請假狀態；Ticket 不提供開單班別欄位，也不允許手動修改 Ticket 歸屬班別
- [ ] 5.5 On-call 排班以時間窗口（起訖時間，精度到分鐘）為單位記錄，支援跨天，並防止同一人的 On-call 時間重疊
- [ ] 5.6 排班異動（新增、修改、刪除）全程記錄審計日誌（`user_schedule_history`），含操作人員、時間、異動前後快照
- [ ] 5.7 Ticket 列表與報表可依班別篩選與統計，但班別條件需由 Ticket 建立人 / 活動操作人與對應時間查詢排班資料推導；班別選項來自系統班制設定（`duty_shifts`），不硬編碼
- [ ] 5.8 組織架構支援：SRE 經理 → 值班組長 → 工程師（依班制歸屬）
- [ ] 5.9 三班制適用的變形工時制（四週 / 八週）由 Global Setting（`schedule.rotating_labor_law`）統一設定，作為合規記錄與假期計算基準；單日最長工時上限亦可由設定調整（預設 10 小時）
- [ ] 5.10 系統維護國定假日行事曆（`public_holidays`），預設以手動匯入方式維護；預留對接人事行政總處 API 的擴充介面，可自動取得當年度或下年度行事曆
- [ ] 5.11 系統支援人員假別記錄，假別類型包含：例假、休假、國定假、年假、公假、病假、事假、其他（婚/喪）；目前以手動方式登記，預留對接人事系統匯入的擴充介面
- [ ] 5.12 假日/休假判斷由人員自行在排班記錄中選定；系統以 `day_type` 欄位統一記錄出勤與假別，不需跨表查詢
- [ ] 5.13 系統以「排班週期（`shift_periods`）」管理班表，對應現有 Google Sheet Tab，格式為 `YYYYMMDD-MMDD`，以班制為單位（早/中/夜各一個週期），預設以四週為一週期
- [ ] 5.14 所有組員可查看同一週期內其他人的例假/休假狀況；同一天同一班制不得有超過一人同時排例假或休假，系統於儲存時驗證並拒絕衝突請求（回傳 409）
- [ ] 5.15 法規驗證：每 7 天必須有一個例假（一週一例）；系統在週期統計欄位顯示每週例假檢查結果（`✅ 全部正確` / `⚠ 第N週缺例假`）
- [ ] 5.16 排班週期完成後由各班組長各自簽名確認（`shift_period_confirmations`），確認後週期狀態變更為 `confirmed`，鎖定後不可修改
- [ ] 5.17 週期統計資訊（對應圖片右側）：每人出勤天數、國/例/休計數、各假別天數；底部彙總：總休假日、總例假日、總國定假
- [ ] 5.18 支援匯出排班週期為 Excel（`.xlsx`）或 CSV 格式，還原現有 Google Sheet 格式：橫軸為日期、縱軸為人員（依班制分組）、右側統計欄、底部彙總列、組長簽名欄

---

### 需求 6：Ticket 狀態流轉

**使用者故事**：身為值班工程師，我希望 Ticket 有清楚的狀態流程，以便團隊了解目前處理進度。

#### 驗收條件

- [ ] 6.1 Ticket 狀態包含：`Open` → `In Progress` → `Resolved` → `Closed`；`Closed` 與 `Cancelled` 為終態
- [ ] 6.2 `ticket_types.supports_escalation = TRUE` 的事件類型支援額外狀態：`Escalated`（從 In Progress 可轉入，表示需要升級處理）；預設 Issue 支援，Daily 不支援
- [ ] 6.3 `Open`、`In Progress`、`Escalated` 可轉為 `Cancelled`（需填寫原因）；`Closed` 後不可再轉為 `Cancelled`
- [ ] 6.4 狀態變更需記錄操作人員與時間戳（UTC）
- [ ] 6.5 `Resolved` 狀態需填寫處理方式摘要（Resolution Summary）
- [ ] 6.6 `Closed` 狀態需由值班人員確認

---

### 需求 7：指派與協作

**使用者故事**：身為值班組長，我希望能將 Ticket 指派給特定工程師，並支援多人協作處理。

#### 驗收條件

- [ ] 7.1 Ticket 可指派給單一負責人（Assignee），負責人需為該專案成員
- [ ] 7.2 Ticket 可加入多位協作者（Collaborators），協作者需為該專案成員
- [ ] 7.3 支援 Ticket 轉交（Reassign），並記錄轉交歷史

---

### 需求 8：留言與活動記錄

**使用者故事**：身為工程師，我希望能在 Ticket 上留言並查看完整的操作歷史，以便了解問題處理脈絡。

#### 驗收條件

- [ ] 8.1 使用者可在 Ticket 上新增留言（支援 Markdown 格式），留言以 `ticket_activities` 的 `action_type = comment_added` 與 `content` 儲存
- [ ] 8.2 留言可標記為「內部備註」（Internal Note，僅專案成員可見），由 `ticket_activities.is_internal` 記錄
- [ ] 8.3 所有狀態變更、指派變更、欄位修改、留言新增、附件異動與刪除操作皆自動記錄於 `ticket_activities`；Ticket 欄位修改需記錄全部異動欄位的修改前 / 修改後內容（before / after）
- [ ] 8.4 `ticket_activities` 依時間排序，顯示操作人員、時間、操作類型與變更內容；同一張 Ticket 可有多位使用者、多次更新紀錄
- [ ] 8.5 留言支援 @mention 通知特定成員（限同專案成員）
- [ ] 8.6 `ticket_activities.action_type` 僅允許下列操作類型：`created`、`field_updated`、`status_changed`、`assigned`、`collaborator_added`、`collaborator_removed`、`comment_added`、`attachment_added`、`attachment_deleted`、`deleted`、`sla_breached`；其中 `sla_breached` 由 SLA 檢查排程寫入，屬 Phase 2 SLA 功能
- [ ] 8.7 `ticket_activities.field_changes` 統一使用 JSON object 記錄欄位異動，key 為資料欄位名稱，value 至少包含 `before` / `after`；FK 或 enum 欄位可額外包含 `before_label` / `after_label` 作為顯示快照。一次 API 更新多個欄位時寫入同一筆 activity，並在 `field_changes` 中包含所有異動欄位；純系統欄位（例如 `updated_at`）不需記錄

---

### 需求 9：搜尋與篩選

**使用者故事**：身為值班工程師，我希望能快速找到相關 Ticket，以便查閱歷史案例或追蹤進行中的問題。

#### 驗收條件

- [ ] 9.1 支援全文搜尋（標題、描述、`ticket_activities.content` 留言 / 處理內容），可限定在單一專案/子專案或跨專案搜尋
- [ ] 9.2 支援多條件篩選：專案、子專案、狀態、類型、優先級、班別（由人員與排班資料推導）、指派人、資訊來源、標籤、建立時間範圍
- [ ] 9.3 支援依欄位排序（建立時間、更新時間、優先級）
- [ ] 9.4 提供預設視圖：「我的 Ticket」、「未指派」、「今日 Issue」（可跨專案或限單一專案）
- [ ] 9.5 搜尋結果支援分頁（每頁預設 20 筆）

---

### 需求 10：通知機制

> **注意：此需求保留設計，MVP 階段暫不實作。站內通知與 Email 通知屬 Phase 2；Webhook 通知屬 Phase 3。**

**使用者故事**：身為工程師，我希望在 Ticket 有重要更新時收到通知，以便即時回應。

#### 驗收條件

- [ ] 10.1 支援站內通知（In-App Notification）
- [ ] 10.2 支援 Email 通知（可依個人偏好設定開關）
- [ ] 10.3 支援 Webhook 通知（可設定外部端點，例如 TG Bot / Slack），Webhook 設定為專案層級
- [ ] 10.4 P1 Issue 建立時，自動觸發緊急通知給該專案值班組長與相關人員
- [ ] 10.5 通知事件包含：Ticket 建立、狀態變更、被指派、被 @mention、留言新增

---

### 需求 11：SLA 管理

> **注意：SLA 管理屬 Phase 2，MVP 階段暫不實作。MVP 僅保留 Issue / Daily 是否套用 SLA 的事件類型設定。**

**使用者故事**：身為值班組長，我希望系統能追蹤 SLA 達成狀況，以便評估服務品質。

#### 驗收條件

- [ ] 11.1 SLA 設定為專案層級，各專案可獨立設定不同的 SLA 時間
- [ ] 11.2 預設依優先級定義 SLA 回應時間：P1=15min、P2=1hr、P3=4hr、P4=24hr
- [ ] 11.3 預設依優先級定義 SLA 解決時間：P1=2hr、P2=8hr、P3=24hr、P4=72hr
- [ ] 11.4 Ticket 超過 SLA 回應時間未處理時，系統需標記為 SLA 超時並於列表 / 詳情頁顯示；自動升級與通知值班組長屬 Phase 2，待通知機制啟用後實作
- [ ] 11.5 Ticket 詳情頁顯示 SLA 剩餘時間（倒數計時）
- [ ] 11.6 提供 SLA 達成率報表（依時間區間、優先級、專案、班別分類）

---

### 需求 12：API 介面

**使用者故事**：身為整合開發者，我希望系統提供完整的 REST API，以便與其他工具整合。

#### 驗收條件

- [ ] 12.1 提供 RESTful API，遵循統一回應格式（`code`, `message`, `data`, `trace_id`）
- [ ] 12.2 API 版本採 URL Path 方式（`/api/v1/`）
- [ ] 12.3 提供 Swagger / OpenAPI 3.0 文件
- [ ] 12.4 API Key 認證保留設計，屬 Phase 3，MVP 階段暫不實作；後續版本再定義 API Key 權限範圍、IP 限制與到期日
- [ ] 12.5 關鍵操作 API 需有 Rate Limiting（每分鐘 60 次）

---

### 需求 13：儀表板

**使用者故事**：身為值班組長，我希望有一個儀表板能快速掌握目前運維狀況。

#### 驗收條件

- [ ] 13.1 儀表板支援「全專案彙總」與「單一專案/子專案」兩種視圖切換
- [ ] 13.2 儀表板顯示：進行中 Ticket 數量、各優先級分佈、今日新增 / 解決數量
- [ ] 13.3 顯示 SLA 違反警示（標示超時 Ticket，屬 Phase 2，MVP 階段不顯示）
- [ ] 13.4 儀表板資料每 30 秒自動刷新
- [ ] 13.5 儀表板版面需依可用寬度自動重排，確保統計卡片、圖表與最近 Ticket 列表在桌面、平板與手機皆不產生非必要的水平捲動；不得以縮小整頁字級或整體 CSS scale 取代響應式佈局
- [ ] 13.6 全域與專案儀表板需提供版面編輯模式；桌面版可調整面板順序與支援的尺寸，並提供鍵盤操作、取消編輯及重設預設版面。手機版維持單欄自動重排，不提供自由拖曳
- [ ] 13.7 使用者自訂版面需依全域儀表板與各專案範圍分開保存；第一階段採目前瀏覽器 `localStorage` 保存並包含版面 schema version，不宣稱支援跨瀏覽器或跨裝置同步

---

### 需求 14：自訂報表與範本

> **注意：報表設計器與範本屬 Phase 3，MVP 階段暫不實作；Phase 2 僅提供自訂報表基礎查詢。**
> **文件來源調整：** Report 後續實作以 `.kiro/specs/2026-06-22_11-06_Report` 為準；本段保留舊版需求背景，不再作為 task 切分與 API 契約的唯一來源。

**使用者故事**：身為值班組長，我希望能自訂報表維度並儲存為範本，之後直接套用產出週報，取代手動填寫 Google Sheet 的流程。

#### 驗收條件

- [ ] 14.1 提供報表設計器，使用者可自由選擇以下統計維度組合：
  - 橫軸：日期（日 / 週 / 月）
  - 縱軸分組：班制（依 `shift_groups.name`）、個人（班制-班別-姓名）
  - 統計指標：Ticket 數量，可依以下條件拆分：
    - Ticket 事件類型（來自 `ticket_types`，預設 Daily / Issue）
    - 資訊來源（TG Alert / Mail / Signal / Signal 項目群 / WhatsApp / WhatsApp 項目群 / Zabbix Alert / 支付or業務項目域名更換）
    - 子專案
- [ ] 14.2 報表設計器支援即時預覽，選擇時間區間後可直接看到結果
- [ ] 14.3 設計完成後可儲存為報表範本（Template），需填寫範本名稱與描述
- [ ] 14.4 報表範本為專案層級共用，同專案成員皆可查看與執行範本
- [ ] 14.5 範本的建立、修改、刪除僅限 `Project Manager` 以上層級
- [ ] 14.6 執行範本時，使用者選擇時間區間（支援：本週、上週、本月、上月、自訂區間），系統自動產出報表
- [ ] 14.7 週報表以 7 天為單位，橫軸顯示日期與星期，符合現有 Google Sheet 格式
- [ ] 14.8 月報表以整月為單位（標題格式例如 `2026年0501-0531運維處理事件數量`），支援以下四種呈現模式（A / B / C 為內建版型，D 透過報表設計器自訂）：
  - **模式 A（指標導向）**：橫軸顯示週區間（例如 5/1-5/7、5/8-5/14）並附總計欄；每個統計指標獨立一張圖表，縱軸為個人（班別-姓名），包含長條圖 + 明細數字表，對應 OP 專案的 Jira 開單數 / 告警通知數 / 域名更換數格式
  - **模式 B（任務導向）**：縱軸為任務內容（Ticket 標題），橫軸為個人，呈現堆疊長條圖 + 交叉明細表（作業or問題 × 人員），對應 Juyou 專案格式
  - **模式 C（人員 × 子專案堆疊）**：整月彙總為單一圖表；橫軸為人員（依 `created_by` 或活動操作人統計，可於範本設定），堆疊區段為子專案（或專案內業務項目），呈現堆疊長條圖 + 圖例；對應 OP 月報「運維處理事件數量」格式（人員 × 各項目 Ticket 數）
  - **模式 D（設計器自訂維度）**：透過報表設計器（14.1）自由組合橫軸、縱軸、統計指標與圖表類型（長條 / 堆疊長條等），儲存為範本後可重複套用；時間區間支援整月或自訂區間，不受 A / B / C 固定版型限制
- [ ] 14.9 報表以圖表形式呈現（使用前端報表套件渲染，例如 ECharts / Recharts），支援長條圖與堆疊長條圖
- [ ] 14.10 匯出 CSV 功能所有專案成員皆可操作
- [ ] 14.11 報表查詢 API 由 Go 後端提供，前端負責渲染與互動
- [x] 14.12 「值班統計－告警通知」、「值班統計－域名更換」與「值班統計－支付域名更換」矩陣，需在第一個日期欄位前顯示「總計」欄；總計為該列在目前查詢日期區間內所有日期數值的合計，適用於指標及人員列。畫面不顯示早班、中班、晚班的班別彙總列，但保留含班別前綴的個別人員列；不改變既有每日數值與 API 契約
- [x] 14.13 值班報表顯示名稱統一使用「值班統計」，三張指定報表標題分別為「值班統計－告警通知」、「值班統計－域名更換」與「值班統計－支付域名更換」；僅調整顯示名稱，不變更 API、程式識別值或資料庫
- [x] 14.14 報表控制列將值班統計的統計項目移至「數值」欄位：報表模式僅顯示「值班統計」，選取後「數值」提供告警通知、域名更換及支付域名更換；送出的 `report_mode = E`、`metric_groups` 與既有範本設定格式維持不變

---

### 需求 15：Jira CSV 匯入與統計報表

> **注意：Jira CSV 匯入與統計報表屬 Phase 2，MVP 階段暫不實作。**

**使用者故事**：身為值班組長，我希望能匯入 Jira 匯出的 CSV 檔案，讓系統儲存並統計運維執行的 Jira Ticket 數量，產出獨立的 Jira 執行報表。

#### 驗收條件

- [ ] 15.1 提供 Jira CSV 檔案上傳介面，所有專案成員皆可操作
- [ ] 15.2 匯入時解析 Jira 簡中匯出 CSV 欄位並完整存入獨立的資料庫表（`jira_issues`）；Jira CSV 匯入屬 Phase 2，固定支援以下簡中欄位，不提供欄位 mapping 設定：
  - 概要、问题关键字、问题ID、问题类型、状态、项目关键字、项目名称、项目类型、项目主管、项目描述、项目URL
  - 优先级、解决结果、经办人、报告人、创建者、创建日期、已更新、最近查看的、已解决、到期日
  - 表决、标签、描述、环境、管理关注列表
  - 初始预估、剩余的估算、耗费时间、工作量比率、Σ 原预估时间、Σ 预估剩余时间、Σ 耗费时间
  - 安全级别
  - 自定义字段(Epic状态)、自定义字段(Epic颜色)、自定义字段(Story Point)、自定义字段(史诗名称)、自定义字段(史诗链接)、自定义字段(等级)
- [ ] 15.3 以 **Issue Key**（例如 `QIEZ-1648`）與**创建日期**作為複合唯一鍵，重複匯入時自動跳過已存在的資料（不覆蓋、不報錯）
- [ ] 15.4 每次匯入完成後回傳匯入結果摘要：總筆數、新增筆數、跳過（重複）筆數、失敗筆數
- [ ] 15.5 匯入的 Jira 資料與系統內 Ticket 完全獨立，不建立關聯
- [ ] 15.6 提供 Jira 執行統計報表，統計維度包含：
  - 依经办人（Assignee）統計 Ticket 數量
  - 依时间区间篩選（创建日期）
  - 依专案（项目关键字）篩選
  - 依状态、优先级篩選
- [ ] 15.7 Jira 執行統計報表支援長條圖呈現（個人 × 數量）
- [ ] 15.8 Jira 執行統計報表可匯出 CSV，所有專案成員皆可操作

---

## Glossary

| 術語            | 說明                                                                                                                                                    |
| --------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 表單樹          | 以父子層級組織的表單目錄結構，每個節點具唯一 `form_key` 識別碼                                                                                          |
| form_key        | 表單樹節點的路徑式唯一識別碼，用於定位表單與綁定群組操作層級                                                                                            |
| 群組            | 用於批次賦予使用者表單操作層級的權限集合                                                                                                                |
| 操作層級        | 對表單節點的權限類型：`read`、`create`、`update`、`delete`                                                                                              |
| OIDC 群組對應   | 將外部 IdP 回傳的群組名稱轉換為內部 `global_role` 或權限群組成員關係的登入同步規則；不取代內部權限模型                                                     |
| Ticket          | 一筆運維事件 / 工作項目，事件類型由 `ticket_types` 管理，預設包含 Daily（日常巡檢）與 Issue（問題處理）                                                   |
| Ticket Type     | Ticket 事件類型主檔，用於定義 Daily / Issue 等事件分類，以及是否支援升級、SLA、允許優先級與預設優先級                                                   |
| Project         | 獨立的工作空間，包含成員、子專案、Ticket 與設定                                                                                                         |
| Sub Project     | 專案下的子分類，對應特定服務或業務線，例如 QIEZ、QT、DSP                                                                                                |
| Sub Project Key | 子專案在 Project 內的唯一識別碼，用於分類與篩選（例如 `QIEZ`、`QT`、`DSP`），不作為 Ticket 編號前綴                                                       |
| Daily           | 日常巡檢類任務，例如超簽後台地址檢查、每日簡訊數量檢查                                                                                                  |
| Issue           | 需要處理的問題或異常事件                                                                                                                                |
| Shift Group     | 班制，定義排班制度類型（fixed/rotating/oncall），例如正常班、三班制、四班二輪、On-call；對應 `shift_groups` 資料表                                      |
| Duty Shift      | 具體班別，定義班別名稱與起訖時間（例如早班 07:00-15:00），歸屬於某個班制；對應 `duty_shifts` 資料表                                                     |
| Shift           | 班別的統稱，屬於人員排班 / 假勤資料；Ticket 本身不保存班別快照，班別統計需依 `created_by` / `ticket_activities.actor_id` 與對應時間查詢人員班制或排班資料 |
| 代班            | 某工程師代替請假同事執行其班別任務，在 `user_schedules` 以代班記錄呈現（`is_substitute=TRUE`）；Ticket 僅記錄實際開單 / 更新人員                         |
| 變形工時制      | 依勞基法規定的彈性工時制度，三班制適用四週或八週變形工時，由 Global Setting 統一設定；影響單日最長工時上限與假期合規計算                                |
| 國定假日        | 人事行政總處公告的例假日，儲存於 `public_holidays`，預設手動匯入，預留 API 擴充                                                                         |
| 假別            | 個人請假類型（病假 / 事假 / 年假 / 補休），儲存於 `user_leave_records`，支援對接人事系統                                                                |
| Assignee        | Ticket 的主要負責人                                                                                                                                     |
| Collaborator    | 協助處理 Ticket 的協作成員                                                                                                                              |
| Ticket Activity | 記錄 Ticket 所有操作、留言、欄位異動與處理歷程的明細表（`ticket_activities`），包含操作人員、before / after 與內容                                      |
| Internal Note   | 僅專案成員可見的內部備註留言                                                                                                                            |
| Escalated       | Issue 升級狀態，表示問題需要更高層級介入                                                                                                                |
| 外部單號        | 外部系統的追蹤編號，例如 Jira 單號，選填                                                                                                                |
| 資訊來源        | Ticket 的開單觸發來源，例如 TG Alert、Mail、Zabbix Alert 等                                                                                             |
| Jira Issue      | 從 Jira 匯入的外部工單資料，儲存於獨立資料表，與系統 Ticket 無關聯                                                                                      |
| Issue Key       | Jira 工單的唯一識別碼，例如 `QIEZ-1648`，作為去重依據                                                                                                   |
| 報表範本        | 使用者自訂的報表設定組合（維度、指標、分組），儲存後可重複套用產出報表                                                                                  |
| 報表設計器      | 讓使用者選擇統計維度與指標、預覽並儲存範本的互動介面；模式 D 透過此設計器自訂維度                                                                         |
| 月報模式 C      | 整月人員 × 子專案堆疊長條圖，對應 OP「運維處理事件數量」月報版型                                                                                         |
| 月報模式 D      | 報表設計器自訂維度的月報 / 區間報表，可儲存為範本、不受 A / B / C 固定版型限制                                                                           |
| MFA             | Multi-Factor Authentication，多因子驗證，登入時需完成第二因子（TOTP）驗證                                                                               |
| TOTP            | Time-based One-Time Password，基於時間的一次性密碼，相容 Google Authenticator                                                                           |
| 附件            | Ticket 上傳的截圖或圖片檔案，儲存於本機或 S3 Private Bucket；Browser 僅透過 API 串流存取                                                                 |
| storage_key     | Local / S3 共用的儲存後端 key，格式為 `yyyy/mm/dd/{attachment_ulid}.{ext}`，`ext` 僅允許 `avif` 或 `webp`                                                |

---

## 非功能需求

- **效能**：
  - 一般 CRUD 與單筆查詢 API：P99 < 500ms
  - 全文搜尋、報表查詢、跨專案彙總等聚合查詢：P95 < 3s
  - 大量資料匯入 / 匯出（例如 Jira CSV 匯入、報表匯出、排班 Excel 匯出）採非同步 Job 處理，API 先回傳任務狀態與查詢識別碼
- **可用性**：系統可用性目標 99.9%
- **使用者回饋**：
  - 所有前端新增、修改、刪除、批量操作、立即執行、登入 / 登出、語系 / 主題切換、MFA 操作與業務狀態變更，成功與失敗皆必須顯示 Toast
  - Toast 需使用一致的位置、顏色、關閉行為與 i18n 文案；不得遮擋目前操作中的表單、Dialog 主要按鈕或必要輸入欄位
  - 失敗 Toast 優先顯示後端錯誤訊息；無後端訊息時依錯誤碼對應前端 i18n 預設文案
- **國際化**：
  - 前端所有 table、grid、Data Grid 與資料列表的欄位標題、分頁文字、排序選單、篩選選單、欄位管理選單、空資料提示、載入文字、錯誤提示與操作文字皆必須跟隨目前 i18n 語系切換
  - 使用 MUI X Data Grid 時必須設定對應 `localeText` 或共用 locale mapper，不得保留英文內建文案（例如 Sort by ASC、Filter、Manage columns）
  - 語系切換後已掛載的 table / grid 需即時更新顯示文字，不得要求重新登入或重新整理頁面
- **時區**：
  - 所有狀態變更、審計日誌與資料庫時間欄位以 UTC 儲存
  - 系統業務時區固定為 `Asia/Taipei`，排班、報表、日期篩選與 UI 日期顯示皆以此時區計算；Ticket ULID 由 PostgreSQL 產生，不依業務時區格式化
  - Docker runtime 需設定環境變數 `TZ=Asia/Taipei`
- **識別碼**：
  - 系統內部主鍵與對外 API resource id 統一使用 ULID（26 字元 Crockford Base32），由 PostgreSQL 透過 ULID extension / function 產生
  - 資料庫以 `CHAR(26)` 儲存 ULID，欄位預設值使用 `generate_ulid()`；外部系統 ID（例如 Jira Issue Key、外部單號）維持原始文字格式
  - 部署與 migration 前需先確認 PostgreSQL 已安裝選定的 ULID extension，並已建立 `generate_ulid()` wrapper function；若 extension 或 function 不存在，migration / app 啟動需失敗並提示安裝
- **安全性**：
  - 所有 API 需認證；敏感資料加密儲存；日誌保留期限由 Global Setting 控制，Ticket Activity Log 與安全 / 系統稽核日誌需分開設定（預設 Ticket Activity 長期保留，安全 / 系統稽核日誌保留 180 天）
  - Admin 操作日誌 API 不得只以 `detail` JSON 暫存 UI 欄位；後端需持久化並回傳 `username`、`method`、`path`、`status_code`、`result`、`duration_ms` 等列表欄位，並支援 `username`、`module`、`ip`、`method`、`status_code`、`date_range` 篩選。前端不得自行推測或偽造缺失欄位。
  - Admin 管理端非 GET 操作需統一寫入 `system_audit_logs`，涵蓋 users、roles、schedulers、settings、forms、groups、permissions 等管理路由；成功與失敗都要記錄，但不得保存密碼、secret、token、MFA secret 等敏感 request body。
  - 後端所有應用程式 log 與第三方 runtime / native library log 必須統一透過 `zlogger` 結構化輸出；`govips` / `libvips` log 不得直接寫入標準 `log` 或 stderr，需保留原始 domain、level 與 message，並帶入 `subsystem` / `component` 欄位
  - 安全政策類 API 需最小揭露且由後端做最終驗證；不得把前端可解密的混淆或加密 payload 當作安全控制
  - JWT secret、OIDC client secret、S3 secret、DB / Redis 密碼等機密設定不得經由前端 API 明文回傳；`system_settings.is_secret = TRUE` 的值僅可遮罩顯示或回傳是否已設定
  - 強制 HTTPS（TLS 1.2+），禁止 HTTP 存取
  - JWT 放於 `Authorization: Bearer` header，不使用 cookie 傳遞
  - CORS 設定白名單，僅允許前端 domain 跨域請求
  - API Gateway 層實施 Rate Limiting 防暴力攻擊
  - 圖片轉檔需限制原始檔大小、圖片寬高與總像素數，避免 decompression bomb 與 native memory 耗盡
- **部署架構**：
  - 前端（React）與後端（Go）使用 Multi-stage Dockerfile 打包為單一 image
  - Go backend 透過 `embed` 直接 serve React 靜態檔案，無跨域問題
  - 對外僅暴露單一 port（9898），由 Nginx/API Gateway 做 TLS termination
  - Go backend 使用 `govips` 並以 `CGO_ENABLED=1` 建置；build stage 與 runtime image 必須提供 libvips、WebP 所需的 libwebp，以及 AVIF 所需的 libheif
  - 應用程式啟動時需驗證 WebP / AVIF 輸出能力；缺少必要 native library 或 encoder 時，應用程式必須啟動失敗，不可進入 readiness
- **擴展性**：支援水平擴展；資料庫使用 PostgreSQL；快取使用 Redis
- **語言**：後端使用 Go 1.25+；前端使用 React 18+；遵循專案開發規範
