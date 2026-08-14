# 需求文件：Policy Validation CLI

## 來源

- Draft: 無
- Type: Feature
- Owner: Vincent
- Status: Complete

## 文件定位

本 spec 接續 `.specs/2026-08-04-15-14_Feature-policy-directory-deployment`，為既有 workload policy loader 提供部署前 CLI 與 CI validation gate。只重用並補強 `domain.LoadPolicy` 的錯誤定位，不重寫授權決策、policy schema、啟動流程或 Kubernetes deployment。

參考來源：

- 需求來源: 使用者確認下一階段實作 Policy Validation CLI 與 CI gate
- 既有文件: `docs/config.zh-Hant.md`、`docs/deploy.zh-Hant.md`
- 既有程式碼: `cmd/vault-agent/main.go`、`internal/syncer/domain/policy.go`、`.github/workflows/docker-publish.yml`

## 背景

Kubernetes deployment 已預設從 `/app/policy` 載入多個 YAML。現況只有 application startup 會驗證 policy；錯誤 policy 進入 ConfigMap 後才會使 Pod 啟動失敗。既有 loader 已具備 strict YAML、版本、規則欄位、目錄排序、跨檔版本與重複 rule ID 驗證，可作為 CLI 的唯一驗證來源。

## 問題陳述

缺少不啟動 server 即可執行的 policy validation 入口，也缺少 CI gate。現有 semantic error 未一致標示來源檔名與規則位置，分檔後排錯成本偏高。

## 目標

1. 提供 `vault-agent policy validate --file <path>`。
2. 提供 `vault-agent policy validate --dir <path>`。
3. 成功回傳 exit code `0`，policy 錯誤回傳 `1`，CLI 使用方式錯誤回傳 `2`。
4. 單檔與目錄模式直接重用 `domain.LoadPolicy`，避免驗證語意分叉。
5. semantic error 包含 basename 與 `webhook_rules[n]` 或 `sync_rules[n]`，但不輸出 backend path、keys 或完整 YAML。
6. Makefile 與 PR CI 同時驗證單檔與目錄範例。

## 非目標

1. 不新增或變更 policy schema。
2. 不新增 hot reload、auto-fix、formatter 或 policy generator。
3. 不驗證 Vault/AWS path 是否實際存在或 caller 是否有 server-side permission。
4. 不輸出完整解析後 policy。
5. 不導入第三方 CLI framework。

## 已定決策

- CLI 整合至既有 `vault-agent` binary，子命令固定為 `policy validate`。
- `--file` 與 `--dir` 必須且只能指定一個。
- runtime CLI 訊息維持英文；使用者文件使用繁體中文。
- policy validation 不載入 application config，也不初始化 logger、Kubernetes、Vault 或 AWS client。
- CI 使用 repository 內的 `configs/policy.sample.yaml` 與 `configs/policies/`。

## 待確認項目

- 無。

## 現有行為

- `vault-agent` 不解析 command arguments，執行後直接載入完整 application config。
- policy 只在 authorization 啟用時於 server startup 載入。
- 目錄檔案使用排序後的 `.yaml`、`.yml` 清單合併。
- strict YAML 與 semantic validation 已存在，但部分錯誤只有 rule ID，未標示來源檔案與規則索引。

## 新行為

- 有 command arguments 時進入 CLI dispatcher，不啟動 server。
- `policy validate` 解析互斥的 `--file` 或 `--dir`，呼叫 `domain.LoadPolicy`。
- 成功時 stdout 輸出 `policy validation succeeded`。
- policy 無效時 stderr 輸出安全的錯誤摘要並回傳 `1`。
- command 或 flags 無效時顯示 usage 並回傳 `2`。
- PR workflow 在 container build 前執行 `make policy-validate`。

## 影響範圍

- 使用者: policy 維護者、平台工程師、CI 維護者
- 功能: binary CLI dispatch、policy error location、CI release gate
- API / CLI: 新增 `vault-agent policy validate`
- Data / Storage: 無 migration；policy schema 不變
- 文件 / 安裝 / 發布: Makefile、GitHub Actions、繁中設定與部署文件

