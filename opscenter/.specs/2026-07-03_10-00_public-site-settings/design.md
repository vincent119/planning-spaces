# 公開站台設定設計

## 文件定位

本文件描述前端如何透過公開白名單 API 取得站台名稱與 Logo。需求來源為本 spec 的 `requirements.md`，並對齊原始 spec `../2026-06-01_10-22_oncall-ticket-system` 已規劃的 `general.site_name` 與 `general.site_logo`。

原始設計稿不在本次修改範圍。

## 設計目標

- 前端不再以硬寫文字作為站台名稱唯一來源。
- 未登入頁面可以取得站台品牌設定。
- 後端只暴露公開白名單欄位，不暴露 Global Setting 管理資料。
- 公開設定失敗時不影響登入、MFA setup 與主系統使用。
- 保留既有 Global Setting 管理端，不新增另一套品牌設定 CRUD。

## API 設計

新增公開端點：

```text
GET /api/v1/public/site-settings
```

路由位置：

- 掛在 `/api/v1/public`。
- 不套用一般 auth middleware。
- 可套用 request id、access log、rate limit 與 CORS。

不建議塞入既有 `/api/v1/auth/config`，原因如下：

- `/auth/config` 目前語意是登入驗證相關設定，例如 OIDC 入口。
- 站台名稱與 Logo 屬於系統外觀資料，不應綁在 Auth namespace。
- 未來若新增公開 help URL、support URL 或 favicon，可以維持在 public namespace 擴充。

## Response 契約

成功回應沿用既有 `httpx.APIResponse`：

```json
{
  "code": 0,
  "message": "ok",
  "data": {
    "site": {
      "name": "Opscenter",
      "logo_url": null
    }
  },
  "trace_id": "01..."
}
```

欄位定義：

| 欄位 | 型別 | 說明 |
| --- | --- | --- |
| `site.name` | string | 顯示用站台名稱 |
| `site.logo_url` | string 或 null | 顯示用 Logo URL |

不得回傳：

- `system_settings.key`
- `system_settings.category`
- `system_settings.value_type`
- `system_settings.description`
- `system_settings.sort_order`
- `system_settings.is_secret`
- `system_settings.updated_by`
- `system_settings.updated_at`
- 設定來源或 fallback 來源

## 後端資料來源

固定查詢白名單 key：

```sql
SELECT key, value
FROM system_settings
WHERE key IN ('general.site_name', 'general.site_logo')
  AND is_active = TRUE;
```

解析規則：

| key | response 欄位 | 預設值 | 規則 |
| --- | --- | --- | --- |
| `general.site_name` | `site.name` | `Opscenter` | trim 後非空才使用 |
| `general.site_logo` | `site.logo_url` | `null` | trim 後通過 URL 白名單才使用 |

Logo URL 白名單：

| 格式 | 生產環境 | 開發環境 | 說明 |
| --- | --- | --- | --- |
| `/assets/logo.png` | 允許 | 允許 | 相對路徑 |
| `https://example.com/logo.png` | 允許 | 允許 | HTTPS |
| `http://127.0.0.1/logo.png` | 不允許 | 允許 | 僅本機開發 |
| `javascript:...` | 不允許 | 不允許 | 直接捨棄 |
| `data:image/...` | 不允許 | 不允許 | 第一版不支援 |

若 Logo URL 無效，後端回 `logo_url: null`，不讓前端自行判斷危險 scheme。

## 後端模組規劃

可採用 `internal/setting` 擴充公開讀取服務，或新增輕量 `internal/publicsetting`。第一版建議放在 `internal/setting`，原因是資料表與驗證規則已集中於 setting domain。

新增介面：

```go
type PublicSiteSettings struct {
	Site PublicSite `json:"site"`
}

type PublicSite struct {
	Name    string  `json:"name"`
	LogoURL *string `json:"logo_url"`
}
```

Repository 增加固定白名單讀取：

```go
FindActiveValuesByKeys(ctx context.Context, keys []string) (map[string]string, error)
```

Service 增加：

