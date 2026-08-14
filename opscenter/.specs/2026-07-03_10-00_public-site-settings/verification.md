# 公開站台設定整合驗收紀錄

## 驗收範圍

- 後端公開 API：`GET /api/v1/public/site-settings`
- 管理端相容性：既有 Global Setting 管理頁與 `/admin/global-settings` API
- 前端接線：登入頁、強制 MFA setup 頁、主版面、`document.title`
- 安全邊界：公開 API 僅回傳 `site.name` 與 `site.logo_url`

## 驗收結果

| 項目 | 結果 | 依據 |
| --- | --- | --- |
| Global Setting 管理端相容 | 通過 | 管理端仍使用 `listGlobalSettings`、`getGlobalSetting`、`createGlobalSetting`、`updateGlobalSetting` 串接 `/admin/global-settings`；未新增另一套站台設定 CRUD |
| 公開 API 白名單 | 通過 | `GetPublicSiteSettings` 固定只讀 `general.site_name` 與 `general.site_logo` |
| 公開 API 不需登入 | 通過 | `RegisterPublicSiteSettings` 註冊於 auth middleware 前；`TestPublicSiteSettingsAllowsAnonymousAndHidesSettingMetadata` 覆蓋未登入情境 |
| 敏感設定不外洩 | 通過 | delivery 測試確認 response 不包含 `category`、`value_type`、`description`、`sort_order`、`is_secret`、`updated_by`、`updated_at` 與 `security.mfa.force_enable` |
| API 失敗 fallback | 通過 | `usePublicSiteSettings` 捕捉 API 錯誤並回 `Opscenter` 預設設定 |
| Logo 載入失敗 fallback | 通過 | `SiteBrandMark` 的圖片 `onError` 會呼叫 `markLogoFailed`，改顯示站名縮寫 |

## 情境紀錄

### 1. 設定正常

API response 契約：

```json
{
  "code": 0,
  "message": "ok",
  "data": {
    "site": {
      "name": "Ops Portal",
      "logo_url": "/brand/logo.svg"
    }
  }
}
```

畫面結果：登入頁、強制 MFA setup 頁、主版面品牌文字會使用 `Ops Portal`；`document.title` 會同步為 `Ops Portal`。

### 2. 設定缺值

API response 契約：

```json
{
  "code": 0,
  "message": "ok",
  "data": {
    "site": {
      "name": "Opscenter",
      "logo_url": null
    }
  }
}
```

畫面結果：品牌文字與 `document.title` 回到 `Opscenter`，Logo 使用文字縮寫 fallback。

### 3. API 失敗

API response 契約：

```json
{
  "code": 500,
  "message": "public site settings load failed"
}
```

畫面結果：前端資料層捕捉錯誤後回 `Opscenter` 預設設定，不中斷登入、強制 MFA setup 與主版面流程。

### 4. Logo 載入失敗

API response 契約：

```json
{
  "code": 0,
  "message": "ok",
  "data": {
    "site": {
      "name": "Custom Portal",
      "logo_url": "/broken-logo.png"
    }
  }
}
```

畫面結果：圖片載入失敗後改顯示 `CP` 站名縮寫，不造成白畫面。

## 執行紀錄

- `go test ./internal/setting`：通過
- `npm run typecheck`：通過
- `npm run build`：通過
- `git diff --check`：通過

## 限制

- 本機後端 `127.0.0.1:9998` 未啟動，因此沒有使用本機 HTTP server 做 curl 驗收。
- 目前環境無法啟動可用的瀏覽器自動化；畫面驗收以型別檢查、production build 與前端 fallback 接線檢查為準。
