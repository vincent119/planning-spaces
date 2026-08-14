# 需求文件：OTLP 傳輸安全預設值

## 來源

- Type：BugFix
- Status：Planned
- 需求來源：2026-08-14 全域程式碼審查的 High 發現「OTLP 無條件使用明文傳輸」

## 文件定位

本 spec 修正 `internal/infra/telemetry/tracer.go` 對所有遠端 OTLP exporter 無條件使用 `WithInsecure()` 的行為。它接續既有 telemetry 設定、OTLP header 與 Basic Auth 功能；不改變 Webhook TLS、Vault 連線、metrics server、Secret Fetch、授權或 Kubernetes 部署拓撲。

既有契約：

- `telemetry.otlp_endpoint` 為空時使用 stdout exporter。
- endpoint 非空時，依 `telemetry.otlp_transport` 使用 `grpc` 或 `http` exporter。
- `telemetry.otlp_headers` 與 `telemetry.otlp_basic_auth` 可攜帶 OTLP 認證資訊。
- `app.env=prod` 已用於限制部分不安全設定。

## 背景

目前 HTTP 與 gRPC OTLP exporter 都設定不安全傳輸。使用者只要設定 OTLP header 或 Basic Auth，就會將認證資訊及遙測資料經由未加密連線傳送。這使網路路徑上的攻擊者可讀取或竄改資料，屬於 CWE-319。

## 目標

1. 遠端 OTLP exporter 預設使用 TLS。
2. `telemetry.otlp_endpoint` 支援以完整 `https://` URL 指定 OTLP collector；HTTP transport 保留 URL 的 path，gRPC transport 取出 host 與 port 作為連線 endpoint。
3. 支援使用系統信任根或指定 CA 檔案驗證 OTLP collector。
4. 支援設定 TLS ServerName，以處理 endpoint 使用內部位址但憑證使用不同 DNS 名稱的情境。
5. 僅允許非 production 環境以明確設定使用不安全傳輸。
6. 在設定載入階段 fail fast，避免 production 於啟動後才以不安全模式送出資料。
7. 保持 stdout exporter、既有 compression、header 與 Basic Auth 語意不變。
8. 不在 log、error 或 metrics 輸出 CA 內容、header、Basic Auth 值或其他 credential。

## 非目標

1. 不新增雙向 TLS、用戶端憑證、憑證輪替或 SPIFFE 整合。
2. 不變更 OTLP transport 種類、compression 或 header 解析規則；僅擴充 endpoint 以接受完整 URL。
3. 不替 OTLP exporter 新增重試、buffer、queue、sampling 或 telemetry health check。
4. 不修改 Webhook、operations server、Vault/OpenBao 或 AWS 的 TLS 行為。
5. 不修改既有 logger、metrics label 或 trace 資料內容。

## 已定決策

新增 telemetry 設定：

```yaml
telemetry:
  otlp_insecure: false
  otlp_tls_ca_file: ""
  otlp_tls_server_name: ""
```

- `otlp_insecure` 預設為 `false`；endpoint 非空時即使用 TLS。
- `otlp_endpoint` 向後相容既有 `host:port` 格式，並新增完整 URL 格式。`https://` 表示 TLS；`http://` 僅在 `otlp_insecure=true` 且非 production 時有效。含 scheme 的 gRPC endpoint 不得有 path、query 或 fragment。
- `otlp_insecure=true` 只允許 `app.env` 不是 `prod` 的環境；production 必須在設定驗證時拒絕啟動。
- `otlp_tls_ca_file` 為空時使用系統信任根；非空時讀取 PEM CA 檔並加入 TLS trust store。
- `otlp_tls_server_name` 為空時由 TLS 依 endpoint 推導；非空時作為憑證名稱驗證目標。
- endpoint 為空時，TLS 相關設定不影響 stdout exporter；為降低非必要啟動阻礙，不拒絕它們。
- 環境變數固定為 `TELEMETRY_OTLP_INSECURE`、`TELEMETRY_OTLP_TLS_CA_FILE`、`TELEMETRY_OTLP_TLS_SERVER_NAME`。

