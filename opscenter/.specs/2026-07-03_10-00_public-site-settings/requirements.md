# 公開站台設定需求

## 文件定位

本 spec 接續 `../2026-06-01_10-22_oncall-ticket-system` 已規劃的 `general.site_name` 與 `general.site_logo`，補齊前端公開讀取站台品牌設定的需求。

原始設計稿為初始設計基準，本文件不修改原始 spec。

## 背景

目前 `system_settings` 已存在一般設定類別，且原始設計已規劃下列設定：

- `general.site_name`
- `general.site_logo`

但前端仍在登入頁、側邊欄與頁面標題使用固定文字，例如 `Opscenter` 與 `Opscenter System`。若管理者已在 Global Setting 調整網站名稱，使用者介面不會同步反映。

站台名稱與 Logo 屬於可公開顯示資訊，但 `system_settings` 內同時包含安全、JWT、Webhook、儲存與排程設定。前端不得直接讀取完整 Global Setting 清單作為站台顯示來源，必須由後端提供最小化公開白名單 API。

## 範圍

本次範圍包含：

- 定義公開站台設定 API，只回傳前端顯示必要欄位。
- 站台名稱來源為 `system_settings.general.site_name`。
- 站台 Logo 來源為 `system_settings.general.site_logo`。
- 前端可於登入頁、強制 MFA setup 頁、主版面側邊欄與瀏覽器標題使用公開站台設定。
- 當設定缺值、停用、空白或 API 失敗時，前端需使用既有預設值，避免阻塞登入與主畫面。

本次範圍不包含：

- 完整品牌管理後台。
- Theme、色票、字型、favicon、背景圖管理。
- 讓未登入使用者查詢任意 `system_settings`。
- 讓前端直接讀取或快取敏感 Global Setting。
- 變更既有 Global Setting CRUD 權限模型。

## 需求 1：公開站台設定 API

系統需要提供不需登入即可讀取的公開站台設定 API，供登入前與登入後 UI 使用。

### 驗收條件

- [ ] 1.1 後端需提供 `GET /api/v1/public/site-settings`。
- [ ] 1.2 API 不需 access token、refresh token 或 `temp_token`。
- [ ] 1.3 API 只能回傳白名單欄位，不得回傳 `system_settings` 原始資料列。
- [ ] 1.4 API 不得回傳 `security.*`、`jwt.*`、`webhook.*`、`storage.*`、`system.*` 等設定。
- [ ] 1.5 API 不得回傳 `category`、`value_type`、`description`、`sort_order`、`is_secret`、`updated_by`、`updated_at`。
- [ ] 1.6 API 回應需符合既有 `httpx.APIResponse` envelope。

## 需求 2：站台名稱

前端需要從公開站台設定取得站台名稱，避免硬寫顯示文字。

### 驗收條件

- [ ] 2.1 `site.name` 來源為啟用中的 `general.site_name`。
- [ ] 2.2 `general.site_name` 缺值、停用、空白或只有空白字元時，後端需回傳預設值 `Opscenter`。
- [ ] 2.3 前端 API 失敗或資料格式不符時，需使用預設值 `Opscenter`。
- [ ] 2.4 `site.name` 為設定值，不走 i18n 翻譯；周邊 label 與錯誤訊息仍需走 i18n。
- [ ] 2.5 前端需逐步取代登入頁、強制 MFA setup 頁、主版面側邊欄與 `document.title` 的硬寫站名。

## 需求 3：站台 Logo

前端需要從公開站台設定取得站台 Logo URL，但不得造成不安全 URL 注入。

### 驗收條件

- [ ] 3.1 `site.logo_url` 來源為啟用中的 `general.site_logo`。
- [ ] 3.2 `general.site_logo` 缺值、停用或空白時，`site.logo_url` 需為 `null`。
- [ ] 3.3 後端只允許相對路徑、`https://` 或開發環境明確允許的 `http://` URL。
- [ ] 3.4 `javascript:`、`data:`、`file:` 與其他不支援 scheme 不得回傳給前端。
- [ ] 3.5 前端載入 Logo 失敗時需回退到既有文字或縮寫 avatar，不得讓版面破裂。

## 需求 4：前端載入與回退

公開站台設定不得阻塞登入流程或主系統初始化。

### 驗收條件

- [ ] 4.1 前端需在登入頁與主系統 shell 載入公開站台設定。
- [ ] 4.2 載入期間先顯示預設值，不顯示空白品牌文字。
- [ ] 4.3 API 失敗時不得將使用者導回登入頁，也不得中斷 MFA setup。
- [ ] 4.4 前端可使用短時間快取，但使用者重新整理頁面後需可取得最新設定。
- [ ] 4.5 若使用者已登入，公開站台設定 API 仍不得依使用者權限回傳不同敏感資料。

## 需求 5：安全與驗證

公開 API 需要保持最小揭露，並避免未登入環境成為設定探測入口。

### 驗收條件

- [ ] 5.1 後端需以固定 key 白名單查詢，不接受 client 指定 key。
- [ ] 5.2 DB 查詢錯誤需回傳標準錯誤 envelope，且前端需自行回退預設值。
- [ ] 5.3 後端日誌不得輸出完整 `system_settings` 清單。
- [ ] 5.4 後端測試需覆蓋正常值、缺值、停用值、空白值、無效 Logo URL 與 DB 錯誤。
- [ ] 5.5 前端驗證需覆蓋成功載入、API 失敗回退與 Logo 載入失敗回退。
