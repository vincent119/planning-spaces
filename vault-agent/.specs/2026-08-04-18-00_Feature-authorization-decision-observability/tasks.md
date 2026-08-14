# 任務文件：Authorization Decision Observability 與安全稽核

Status: Complete

## Execution Context

- 意圖：為Webhook／Sync workload authorization建立固定低基數decision metrics、選配安全audit log，並移除相鄰operational log的secret path/key洩漏路徑。
- 非目標：不改caller authentication、Admission contract、policy matching/hot reload、external policy source、dashboard/alert、tracing或演練。
- 已定決策：單一24-series counter；surface/decision/reason/backend固定taxonomy；backend normalize；audit預設false；identity第二層opt-in；category-only operational log。
- 邊界：config、metrics、application recorder與use cases、delivery safe log、Vault/AWS error redaction、main wiring、deployment、繁中文件與本spec。
- 關鍵檔案：`internal/syncer/application/authorization.go`、`mutate_usecase.go`、`sync_worker_usecase.go`、`internal/infra/metrics/metrics.go`、`internal/syncer/delivery/webhook_handler.go`
- 完成條件：所有驗收情境、24-series schema、exactly-once/pre-fetch、audit opt-in、marker redaction、race/vet/lint/policy validation/Kustomize/diff checks完成。

### Protected Behavior

- Caller authentication與AdmissionReview validation順序不變。
- `SecretAuthorizer`介面、policy matching、deny-by-default、RuleID與hot reload不變。
- Deny/not initialized在external fetch前停止；authorization disabled只bypass workload policy。
- Admission response維持generic `secret injection failed`。
- Sync conflict retry、Secret刪鍵、readiness、graceful shutdown與metrics basic auth不變。
- Metrics不得包含任何workload identity；namespace／ServiceAccount只能出現在明確identity opt-in的audit log。
- Metric與audit都不得包含path、keys、RuleID、Pod/Secret name、UID、fingerprint、credential或raw error。
- Runtime、error、log與Go測試失敗訊息使用英文；註解與文件使用繁體中文。
- `_workspace/`與外部Policy Source draft不得修改或納入提交。

### 邊界

#### Allowed Changes

- `internal/configs/config.go`
- `internal/configs/config_test.go`
- `internal/infra/metrics/metrics.go`
- `internal/infra/metrics/metrics_test.go`
- `internal/syncer/application/authorization.go`
- `internal/syncer/application/authorization_test.go`
- `internal/syncer/application/mutate_usecase.go`
- `internal/syncer/application/mutate_usecase_test.go`
- `internal/syncer/application/sync_worker_usecase.go`
- `internal/syncer/application/sync_worker_usecase_test.go`
- `internal/syncer/delivery/webhook_handler.go`
- `internal/syncer/delivery/webhook_handler_test.go`
- `internal/syncer/infra/vault_client.go`
- `internal/syncer/infra/vault_client_test.go`
- `internal/syncer/infra/aws_client.go`
- `internal/syncer/infra/aws_client_test.go`
- `cmd/vault-agent/main.go`
- `cmd/vault-agent/main_test.go`
- `configs/config.sample.yaml`
- `deployments/kustomize/base/deployment.yaml`
- `docs/config.zh-Hant.md`
- `docs/deploy.zh-Hant.md`
- `docs/annotations.zh-Hant.md`
- `.specs/2026-08-04-18-00_Feature-authorization-decision-observability/`

#### Forbidden

- 修改policy YAML schema、matching、RuleID或SecretAuthorizer介面
- 新增namespace、ServiceAccount、path、keys、RuleID、resource ID、UID、error text等metric label
- 將raw backend或Decision.Reason直接作label
- 預設啟用audit或identity audit
- 記錄path、keys、RuleID、policy content/fingerprint、credential或raw backend error
- 新增PrometheusRule、dashboard、SIEM、tracing或dependency
- 修改RBAC、webhook failurePolicy、volume mount或external policy draft
- 修改`_workspace/`

## 任務依賴