## 影響範圍

- Config：TelemetryConfig、環境變數綁定、預設值與 production validation。
- Infra：HTTP 與 gRPC OTLP exporter 的 TLS options。
- Tests：設定驗證、TLS config 建構、HTTP／gRPC exporter 初始化與既有 header 行為。
- Deployment：範例與 base manifest 顯式宣告安全預設。
- 文件：config、observability、README 的中英文說明。

## 使用情境

- 作為平台工程師，我希望只設定 OTLP endpoint 就安全地連線到 TLS collector，不必額外指定不安全開關。
- 作為使用私有 CA 的維運人員，我希望指定 CA 檔與 ServerName，使內部 collector 的憑證可正確驗證。
- 作為開發者，我需要連至本機明文 collector 時，必須明確設定 `otlp_insecure=true`。
- 作為資安人員，我希望 production 使用明文 OTLP 時在啟動前失敗，避免認證 header 外洩。

## 驗收情境

### 場景：TLS 為遠端 exporter 預設行為

- 測試：`TestInitTracer_UsesTLSByDefault`
- 假設：提供非空 endpoint，未設定 `otlp_insecure`
- 當：初始化 HTTP 或 gRPC exporter
- 那麼：兩種 exporter 都使用 TLS 設定，且不呼叫不安全傳輸 option。

### 場景：HTTPS URL 可作為 OTLP endpoint

- 測試：`TestInitTracer_AcceptsHTTPSURL`
- 假設：HTTP transport 使用含 path 的 `https://collector.example:4318/v1/traces`，gRPC transport 使用 `https://collector.example:4317`
- 當：初始化 exporter
- 那麼：HTTP 使用完整 HTTPS URL 與 path；gRPC 以 `collector.example:4317` 建立 TLS 連線；兩者都不使用明文傳輸。

### 場景：不安全 URL 與 TLS 模式不相容

- 測試：`TestTelemetryConfigValidate_RejectsHTTPURLWithoutExplicitInsecureMode`
- 假設：endpoint 使用 `http://` 且 `otlp_insecure=false`
- 當：驗證設定
- 那麼：設定被拒絕；錯誤不包含 endpoint credential 或 header。

### 場景：非 production 明確允許本機明文 collector

- 測試：`TestTelemetryConfigValidate_AllowsExplicitInsecureOutsideProduction`
- 假設：`app.env=dev`、endpoint 非空且 `otlp_insecure=true`
- 當：載入並驗證設定
- 那麼：設定通過，HTTP 與 gRPC exporter 使用不安全傳輸。

### 場景：production 拒絕明文 OTLP

- 測試：`TestTelemetryConfigValidate_RejectsInsecureInProduction`
- 假設：`app.env=prod`、endpoint 非空且 `otlp_insecure=true`
- 當：驗證設定
- 那麼：回傳不包含 credential 的固定錯誤，且 server 不開始監聽。

### 場景：使用私有 CA 與自訂 ServerName

- 測試：`TestBuildOTLPTLSConfig_CustomCAAndServerName`
- 假設：提供有效 PEM CA 檔與 ServerName
- 當：建立 OTLP TLS config
- 那麼：RootCAs 含指定 CA，ServerName 等於設定值，且未關閉憑證驗證。

### 場景：無效 CA 在啟動前被拒絕

- 測試：`TestBuildOTLPTLSConfig_RejectsInvalidCAFile`
- 假設：CA 路徑不存在或內容不是有效 PEM
- 當：初始化 tracer
- 那麼：回傳安全的初始化錯誤；錯誤不得包含 header、Basic Auth 或 CA 內容。

## 完成條件

1. 所有驗收情境有對應單元測試並通過。
2. `go test ./...`、`go test -race ./...`、`go vet ./...` 與 `golangci-lint run` 通過。
3. config sample、base deployment、繁中與英文文件皆採 TLS 預設，並記錄明文模式僅限非 production 的限制。
4. `git diff --check` 通過，且實際變更未超出 `tasks.md` 的 Boundary。
