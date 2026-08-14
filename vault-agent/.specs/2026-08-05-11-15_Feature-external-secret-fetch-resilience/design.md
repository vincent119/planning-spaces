# 設計文件：External Secret Fetch 韌性與隔離

## 設計摘要

在既有 Vault／AWS adapter 外新增每個 backend 一份的 `ResilientSecretFetcher` decorator。Decorator 先以 reference-counted flight coordinator 合併相同進行中讀取，再讓 leader 通過有界 queue 與 active semaphore，在單一總時間預算內執行 capped exponential full-jitter retry。Adapter 負責在 raw SDK error 邊界完成 retryable／terminal 分類並只回傳安全 error。Vault Kubernetes re-auth 另外以 token generation 收斂並行刷新。所有工作都連結 process fetch lifecycle，graceful shutdown 可主動取消。

## 文件定位

本設計實現同目錄 `requirements.md`。它位於 application authorization 之後、backend adapter 之外，不改變 caller authentication、Admission contract、policy decision、Secret replacement、Kubernetes update retry 或 AdmissionReview contract。

## 已知契約狀態

- `domain.SecretFetcher` 已以 `context.Context`、path、keys 為輸入，適合由 decorator 實作同一介面。
- Webhook 與 Sync 共用 `map[string]domain.SecretFetcher`，可在 main wiring 一次套用 decorator，不必修改 use case constructor。
- `MutateUseCase` 與 `SyncWorkerUseCase` 已在 fetch 前完成 workload authorization。
- Vault `v1.22.0` 的 `DefaultConfig` 預設 `MaxRetries=2`，共三次 attempts；`MaxRetries=0` 可關閉。
- AWS SDK `v1.41.3` 支援 `config.WithRetryMaxAttempts(1)`，且 standard retryer 可判定 transport、throttling 與 server error。
- Vault Kubernetes auth 現在在每個 403 caller 內直接 login，沒有並行收斂。
- `SecretFetchDuration` 已在 application layer量測 caller latency，無須移到 decorator。
- Main 的 graceful task context 目前只直接傳給 Sync Worker 與 policy reloader；fetcher 在 task 建立前初始化，需要獨立 lifecycle handle。

## 架構

```text
Webhook／Sync caller
        │
        ├─ caller authentication／contract／authorization
        │
        ▼
ResilientSecretFetcher（每個 backend 一份）
        │
        ├─ opaque flight identity
        ├─ join／create in-flight operation
        ├─ bounded queue
        ├─ active semaphore
        ├─ total timeout + retry + full jitter
        └─ per-caller result clone／final metric
        │
        ▼
VaultClient 或 AWSClient
        │
        ├─ SDK internal attempts = 1
        ├─ raw error typed classification
        └─ safe domain error only
        │
        ▼
Vault／AWS Secrets Manager
```

Vault Kubernetes auth 另外在 `VaultClient` 內使用 re-auth coordinator；它不跨 backend，也不與 Secret in-flight identity 共用。

## 元件設計

### Config

在 `configs.Config` 新增：

```go
type SecretFetchConfig struct {
    TimeoutSeconds             int `mapstructure:"timeout_seconds"`
    MaxAttempts               int `mapstructure:"max_attempts"`
    InitialBackoffMilliseconds int `mapstructure:"initial_backoff_milliseconds"`
    MaxBackoffMilliseconds     int `mapstructure:"max_backoff_milliseconds"`
    MaxConcurrentRequests      int `mapstructure:"max_concurrent_requests"`
    MaxQueuedRequests          int `mapstructure:"max_queued_requests"`
}
```

`Config.SecretFetch` 使用 `mapstructure:"secret_fetch"`。Defaults 與 env names 依 requirements 固定。`Validate` 不調整不合法值，直接回傳英文錯誤，避免 production 靜默降級。

轉換成 runtime options 時才建立 `time.Duration`，且應集中在 constructor，避免多處秒／毫秒換算。

### Safe domain errors

保留既有 sentinel，新增兩個安全分類：

