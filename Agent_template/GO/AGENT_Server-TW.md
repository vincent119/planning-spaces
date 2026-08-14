# Go 後端代理貢獻者指南

## 適用範圍與優先順序

- 本規範適用於此目錄及其子目錄的 Go 後端程式碼。
- 先遵循 repository 根目錄規範，再遵循本檔；更深層目錄的 `AGENTS.md` 可補充專屬規範。
- 不確定現有慣例時，先查程式碼、測試、Makefile 與開發文件；不得以記憶或猜測取代驗證。
- 資料庫、DDD、gRPC、可觀測性、CI/CD 與部署等進階議題，使用相應 skill 或專案文件；本檔只保留必要的品質與邊界規則。

## 修改前

- 先閱讀同一 package 的兩到三個相鄰檔案與測試，沿用既有目錄、檔名、型別與錯誤處理模式。
- 使用 `make help`、README、CONTRIBUTING 或開發文件確認可執行的 format、test、lint、build 與 generate 命令。
- 確認改動屬於既有需求與 task 範圍；不在範圍內時，依根目錄規範先補文件。
- 優先修正根本原因；不得以關閉 lint、弱化測試、忽略錯誤或大量型別斷言掩蓋問題。

## 程式碼風格

- 程式格式化與靜態檢查以專案既有命令為準，例如 `make fmt` 與 `make lint`。
- 註解回答「為什麼」，而不是「是什麼」；只記錄約束、過往事件、規範引用、相容性或例外。註解以完整句子與句號結尾。
- 斷言訊息應保持簡潔；測試名稱應提供主要上下文。不得以冗長訊息取代清楚的測試名稱。
- 在專案 Go 版本支援時，優先使用標準函式庫的 `slices`、`maps`、`cmp` 等套件，避免重複實作輔助函式或無必要的第三方依賴。
- 新檔案沿用資料夾既有 package；package 名稱全小寫、單字、無底線，避免 `util` 與 `common`。
- imports 依標準函式庫、第三方套件、專案內部套件分組；不得保留未使用 import 或引入循環依賴。
- 使用 Tab 縮排、UTF-8 無 BOM 與單一檔尾換行。
- 為所有導出的函數、類型和常數添加 godoc 註釋。
- 不要僅僅為了複述程式碼的功能而添加註解。

## 程式碼品質與錯誤處理

- 函式維持單一責任，使用 early return，避免不必要的巢狀 `else`。
- `context.Context` 為 I/O、HTTP、資料庫、queue 與外部服務邊界函式的第一個參數，且必須向下傳遞。
- 不得把 `context.Context` 存入 struct，也不得在 request path 任意使用 `context.Background()`。
- 立即處理錯誤；需補上下文時使用 `fmt.Errorf("情境: %w", err)`，跨層判斷使用 `errors.Is` 或 `errors.As`。
- 可預期錯誤以 `error` 回傳；不得以 `panic`、`log.Fatal` 或 `os.Exit` 處理 library、handler 或 worker 的正常錯誤流程。
- error string 以小寫開頭且不加結尾標點，除非專案既有規範不同。
- 縮寫遵循 Go 慣例，例如 `URL`、`HTTP`、`ID`、`API`。
- 避免 `any`、寬鬆型別斷言、全域可變狀態與未受控 goroutine；確有必要時以最小範圍說明原因。
- 公開 API 的設計優先清晰與零值可用；多個參數不表示必須使用 Functional Options，應依複雜度與既有慣例決定。

## 並行、資源與 HTTP Client

- 每個 goroutine 必須有結束機制，例如 `context.Context`、`WaitGroup` 或關閉 channel；關機時必須回應 `ctx.Done()`。
- 共享可變狀態必須受 `sync.Mutex`、`sync.RWMutex` 或經量測確認的無鎖設計保護。
- `defer Close()` 應放在取得資源後的適當 scope；不得在大型迴圈中累積 defer，必要時以匿名函式縮小 scope。
- slice 或 map 跨越可變所有權邊界時，判斷是否需要複製，避免呼叫端與被呼叫端意外共享可變資料。
- HTTP Client 僅保存設定與可重用的 `*http.Client`；不得保存單一請求狀態。
- HTTP 呼叫使用 `http.NewRequestWithContext`、設定合理 timeout，並在讀取 response 後關閉 `resp.Body`。
- 重用 Transport；重試只適用於冪等操作，必須有退避、上限與明確的可重試錯誤範圍。
- 重讀 `req.Body` 前必須建立安全副本並正確設定 `GetBody`；`io.Pipe` 與 multipart 寫入需維持明確的同步與關閉責任。

## JSON、時間與資料邊界

