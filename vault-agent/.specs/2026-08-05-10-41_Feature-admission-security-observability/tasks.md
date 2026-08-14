# 任務文件：Caller Authentication 與 Admission Contract 安全觀測

Status: Complete

## Execution Context

- 意圖：為caller authentication與Admission contract建立獨立固定低基數decision metric及安全deny log，保留既有fail-closed順序與HTTP contract。
- 非目標：不改認證機制、Admission schema、workload authorization、backend fetch、RBAC、failurePolicy、dashboard、alert、trace或演練。
- 已定決策：單一14-series counter；gate/decision/reason固定taxonomy；不加auth mode或identity；allow只記metric；deny記category-only log；recorder無error且nil-safe。
- 邊界：metrics、delivery security observation／authenticator／handler、main wiring、繁中README與deploy文件、本spec及對應tests。
- 關鍵檔案：`internal/syncer/delivery/authenticator.go`、`webhook_handler.go`、新`security_observation.go`、`internal/infra/metrics/metrics.go`、`cmd/vault-agent/main.go`。
- 完成條件：14series、fixed reason、exactly-once、gate order、deny short-circuit、marker redaction、HTTP regression、race/vet/lint/policy/Kustomize/diff checks全數通過。

### Protected Behavior

- 執行順序維持method → caller authentication → body limit/decode → Admission contract → MutateUseCase → workload authorization → external fetch。
- 非POST、authenticator nil、auth deny、decode deny與contract deny的HTTP status及response body不變。
- Bearer constant-time comparison、mTLS verified chain與principal allowlist語意不變。
- DisabledAuthenticator維持拒絕所有secret access，不變成anonymous bypass。
- AdmissionReview type meta、request fields、resource、kind、operation、namespace、Pod object與namespace一致性規則及順序不變。
- `MutateRequestsTotal`與`AuthorizationDecisionsTotal`名稱、labels與記錄時機不變。
- Metric／log不得包含auth mode、principal、token、certificate、header、body、UID、namespace、Pod name、path、keys、RuleID、credential或raw error。
- Runtime、error、log與Go測試失敗訊息使用英文；註解與文件使用繁體中文。
- `.specs/drafts/`與`_workspace/`不得修改或納入提交。

### 邊界

#### Allowed Changes

- `internal/infra/metrics/metrics.go`
- `internal/infra/metrics/metrics_test.go`
- `internal/syncer/delivery/security_observation.go`
- `internal/syncer/delivery/security_observation_test.go`
- `internal/syncer/delivery/authenticator.go`
- `internal/syncer/delivery/authenticator_test.go`
- `internal/syncer/delivery/webhook_handler.go`
- `internal/syncer/delivery/webhook_handler_test.go`
- `internal/syncer/delivery/webhook_handler_internal_test.go`
- `cmd/vault-agent/main.go`
- `cmd/vault-agent/main_test.go`
- `docs/README.zh-Hant.md`
- `docs/deploy.zh-Hant.md`
- `.specs/2026-08-05-10-41_Feature-admission-security-observability/`

#### Forbidden

- 修改`RequestAuthenticator`的認證成功／失敗語意或接受credential格式
- 記錄／hash principal、token、certificate、Authorization header或AdmissionReview內容
- 修改workload authorization recorder、audit config、policy、SecretFetcher或sync worker
- 將raw error、HTTP status、request metadata或外部輸入直接作label
- 新增auth mode、namespace、UID、resource、path等metric label
- 修改Kubernetes RBAC、Deployment、Service、failurePolicy、TLS、volume或probe
- 新增dependency、dashboard、PrometheusRule、alert、trace或SIEM integration
- 修改`.specs/drafts/`或`_workspace/`

## 任務依賴

| 任務 | Depends | 狀態 | 備註 |
|------|---------|------|------|
| T1 建立taxonomy與flow red tests | 無 | Complete | 14series、reason、order、redaction |
| T2 實作security recorder與counter | T1 | Complete | fixed tuple、nil-safe、deny log |
| T3 實作auth與contract固定分類 | T1 | Complete | sentinel、typed reason、順序不變 |
| T4 串接Webhook observation points | T2、T3 | Complete | exactly-once、short-circuit |
| T5 串接main與繁中文件 | T4 | Complete | production recorder與PromQL |
| V1 驗收與回歸驗證 | T1至T5 | Complete | 全套quality gates |

## 實作任務

