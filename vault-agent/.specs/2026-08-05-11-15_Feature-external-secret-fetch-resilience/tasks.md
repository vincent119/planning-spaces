# 任務文件：External Secret Fetch 韌性與隔離

Status: Complete

## Execution Context

- 意圖：為Vault／AWS external Secret reads建立統一總逾時、bounded retry、per-backend bulkhead、有界queue、in-flight合併、Vault re-auth收斂與低基數觀測。
- 非目標：不做Secret cache、circuit breaker、backend fallback、surface priority、跨replica dedupe、dashboard／alert或failure drill。
- 已定決策：timeout涵蓋queue/backoff/re-auth/attempts；SDK內層attempt固定1；`max_attempts`包含第一次；queue滿fail fast；reference-counted flight；完成即刪除；每caller clone map。
- 邊界：config、domain safe errors、metrics、Vault/AWS adapters、新resilient decorator、main lifecycle wiring、base Deployment、繁中文件與本spec。
- 關鍵檔案：`internal/syncer/infra/resilient_fetcher.go`、`vault_client.go`、`aws_client.go`、`internal/configs/config.go`、`internal/infra/metrics/metrics.go`、`cmd/vault-agent/main.go`。
- 完成條件：requirements全部驗收情境、bounded actual requests、bulkhead isolation、no-cache、waiter cancellation、single re-auth、fixed metrics、redaction、race/vet/lint/policy/Kustomize/diff checks完成。

### Protected Behavior

- 執行順序維持caller authentication → Admission contract → workload authorization → external fetch。
- Authorization deny／not initialized時不得呼叫raw或resilient fetcher。
- `domain.SecretFetcher` method signature、annotation、backend選擇與policy contract不變。
- Webhook backend failure回應維持generic；Sync cancellation與`SyncErrorsTotal`既有語意不變。
- Sync successful fetch仍以remote完整snapshot取代Secret.Data，撤銷key與空集合語意不變。
- Existing `SecretFetchDuration{backend}`繼續量測caller完整latency。
- Vault static token與Kubernetes auth選擇、AWS region／credential provider chain不變。
- Policy reload、readiness、HTTP timeout、graceful總timeout與cleanup職責不變；只新增fetch lifecycle cancellation。
- 不得建立completed result cache或把Secret value留在flight entry。
- Metric／log／error／trace不得包含path、keys、Secret value、ARN、URL、credential、raw error或flight digest。
- Runtime、error、log與Go測試失敗訊息使用英文；註解與文件使用繁體中文。
- `.specs/drafts/`與`_workspace/`不得修改或納入提交。

### 邊界

#### Allowed Changes

- `internal/configs/config.go`
- `internal/configs/config_test.go`
- `internal/infra/metrics/metrics.go`
- `internal/infra/metrics/metrics_test.go`
- `internal/syncer/domain/secret.go`
- `internal/syncer/domain/secret_test.go`
- 新增 `internal/syncer/infra/resilient_fetcher.go`
- 新增 `internal/syncer/infra/resilient_fetcher_test.go`
- `internal/syncer/infra/vault_client.go`
- `internal/syncer/infra/vault_client_test.go`
- `internal/syncer/infra/aws_client.go`
- `internal/syncer/infra/aws_client_test.go`
- `internal/syncer/application/mutate_usecase_test.go`
- `internal/syncer/application/sync_worker_usecase_test.go`
- `cmd/vault-agent/main.go`
- `cmd/vault-agent/main_test.go`
- `configs/config.sample.yaml`
- `deployments/kustomize/base/deployment.yaml`
- `docs/README.zh-Hant.md`
- `docs/config.zh-Hant.md`
- `docs/deploy.zh-Hant.md`
- `.specs/2026-08-05-11-15_Feature-external-secret-fetch-resilience/`

#### Forbidden

- 修改caller authentication、Admission contract、authorization policy／matching／reload或安全gate順序
- 修改`SecretFetcher` signature、annotations、Secret replacement或Kubernetes update retry
- 新增completed result cache、TTL、distributed state、backend failover或fallback value
- 對非冪等操作、not found、authn/authz、invalid response或context error重試
- 保留Vault／AWS SDK內層多次retry
- 以error string或external input決定retry／metric label
- 新增surface、namespace、ServiceAccount、path、keys、resource、status、attempt或error metric label
- 記錄Secret value、path、keys、ARN、Vault URL、credential、raw backend error或flight digest
- 修改Dockerfile、RBAC、volume、ConfigMap、Secret、probe、PDB或webhook failurePolicy
- 新增runtime dependency
- 修改`.specs/drafts/`或`_workspace/`

## 任務依賴

