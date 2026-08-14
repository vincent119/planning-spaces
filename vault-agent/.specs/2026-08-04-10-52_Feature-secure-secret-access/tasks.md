# 任務文件：安全機密存取

Status: Complete

## Execution Context

- 意圖: 在不重寫 SecretFetcher 與 JSON Patch 行為的前提下，確保所有 Webhook 外部機密讀取都先經過 caller authentication 與 Admission contract validation；`authorization.enabled=true` 時再經過 workload authorization，並將 Kubernetes RBAC 收斂到受管 namespace 的 Secret `list/update`。
- 非目標:
  - 不處理 cache、singleflight、bulkhead。
  - 不修正 Secret 刪鍵語意或 Worker shutdown。
  - 不導入 CSI Driver、sidecar 或 init container。
  - 不自動建立 Vault/AWS policy。
- 已定決策:
  - production fail closed，不提供匿名相容模式。
  - 認證模式支援 mTLS 與 bearer；mTLS 優先。
  - `disabled` 代表關閉機密讀取。
  - Webhook 與 operations listener 分離。
  - 授權採 default deny，Webhook 與 Sync 規則用途分離。
  - RBAC 使用最小 ClusterRole 加受管 namespace RoleBinding，不使用全叢集 ClusterRoleBinding。
  - Policy 啟動時載入，第一版不支援 hot reload。
  - `authorization.enabled` 預設為 `true`；設為 `false` 時只略過 Webhook/Sync workload policy，不影響 caller authentication、Admission validation 或 RBAC。
  - Policy 載入失敗不得自動把 authorization 切換為 disabled。
  - 同時支援互斥的 `policy_file` 與 `policy_dir`；目錄模式採排序、全域唯一 rule ID 與 all-or-nothing 載入。
  - Policy 同時支援精確 namespace/path prefix 與 `namespace_globs`/`path_prefix_templates`，模板第一版只允許 `{{namespace}}`。
- 邊界: 主要變更限制在 `cmd/vault-agent/`、`internal/configs/`、`internal/syncer/`、`internal/infra/metrics/`、`deployments/kustomize/`、`configs/`、`docs/`、`go.mod` 與 `go.sum`。任何超出範圍的變更必須先更新本文件。
- 關鍵檔案:
  - `internal/syncer/delivery/webhook_handler.go`
  - `internal/syncer/application/mutate_usecase.go`
  - `internal/syncer/application/sync_worker_usecase.go`
  - `internal/syncer/domain/secret.go`
  - `internal/configs/config.go`
  - `cmd/vault-agent/main.go`
  - `deployments/kustomize/base/rbac.yaml`
  - `deployments/kustomize/base/admission.yaml`
- 完成條件:
  - AC-1 至 AC-13 全部通過或有明確環境限制證據。
  - 所有拒絕路徑以測試證明 SecretFetcher 呼叫次數為零。
  - production 缺少 authentication 時無法啟用 Webhook；authorization 啟用且缺少 policy 時亦無法啟用。
  - 渲染 manifest 不含全叢集 Secret ClusterRoleBinding、ConfigMap 權限與長效 SA token。
  - `go test -race ./...`、`go vet ./...`、`gofmt`、lint 與 manifest 驗證通過。

### Protected Behavior

- 已授權 Pod 的 Vault/AWS annotation 格式與注入 patch 語意維持相容。
- 未 opt-in 的 Pod 維持空 patch，不得觸發 fetch。
- `/healthz` 與已啟用的 `/metrics` 維持可供 kubelet/Prometheus 使用。
- Vault token auth 與 Kubernetes auth 的 SecretFetcher 實作本身不改寫。
- Sync Worker 對已授權 Secret 的 fetch/update 正常流程維持相容。

### 邊界

#### Allowed Changes

- `cmd/vault-agent/`
- `internal/configs/`
- `internal/syncer/`
- `internal/infra/metrics/`
- `deployments/kustomize/`
- `configs/`
- `docs/`
- `go.mod`
- `go.sum`
- 本 spec 的 `tasks.md`，僅供狀態與 Implementation Notes 回填

#### Forbidden