## 使用情境

- 作為 policy 維護者，我想在提交或部署前驗證單檔或目錄，以便阻止無效 policy 進入 ConfigMap。

## 驗收情境

### 情境：驗證有效單檔

- 場景：使用者指定有效 policy YAML
- 測試：`TestRunCLI_ValidatePolicyFileSuccess`
- 假設：檔案符合 v1 schema 與 semantic rules
- 當：執行 `vault-agent policy validate --file <path>`
- 那麼：exit code 為 `0`，stdout 顯示成功訊息，stderr 為空

### 情境：驗證有效目錄

- 場景：目錄包含可合併的多個 policy YAML
- 測試：`TestRunCLI_ValidatePolicyDirectorySuccess`
- 假設：檔案依名稱排序後版本一致且 rule ID 不重複
- 當：執行 `vault-agent policy validate --dir <path>`
- 那麼：exit code 為 `0`

### 情境：拒絕無效 policy 且不洩漏內容

- 場景：rule 使用無效 backend 且 policy 內含敏感 path marker
- 測試：`TestRunCLI_InvalidPolicyReportsLocationWithoutSensitiveValues`
- 假設：rule 位於已知 basename 與規則索引
- 當：執行 validation
- 那麼：exit code 為 `1`，錯誤含 basename 與規則位置，不含 path marker 或完整 YAML

### 情境：拒絕無效 CLI 使用方式

- 場景：未提供來源、同時提供兩種來源或傳入未知 command
- 測試：`TestRunCLI_UsageErrors`
- 假設：沒有啟動 application dependencies
- 當：執行 CLI
- 那麼：exit code 為 `2` 並輸出 usage

### 情境：既有 server 與授權行為不被破壞

- 場景：無 command arguments 啟動既有 server path，並執行 policy/domain 回歸測試
- 測試：`go test -race -count=1 ./...`
- 假設：既有 application config 與 policy 不變
- 當：執行全套測試
- 那麼：既有啟動 helper、授權決策與 loader 測試維持通過

## 驗收條件

1. 兩個 CLI contract、互斥來源與 exit codes 由 unit tests 覆蓋。
2. CLI 不呼叫 `configs.LoadConfig` 或初始化外部 client。
3. 單檔及目錄共用 `domain.LoadPolicy`。
4. semantic error 提供 basename 與 rule location，且敏感 path marker 不出現在輸出。
5. `make policy-validate` 驗證 repository 內單檔與目錄範例。
6. GitHub Actions 在 image build 前執行 policy validation。
7. race test、vet、CLI smoke test、Kustomize render 與 diff 檢查通過。

## 驗證需求

- Unit / Integration: `go test ./cmd/vault-agent ./internal/syncer/domain`
- CLI / Dry-run: `go run ./cmd/vault-agent/ policy validate --file configs/policy.sample.yaml` 與 `--dir configs/policies`
- 文件檢查: CLI、exit codes、CI command、file/dir 互斥與安全錯誤說明
- 回歸驗證: `go test -race -count=1 ./...`、`go vet ./...`、base/prod Kustomize render

## 風險與假設

| 類型 | 內容 | 處理方式 |
|------|------|----------|
| 風險 | CLI 與 startup 使用不同驗證路徑 | 兩者只呼叫 `domain.LoadPolicy` |
| 風險 | 錯誤輸出洩漏 policy resource 值 | error 只輸出 basename、規則位置、rule ID 與錯誤類型；以 marker test 驗證 |
| 風險 | CLI dispatch 破壞無參數 server path | 只在 arguments 非空時 dispatch，執行 main package 回歸測試 |
| 假設 | repository 範例代表 CI 最低驗證集合 | Makefile 同時驗證 canonical file 與 policies directory |

## 摘要

- 關鍵決策: 單一 binary、重用 loader、file/dir 互斥、三種 exit code、安全錯誤定位
- 待確認項目: 無
- 風險: CLI dispatch 回歸與錯誤資訊洩漏
- 下一步: 依 tasks 實作測試、CLI、定位資訊、Makefile、CI 與文件
