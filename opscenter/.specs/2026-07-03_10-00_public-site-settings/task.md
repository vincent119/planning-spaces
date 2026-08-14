# 公開站台設定 Task

## 1. 後端公開 API

- [x] 1.1 建立公開站台設定 DTO
  - 定義 `PublicSiteSettingsResponse` 與 `PublicSiteResponse`
  - response 只包含 `site.name` 與 `site.logo_url`
  - 不暴露 `system_settings` metadata
  - 對應需求：1.3、1.5、1.6

- [x] 1.2 擴充 setting repository 固定白名單讀取
  - 新增依 key 清單讀取啟用設定值的方法
  - 僅查詢 `general.site_name` 與 `general.site_logo`
  - 不接受 client 指定任意 key
  - 對應需求：1.3、5.1

- [x] 1.3 建立公開站台設定 service
  - `general.site_name` 缺值、停用或空白時回 `Opscenter`
  - `general.site_logo` 缺值、停用或空白時回 `null`
  - Logo URL 無效時回 `null`
  - 對應需求：2.1、2.2、3.1、3.2、3.4

- [x] 1.4 建立 `GET /api/v1/public/site-settings`
  - route 不套用一般 auth middleware
  - route 仍套用 request id 與 access log
  - DB 錯誤時回標準錯誤 envelope
  - 對應需求：1.1、1.2、5.2

- [x] 1.5 補 OpenAPI 註解
  - 標示公開端點不需登入
  - 標示 response envelope 與欄位
  - 明確說明不回傳敏感設定
  - 對應需求：1.6、5.1

## 2. 後端驗證

- [x] 2.1 補 service 單元測試
  - 正常站名回設定值
  - 缺值、停用、空白站名回 `Opscenter`
  - 正常 Logo 相對路徑與 HTTPS 可回傳
  - 無效 Logo scheme 回 `null`
  - 對應需求：5.4

- [x] 2.2 補 delivery 測試
  - 未登入可呼叫 `GET /api/v1/public/site-settings`
  - response 不包含 `category`、`value_type`、`description`、`is_secret` 等欄位
  - DB 錯誤時回標準錯誤 envelope
  - 對應需求：1.2、1.5、5.2、5.4

- [x] 2.3 執行後端驗證
  - 執行相關 Go 測試
  - 確認不影響既有 setting 與 auth 測試
  - 對應需求：5.4

## 3. 前端公開設定資料層

- [x] 3.1 建立公開站台設定型別與 API client
  - 新增 `PublicSiteSettingsResponse`
  - 使用 `auth: false` 呼叫 `/public/site-settings`
  - 對應需求：4.1、4.5

- [x] 3.2 建立 `usePublicSiteSettings`
  - 載入中回預設值
  - API 失敗回預設值
  - 可設定短時間 `staleTime`
  - 對應需求：2.3、4.2、4.3、4.4

- [x] 3.3 建立 Logo 顯示回退邏輯
  - `logo_url = null` 時顯示文字或縮寫 avatar
  - 圖片載入失敗時回退
  - 對應需求：3.5

## 4. 前端畫面接線

- [x] 4.1 登入頁接站台設定
  - 登入頁標題使用 `site.name`
  - API 失敗時顯示 `Opscenter`
  - 對應需求：2.3、2.5、4.1

- [x] 4.2 強制 MFA setup 頁接站台設定
  - MFA setup 頁品牌文字使用 `site.name`
  - 公開設定失敗不得中斷 setup 流程
  - 對應需求：2.5、4.3

- [x] 4.3 主版面側邊欄接站台設定
  - 側邊欄主標使用 `site.name`
  - 副標第一版維持既有固定文字
  - 對應需求：2.5

- [x] 4.4 `document.title` 接站台設定
  - 有站台設定時使用 `site.name`
  - 載入失敗時使用 `Opscenter`
  - 對應需求：2.5、4.2

## 5. 前端驗證

- [x] 5.1 補前端行為驗證
  - API 成功時顯示設定站名
  - API 失敗時顯示 `Opscenter`
  - Logo 載入失敗時顯示 fallback
  - 對應需求：5.5

- [x] 5.2 執行前端驗證
  - 執行 `npm run typecheck`
  - 執行 `npm run build`
  - 確認登入頁、MFA setup 頁與主版面不因公開設定失敗而白畫面
  - 備註：目前專案未配置前端測試框架；本次以既有型別檢查、production build 與公開設定 fallback 接線檢查驗收
  - 對應需求：4.3、5.5

## 6. 整合檢查

- [x] 6.1 確認 Global Setting 管理端相容
  - `general.site_name` 與 `general.site_logo` 仍由既有管理端修改
  - 不新增另一套品牌設定 CRUD
  - 對應需求：範圍

- [x] 6.2 確認安全揭露邊界
  - 未登入呼叫公開 API 不可取得安全、JWT、Webhook、Storage、System 設定
  - response 不包含 `system_settings` 原始 metadata
  - 對應需求：1.4、1.5、5.1

- [x] 6.3 補手動驗收紀錄
  - 記錄設定正常、設定缺值、API 失敗、Logo 失敗四種情境
  - 附上實際 API response 與畫面驗收結果
  - 驗收紀錄：`verification.md`
  - 對應需求：4.1、4.2、4.3、5.5