| 任務 | Depends | 狀態 | 備註 |
|------|---------|------|------|
| T1 建立observation與metric red tests | 無 | Complete | 固定taxonomy、24 series與order |
| T2 實作decision recorder與counter | T1 | Complete | no-error、normalize、audit off |
| T3 串接Webhook與Sync observation points | T2 | Complete | exactly-once、pre-fetch |
| T4 實作audit config與identity opt-in | T2、T3 | Complete | structured safe fields |
| T5 收斂operational error與backend redaction | T3、T4 | Complete | category-only logs |
| T6 串接main、deployment與繁中文件 | T4、T5 | Complete | enabled/disabled都有recorder |
| V1 驗收與回歸驗證 | T1至T6 | Complete | 全套品質gate |

## 實作任務

- [x] T1 建立observation與metric red tests
  - Status: Complete
  - Boundary:
    - Allowed Changes: application、metrics與config測試檔
    - Forbidden: production code、deployment、docs、infra adapters
  - Depends: 無
  - Context: 建立fake recorder與ordered event；固定allow/deny/not_initialized/disabled、backend unknown、24 labelsets、audit config default/invalid組合。
  - Verify:
    - `go test ./internal/configs ./internal/infra/metrics ./internal/syncer/application`應先因新contract不存在而失敗
    - `git diff --check`

- [x] T2 實作decision recorder與counter
  - Status: Complete
  - Boundary:
    - Allowed Changes: `internal/infra/metrics/metrics.go`、`internal/syncer/application/authorization.go`、對應tests
    - Forbidden: use case control flow、config、logger raw fields
  - Depends: T1
  - Context: CounterVec四labels；只初始化四個合法decision/reason pair×surface×backend；recorder normalize backend與固定enum；audit先維持off/no-op。
  - Verify:
    - `go test ./internal/infra/metrics ./internal/syncer/application -run 'Test(InitMetrics|AuthorizationRecorder)'`
    - gathered family恰有24series且無動態marker

- [x] T3 串接Webhook與Sync observation points
  - Status: Complete
  - Boundary:
    - Allowed Changes: application authorization/options、mutate/sync use cases與tests、必要main test fixture
    - Forbidden: policy matching、fetcher、retry、delivery response、audit identity
  - Depends: T2
  - Context: 有效SecretRef後恰記一次；allow先record再fetch；deny/nil record後return；disabled記bypass再沿用既有fetch。
  - Verify:
    - `go test -race ./internal/syncer/application -run 'TestAuthorizationObservation|TestMutateUseCase|TestSyncWorker'`
    - ordered fake證明deny無fetch、allow observation先於fetch

- [x] T4 實作audit config與identity opt-in
  - Status: Complete
  - Boundary:
    - Allowed Changes: config、application recorder、main wiring與對應tests
    - Forbidden: 將identity放入metrics、預設啟用audit、修改deployment
  - Depends: T2、T3
  - Context: nested audit config與env；include identity需要audit enabled；structured message固定；Webhook identity完整、Sync只namespace；所有禁用fields缺席。
  - Verify:
    - `go test ./internal/configs ./internal/syncer/application ./cmd/vault-agent -run 'Test.*(Audit|Authorization)'`
    - zap capture marker與field absence assertions

- [x] T5 收斂operational error與backend redaction
  - Status: Complete
  - Boundary:
    - Allowed Changes: application error categories、delivery handler、Vault/AWS adapters與對應tests
    - Forbidden: 改Admission response、移除errors.Is sentinel、記錄raw error
  - Depends: T3、T4
  - Context: Webhook/Sync logs只記固定category；adapter not-found/fetch errors不含path/key/ARN/URL marker；sentinel仍可用errors.Is。
  - Verify:
    - `go test ./internal/syncer/delivery ./internal/syncer/infra ./internal/syncer/application -run 'Test.*(Redact|Error|Fetch|Webhook)'`
    - captured log與error text不含所有sensitive markers

- [x] T6 串接main、deployment與繁中文件
  - Status: Complete
  - Boundary:
    - Allowed Changes: main/config sample/base Deployment、三份繁中docs
    - Forbidden: RBAC、failurePolicy、volume、英文docs、external policy draft
  - Depends: T4、T5
  - Context: Production enabled/disabled authorization均注入recorder；Deployment明確false audit env；文件記錄24series、query範例、audit容量與資料禁區。
  - Verify:
    - `rg -n 'authorization_decisions_total|audit|include_identity|AUTHORIZATION_AUDIT' configs deployments docs`
    - `kubectl kustomize deployments/kustomize/base`
    - `kubectl kustomize deployments/kustomize/overlays/prod`