- [x] T1 建立taxonomy與flow red tests
  - Status: Complete
  - Boundary:
    - Allowed Changes: metrics與delivery測試檔、本spec Implementation Notes
    - Forbidden: production code、main、文件、deployment
  - Depends: 無
  - Context: 先以測試固定14個合法tuple、三個label names、auth control states、十種contract結果、exactly-once、order與sensitive marker absence。
  - Verify:
    - `go test ./internal/infra/metrics ./internal/syncer/delivery -run 'Test.*(AdmissionSecurity|Authentication.*Observation|ContractReason)'`應先因新contract不存在而失敗
    - `git diff --stat`
    - `git diff --check`

- [x] T2 實作security recorder與counter
  - Status: Complete
  - Boundary:
    - Allowed Changes: `internal/infra/metrics/metrics.go`、新`security_observation.go`與對應tests
    - Forbidden: handler control flow、authenticator、validator、main、docs
  - Depends: T1
  - Context: CounterVec labels固定gate/decision/reason；`InitMetrics`只建立14個完整合法tuple；recorder invalid-safe、nil-safe、無error；allow只metric，deny固定fields log。
  - Verify:
    - `go test ./internal/infra/metrics ./internal/syncer/delivery -run 'Test.*(InitMetrics.*AdmissionSecurity|SecurityRecorder)'`
    - Gathered family恰14series，無marker或invalid tuple
    - Captured security-specific deny fields只有gate、decision、reason

- [x] T3 實作auth與contract固定分類
  - Status: Complete
  - Boundary:
    - Allowed Changes: `authenticator.go`、`webhook_handler.go`內validator型別、`authenticator_test.go`、handler internal tests
    - Forbidden: Authenticate成功條件、validator檢查條件／順序、HTTP response、metric直接寫入
  - Depends: T1
  - Context: DisabledAuthenticator使用可`errors.Is`判斷的固定sentinel；其他未知auth error normalize為authentication_failed。Validator以typed fixed reason回傳，Pod unmarshal raw error不進message/log。
  - Verify:
    - `go test ./internal/syncer/delivery -run 'Test.*(Bearer|MTLS|Disabled|AdmissionContractReason|ValidateAdmissionReview)'`
    - Table涵蓋八種validator deny reasons且first-failure順序不變
    - Runtime errors與test failure messages為英文

- [x] T4 串接Webhook observation points
  - Status: Complete
  - Boundary:
    - Allowed Changes: handler、security observation helper與delivery tests
    - Forbidden: 移動安全gate、改mutator、workload authorization、fetch或HTTP contract
  - Depends: T2、T3
  - Context: POST到達auth gate恰記一次；auth allow後contract恰記一次；deny先record再return；valid依序auth allow、contract allow、mutate。Reject log移除`zlogger.Err`。
  - Verify:
    - `go test -race ./internal/syncer/delivery -run 'TestWebhookHandler_.*(SecurityObservation|Authentication|Admission|Contract)'`
    - Ordered fake證明allow sequence與deny short-circuit
    - Nil recorder不panic且不改HTTP結果
    - Existing handler tests全通過

- [x] T5 串接main與繁中文件
  - Status: Complete
  - Boundary:
    - Allowed Changes: `cmd/vault-agent/main.go`、main tests、`docs/README.zh-Hant.md`、`docs/deploy.zh-Hant.md`
    - Forbidden: config schema、Deployment、英文文件、Kubernetes manifests、external draft
  - Depends: T4
  - Context: Main明確建立並注入production recorder；README說明三層安全流程與獨立taxonomy；deploy文件提供PromQL、14series budget與資料禁區。
  - Verify:
    - `go test ./cmd/vault-agent -run 'Test.*Webhook.*Security'`
    - `rg -n 'admission_security_decisions_total|caller_authentication|admission_contract|14' docs/README.zh-Hant.md docs/deploy.zh-Hant.md`
    - Main production constructor不得傳nil recorder

## 驗證任務

- [x] V1 驗收情境覆蓋
  - Status: Complete
  - Boundary:
    - Allowed Changes: task邊界內測試與本spec驗證紀錄
    - Forbidden: 為通過測試放寬認證、contract或資料禁區
  - Depends: T1至T5
  - Verify:
    - Requirements的14series、auth allow/deny/nil/disabled、decode、八種contract reason、order、short-circuit、redaction與taxonomy separation均有automated tests