- 不修改 Vault/AWS 外部 policy provisioning 流程。
- 不改寫 Vault/AWS SecretFetcher 的取值語意。
- 不導入 cache、singleflight、bulkhead、CSI Driver、sidecar 或 init container。
- 不修正本 spec 以外的 Secret 刪鍵與 Worker shutdown 問題。
- 不提交真實 credential、token、client private key 或 production path 清單。
- 不以測試通過為由放寬 caller authentication、Admission validation、authorization 或 RBAC。

## 任務依賴

| 任務 | Depends | 狀態 | 備註 |
|------|---------|------|------|
| T1 建立安全設定與啟動驗證 | 無 | Complete | 設定驗證與單元測試完成 |
| T2 建立 policy model、loader 與 backend matcher | T1 | Complete | domain 測試通過 |
| T3 實作 caller authentication 與 Admission validation | T1 | Complete | delivery 測試通過 |
| T4 在 Webhook 與 Sync fetch 前強制授權 | T2、T3 | Complete | deny-before-fetch 測試通過 |
| T5 分離 listener 並組裝安全依賴 | T1、T2、T3、T4 | Complete | 全套 Go 測試通過 |
| T6 收斂 Service、RBAC 與 ServiceAccount token | T5 | Complete | Kustomize 驗證通過 |
| T7 加入 credential 與 policy deployment contract | T1、T6 | Complete | 安全範例已加入 |
| T8 更新文件與 migration runbook | T6、T7 | Complete | 繁體中文文件已更新 |
| V1 驗收情境覆蓋 | T1 至 T8 | Complete | AC 自動與靜態驗證完成 |
| V2 回歸驗證 | T1 至 T8 | Complete | race tests 通過 |
| V3 品質檢查清單 | V1、V2 | Complete | 測試、lint、manifest 與差異檢查通過 |

## 實作任務

- [x] T1 建立安全設定與啟動驗證
  - Status: Complete
  - Boundary:
    - Allowed Changes: `internal/configs/config.go`、新增 `internal/configs/config_test.go`、`configs/config.sample.yaml`
    - Forbidden: HTTP handler、use case、deployment manifests
  - Depends: 無
  - Context: 新增 Webhook auth mode、client CA、allowed principals、bearer token file、`authorization.enabled`、`policy_file`、`policy_dir`、Webhook/operations ports。authorization 預設啟用；只有明確設為 false 時 policy 來源才可省略。file/dir 同時設定時一律報錯；兩者皆未設定時，只有 authorization 啟用才報錯。production 缺少 authentication 必須報錯；`disabled` 不得變成匿名模式。
  - Verify:
    - `go test ./internal/configs -run 'TestConfig_(ProductionWebhookAuthRequired|AuthorizationSwitch|PolicySourceMutualExclusion|InvalidAuthMode|ListenerPorts)'`
    - 驗證 credential 僅接受檔案路徑，不接受 config literal。
    - 確認 AC-7 的 production fail-closed 行為。

- [x] T2 建立 policy model、loader 與 backend matcher
  - Status: Complete
  - Boundary:
    - Allowed Changes: 新增 `internal/syncer/domain/policy*.go` 與測試、必要的 `go.mod`/`go.sum`
    - Forbidden: Handler、Use Case、Kubernetes manifests
  - Depends: T1
  - Context: 定義 Webhook/Sync rules、WorkloadIdentity、Decision、SecretAuthorizer；authorization 啟用時於啟動階段驗證 schema。實作單檔與目錄 loader、排序、ConfigMap volume symlink 安全解析、版本一致及全域 rule ID 唯一檢查。支援精確 namespace/path 與受控 glob/template。Vault 與 AWS matcher 分開，禁止空規則與全域 wildcard。
  - Verify:
    - `go test ./internal/syncer/domain -run 'Test(Policy|PolicyLoader|VaultPathMatcher|AWSSecretMatcher|Authorizer)'`
    - 測試 default deny、namespace、namespace glob、ServiceAccount、backend、keys、delimiter boundary、path template、AWS ARN/name。
    - 測試 file/dir 等價、排序、重複 ID、版本衝突、部分檔案失敗、symlink 逃逸與 ConfigMap volume symlink。
    - 確認 AC-4、AC-5、AC-6、AC-11、AC-12、AC-13 的 policy 行為。