- `ErrSecretFetchRetryable`：可由 `errors.Is(err, ErrSecretFetchFailed)` 與自身辨識，只允許 adapter 在已判定 transient 後回傳。
- `ErrSecretFetchOverloaded`：可由 `errors.Is(err, ErrSecretFetchFailed)` 與自身辨識，表示 queue 已滿。

Error text 只能是固定英文，不包 raw SDK error、path、key 或 resource identifier。`context.Canceled` 與 `context.DeadlineExceeded` 保持原 error chain，不能轉為 retryable。

### Adapter error classification

分類順序必須由最具體到最一般：

1. Caller／operation context cancellation 或 deadline：原樣返回。
2. Backend not found：`ErrSecretNotFound`。
3. 明確 transient：`ErrSecretFetchRetryable`。
4. 其他：`ErrSecretFetchFailed`。

Vault 使用 `errors.As(*vault.ResponseError)` 與 status code；network error 使用型別判斷，不解析文字。AWS 先辨識 `types.ResourceNotFoundException`，再使用 pinned SDK standard retry classifier；其餘為 permanent。Adapter logs 只記固定 category 與 backend，不寫 raw error。

Vault 與 AWS constructor 都接收 fetch timeout／retry ownership options：

- Vault：`cfg.MaxRetries = 0`，`cfg.Timeout` 不得超過 logical fetch timeout；context 較早 deadline仍優先。
- AWS：`config.WithRetryMaxAttempts(1)`。

Constructor／初始 Kubernetes login 不納入 runtime fetch retry loop；啟動失敗維持 fail fast。

### ResilientSecretFetcher

建議結構：

```go
type ResilientSecretFetcher struct {
    backend   string
    next      domain.SecretFetcher
    options   ResilienceOptions
    lifecycle context.Context
    shutdown  context.CancelFunc
    flights   *fetchFlightGroup
    active    chan struct{}
    queued    chan struct{}
    clock     Clock
    jitter    JitterSource
}
```

Production 使用 real clock 與 concurrency-safe random source；tests 注入 fake clock／deterministic jitter。Constructor 只接受固定 backend `vault` 或 `aws`，其他值直接回傳 error，避免 metric cardinality 漂移。

每次 `FetchSecret` 的流程：

1. 若 caller context 已取消，記錄 caller final result 後返回。
2. 對 path 與 canonical keys 產生 opaque identity。
3. 加入既有 flight 或建立新 flight；follower 增加 shared counter。
4. Leader 建立 operation context，總 timeout 從 flight 建立開始計算，並連結 fetch lifecycle。
5. Leader 進入 bounded bulkhead，成功後執行 retry loop。
6. Operation 完成時儲存短暫 result、關閉 done channel並從 flight map 刪除。
7. 每個仍在等待的 caller clone map 後返回；已取消 caller 不等待 operation。
8. 每個 caller 恰記一次 `requests_total` final result。

### Opaque flight identity

Identity 不直接串接字串，避免分隔符碰撞。編碼內容為：

- path byte length + path bytes
- mode marker：all keys 或 selected keys
- 每個排序、去重後 key 的 byte length + key bytes

對 bytes 計算 SHA-256，flight map 只保存固定長度 digest。Backend 已由每個 decorator instance 隔離，不需放入 digest。原始 path／keys 只存在 caller 與 backend call 必要生命週期；digest 不可出現在任何觀測資料。

### Reference-counted flight coordinator

不使用傳統 first-caller-owned singleflight，避免 leader context 取消導致其他 waiter 無條件失敗。每個 flight 保存：

- `done chan struct{}`
- operation context／cancel
- waiter count
- 完成後的短暫 result／error

第一個 caller 建立 operation；後續 caller在同一 lock 內增加 waiter count。Caller select `done` 與自身 `ctx.Done()`：

- 收到 `done`：讀取 result，減少 waiter，clone 後返回。
- Caller 先取消：減少 waiter並立即返回。
- Waiter count 歸零且 operation 未完成：呼叫 operation cancel。

