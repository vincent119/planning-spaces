# 任務文件：Vault Kubernetes Re-auth ServiceAccount Token 輪替支援

Status: InProgress

## Execution Context

- 意圖：修正 `reAuthenticate()` 重用啟動時 ServiceAccount token，確保每次實際 Vault Kubernetes login 都重新讀取目前 token 並建立新 authenticator。
- 非目標：不新增 token path config、背景 watcher、主動 login、retry policy、metrics、RBAC、Deployment或 Dockerfile 變更。
- 已定決策：initial login與re-auth共用factory；client不保存長生命週期authenticator；factory只由single leader呼叫；任何refresh失敗均fail closed且不回退舊JWT。
- 邊界：Vault Kubernetes auth factory、Vault adapter、對應tests、三份繁中文件與本spec。
- 關鍵檔案：`internal/syncer/infra/vault_client.go`、可新增`vault_kubernetes_auth.go`、對應tests。
- 完成條件：rotated token payload、single read/create/login、fail-closed、static mode、generation、waiter cancellation、redaction與全套品質gate通過。

### Protected Behavior

- Vault Secret read只有在Kubernetes auth mode回傳403時觸發一次auth recovery。
- 每個logical fetch最多一次re-auth與一次驗證重讀；驗證重讀仍403時不得循環。
- `authMu`、token generation、single leader、共享result與waiter cancellation語意保持不變。
- Leader使用共享lifecycle context；單一waiter取消不得取消leader或其他waiter。
- Static Vault token mode不得讀取ServiceAccount token或建立KubernetesAuth。
- External fetch timeout、retry、bulkhead、queue、in-flight合併與metrics不變。
- `domain.SecretFetcher`、safe errors、authorization-before-fetch與Webhook generic response不變。
- JWT、token path、Vault address、auth mount path、role與raw backend error不得進入error或log。
- Runtime、error、log與Go測試失敗訊息使用英文；註解與文件使用繁體中文。
- `.specs/drafts/`與`_workspace/`不得修改或納入提交。

### 邊界

#### Allowed Changes

- `internal/syncer/infra/vault_client.go`
- `internal/syncer/infra/vault_client_test.go`
- 新增 `internal/syncer/infra/vault_kubernetes_auth.go`
- 新增 `internal/syncer/infra/vault_kubernetes_auth_test.go`
- `docs/README.zh-Hant.md`
- `docs/deploy.zh-Hant.md`
- `docs/architecture-diagrams.zh-Hant.md`
- `.specs/2026-08-05-11-52_BugFix-vault-kubernetes-reauth-token-refresh/`

#### Forbidden

- 新增ServiceAccount token path設定、環境變數或公開constructor參數
- 新增token watcher、timer、goroutine、主動refresh或無界login retry
- 保留或回退使用先前ServiceAccount JWT
- 修改403觸發條件、general retry classifier或每個logical fetch的auth recovery上限
- 讓waiter自行讀檔、建立authenticator或登入Vault
- 修改resilient fetcher、AWS adapter、domain、application、delivery、authorization或Admission contract
- 修改config、main、Dockerfile、Kubernetes manifest、RBAC、volume、probe、PDB或failurePolicy
- 新增runtime dependency或metric
- 記錄JWT、token path、Vault address、auth mount path、role或raw SDK error
- 修改`.specs/drafts/`或`_workspace/`

## 任務依賴

| 任務 | Depends | 狀態 | 備註 |
|------|---------|------|------|
| T1 建立token rotation與factory red tests | 無 | Complete | payload freshness、single construction path |
| T2 實作可重建Kubernetes auth factory | T1 | Complete | 每次New讀目前token，不保存JWT |
| T3 將initial login與re-auth切換至factory | T2 | Complete | 保留generation與single leader |
| T4 補齊fail-closed、redaction與併發回歸 | T3 | Complete | failure sharing、static mode、waiter |
| T5 更新繁中文件 | T4 | Complete | rotation、觸發與限制 |
| V1 驗收與回歸驗證 | T1至T5 | InProgress | lint package loading未通過，其餘gate通過 |

## 實作任務

- [x] T1 建立token rotation與factory red tests
  - Status: Complete
  - Boundary:
    - Allowed Changes: Vault adapter測試檔、新factory測試檔、本spec Implementation Notes
    - Forbidden: production code、user-facing docs、其他package
  - Depends: 無
  - Context: 以暫存token檔及capturing Vault login transport固定token A到token B輪替；以fake factory固定create count與initial/re-auth共用contract。測試token只能使用明確虛構marker。
  - Verify:
    - `go test ./internal/syncer/infra -run 'TestVault(KubernetesAuthFactory|Client)_.*(Token|Rotation|Factory)'`應先因新contract不存在或仍送出token A而失敗
    - Red tests證明現行`Login()`不會自行重讀token檔
    - 所有新Go測試失敗訊息使用英文
    - `git diff --check`