- [x] T3 實作 caller authentication 與 Admission validation
  - Status: Complete
  - Boundary:
    - Allowed Changes: 新增或修改 `internal/syncer/delivery/` 下 authenticator、validator、handler 與測試
    - Forbidden: SecretFetcher 實作、RBAC、Sync Worker
  - Depends: T1
  - Context: Handler 順序固定為 authenticate、decode、validate、call use case。mTLS 驗證 chain 後限制 principal；bearer 使用 constant-time compare。任何錯誤都不得呼叫 Mutator。
  - Verify:
    - `go test ./internal/syncer/delivery -run 'TestWebhookHandler_(Unauthenticated|InvalidAdmissionReview|Authenticated)'`
    - `go test ./internal/syncer/delivery -run 'TestMTLSAuthenticator|TestBearerAuthenticator'`
    - 使用 `httptest` 驗證合法與非法 client certificate。
    - 確認 AC-1、AC-2。

- [x] T4 在 Webhook 與 Sync fetch 前強制授權
  - Status: Complete
  - Boundary:
    - Allowed Changes: `internal/syncer/application/mutate_usecase.go`、`internal/syncer/application/sync_worker_usecase.go` 及其測試；必要時調整 delivery/domain 契約
    - Forbidden: Vault/AWS client 內部實作、deployment manifests
  - Depends: T2、T3
  - Context: Mutate use case 接收已驗證 admission 與 WorkloadIdentity；authorization 啟用時，在 SecretFetcher 前呼叫 authorizer，停用時明確略過。Sync Worker 使用同一開關及獨立 sync rule。拒絕錯誤使用安全 reason code，不回傳完整敏感 path。
  - Verify:
    - `go test ./internal/syncer/application -run 'TestMutateUseCase_(AuthorizedWorkloadFetchesSecret|UnauthorizedPathDoesNotFetch)'`
    - `go test ./internal/syncer/application -run 'TestSyncWorker_UnauthorizedReferenceDoesNotFetchOrUpdate'`
    - `go test ./internal/syncer/application -run 'TestMutateUseCase_AuthorizationDisabledBypassesPolicyOnly'`
    - 保留既有 application tests 並通過 race detector。
    - 確認 AC-3、AC-4、AC-6、AC-10。

- [x] T5 分離 listener 並組裝安全依賴
  - Status: Complete
  - Boundary:
    - Allowed Changes: `cmd/vault-agent/main.go`、必要的 `cmd/vault-agent/*_test.go`、`internal/infra/metrics/metrics.go` 與測試
    - Forbidden: RBAC、Vault/AWS client 行為
  - Depends: T1、T2、T3、T4
  - Context: 組裝 authenticator、validator、authorizer；Webhook server 只提供 `/mutate`，operations server 提供 health/readiness/metrics。加入 bounded denial metrics 與安全稽核欄位。所有 server 納入既有 graceful lifecycle。
  - Verify:
    - 測試 operations mux 不含 `/mutate`，Webhook mux 不含 `/metrics`。
    - 測試 production 設定錯誤時不啟動 Webhook listener。
    - `go test -race ./cmd/vault-agent ./internal/infra/metrics`
    - 確認 AC-7 與 FR-6、FR-7。

- [x] T6 收斂 Service、RBAC 與 ServiceAccount token
  - Status: Complete
  - Boundary:
    - Allowed Changes: `deployments/kustomize/base/`、`deployments/kustomize/overlays/prod/`、新增必要 manifest 驗證腳本或測試 fixture
    - Forbidden: Go 程式碼、Vault Server policy
  - Depends: T5
  - Context: Service 改為 `443 -> 8443`，probes 使用 operations port。RBAC 僅保留 Secret `list/update`；移除全叢集 ClusterRoleBinding、ConfigMap 權限、`system:auth-delegator` 與長效 token Secret。受管 namespace 由 overlay 顯式 RoleBinding。
  - Verify:
    - `kubectl kustomize deployments/kustomize/base` 與 production overlay 可成功渲染。
    - Manifest assertion 驗證無 `kubernetes.io/service-account-token` Secret、無 Secret ClusterRoleBinding、無 ConfigMap rule、無 delete verb。
    - 測試叢集執行 `kubectl auth can-i` 權限矩陣。
    - 確認 AC-8、AC-9。