- [x] V1 回歸驗證
  - Status: Complete
  - Boundary:
    - Allowed Changes: 僅修正本Feature直接造成的邊界內失敗，先記錄Implementation Notes
    - Forbidden: 修改其他模組或把本地結果描述為target cluster evidence
  - Depends: T1至T5
  - Verify:
    - `go test -race -count=1 ./...`
    - `go vet ./...`
    - `go list ./...`
    - `make lint`
    - `make policy-validate`
    - `kubectl kustomize deployments/kustomize/base`
    - `kubectl kustomize deployments/kustomize/overlays/prod`
    - `git diff --stat`
    - `git diff --check`

- [x] V1 品質檢查清單
  - [x] Metric名稱與gate/decision/reason labels固定
  - [x] 合法labelsets恰為14組
  - [x] Invalid tuple不建立series或log
  - [x] 非POST不記security gate且維持405
  - [x] Auth nil、disabled、failure、success均exactly-once
  - [x] Auth deny不decode body或呼叫mutator
  - [x] Decode與八種validator deny reason固定
  - [x] Contract deny不呼叫mutator
  - [x] Valid flow依序auth allow、contract allow、mutate
  - [x] Allow不產生逐筆security log
  - [x] Deny log的security-specific fields只有固定gate、decision、reason
  - [x] Principal、token、certificate、header、body、UID、namespace與secret metadata未進metric/log
  - [x] HTTP status與response body不變
  - [x] Bearer、mTLS與DisabledAuthenticator語意不變
  - [x] Admission validator條件與first-failure順序不變
  - [x] Mutate與Authorization既有metrics不變
  - [x] Main production recorder非nil
  - [x] Race、vet、lint與policy validation通過
  - [x] Base/prod Kustomize render通過且manifest無diff
  - [x] 文件一致性已確認
  - [x] `_workspace/`與draft未納入
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
rg -n "^#|^##|^###|Boundary:|Depends:|Implementation Notes|Status:" .specs/2026-08-05-10-41_Feature-admission-security-observability
```

## Implementation Notes

- 2026-08-05：Discovery確認Webhook既有順序為method、caller authentication、body limit/decode、Admission contract、MutateUseCase；未認證與contract invalid都在機密fetch前return。
- 2026-08-05：現有`MutateRequestsTotal`只記mutator結果，`AuthorizationDecisionsTotal`只記workload authorization，兩者都不能表示delivery security gate結果。
- 2026-08-05：Bearer與mTLS一般失敗共用固定authentication error；DisabledAuthenticator有不同error text但尚無sentinel，handler nil則直接503。
- 2026-08-05：Admission validator有八個依序執行的拒絕類別；JSON body decode在validator外，形成第十個contract合法tuple中的`decode_failed` deny。
- 2026-08-05：設計固定4個caller authentication tuples與10個Admission contract tuples，總計14series，不加入auth mode、principal、HTTP status或request metadata。
- 2026-08-05：本階段只建立requirements/design/tasks，尚未修改production code、tests、main或user-facing docs。
- 2026-08-05：T1 red tests確認新metric family與delivery recorder contract不存在；T2新增固定14-series counter、完整tuple allowlist與invalid-safe recorder。
- 2026-08-05：T3以disabled sentinel及typed contract error固定分類，保留Bearer、mTLS與八個validator條件及first-failure順序。
- 2026-08-05：T4在既有return points串接exactly-once observation；auth deny不讀body，contract deny不呼叫mutator，有效流程順序為auth allow、contract allow、mutate。
- 2026-08-05：T5 main注入production recorder；繁中README與deploy文件補上14-series taxonomy、PromQL與資料禁區。
- 2026-08-05：V1通過全專案race、vet、固定v2.12.2 lint、go list、policy validation、base/prod Kustomize render與diff checks。
- `.specs/drafts/`與`_workspace/`不屬於本spec，不納入未來implementation commit。

## 驗證結果摘要

- 新行為驗證：14series、fixed tuple、auth control states、contract taxonomy、order、short-circuit、safe fields與main wiring tests通過
- 回歸驗證：`go test -race -count=1 ./...`、`go vet ./...`、`go list ./...`、`make lint`、`make policy-validate`與base/prod Kustomize render通過
- 文件一致性：requirements、design、tasks、繁中README與deploy文件使用相同14-series taxonomy與資料禁區
- 剩餘風險：本Feature不含TLS handshake、API Server transport telemetry、recording rule、alert或failure drill，維持後續項目

## 後續改善

- [ ] 依production traffic評估authentication／contract deny rate recording rule與alert threshold。
- [ ] 平台可提供control-plane telemetry後，再評估TLS handshake與API Server webhook transport observation。
- [ ] Failure/recovery drills維持最後整合階段，不納入本Feature。