Operation 完成時必須在 lock 下標記完成、刪除 map entry，再關閉 `done`。所有共享狀態需由 mutex 或 atomic 清楚保護，並以 race test、重複 cancel test、完成與取消競爭 test 驗證。完成後不得留下 TTL entry。

Operation context 以 lifecycle 為 cancellation root，保留 first caller 的 tracing values但移除其 cancellation，再套用總 timeout；另以 cancellation hook連結 lifecycle。任何 context hook 都必須在 operation 完成時停止並釋放。

### Per-backend bulkhead

每個 decorator 建立自己的 active 與 queue tokens：

1. 先嘗試取得 active token。
2. Active 滿時嘗試取得 queue token；queue 滿立即回傳 `ErrSecretFetchOverloaded`。
3. 已取得 queue token者等待 active token或 operation context cancellation。
4. 取得 active後立即釋放 queue token並增加 inflight gauge。
5. Backend operation 結束時以 `defer` 釋放 active token並減少 gauge。

`max_queued_requests=0` 表示 active 滿時不等待。Queue depth 只計已接受且等待 active 的 unique flight leaders；followers 不消耗 queue token。實作不承諾 strict FIFO fairness，但不得讓 token／gauge 遺漏。

### Retry loop

`max_attempts` 是 decorator 對 adapter `FetchSecret` 的呼叫上限。流程為：

1. 呼叫 attempt。
2. Success、not found、context error或 permanent error立即結束。
3. 只有 `ErrSecretFetchRetryable` 且尚有 attempts 時才計算 backoff。
4. 增加 retry counter，使用 full jitter：`delay ∈ [0, min(maxBackoff, initialBackoff × 2^(attempt-1))]`。
5. 以 timer select operation context；取消時停止 timer並返回。
6. 開始下一 attempt 前再次檢查 context。

Delay 算術必須防 overflow。Retry counter 在 retry 已排定時增加，即使稍後在 backoff 中被取消；這可反映系統確實進入 retry path。

### Vault Kubernetes re-auth 收斂

`VaultClient` 新增 token generation 與 re-auth coordination。每個讀取先取得 generation snapshot；收到 403 時：

1. Static token 模式直接回 permanent failure。
2. Kubernetes auth 模式進入 re-auth coordinator。
3. 若目前 generation 已高於 request snapshot，表示其他 goroutine 已刷新，直接使用新 token重讀。
4. 否則只有一個 leader 執行 login；其他 goroutine等待同一結果。
5. Login 成功後更新 token及 generation，再讓各 request 重讀一次。
6. Login failure、等待 context 取消或重讀仍為 403 時，不進一般 transient retry。

每個 logical fetch 最多發生一次 auth recovery；因此一般 adapter attempts 上限是 `max_attempts`，Vault Secret read HTTP requests 上限是 `max_attempts + 1`。Kubernetes login request 不算 Secret read，但包含在同一 logical timeout 內。Tests 必須分別驗證 adapter call、Secret read request與login request計數。

Re-auth 不記 credential、role、auth path或 raw error。可增加固定 category log，但本 Feature 不新增高基數 re-auth metric。Vault client 的 token 更新與 generation 讀寫必須通過 race test。

### Lifecycle 與 graceful shutdown

Main 在建立 decorators 時建立 fetch lifecycle manager，並暴露 idempotent `Shutdown()`。Graceful cleanup 順序固定為：

1. readiness 已由 task termination 設為 not ready。
2. 取消 fetch lifecycle，使 active、queued、backoff、re-auth 及 flights 立即收斂。
3. Shutdown HTTP servers。
4. 等待 policy reloader／Sync Worker。
5. Shutdown tracer。

實作必須依 `graceful` 的 LIFO 特性安排 registration，並以 ordered cleanup test 固定。`Shutdown()` 重複呼叫不得 panic。服務正常運行時，單一 caller取消只依 waiter ref count影響其 flight，不取消其他 backend 或其他 identity。

## Metrics 與結果映射

### Final result