- [x] T2 實作可重建Kubernetes auth factory
  - Status: Complete
  - Boundary:
    - Allowed Changes: 新`vault_kubernetes_auth.go`、對應test，必要的Vault adapter內部interface調整
    - Forbidden: re-auth lock流程、config、main、deployment、docs
  - Depends: T1
  - Context: Factory保存role、mount path與預設token path；每次`New(ctx)`重新讀取檔案並建立獨立`KubernetesAuth`。讀檔前後檢查context；不將raw token寫入client、error或log。
  - Verify:
    - `go test ./internal/syncer/infra -run 'TestVaultKubernetesAuthFactory_.*'`
    - 同一factory在檔案更新前後建立的兩次login payload依序使用token A、token B
    - Cancelled context不呼叫SDK constructor或Login
    - Factory error text不包含token、path、role或raw reader error marker

- [x] T3 將initial login與re-auth切換至factory
  - Status: Complete
  - Boundary:
    - Allowed Changes: `vault_client.go`、`vault_client_test.go`與factory files
    - Forbidden: 改變FetchSecret一般retry、resilient decorator、public config或main wiring
  - Depends: T2
  - Context: 以factory欄位取代長生命週期`k8sAuth`；initial login與`reAuthenticate()`都先`factory.New(ctx)`再Login。Factory invocation必須位於既有single leader path，成功SetToken後才增加generation。
  - Verify:
    - `go test -race ./internal/syncer/infra -run 'TestVaultClient_.*(Reauth|Generation|Concurrent403|RotatedServiceAccountToken)'`
    - N個同generation並行403在re-auth階段只有一次token read、factory create與Login
    - 成功只增加一次generation；已更新generation的caller不再建立authenticator
    - Source inspection確認`VaultClient`不保存authenticator或rawServiceAccount token

- [x] T4 補齊fail-closed、redaction與併發回歸
  - Status: Complete
  - Boundary:
    - Allowed Changes: Vault adapter、factory與其tests
    - Forbidden: 為通過測試放寬安全錯誤、重試上限或single leader語意
  - Depends: T3
  - Context: 覆蓋token read、factory、Login、nil response、nil Auth、empty ClientToken與驗證重讀403。所有失敗不fallback、不更新token/generation；相鄰initial empty-response log移除Vault metadata。
  - Verify:
    - `go test -race ./internal/syncer/infra -run 'TestVaultClient_.*(Reauth|Failure|Redact|Static|Waiter|Second403)'`
    - Token read失敗時Login count=0
    - Leader失敗時waiter共用結果且下一個獨立週期可再嘗試
    - Static token 403時factory/read/Login count=0
    - Cancelled waiter立即返回且其他waiter仍完成
    - Captured error/log不含JWT、path、address、auth path、role或raw error markers

- [x] T5 更新繁中文件
  - Status: Complete
  - Boundary:
    - Allowed Changes: 三份繁中文件與本spec Implementation Notes
    - Forbidden: 英文文件、config sample、Kubernetes manifests
  - Depends: T4
  - Context: 說明initial login及每個實際re-auth會重新讀取projected token；同generation只由leader讀取一次；觸發仍為Secret read 403；無背景watcher與新設定。
  - Verify:
    - `rg -n 'ServiceAccount token|重新認證|re-auth|403|輪替' docs/README.zh-Hant.md docs/deploy.zh-Hant.md docs/architecture-diagrams.zh-Hant.md`
    - 文件不得宣稱每個請求都讀token、主動refresh或支援自訂token path
    - `git diff --check`

## 驗證任務

- [x] V1 驗收情境覆蓋
  - Status: Complete
  - Depends: T1至T5
  - Verify: requirements的token A到B、single read/create/login、read failure、login failure、no fallback、static mode、second 403、waiter cancellation與redaction均有automated tests。

- [ ] V1 回歸驗證
  - Status: InProgress
  - Depends: T1至T5
  - Verify:
    - `go test -race -count=1 ./...`
    - `go vet ./...`
    - `go list ./...`
    - `make lint`
    - `make policy-validate`
    - `kubectl kustomize deployments/kustomize/base`
    - `kubectl kustomize deployments/kustomize/overlays/prod`
    - Rendered manifests與本修正前一致
    - `git diff --stat`
    - `git diff --check`