| 任務 | Depends | 狀態 | 備註 |
|------|---------|------|------|
| T1 建立韌性contract red tests | 無 | Complete | config、errors、metrics、protected behavior |
| T2 實作設定與safe error contract | T1 | Complete | defaults、env、validation、errors.Is |
| T3 收斂adapter retry ownership與分類 | T2 | Complete | SDK單次、typed classifier、redaction |
| T4 實作總預算與bounded retry | T2、T3 | Complete | injectable timeout、full jitter、attempt cap |
| T5 實作bulkhead與in-flight coordinator | T4 | Complete | per-backend、bounded queue、no-cache |
| T6 收斂Vault Kubernetes re-auth | T3、T5 | Complete | generation、single login、403 contract |
| T7 串接metrics與fetch lifecycle | T4、T5、T6 | Complete | fixed series、LIFO shutdown |
| T8 更新deployment與繁中文件 | T7 | Complete | env defaults、調校、無cache |
| V1 驗收與回歸驗證 | T1至T8 | Complete | 全套quality gates |

## 實作任務

- [x] T1 建立韌性contract red tests
  - Status: Complete
  - Boundary:
    - Allowed Changes: config、domain、metrics、infra、application與main測試檔；本spec Implementation Notes
    - Forbidden: production code、Deployment、user-facing docs
  - Depends: 無
  - Context: 先固定六個設定欄位、safe retryable／overload errors、五個metric families、authorization-before-fetch、retry count、bulkhead、flight、re-auth與shutdown contracts。所有測試失敗訊息用英文。
  - Verify:
    - `go test ./internal/configs ./internal/infra/metrics ./internal/syncer/domain ./internal/syncer/infra ./internal/syncer/application ./cmd/vault-agent`應先因新contract不存在而失敗
    - `git diff --stat`
    - `git diff --check`

- [x] T2 實作設定與safe error contract
  - Status: Complete
  - Boundary:
    - Allowed Changes: `internal/configs/config.go`、`internal/syncer/domain/secret.go`及對應tests
    - Forbidden: adapters、decorator、metrics、main、manifest
  - Depends: T1
  - Context: 新增`SecretFetchConfig`、defaults、六個env bindings及fail-fast validation；新增retryable與overloaded safe sentinels，兩者保留`errors.Is(ErrSecretFetchFailed)`，不得攜帶raw error。
  - Verify:
    - `go test ./internal/configs ./internal/syncer/domain -run 'Test.*(SecretFetch|Retryable|Overloaded)'`
    - Table涵蓋zero／negative、initial大於max及合法queue zero
    - Error marker不得出現在safe error text

- [x] T3 收斂adapter retry ownership與分類
  - Status: Complete
  - Boundary:
    - Allowed Changes: Vault／AWS adapters及tests，必要constructor call sites的compile-only調整
    - Forbidden: resilient decorator、metrics、application control flow、deployment
  - Depends: T2
  - Context: Vault設`MaxRetries=0`，AWS設`RetryMaxAttempts=1`；先處理context與not found，再以typed/status classifier映射retryable，其他固定permanent。Capturing transport證明持續5xx時SDK不自行追加requests。
  - Verify:
    - `go test ./internal/syncer/infra -run 'Test(Vault|AWS)Client_.*(Retry|Classif|Redact|Context|NotFound)'`
    - Vault statuses涵蓋408、412、429、500、502、503、504與400／401／403
    - AWS涵蓋ResourceNotFound、standard retryable與terminal API error
    - Raw marker不出現在returned error或captured log

- [x] T4 實作總預算與bounded retry
  - Status: Complete
  - Boundary:
    - Allowed Changes: 新`resilient_fetcher.go`與tests
    - Forbidden: bulkhead queue、flight coalescing、Vault re-auth、main wiring
  - Depends: T2、T3
  - Context: 先以單operation實作total timeout、max attempts、capped exponential full jitter、context-aware timer與result mapping。注入clock／jitter，算術防overflow，禁止wall-clock sleep tests。
  - Verify:
    - `go test ./internal/syncer/infra -run 'TestResilientSecretFetcher_.*(Budget|Retry|Backoff|Terminal|Context)'`
    - Retryable前兩次失敗第三次成功時attempts=3、retries=2
    - Not found、permanent、cancel、deadline均attempts=1
    - Queue尚未實作前不預先修改application或main