- [x] T7 加入 credential 與 policy deployment contract
  - Status: Complete
  - Boundary:
    - Allowed Changes: `deployments/kustomize/overlays/prod/`、`configs/`、必要 Secret/ConfigMap volume reference
    - Forbidden: 將真實 credential、token、client key 或 production path 清單提交版本控制
  - Depends: T1、T6
  - Context: 只提交 Secret reference 與 policy 範例，不提交值。提供單檔與多檔目錄兩種 ConfigMap 掛載範例，overlay 只能選擇一種來源並明確設定 `authorization.enabled`；若停用，必須在部署文件標示只依賴 Vault/AWS policy 的風險。managed control plane credential 設定屬外部 prerequisite。
  - Verify:
    - `rg -n '(Bearer |client-key-data|token:|BEGIN .*PRIVATE KEY)' deployments configs` 不得找到真實 credential。
    - 缺少掛載檔案時 production 啟動驗證失敗。
    - Kustomize 渲染不包含 secret literal credential。

- [x] T8 更新繁體中文文件與 migration runbook
  - Status: Complete
  - Boundary:
    - Allowed Changes: `docs/config.zh-Hant.md`、`docs/deploy.zh-Hant.md`、`docs/annotations.zh-Hant.md`、`docs/README.zh-Hant.md`，必要時同步英文既有對照文件
    - Forbidden: 宣稱未經測試的平台能力、放入真實 credential
  - Depends: T6、T7
  - Context: 文件包含 API Server credential prerequisite、authorization 開關、單檔/目錄 policy、glob/template schema、受管 namespace RoleBinding、disabled 行為、rollback 與 managed control plane 限制。修正現有文件引用未納入 kustomization 的 `webhook.yaml` 問題。
  - Verify:
    - 文件範例與實際 config keys、ports、resource names 一致。
    - 所有 production 範例預設 fail closed。
    - Migration 步驟先配置 credential/RBAC，再切換 Service 與 Webhook。

## 驗證任務

- [x] V1 驗收情境覆蓋
  - Status: Complete
  - Boundary:
    - Allowed Changes: 測試檔、manifest assertions、必要測試 fixture、`tasks.md` Implementation Notes
    - Forbidden: 修改 production 行為以迴避驗收條件
  - Depends: T1 至 T8
  - Context: 將 requirements.md 的 AC-1 至 AC-13 綁定至自動測試、manifest dry-run 或具環境證據的人工驗證。
  - Verify:
    - AC-1 至 AC-13 每項皆有測試 selector、執行結果或明確環境限制證據。
    - 所有 authentication/validation/authorization 拒絕案例均證明 SecretFetcher 呼叫次數為零。
    - `policy_file` 與 `policy_dir` 等價、互斥、全域唯一 ID、glob/template 隔離皆有測試。

- [x] V2 回歸驗證
  - Status: Complete
  - Boundary:
    - Allowed Changes: 測試檔、測試 fixture、`tasks.md` Implementation Notes
    - Forbidden: 改寫 Protected Behavior 以符合新測試
  - Depends: T1 至 T8
  - Context: 確認已授權 Pod 注入、未 opt-in 空 patch、health/metrics、Vault/AWS fetcher 與已授權 Sync Worker 行為未被破壞。
  - Verify:
    - `go test -race -count=1 ./...`
    - 既有 Vault/AWS annotation、patch 與 Sync Worker 測試全部通過。
    - Webhook 與 operations mux 隔離後，health/metrics probes 仍可使用。