- [ ] V1 品質檢查清單
  - [x] Initial login與re-auth共用同一factory
  - [x] 每次factory.New都重新取得目前ServiceAccount token
  - [x] VaultClient不保存長生命週期authenticator或rawJWT
  - [x] Token A輪替為B後只送出B進行re-auth
  - [x] 同generation並行403只有一次read、create與Login
  - [x] Token read／factory／Login／response失敗均fail closed
  - [x] 失敗不更新Vault token或generation
  - [x] 失敗不fallback至舊JWT
  - [x] 下一個獨立re-auth週期可重新讀檔再嘗試
  - [x] 每個logical fetch仍最多一次auth recovery與一次驗證重讀
  - [x] Waiter cancellation不影響leader或其他waiter
  - [x] Static token mode不接觸ServiceAccount token
  - [x] JWT、token path、Vault metadata與raw error未進error或log
  - [x] Initial empty auth response log已移除Vault metadata
  - [x] Config、main、Dockerfile、Kubernetes manifests與metrics未變更
  - [ ] Race、vet、lint與policy validation全部通過；目前只有lint package loading失敗
  - [x] Base/prod Kustomize render通過且manifest無diff
  - [x] 三份繁中文件與實作一致
  - [x] `.specs/drafts/`與`_workspace/`未納入
  - [x] `git diff --stat`已檢查
  - [x] `git diff --check`已通過

## 實作中斷恢復

恢復時優先讀取：

1. 本文件的`Execution Context`
2. 目前未完成task
3. `Protected Behavior`
4. `Implementation Notes`

不得預設掃描整個`.specs`目錄。需要定位時使用：

```bash
rg -n "^#|^##|^###|Boundary:|Depends:|Implementation Notes|Status:" .specs/2026-08-05-11-52_BugFix-vault-kubernetes-reauth-token-refresh
```

## Implementation Notes

- 2026-08-05：Discovery確認目前`VaultClient`只在constructor建立一次`KubernetesAuth`，`reAuthenticate()`重複呼叫同一物件的`Login()`。
- 2026-08-05：專案固定的HashiCorp Kubernetes auth套件會在`NewKubernetesAuth()`或套用token path option時讀檔，`Login()`使用物件內保存的token，不會自行重讀。
- 2026-08-05：既有generation與single leader機制可正確收斂並行403，本BugFix只把leader執行的登入改為fresh authenticator，不重寫協調器。
- 2026-08-05：設計不新增token path config；production沿用Kubernetes預設projected ServiceAccount token檔案。
- 2026-08-05：本階段只建立requirements、design與tasks，尚未修改production code、tests、config、deployment或user-facing docs，所有task維持Planned。
- 2026-08-05：收到`run task`命令，開始T1 token rotation與factory contract red tests。
- 2026-08-05：T1紅燈確認factory contract不存在；T2新增每次construction讀取目前token的factory；T3讓initial login與single-leader re-auth共用factory並移除client內長生命週期authenticator。
- 2026-08-05：T4以實際HTTP login payload驗證token A輪替為token B，並完成factory failure、invalid response、static mode、single leader及waiter cancellation回歸；定向race測試通過。
- 2026-08-05：T5完成繁中README、deploy與architecture diagrams，明確記錄403-driven、single-leader、no fallback及無新增設定。
- 2026-08-05：V1通過`go test -race -count=1 ./...`、`go vet ./...`、`go list ./...`、`make policy-validate`與base/prod Kustomize render；Vault infra新增失敗案例後再次通過race test。
- 2026-08-05：固定`v2.12.2`的`golangci-lint`在官方binary與本機同Go版本build都回報`context loading failed: no go files to analyze`；`go list ./...`及`go list -mod=readonly -deps ./...`均成功，未發現本次程式碼lint finding。V1因此維持InProgress。
- 2026-08-05：診斷用`go mod tidy -diff`曾調整既有direct／indirect分組，已以精確patch還原；`go.mod`最終無diff。
- `.specs/drafts/`與`_workspace/`不屬於本spec，不納入未來implementation commit。

## 驗證結果摘要

- 新行為驗證：token A輪替至token B、initial/re-auth共用factory、同generation single leader、factory/read/login/response fail-closed、static mode、second 403與waiter cancellation測試通過。
- 回歸驗證：全專案race test、vet、package loading、policy validation及base/prod Kustomize render通過。
- 文件一致性：README、deploy與architecture diagrams均記錄403-driven、single-leader、no fallback、無背景watcher及無新增設定。
- 未通過項目：`golangci-lint v2.12.2` package loading回報沒有可分析的Go檔案；此錯誤在啟用單一`govet`且停用專案設定時仍可重現。

## 後續改善

- [ ] 另案評估Vault token主動renewal或背景auth refresh；不得混入本次403-driven修正。
- [ ] 若未來需要自訂ServiceAccount token projection path，再以獨立設定需求評估validation與deployment contract。