- 對外輸入與輸出型別使用一致的 `json` tag；採用 Viper 的設定型別同步提供 `mapstructure` tag。
- 可選欄位依 API 契約使用 `omitempty`；不得為了省略欄位而改變既有 response 語意。
- 外部 JSON 輸入在相容性允許時拒絕未知欄位，避免拼字錯誤被靜默忽略。
- 內部儲存與運算使用 UTC；JSON 時間格式使用 RFC3339，呈現層再依使用者時區格式化。
- 檔案讀取設定大小與格式限制；壓縮檔處理需防止 Zip Slip 與不受控解壓。

## API、輸入與安全性

- 所有外部輸入皆需驗證格式、長度、範圍、列舉值與權限。
- 修改 endpoint 時維持既有 request、response、錯誤碼與權限語意；破壞性變更必須先取得明確確認。
- 使用既有 DTO、驗證、錯誤處理與授權 middleware；不得在 handler 中自行建立不一致的契約。
- API 變更同步更新 OpenAPI、前端 schema／types、規格文件與相關測試。
- 不得記錄密碼、Token、連線字串、個資或其他敏感資訊；錯誤 response 不得洩漏 SQL、堆疊或內部設定。
- 使用參數化查詢；不得將使用者輸入拼接為 SQL、shell command、檔案路徑或 URL。
- 僅使用標準 `crypto/*` 實作密碼學；不得自行設計密碼學演算法。正則表達式、檔案處理與外部 URL 必須評估 ReDoS、路徑穿越與 SSRF 風險。

## Configuration、DI 與 API 文件

- 設定管理強制使用 `spf13/viper`。設定來源與優先順序必須明確定義；至少遵循環境變數高於設定檔，設定檔高於預設值的原則。
- 設定應先 unmarshal 至 typed Config，再集中驗證；不得在業務程式碼分散讀取 Viper。
- 所有選填設定必須有明確預設值；所有必填設定必須於 startup 驗證。
- 環境變數名稱映射、巢狀 key 與空值行為必須有測試，避免不同環境讀到不同結果。
- 敏感資訊由環境變數、Secret 或受控設定服務注入，不得寫入 repository、範例設定或日誌。
- 必要設定缺失或無效時，application startup 必須明確失敗並回傳至統一生命週期管理入口；不得以不安全的隱含預設值繼續提供服務。
- Infrastructure 與 application 層依賴由明確 composition root 組裝。採用 `uber-go/fx` 時，集中於 `cmd/` 或明確 DI composition root；禁止業務邏輯層隨意建立具體 infrastructure。
- Repository 與 service 以小型 interface 表達真正需要的能力；具體實作維持非公開，測試替身依專案既有慣例建立。
- HTTP API 以 OpenAPI 作為可追溯契約；修改 endpoint、DTO、驗證或錯誤 response 時，必須更新專案的 OpenAPI 來源並重新產生文件。
- 採用 `swag` 的服務，必須同步更新 Swagger 註解，並使用專案指定的 `swag init` 命令產生文件；不得手動修改產生檔。

## 採用 gRPC 時

- Proto 檔案集中於專案指定目錄，產生碼不得手動修改。
- 新增或修改 RPC 時，必須重新產生程式碼並一併提交；server 必須尊重 client deadline 與 `context.Context` 取消。
- Interceptor 順序維持一致：recovery、observability、logging、authentication／authorization。
- 實作 gRPC health checking；將 domain error 映射為正確的 gRPC status code。
- 服務關閉時使用 `GracefulStop` 或具 timeout 的停止流程，避免中斷進行中的 RPC。

## 採用 Domain Events 時

- Event 為不可變 struct，至少包含事件識別碼、發生時間、Aggregate 識別碼與事件類型。
- 事件命名使用過去式動詞，例如 `OrderCreated`、`PaymentCompleted`。
- 同一 Bounded Context 可同步傳遞事件；跨 Bounded Context 的非同步事件需使用明確訊息邊界。
- 使用 Outbox Pattern 時，業務操作與事件寫入必須位於同一 DB transaction；發布 worker 需可重試並記錄完成狀態。
- Consumer 必須可冪等處理重複事件，以事件識別碼或等效去重鍵去重。

## 生命週期與日誌