```go
GetPublicSiteSettings(ctx context.Context) (PublicSiteSettings, error)
```

Delivery 增加：

```go
func (h PublicHandler) SiteSettings(c *gin.Context)
```

錯誤處理：

| 情境 | HTTP | code | 前端行為 |
| --- | --- | --- | --- |
| DB 正常但設定缺值 | 200 | 0 | 使用後端預設值 |
| DB 正常但 Logo 無效 | 200 | 0 | `logo_url = null` |
| DB 查詢失敗 | 500 | 500 | 使用前端預設值 |

DB 查詢失敗不應偽裝成成功，避免維運無法觀測真正問題。

## 前端設計

新增 API client：

```ts
export interface PublicSiteSettingsResponse {
  site: {
    name: string;
    logo_url: string | null;
  };
}
```

新增 hook：

```ts
usePublicSiteSettings()
```

行為：

- `auth: false` 呼叫 `/public/site-settings`。
- 預設值為 `{ name: 'Opscenter', logo_url: null }`。
- 載入中直接回預設值。
- API 失敗時回預設值並保留錯誤供 console 或診斷使用。
- 可設定短時間 `staleTime`，避免每次 route 切換重打 API。

前端整合點：

| 位置 | 使用欄位 | 回退 |
| --- | --- | --- |
| 登入頁標題 | `site.name` | `Opscenter` |
| 強制 MFA setup 頁 | `site.name` | `Opscenter` |
| 主版面側邊欄主標 | `site.name` | `Opscenter` |
| 主版面側邊欄副標 | 保留固定產品副標或後續另開設定 | `Opscenter System` |
| `document.title` | `site.name` | `Opscenter` |
| Logo 顯示 | `site.logo_url` | 文字或縮寫 avatar |

第一版不要求全站所有字串一次替換，但登入頁、MFA setup 與主 shell 需優先完成，避免使用者最常看到的位置與設定不一致。

## 快取策略

後端：

- 不需要 Redis cache。
- 每次查詢只讀兩個 key，成本低。
- 可先不加 HTTP cache header，避免管理端修改後使用者長時間看不到變更。

前端：

- 可使用 React Query 或現有資料抓取工具。
- 建議 `staleTime` 為 5 分鐘。
- 頁面重新整理時重新讀取 API。

## 安全設計

安全邊界：

- API 不接受任意 key 查詢。
- API 不回傳原始 setting metadata。
- API 不回傳 `is_secret = TRUE` 的值。
- 即使未來 `general.site_logo` 被誤設為 secret，也不得在 public API 回傳。
- Logo URL 由後端正規化，前端只負責顯示與載入失敗回退。

授權：

- 讀取 public site settings 不需登入。
- 修改 `general.site_name` 與 `general.site_logo` 仍使用既有 Global Setting 管理端權限。
- 第一版不新增 `op_admin` 特殊覆寫規則。

## 測試規劃

後端測試：

- `general.site_name` 正常時回設定值。
- `general.site_name` 缺值時回 `Opscenter`。
- `general.site_name` 停用時回 `Opscenter`。
- `general.site_name` 空白時回 `Opscenter`。
- `general.site_logo` 正常相對路徑時回 URL。
- `general.site_logo` 正常 HTTPS 時回 URL。
- `general.site_logo` 無效 scheme 時回 `null`。
- DB 錯誤時回 500。
- response 不包含 `system_settings` metadata 與敏感 key。

前端驗證：

- API 成功時顯示設定站名。
- API 失敗時顯示 `Opscenter`。
- Logo 載入失敗時回退文字或縮寫 avatar。
- `document.title` 能隨站名更新。
- `npm run typecheck` 與 `npm run build` 通過。

## 實作順序建議

1. 後端新增 public site settings DTO、service、repository 與 route。
2. 補後端單元測試與 delivery 測試。
3. 前端新增 API client 與 `usePublicSiteSettings()`。
4. 前端接登入頁、強制 MFA setup 頁、主 shell 與 `document.title`。
5. 補前端型別檢查與 build 驗證。