- [x] T5 實作bulkhead與in-flight coordinator
  - Status: Complete
  - Boundary:
    - Allowed Changes: resilient fetcher與tests
    - Forbidden: Vault/AWS classifier、re-auth、metrics globals、main、deployment
  - Depends: T4
  - Context: 每個instance獨立active／queued tokens；queue full fail fast。建立SHA-256 opaque identity與reference-counted flights；followers不占queue；last waiter取消operation；完成即刪除；每caller clone map。
  - Verify:
    - `go test -race ./internal/syncer/infra -run 'TestResilientSecretFetcher_.*(Bulkhead|Queue|Coalesce|Flight|Waiter|Cache|Clone|Identity)'`
    - N equivalent callers只執行一次backend call
    - 完成後相同fetch再次呼叫backend
    - 不同decorator的active／queue token互不影響
    - 完成／取消競爭重複執行至少100次且無race、deadlock、token leak

- [x] T6 收斂Vault Kubernetes re-auth
  - Status: Complete
  - Boundary:
    - Allowed Changes: Vault adapter、其auth test seams與tests
    - Forbidden: static token認證選擇、一般retry policy、metrics schema、main
  - Depends: T3、T5
  - Context: 為每次request保留token generation snapshot；同generation並行403只允許一個login，等待者共用結果。已更新generation者跳過重複login；每個logical fetch最多執行一次auth recovery與一次驗證重讀；重讀403與login failure皆不進一般retry。一般adapter attempts上限是`max_attempts`，Vault Secret read requests上限是`max_attempts + 1`。
  - Verify:
    - `go test -race ./internal/syncer/infra -run 'TestVaultClient_.*(Reauth|Generation|Concurrent403)'`
    - N個不同path並行403時login count=1
    - Static token 403時login count=0
    - Re-auth success後重讀仍403時結束，不循環
    - Waiter context cancel立即返回且不取消其他waiter

- [x] T7 串接metrics與fetch lifecycle
  - Status: Complete
  - Boundary:
    - Allowed Changes: metrics、resilient fetcher、main、application regression tests及對應tests
    - Forbidden: Deployment、docs、HTTP／authorization contract
  - Depends: T4、T5、T6
  - Context: 新增requests/retries/shared/inflight/queue collectors與固定constants；每caller final result exactly once；leader的active/queue gauge balance。Main包裝每個raw client並建立idempotent fetch lifecycle shutdown，依graceful LIFO固定cleanup順序。
  - Verify:
    - `go test -race ./internal/infra/metrics ./internal/syncer/infra ./internal/syncer/application ./cmd/vault-agent -run 'Test.*(SecretFetch|FetchLifecycle|Cleanup|Authorization)'`
    - Requests counter恰12 label combinations，其餘collector只使用vault／aws
    - Shutdown可取消active、queued、backoff與re-auth，gauges歸零
    - Authorization deny時raw與resilient fetch call count均為0
    - Webhook generic failure、Sync cancellation與existing duration metric不變

- [x] T8 更新deployment與繁中文件
  - Status: Complete
  - Boundary:
    - Allowed Changes: config sample、base Deployment、三份繁中文件
    - Forbidden: Dockerfile、RBAC、volume、probe、PDB、failurePolicy、英文文件
  - Depends: T7
  - Context: 加入六個明確env defaults；說明4秒為logical total budget、`max_attempts`包含首次、per-backend容量、queue zero、no-cache、metrics taxonomy、調校與敏感資料禁區。
  - Verify:
    - `rg -n 'secret_fetch|SECRET_FETCH_|secret_fetch_(requests|retries|shared|inflight|queue)' configs deployments docs`
    - `kubectl kustomize deployments/kustomize/base`
    - `kubectl kustomize deployments/kustomize/overlays/prod`
    - Rendered manifest只新增預期env，不改RBAC、volume、probe、PDB或failurePolicy

## 驗證任務

- [x] V1 驗收情境覆蓋
  - Status: Complete
  - Depends: T1至T8
  - Verify: requirements中的總預算、retry-only-transient、nested retry disabled、backend isolation、bounded queue、coalescing、no-cache、waiter cancellation、single re-auth、shutdown與redaction均有automated tests。

- [x] V1 回歸驗證
  - Status: Complete
  - Depends: T1至T8
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
  - [x] Config defaults、env、validation與文件一致
  - [x] Timeout涵蓋queue、backoff、re-auth與attempts
  - [x] Caller較早deadline優先，shutdown可取消所有operation
  - [x] Vault與AWS SDK內層attempt固定1
  - [x] 一般adapter calls不超過`max_attempts`，Vault auth recovery最多另有一次Secret驗證讀取
  - [x] 只有typed/status-classified transient read可retry
  - [x] Not found、authn/authz、invalid、cancel與deadline不retry
  - [x] Vault/AWS active與queue tokens互相獨立
  - [x] Queue full fail fast，cancel／completion無token leak
  - [x] Equivalent in-flight calls只打一個backend request
  - [x] Completed result與error都不cache
  - [x] 每caller取得獨立result map
  - [x] Last waiter取消shared operation，其他waiter不受單一caller取消影響
  - [x] 同Vault token generation並行403只login一次
  - [x] Requests metric固定12series，其餘labels固定vault／aws
  - [x] Counters exactly once，inflight／queue gauges平衡
  - [x] Authorization deny仍不執行fetch
  - [x] Webhook generic response、Sync replacement／cancel、policy與readiness不變
  - [x] Path、keys、value、ARN、URL、credential、raw error與flight digest未進觀測資料
  - [x] Runtime／error／log／test failure messages為英文
  - [x] Go race、vet、lint、go list與policy validation通過
  - [x] Base/prod Kustomize render通過且只含預期env差異
  - [x] Dockerfile、RBAC、volume、probe、PDB與failurePolicy未變更
  - [x] `.specs/drafts/`與`_workspace/`未納入
  - [x] `git diff --check`通過