## 驗證任務

- [x] V1 驗收情境覆蓋
  - Verify: requirements的allow、deny、nil、disabled、unknown backend、24series、audit default/identity與log redaction均有automated tests。

- [x] V1 回歸驗證
  - Verify: `go test -race -count=1 ./...`、`go vet ./...`、`make lint`、`make policy-validate`、`go list ./...`、base/prod Kustomize render。

- [x] V1 品質檢查清單
  - [x] Metric名稱與四個label固定
  - [x] 合法labelsets恰為24組
  - [x] Backend raw value只能變成vault/aws/unknown
  - [x] Allow、deny、nil、disabled都exactly-once
  - [x] Deny/not initialized在external fetch前停止
  - [x] Disabled明確記bypass而非allow
  - [x] Audit預設關閉
  - [x] Identity需要第二層opt-in且不進metrics
  - [x] Sync不捏造ServiceAccount
  - [x] Path、keys、RuleID、fingerprint、credential、raw error marker未出現在metric/log
  - [x] Vault/AWS errors.Is語意保留
  - [x] Webhook response維持generic
  - [x] Main enabled/disabled都注入recorder
  - [x] Go race test、vet、lint通過
  - [x] Policy validation與go list通過
  - [x] Base/prod Kustomize render通過
  - [x] RBAC、failurePolicy與volume未變更
  - [x] 文件一致性已確認
  - [x] `git diff --stat`已檢查
  - [x] `git diff --check`已通過

## 實作中斷恢復

恢復時優先讀取：

1. 本文件的`Execution Context`
2. 目前未完成task
3. `Protected Behavior`
4. `Implementation Notes`

不得預設掃描整個`.specs`目錄。若文件很大，先定位：

```bash
rg -n "^#|^##|^###|Boundary:|Depends:|Implementation Notes|Status:" .specs/2026-08-04-18-00_Feature-authorization-decision-observability
```

## Implementation Notes

- 2026-08-04：已完成discovery與正式spec，尚未修改production code、tests、config、deployment或user-facing docs。
- Webhook與Sync目前都在有效SecretRef解析後、external fetch前執行authorization，適合在application layer插入共用recorder。
- `domain.Decision.RuleID`不得加入metric/audit；`Decision.Reason`也不得直接作label，必須由application固定映射。
- Backend annotation是不受信任輸入，必須在recorder內normalize為vault/aws/unknown。
- Discovery確認Vault/AWS not-found errors與Webhook fetch wrapper可能包含path/key，且上層存在raw error logging；T5會一併建立category-only安全邊界。
- Prometheus官方提醒每個唯一label組合都會產生time series，且應避免user ID等高cardinality label；本設計固定24series，不使用workload identity。
- 外部Policy Source draft與`_workspace/`不屬於本spec，不納入未來implementation commit。
- 2026-08-04：完成24-series counter、Webhook／Sync exactly-once observation、audit雙層開關、安全error category、Vault／AWS fetch redaction、production wiring、deployment與繁中文件。
- `golangci-lint`依專案固定版本`v2.12.2`從暫存路徑執行；系統路徑的`v2.12.1`未修改。

## 驗證結果摘要

- 新行為驗證：Webhook／Sync allow、deny、nil、disabled、pre-fetch順序、24 series、backend normalize、audit identity與marker redaction tests通過
- 回歸驗證：`go test -race -count=1 ./...`、`go vet ./...`、`go list ./...`、`make lint`、`make policy-validate`與base/prod Kustomize render通過
- 文件一致性：config sample、Deployment與三份繁中文件已同步audit開關、metric taxonomy與資料禁區
- 剩餘風險：Audit identity啟用後的log容量、retention與存取控制需由部署環境負責

## 後續改善

- [ ] 依production traffic評估deny rate recording rule與alert threshold。
- [ ] 另案評估caller authentication與Admission contract的安全觀測，避免與workload authorization taxonomy混用。
- [ ] 外部Policy Source完成後，再設計source-independent revision與跨replica收斂監控。
- [ ] Policy validation JSON output與failure/recovery drills維持最後階段。