| 條件 | `requests_total` result | 上層語意 |
|------|-------------------------|----------|
| 成功 | `success` | 回傳 clone map |
| `ErrSecretNotFound` | `not_found` | 保留 sentinel |
| Caller／operation deadline | `timeout` | 保留 deadline chain |
| Caller／lifecycle canceled | `canceled` | 保留 cancellation chain |
| Queue full | `overloaded` | `errors.Is(ErrSecretFetchFailed)` 為 true |
| 其他終止錯誤 | `failed` | `ErrSecretFetchFailed` |

Metrics helper 只接受固定 backend 與 result constants。`InitMetrics` 預初始化所有 counter label 組合及兩個 backend gauges。Gauge tests 必須涵蓋 success、timeout、cancel、panic-safe defer path；production code 不以 recover 隱藏 backend panic。

### Logging

- Retry 正常路徑以 metric 為主，不逐次輸出 warn，避免故障時 log storm。
- Final adapter failure延用固定英文訊息與 `error.category`。
- Overload可輸出 rate-controlled／既有 request-level固定 category；不得輸出 path、keys、flight digest、queue identity或 raw error。
- Secret value 永遠不得進 log、metric、trace或 error。

## Application 與 main wiring

- Main 先建立 raw Vault／AWS clients，再各自包成 resilient decorator後放入 fetchers map。
- Webhook／Sync 不知道 decorator，`domain.SecretFetcher` 介面不變。
- Authorization 仍在 fetcher lookup／call 前執行；既有 deny tests增加 decorator call count assertion。
- Existing `SecretFetchDuration` 不移動，讓 follower waiting、queue、backoff與attempts都包含在 caller latency。
- Sync 既有 context cancellation與 `SyncErrorsTotal` 保留；overload與permanent failure仍算 sync error，shutdown cancellation不算。
- Webhook handler仍將所有 backend failure轉為 generic admission response。

## 設定與部署

- `configs/config.sample.yaml` 加入完整 `secret_fetch` 範例與繁中註解。
- Base Deployment加入六個明確 env defaults，使 production manifest不依賴程式隱含值。
- 不修改 Dockerfile、RBAC、Secret、ConfigMap、volume、probe、PDB或 webhook `failurePolicy`。
- `docs/config.zh-Hant.md` 說明欄位、環境變數、validation及 max attempts 定義。
- `docs/deploy.zh-Hant.md` 說明 4 秒總預算與 admission `timeoutSeconds=5` 的關係、容量調校及 shutdown。
- `docs/README.zh-Hant.md` 說明不做 cache、per-backend isolation與觀測指標。

## 測試設計

### Test doubles

- Scripted fetcher：記錄 attempts並依序回傳指定 safe errors／results。
- Blocking fetcher：透過 channels控制 active、完成與 context cancellation。
- Fake clock／timer：不使用實際 sleep 驗證 backoff與總預算。
- Deterministic jitter：固定輸出邊界值，驗證 cap與overflow。
- Capturing transport：計算 Vault／AWS 實際 HTTP requests，證明 nested retry disabled。
- Fake Vault auth：記錄 login count、generation與阻塞點。
- Metric gatherer／captured logger：驗證固定 taxonomy與敏感 marker absence。

### Concurrency matrix

至少涵蓋：

- 1 leader + N followers success
- leader caller取消但 follower存活
- 所有 waiters取消
- operation完成與最後 waiter取消競爭
- active full、queue可用／已滿／取消
- 相同 identity合併、不同 path／keys不合併
- keys reorder／duplicate合併、empty keys不與selected keys合併
- Vault N 個並行403只執行一次login
- shutdown同時存在active、queued、backoff與re-auth

所有 concurrency tests 必須具備 bounded timeout，但 assertions 使用 channel／fake clock，不依賴脆弱的 wall-clock sleep。

## 邊界

### Allowed Changes

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
- 本 spec 目錄

### Forbidden