## 實作中斷恢復

恢復時優先讀取：

1. 本文件的`Execution Context`
2. 目前未完成task
3. `Protected Behavior`
4. `Implementation Notes`

不得預設掃描整個`.specs`目錄。需要定位時使用：

```bash
rg -n "^#|^##|^###|Boundary:|Depends:|Implementation Notes|Status:" .specs/2026-08-05-11-15_Feature-external-secret-fetch-resilience
```

## Implementation Notes

- 2026-08-05：Discovery確認Webhook與Sync都在external fetch前完成workload authorization，適合以`SecretFetcher` decorator共用韌性邏輯。
- 2026-08-05：Vault `DefaultConfig`預設`MaxRetries=2`，AWS使用default retryer；實作前必須先將SDK內層attempt固定為1，避免retry multiplication。
- 2026-08-05：Vault Kubernetes auth目前每個403 caller各自login；本設計以token generation收斂並行refresh。
- 2026-08-05：Existing fetch duration在application layer，將保留以涵蓋followers、queue、backoff與所有attempts。
- 2026-08-05：設計不建立Secret cache；flight完成後立即刪除entry，後續同identity必須重新讀backend。
- 2026-08-05：本階段只建立requirements、design、tasks；尚未修改production code、tests、config、deployment或user-facing docs。
- 2026-08-05：T1至T3完成六個config欄位、safe retryable／overload sentinels、固定metrics schema、Vault status與AWS standard classifier；Vault `MaxRetries=0`、AWS `RetryMaxAttempts=1`。
- 2026-08-05：T4至T5完成單一logical timeout、bounded full-jitter retry、每backend獨立active／queue容量、SHA-256 opaque identity、reference-counted flights、per-caller map clone與completed-result no-cache。
- 2026-08-05：T6以token generation與共享re-auth state收斂並行403；20個並行callers只login一次，驗證重讀仍403時最多兩次Secret reads且不循環，waiter cancellation互不干擾。
- 2026-08-05：T7新增12-series final request counter、retry/shared counters及inflight／queue gauges；main為Vault／AWS包裝decorator並在graceful LIFO cleanup先取消fetch lifecycle。
- 2026-08-05：T8同步config sample、base Deployment六個env defaults及README／config／deploy繁中文件；Kustomize diff只有預期env，未改RBAC、volume、probe、PDB或failurePolicy。
- 2026-08-05：並行flight race cases以`-count=100`通過；完整race、vet、go list、policy validation與base/prod Kustomize render通過。
- 2026-08-05：PATH中的golangci-lint仍為v2.12.1，version guard如預期拒絕；改用既有已校驗的`/private/tmp/vault-agent-bin/golangci-lint` v2.12.2後完整lint為0 issues。
- `.specs/drafts/`與`_workspace/`不屬於本spec，不納入未來implementation commit。

## 驗證結果摘要

- 新行為驗證：config、typed classification、nested retry ownership、bounded retry、per-backend bulkhead、bounded queue、in-flight coalescing、no-cache、waiter cancellation、Vault single re-auth、metrics與shutdown tests通過。
- 回歸驗證：`go test -race -count=1 ./...`、`go vet ./...`、`go list ./...`、固定v2.12.2 `make lint`、`make policy-validate`與base/prod Kustomize render通過。
- 文件一致性：config sample、base Deployment、README、config與deploy文件使用相同defaults、timeout／attempt語意、metric taxonomy與資料禁區。
- 剩餘風險：Webhook與Sync仍共用同一backend容量；circuit breaker、surface priority、跨replica dedupe及production failure drill維持後續項目。

## 後續改善

- [ ] 依production traffic與metrics評估Webhook／Sync獨立capacity pool及priority。
- [ ] 另案規劃circuit breaker、half-open probe與backend availability SLO。
- [ ] 需要跨replica去重時，另行評估不保存Secret value的distributed coordination。
- [ ] Failure／recovery drill維持最後整合階段。