- [x] V3 品質檢查清單
  - Status: Complete
  - Boundary:
    - Allowed Changes: 格式修正、測試檔、manifest assertions、文件、`tasks.md` Implementation Notes
    - Forbidden: 為通過檢查而放寬 authentication、authorization 或 RBAC
  - Depends: V1、V2
  - Context: 執行最終靜態檢查、manifest 渲染、安全掃描與文件一致性驗證，確認 diff 未超出 Execution Context 邊界。
  - Verify:
    - `go test -race -count=1 ./...`
    - `go vet ./...`
    - `gofmt -l cmd internal` 無輸出。
    - `golangci-lint run ./...` 成功；若工具本身失效，需記錄版本與完整錯誤，不得視為通過。
    - `kubectl kustomize deployments/kustomize/base` 與 production overlay 成功並通過 manifest assertions。
    - `git diff --stat` 已檢查。
    - `git diff --check` 已通過。
  - 品質項目:
    - [x] 未認證請求不會觸發 SecretFetcher
    - [x] 未授權請求不會觸發 SecretFetcher 或 Kubernetes Update
    - [x] authorization 預設啟用，停用時只略過 workload policy
    - [x] authorization 停用時仍強制 caller authentication 與 Admission validation
    - [x] policy 載入失敗不會自動降級為 disabled
    - [x] policy file 與 directory 模式皆通過等價性測試且互斥
    - [x] policy directory 具確定排序、全域唯一 ID 與 all-or-nothing 驗證
    - [x] namespace glob 完整匹配，path template 展開後仍通過 backend matcher
    - [x] production 無匿名或 fail-open 設定
    - [x] credential 未出現在日誌、metrics、版本控制或 command line
    - [x] Vault/AWS matcher 有邊界與惡意輸入測試
    - [x] Webhook 與 operations endpoint 已隔離
    - [x] RBAC 只有受管 namespace 的 Secret `list/update`
    - [x] 長效 ServiceAccount token 與非必要 auth-delegator 已移除
    - [x] 既有合法 Vault/AWS 注入與同步測試通過
    - [x] 文件、設定範例與 manifests 一致
    - [x] 主要驗收情境已覆蓋
    - [x] Protected Behavior 回歸驗證通過
    - [x] 風險項目已處理

## 實作中斷恢復

恢復時優先讀取：

1. 本文件的 `Execution Context`
2. 目前未完成 task
3. `Protected Behavior`
4. `Implementation Notes`

不得預設掃描整個 `.specs` 目錄。若文件很大，先用標題與關鍵字定位：

```bash
rg -n "^#|^##|^###|Boundary:|Depends:|Implementation Notes|Status:" .specs/2026-08-04-10-52_Feature-secure-secret-access
```

## Implementation Notes

- T1 完成：新增 Webhook、operations、authorization 設定與 fail-closed 驗證；`go test ./internal/configs` 通過。
- T2 完成：新增 policy file/dir loader、glob/template authorizer 與 Vault/AWS matcher；`go test ./internal/syncer/domain` 通過。
- T3 完成：新增 bearer/mTLS authenticator 與 AdmissionReview validator；未認證及無效契約測試證明 Mutator 呼叫次數為零。
- T4 完成：MutateUseCase 與 Sync Worker 已在 fetch 前套用可選 policy；拒絕路徑測試通過。
- T5 完成：Webhook 與 operations listener 已分離並組裝 authenticator/authorizer；`go test ./...` 通過。
- T6 完成：RBAC 已收斂為受管 namespace 的 Secret list/update，並移除長效 SA token 與 auth-delegator。
- T7 完成：新增安全空 policy、單檔與目錄範例；base/prod Kustomize 渲染成功。
- T8 完成：設定、部署、annotation 與 README 繁體中文文件已更新。
- V1/V2 完成：`go test -race -count=1 ./...`、`go vet ./...`、gofmt、base/prod Kustomize 與安全 manifest assertions 均通過。
- V3 完成：原先 lint 錯誤由受限環境無法寫入 Go 與 golangci-lint 快取所致；改用具正常快取權限的環境後，修正 `errcheck` 與 `staticcheck` 問題，`golangci-lint run ./...` 回報 `0 issues`。
- 實作前必須確認目標 control plane 是否允許配置 Admission Webhook client credential。
- production 採 mTLS 或 bearer 尚待確認；預設建議 mTLS。
- 首批受管 namespace、ServiceAccount 與 backend/path policy 清單尚待確認。
- Admission 是否移除 `UPDATE` operation 尚待確認；目前設計預設只保留 `CREATE`。

## 驗證結果摘要

- 新行為驗證: 通過；domain、application、delivery 與 config 測試均包含安全拒絕案例。
- 回歸驗證: 通過；`go test -race -count=1 ./...` 與 `go vet ./...` 成功。
- 文件一致性: 已確認 config keys、ports、policy schema 與 manifests 一致。
- 剩餘風險: 目標 control plane credential 能力及 production credential 仍需環境驗證。

## 後續改善

- [ ] 評估 policy hot reload 與 generation 一致性。
- [ ] 評估 policy validate CLI 或 CI schema validation。
- [ ] 評估由 GitOps generator 或 controller 自動建立受管 namespace RoleBinding。
- [ ] 另案處理 Secret fetch cache、singleflight 與 bulkhead。
- [ ] 另案處理 Secret 刪鍵語意與 Worker shutdown。