- 修改 caller authentication、Admission validation、authorization policy schema／matching／reload
- 修改 `domain.SecretFetcher` method signature或Secret annotation contract
- 建立成功／失敗 cache、TTL、跨 replica dedupe或backend fallback
- Retry Kubernetes Secret update、非冪等操作或未分類 permanent error
- 讓 SDK retry與decorator retry同時生效
- 使用error string判斷retryability
- 將path、keys、resource identifier、credential、raw error或flight digest輸出到觀測資料
- 新增namespace、ServiceAccount、surface、path、error等metric label
- 修改RBAC、Dockerfile、volume、failurePolicy、probe或PDB
- 修改`.specs/drafts/`或`_workspace/`

## 替代方案

| 方案 | 優點 | 缺點 | 結論 |
|------|------|------|------|
| 只依賴各SDK retry | 程式碼少 | Vault／AWS語意與metrics不一致，無總預算與bulkhead | 不採用 |
| SDK retry外再包retry | 容易加入 | Attempts相乘，故障放大 | 禁止 |
| 在use case各自實作韌性 | 可分Webhook／Sync | 重複邏輯且無per-backend共享隔離 | 不採用 |
| `SecretFetcher` decorator | 保持application contract，可共用測試與容量 | 需處理shared context | 採用 |
| 一般singleflight以leader context執行 | 實作簡單 | Leader取消會傷害其他waiters | 不採用 |
| Reference-counted flight | Caller取消互不干擾，最後waiter可停止operation | 同步狀態較複雜 | 採用 |
| 結果TTL cache | 大幅減少backend load | 延長撤銷值生命週期並增加Secret retention | 禁止 |
| 只限制active且無queue bound | 設定少 | 等待goroutine仍可無界成長 | 不採用 |
| Circuit breaker | 故障時快速拒絕 | State transition與半開探測需另訂SLO | 後續另案 |

## 風險與處理方式

| 風險 | 影響 | 處理方式 | 驗證 |
|------|------|----------|------|
| Flight state race或token leak | deadlock、容量永久降低 | 單一鎖責任、defer release、idempotent cancel | race與競爭tests |
| Shared operation保存Secret過久 | 記憶體暴露面增加 | 無TTL、完成即刪entry、每caller clone | no-cache與lifetime tests |
| Full jitter test不穩定 | flaky tests | 注入fake clock與deterministic source | deterministic unit tests |
| Auth refresh與一般retry交錯 | attempts不可預期 | 403只走generation re-auth，重讀403 permanent | request/login count tests |
| Shutdown cleanup順序錯誤 | handler／worker等待到timeout | 固定LIFO registration與ordered test | main cleanup test |
| Metrics counter與gauge失衡 | on-call誤判 | central recorder與所有exit path tests | gatherer assertions |
| Shared capacity造成surface飢餓 | Webhook或Sync延遲 | 本期以backend隔離並觀測，surface pool另案 | queue／latency metrics |

## 安全與隱私

- Flight identity使用不可逆 digest只降低記憶體中的明文metadata，不視為授權或加密邊界。
- 每個 caller 的 authentication、contract與authorization仍獨立執行，只有已授權後的service-credential backend read可共享。
- Raw backend error只在adapter內用於typed classification，不能包裝到domain error或寫入log。
- Secret values只存在backend result、flight完成前的短暫result與各caller clone，不落盤、不進cache或telemetry。
- Queue、retry與timeout settings來自operator config，不接受Pod annotation覆寫。

## 實作順序

1. 以 red tests 固定 config、safe error、metrics與application protected behavior。
2. 收斂 adapter retry ownership與typed classification。
3. 實作 total budget及retry loop。
4. 實作 bounded bulkhead與reference-counted flight coordinator。
5. 實作 Vault re-auth generation coordination。
6. 串接 lifecycle、metrics、deployment與文件。
7. 執行完整 race、static、policy及Kustomize驗證。

## 摘要

- 核心抽象：每個 backend 一個 `ResilientSecretFetcher`
- Retry ownership：SDK 單次，decorator統一控制
- Isolation：per-backend active + bounded queue
- Deduplication：reference-counted in-flight only，無cache
- Cancellation：caller、last waiter、total timeout、process shutdown四層收斂
- 待確認項目：無
