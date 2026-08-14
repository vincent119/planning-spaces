# 實作任務：OTLP 傳輸安全預設值

Status: Planned

## Execution Context

- 意圖：修正遠端 OTLP exporter 預設使用明文傳輸的安全問題，使 TLS 成為預設，並提供受限的非 production 明文相容模式。
- 非目標：不改變 Webhook、Vault/OpenBao、AWS、metrics server 或 OTLP collector 端；不實作 mTLS。
- 已定決策：`otlp_insecure=false` 為預設；`prod` 禁止 `otlp_insecure=true`；CA 檔與 ServerName 為選用欄位。
- 關鍵檔案：`internal/configs/config.go`、`internal/infra/telemetry/tracer.go`、其測試、config sample、base deployment 與 telemetry 文件。
- 完成條件：滿足 `requirements.md` 的所有驗收情境及品質檢查清單。

## Protected Behavior

- endpoint 為空時繼續使用 stdout exporter。
- OTLP transport 保持 `grpc` 預設與 `http` 選項。
- compression、header 解析與 Basic Auth 覆寫 Authorization header 的語意不變。
- 明文模式在非 production 必須由顯式設定啟用，不得因 endpoint 格式或設定缺漏自動降級。
- log、error、metrics 不得包含 header、Basic Auth、CA 內容或憑證內容。

## 任務

### T1：擴充 telemetry 設定與驗證

Status: Planned

- Boundary:
  - Allowed Changes：`internal/configs/config.go`、`internal/configs/config_test.go`。
  - Forbidden：不得修改 tracer exporter 邏輯、部署 manifest 或文件。
- Depends：無。
- Context：新增 `otlp_insecure`、`otlp_tls_ca_file`、`otlp_tls_server_name`，綁定三個環境變數；驗證 `https://`／`http://` endpoint 與安全模式的組合，並在 production 拒絕不安全模式。
- Verify：新增設定驗證表格測試，涵蓋 HTTPS URL、未明確啟用不安全模式的 HTTP URL、production 拒絕與既有 `host:port` 設定回歸；執行 `go test ./internal/configs/...`。

### T2：實作 HTTP 與 gRPC 共用 TLS config

Status: Planned

- Boundary:
  - Allowed Changes：`internal/infra/telemetry/tracer.go`、`internal/infra/telemetry/tracer_test.go`。
  - Forbidden：不得變更 OTLP header 與 Basic Auth 格式，不得新增 mTLS client certificate 功能。
- Depends：T1。
- Context：新增安全的 TLS config 與 endpoint 正規化函式；預設套用 TLS，支援完整 HTTPS URL，僅在設定明確要求且合法時套用 `WithInsecure()`；HTTP 與 gRPC 使用相同 CA／ServerName 輸入。
- Verify：新增 requirements 中的 HTTPS URL、TLS 預設、私有 CA、ServerName、無效 CA、HTTP 與 gRPC 模式測試；執行 `go test ./internal/infra/telemetry/...`。

### T3：同步範例、部署與文件

Status: Planned

- Boundary:
  - Allowed Changes：`configs/config.sample.yaml`、`deployments/kustomize/base/deployment.yaml`、`docs/config.zh-Hant.md`、`docs/config.en.md`、`docs/observability.zh-Hant.md`、`docs/observability.en.md`、`docs/README.zh-Hant.md`、`docs/README.en.md`。
  - Forbidden：不得變更 RBAC、Service、Webhook 設定、Secret 資料或其他 deployment security context。
- Depends：T1、T2。
- Context：所有範例採 TLS 預設；明文化私有 CA、ServerName 與明文模式只允許非 production 的限制；base deployment 顯式設定安全預設。
- Verify：檢查設定名稱與環境變數和程式一致；執行 `kubectl kustomize deployments/kustomize/base`；確認 diff 未包含不在 Boundary 的 manifest 變更。

### T4：整體回歸與安全驗收

Status: Planned

- Boundary:
  - Allowed Changes：僅測試修正與因驗證結果必要的 T1 至 T3 範圍內檔案。
  - Forbidden：不得擴張至 Webhook、Vault/OpenBao、AWS、metrics server 或其他審查發現。
- Depends：T1、T2、T3。
- Context：確認 production 無法以明文傳送 OTLP，且既有 stdout、HTTP、gRPC 與 header 行為未回歸。
- Verify：`go test ./...`、`go test -race ./...`、`go vet ./...`、`golangci-lint run`、`git diff --check`、`kubectl kustomize deployments/kustomize/base`。

## 品質檢查清單

- [ ] 所有新增 config key 有 mapstructure tag、環境變數綁定、預設值與範例文件。
- [ ] TLS mode 不使用 `InsecureSkipVerify`。
- [ ] production 明文模式在設定驗證階段失敗。
- [ ] CA 檔錯誤不洩漏檔案內容、header 或 credential。
- [ ] HTTP 與 gRPC 都有 TLS 與明文分支的測試。
- [ ] 所有既有 header、Basic Auth、compression 與 stdout exporter 測試仍通過。

## Implementation Notes

- 2026-08-14：依程式碼審查建立。本次僅完成需求、設計與任務文件，尚未開始 T1。