- HTTP server、worker、consumer 與 background goroutine 的生命週期統一使用 `github.com/vincent119/commons/graceful` 管理。
- 啟動後的工作必須可由 `context.Context` 取消；關機時先停止接收新工作，再等待進行中工作完成，最後關閉資料庫、cache、queue、trace 等外部資源。
- 不得在 server goroutine 或 library 層呼叫 `os.Exit`、`log.Fatal`；錯誤應回傳至統一的生命週期管理入口。
- 全部應用程式日誌強制使用 `github.com/vincent119/zlogger`；不得混用標準 `log`、`slog`、`zap`、`logrus` 或其他 logger。
- 日誌採結構化欄位；跨服務或請求路徑保留既有的 `trace_id`、`span_id`、`req_id` 與 `subsystem` 等關聯欄位。
- 可觀測性採既有 OpenTelemetry 設定；跨邊界呼叫必須傳遞 `context.Context`。
- Metrics 命名使用一致的蛇形與單位後綴，例如 `_total`、`_seconds`、`_bytes`；不得使用 `user_id`、email、`trace_id` 等高基數 label。
- 不得將密碼、Token、DSN、個資或完整外部 response 寫入日誌或 metrics label。

## 測試、產生檔與文件

- 新增可觀察行為或修正 bug 時，優先先寫會失敗的測試，再實作最小可行修正，最後重構。
- 修正 bug 必須先加入可重現問題的 regression test；修正前失敗，修正後通過。
- 測試名稱描述可觀察的行為與結果，不描述私有實作細節。
- 優先使用 table-driven tests 覆蓋輸入、預期輸出與錯誤情境；測試檔案使用 `*_test.go`，測試 package 採用專案既有慣例。
- 使用 `t.Helper()` 標記輔助函式、使用 `t.Cleanup()` 清理資源；需要並行或競態驗證時，執行 `-race`。
- 優先隔離 domain、service、parser、validator、轉換函式與 query planner；repository、HTTP、資料庫與外部服務整合測試只驗證邊界契約。
- 關鍵演算法或解析器可視風險新增 benchmark 與 fuzz test；不得把它們當成所有小型修改的固定門檻。
- 不得為了通過測試而放寬 assertion、刪除／跳過測試，或將核心邏輯搬到測試外。
- 文件、註解、純格式化及已由既有測試完整覆蓋的機械調整不強制先寫測試；仍須執行與異動相稱的既有驗證。
- 任何由 schema、API 或程式碼產生的檔案，必須使用專案既有命令重新產生並一併提交。
- 若測試或驗證無法執行，需明確說明原因、未驗證範圍與風險。
- 交付前執行 `git diff --check`。

## 資料與 Migration

- 不得在 application startup 執行 schema migration。
- Production 禁止使用 ORM 的自動 schema migration，例如 GORM `AutoMigrate`。
- 結構異動使用版本化 migration；不得修改已在任何環境套用的 migration。
- 新增索引、View、Materialized View、彙總表或資料回填前，先量測查詢計畫、資料量、鎖定風險與預期效益。
- migration、資料回填、排程與回滾必須有可追溯的部署程序；細節以專案實際工具與文件為準。
- 發現既有架構、資料、migration 或設定不符合本規範時，先回報差異、影響與遷移方案；除非 task 已包含或取得明確授權，不得順帶進行大規模修正。

## 文件、變更與交付

- 面向使用者的功能、API、設定、部署或維運行為變更時，必須更新對應文件。
- 將新內容整合到既有文件結構，例如 `docs/`、README、OpenAPI、runbook 或私有規格工作區；不得建立內容重複的平行文件。
- 文件須描述目前已驗證的行為，不得把規劃中或未驗證的內容寫成既成事實。
- 每個變更保持小而聚焦；不得順帶重構無關模組。
- 新增依賴、變更架構、調整共用設定或影響部署行為前，先說明必要性與影響並取得確認。
- 交付時說明變更檔案、行為影響、驗證結果，以及 migration、設定或部署注意事項。
- commit 與 PR 文字如實描述已驗證範圍；不得把規劃、工具或未驗證結果稱為完成。

##  現況工具與遷移限制

- Go 版本：`1.25.0`，以 `go.mod` 為準。
- API 文件：`go run ./cmd/openapi -output Docs/openapi.json`。
- Swagger：尚未採用 `swag`；若導入，需定義產生指令與輸出位置。
- DI：目前使用既有 constructor／server composition；採用 `uber-go/fx` 時，集中於 `cmd/` 或明確 DI composition root。
- gRPC／Proto：尚未採用；導入時需定義 Proto 位置、Buf／protoc 產生命令與 Health Check。
- Graceful shutdown：`github.com/vincent119/commons/graceful`。
- Logging：`github.com/vincent119/zlogger`。
- Migration：目前使用 `sql/` 的連號版本化 SQL migration；不得修改已套用檔案。Migration 自動化工具尚未定案。
- 日誌／metrics／trace：結構化日誌使用 `zlogger`；metrics／trace 依既有 OpenTelemetry 設定與實作。
- 部署前置條件：先執行尚未套用的版本化 SQL migration，再部署應用程式；不得由 API 啟動流程執行 migration。
