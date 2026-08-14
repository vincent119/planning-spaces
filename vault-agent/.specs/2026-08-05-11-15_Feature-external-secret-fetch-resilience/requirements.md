# 需求文件：External Secret Fetch 韌性與隔離

## 來源

- Draft：無
- Type：Feature
- Owner：Vincent
- Status：Complete

## 文件定位

本 spec 接續既有 caller authentication、Admission contract validation、workload authorization、Secret 同步與 graceful shutdown，補上授權完成後存取 Vault／AWS Secrets Manager 的故障控制。範圍涵蓋總逾時、有限重試、每個 backend 的 bulkhead、同一讀取的 in-flight 合併、Vault 重新認證收斂，以及固定低基數觀測；不新增 Secret 快取，也不改變任何授權邊界。

參考來源：

- 需求來源：使用者要求建立 `requirements.md`、`design.md`、`tasks.md`，暫不實作
- 既有程式碼：`internal/syncer/domain/secret.go`、`internal/syncer/infra/vault_client.go`、`aws_client.go`、application use cases、`internal/infra/metrics/metrics.go`、`cmd/vault-agent/main.go`
- 固定 dependency：`github.com/hashicorp/vault/api v1.22.0`、`github.com/aws/aws-sdk-go-v2 v1.41.3`
- 既有安全文件：secure secret access、authorization observability、admission security observability specs

## 背景

Webhook 與 Sync Worker 都會在通過安全檢查及 workload authorization 後，透過共用的 `domain.SecretFetcher` 直接呼叫外部 backend。目前 Vault 與 AWS adapter 沒有一致的 fetch 總時間預算、應用層重試政策、backend 併發上限或排隊上限。同一 Pod 建立突發、同步輪詢或 backend 變慢時，請求會直接累積到外部系統。

Pinned Vault client 預設對 5xx 額外重試 2 次，AWS SDK 也有內建 retryer。若另加外層重試而不關閉內層策略，單一 logical fetch 的實際呼叫次數會相乘，無法用 `max_attempts` 約束。

Vault Kubernetes auth 遇到 403 時會重新登入並重試一次；多個不同 path 同時收到 403 時，每個 goroutine 都可能各自登入，形成認證風暴。現有 `vault_agent_secret_fetch_duration_seconds{backend}` 可觀察 application fetch 延遲，但看不到重試、過載、併發、排隊或 in-flight 合併。

## 問題陳述

1. 外部 backend 變慢時，單次 fetch 沒有跨排隊、退避與所有嘗試的統一總時間預算。
2. Vault／AWS SDK 的內建重試與新增外層重試可能相乘，放大 backend 故障。
3. Webhook 與 Sync Worker 共用 backend client，但沒有每個 backend 獨立的併發與排隊上限。
4. 同一 backend、path 與 keys 的並行讀取會重複打向外部系統。
5. Vault token 失效時，多個請求可能同時重新執行 Kubernetes login。
6. Caller 或服務關機後，排隊、退避或外部呼叫必須能及時停止，不能拖過 graceful shutdown 預算。
7. 若用 path、keys、ARN、錯誤文字或其他外部輸入做 metrics／logs，會造成機密 metadata 洩漏或無界 cardinality。

## 目標

1. 對每次 logical fetch 套用單一總時間預算，涵蓋 bulkhead 排隊、重試退避、Vault 重新認證與所有 backend attempts。
2. 只對冪等讀取的明確暫時性錯誤重試，並以 `max_attempts` 約束一般 adapter attempts；Vault auth recovery 最多另有一次驗證讀取，整體上限仍可計算。
3. Vault 與 AWS 各自使用獨立的 active／queued 容量，使一個 backend 飽和時不占用另一個 backend 的額度。
4. 合併相同 backend、path 與 canonical keys 的 in-flight fetch；完成後立即移除，不保留結果快取。
5. 同一時間只允許一個 Vault Kubernetes re-auth 執行，等待者共用刷新結果。
6. Caller 取消時立即停止等待；所有 waiter 都取消或服務進入 shutdown 時，取消共用 backend operation。
7. 保留 `ErrSecretNotFound`、`ErrSecretFetchFailed` 與 context cancellation 的既有上層語意，Webhook 回應仍為 generic failure。
8. 新增固定低基數 metrics，觀察 final result、retry、shared fetch、active 與 queue depth。
9. Logs、metrics、errors 不得包含 Secret value、path、keys、ARN、Vault URL、credential、raw backend error 或 in-flight identity。
10. 用 deterministic unit tests、concurrency tests 與 race detector 驗證時間、重試上限、隔離、合併及取消行為。

## 非目標

1. 不建立成功或失敗結果快取，不設定 TTL，也不延長已撤銷 Secret 的可見時間。
2. 不修改 caller authentication、Admission contract、workload authorization、policy schema、matching 或安全 gate 順序。
3. 不新增 circuit breaker、backend health probe、自動 failover、跨 backend fallback 或 fallback value。
4. 不新增每個 namespace、ServiceAccount、surface、path 或 backend 個別 override。
5. 不改為平行 Sync Worker，不建立 Webhook／Sync 優先級或獨立容量池。
6. 不新增 Vault／AWS credential rotation 機制；只收斂既有 Vault Kubernetes re-auth。
7. 不新增 dashboard、PrometheusRule、alert、SLO、distributed cache 或跨 replica request coalescing。
8. 不執行 production failure／recovery 演練；演練保留最後整合階段。

## 已定決策

### 設定契約

新增共用設定區塊，數值會分別套用到 Vault 與 AWS 的獨立 resilient fetcher：

```yaml
secret_fetch:
  timeout_seconds: 4
  max_attempts: 3
  initial_backoff_milliseconds: 100
  max_backoff_milliseconds: 800
  max_concurrent_requests: 16
  max_queued_requests: 32
```

- `timeout_seconds` 是整個 logical fetch 的總預算，不是每個 attempt 的 timeout。
- 每個 caller 的等待 deadline 取 caller context 與設定總預算中較早者；共用 operation 本身不得超過設定總預算，並受 last-waiter 與 process lifecycle cancellation 約束。
- `max_attempts` 包含第一次呼叫；設為 1 表示不重試。
- Backoff 使用 capped exponential full jitter，且等待時間包含在總預算內。
- `max_concurrent_requests` 與 `max_queued_requests` 是每個 backend 各自的限制，不是 Vault／AWS 共用一組 token。
- Queue 已滿時 fail fast，回傳可由 `errors.Is` 辨識且仍屬 `ErrSecretFetchFailed` 的安全 overload error。
- In-flight 合併固定啟用，不另設開關；它只共享進行中的結果，不是 cache。

環境變數固定為：

- `SECRET_FETCH_TIMEOUT_SECONDS`
- `SECRET_FETCH_MAX_ATTEMPTS`
- `SECRET_FETCH_INITIAL_BACKOFF_MILLISECONDS`
- `SECRET_FETCH_MAX_BACKOFF_MILLISECONDS`
- `SECRET_FETCH_MAX_CONCURRENT_REQUESTS`
- `SECRET_FETCH_MAX_QUEUED_REQUESTS`

設定驗證必須 fail fast：timeout、attempts、concurrency 必須大於 0；queue 可為 0；backoff 必須大於 0，且 initial 不得大於 max。所有錯誤訊息使用英文。

### 重試與錯誤分類

- Vault SDK `MaxRetries` 設為 0；AWS SDK `RetryMaxAttempts` 設為 1，由 resilient fetcher 唯一控制 attempts。
- 只重試 adapter 在 raw error 邊界判定為暫時性的錯誤；raw error 不得離開 adapter。
- Vault 可重試：連線暫時錯誤及 HTTP 408、412、429、500、502、503、504。
- AWS 可重試：AWS standard retry classifier 判定為 retryable 的 transport／throttling／server error。
- `context.Canceled`、`context.DeadlineExceeded`、not found、invalid request、authentication／authorization、資料格式錯誤皆不重試。
- Vault Kubernetes auth 的 403 維持一次 re-auth 與一次讀取重試；它是 credential refresh，不計入一般 transient retry。重新讀取仍為 403 時視為 permanent failure。
- 一個 logical fetch 的一般 adapter attempt 永遠不得超過 `max_attempts`。Vault re-auth 後最多另有一次驗證讀取，因此 Vault Secret read HTTP requests 上限為 `max_attempts + 1`；Kubernetes login 是另一個 auth request，仍受同一總時間預算約束。

### In-flight 合併與取消

- 合併 identity 以 backend 內部、path 與 canonical keys 的 length-prefixed bytes 計算 SHA-256；hash 不得輸出到 log、metric 或 trace。
- Keys 使用排序及去重後的副本建立 identity，不修改 caller slice；空 keys 與指定 keys 必須保持不同語意。
- Workload authorization 仍在呼叫 fetcher 前逐一完成，in-flight 合併不得略過任何 caller 的授權。
- 每個 caller 都收到獨立 clone 的結果 map，避免共享可變 map 造成 data race。
- Caller context 取消時立即返回；其他 waiter 不受影響。所有 waiter 都離開時取消共用 operation。
- Process shutdown 取消所有 active、queued、backoff、re-auth 及 shared operations。
- Operation 完成後立即刪除 flight entry，不保留 Secret value 或 failure result。

### Metrics 契約

新增下列 collectors：

- `vault_agent_secret_fetch_requests_total{backend,result}`
  - `backend="vault|aws"`
  - `result="success|not_found|failed|timeout|canceled|overloaded"`
  - 每個 caller 最終結果記錄一次，固定 12 series。
- `vault_agent_secret_fetch_retries_total{backend}`：每次排定的一般 retry 記錄一次，固定 2 series。
- `vault_agent_secret_fetch_shared_total{backend}`：caller 加入既有 in-flight operation 時記錄一次，固定 2 series。
- `vault_agent_secret_fetch_inflight{backend}`：實際占用 active backend slot 的 operation 數，固定 2 series。
- `vault_agent_secret_fetch_queue_depth{backend}`：等待 active slot 的 operation 數，固定 2 series。

既有 `vault_agent_secret_fetch_duration_seconds{backend}` 保留，繼續量測 caller 看到的完整 fetch latency。新 metrics 不增加 path、keys、surface、namespace、ServiceAccount、error、status code 或 attempt number label。

## 現有行為

- Webhook 與 Sync 各自在 application layer 量測 fetch duration，然後直接呼叫 Vault／AWS adapter。
- Vault client 使用 `vault.DefaultConfig()`，預設最多三次 5xx attempts；AWS client 使用 SDK default retryer。
- Vault Kubernetes auth 每個遇到 403 的 goroutine 都可能自行 re-auth。
- 沒有 per-backend active／queue 限制、in-flight 合併或 fetch lifecycle cleanup。
- Adapter 對上層只回傳安全 sentinel，但尚未提供 retryable／overloaded 分類。

## 新行為

- Main 為每個已啟用 backend 建立一個 `ResilientSecretFetcher` decorator，再注入 Webhook 與 Sync。
- 每個 decorator 有獨立 bulkhead、flight coordinator、retry loop 與 metrics backend label。
- Adapter 在 raw SDK error 邊界先分類，再只回傳 context error、not found、retryable safe error或 permanent safe error。
- 相同讀取先通過各 caller 自己的 authorization，再在 decorator 內合併；只有 leader 取得 queue／active slot並呼叫 backend。
- Queue full 時立即拒絕；queue wait、backoff 與 backend call 都受相同總時間預算限制。
- Shutdown 先取消 fetch lifecycle，使 HTTP handler 與 Sync Worker 不會被 detached shared operation拖住，再依既有 graceful cleanup 收斂。

## 影響範圍

- 使用者：平台工程師、on-call 人員、資安人員
- 功能：Vault／AWS fetch retry、timeout、bulkhead、in-flight 合併、Vault re-auth
- API / CLI：AdmissionReview、HTTP path、policy CLI contract 不變
- Data / Storage：不新增 persistence 或 cache；只短暫保存進行中的 result
- Config：新增 `secret_fetch` 與六個環境變數
- Observability：新增五個固定低基數 metric families
- Deployment：Base Deployment 明確設定安全預設值；RBAC、volume、failurePolicy 不變
- Dependency：不新增 runtime dependency；flight coordinator 使用專案內部同步原語與標準庫
- 文件：設定、部署、觀測、取消與無 cache 語意

## 使用情境

- 作為平台工程師，我想限制每個 backend 的同時呼叫與排隊數，避免 Vault 故障拖垮 AWS 讀取或服務記憶體。
- 作為 on-call 人員，我想看見 timeout、overload、retry、queue 與 shared fetch，快速分辨 backend 變慢、容量不足或請求重複。
- 作為資安人員，我想確保合併及觀測不略過 workload authorization，也不儲存或輸出 Secret metadata。
- 作為應用團隊，我想在暫時性 backend 錯誤時取得有限重試，又不因巢狀 SDK retry 造成長時間 Admission 延遲。

## 驗收情境

### 情境：總時間預算涵蓋完整 fetch

- 測試：`TestResilientSecretFetcher_TotalBudgetIncludesQueueBackoffAndAttempts`
- 假設：使用 fake clock、阻塞 backend 與可控 queue
- 當：queue wait、retry backoff 或 backend attempt 耗盡總預算
- 那麼：caller 收到可辨識的 deadline error，後續 attempt 不開始，所有 token 最終釋放

### 情境：暫時性讀取有限重試

- 測試：`TestResilientSecretFetcher_RetriesOnlyRetryableReads`
- 假設：前兩次回傳 retryable safe error，第三次成功
- 當：執行 logical fetch
- 那麼：總共呼叫 backend 三次、retry counter 增加兩次，結果成功

### 情境：永久錯誤與取消不重試

- 測試：`TestResilientSecretFetcher_DoesNotRetryTerminalErrors`
- 假設：表格涵蓋 not found、permission denied、invalid response、canceled 與 deadline
- 當：第一次 attempt 回傳任一錯誤
- 那麼：backend call count 維持 1，final result category 固定且不含 raw error

### 情境：SDK 內層重試不會倍增

- 測試：`TestBackendClients_DisableNestedRetries`
- 假設：Vault／AWS transport 持續回傳 retryable server error
- 當：resilient fetcher 的 `max_attempts=3`
- 那麼：每個 backend 的實際 request 數符合已記錄的 bounded contract，不出現 SDK 額外 attempts

### 情境：Backend bulkhead 相互隔離

- 測試：`TestResilientSecretFetcher_BackendBulkheadsAreIndependent`
- 假設：Vault active 與 queue 都已滿，AWS 尚有容量
- 當：新的 Vault 與 AWS fetch 同時進入
- 那麼：Vault 立即回傳 overload，AWS 可正常開始，不共用 token

### 情境：Queue 容量有界且取消會釋放

- 測試：`TestResilientSecretFetcher_QueueBoundAndCancellationReleaseCapacity`
- 假設：active slot 被占用，queue 到達設定上限
- 當：下一個唯一 fetch 進入，或 queued caller 取消
- 那麼：超額 fetch fail fast；取消的 entry 離開 queue，gauge 與 token 回到正確值

### 情境：相同 in-flight fetch 只打一次 backend

- 測試：`TestResilientSecretFetcher_CoalescesEquivalentInflightRequests`
- 假設：多個已授權 caller 使用相同 path，keys 僅順序或重複不同
- 當：同時執行 fetch
- 那麼：backend 只執行一次，followers 增加 shared counter，每個 caller 取得內容相同但 map identity 不同的結果

### 情境：合併不形成 cache

- 測試：`TestResilientSecretFetcher_DoesNotCacheCompletedResult`
- 假設：第一個 flight 已完成，backend value 隨後變更或被撤銷
- 當：再次執行相同 fetch
- 那麼：必須重新呼叫 backend，不回傳先前完成的 value 或 error

### 情境：Waiter 取消互不干擾

- 測試：`TestResilientSecretFetcher_CancelsSharedOperationAfterLastWaiter`
- 假設：兩個 caller 共用同一 operation
- 當：第一個 caller 取消但第二個仍等待，之後第二個也取消
- 那麼：第一個立即返回、operation 繼續服務第二個；最後 waiter 離開後 backend context 被取消

### 情境：Vault re-auth 只執行一次

- 測試：`TestVaultClient_CoalescesConcurrentKubernetesReauthentication`
- 假設：多個不同 path 同時收到同一 token generation 的 403
- 當：請求進入 re-auth flow
- 那麼：Kubernetes login 只執行一次，等待者使用新 token 各自重讀；重讀 403 不進一般 retry

### 情境：Graceful shutdown 取消所有 fetch 狀態

- 測試：`TestFetchLifecycleShutdown_CancelsActiveQueuedBackoffAndReauth`
- 假設：active、queued、backoff 與 re-auth 各有阻塞工作
- 當：graceful cleanup 取消 fetch lifecycle
- 那麼：所有工作在 shutdown timeout 內結束，queue／inflight gauges 歸零，HTTP 與 Sync cleanup 不被卡住

### 情境：Metrics 與 log 不洩漏 metadata

- 測試：`TestSecretFetchResilience_ObservabilityRedactsSensitiveMetadata`
- 假設：path、keys、ARN、URL 與 raw error 都含唯一 marker
- 當：success、retry、timeout、overload、not found 與 permanent failure 發生
- 那麼：metric labels、log、safe error 及 trace attributes 都不包含任何 marker或 flight hash

## 驗收條件

1. 六個設定欄位具備 defaults、env binding、validation、sample、Deployment 與繁中文件。
2. Timeout 是跨 queue、backoff、re-auth 與所有 attempts 的總預算，caller 較早 deadline 優先。
3. Vault／AWS 內層 retry 被限制為單次；一般 adapter attempts 不超過 `max_attempts`，Vault auth recovery 最多另有一次 Secret驗證讀取。
4. Retry classifier 只在 adapter raw error 邊界使用型別／status，不解析 error string。
5. Not found、authn/authz、invalid response、canceled 與 deadline 不重試。
6. Vault／AWS active 與 queue token 完全獨立；queue full 會 fail fast。
7. Equivalent in-flight calls 只執行一次 backend read，完成後下一次一定重新讀取。
8. Shared result 對每個 caller clone；`go test -race` 不出現 map race。
9. Waiter ref count、caller cancellation、last-waiter cancellation 與 process shutdown 皆有 deterministic tests。
10. Vault 同 token generation 的並行 403 只觸發一次 Kubernetes login。
11. 新 metric label values 是封閉集合，固定 series 數可由 tests 驗證。
12. Path、keys、Secret value、resource identifier、credential、raw error 與 flight hash 不進 metric、log、error 或 trace。
13. Caller authentication、Admission contract、workload authorization 仍先於 fetch；deny call count 維持 0。
14. Admission response、Sync Secret replacement、policy reload、readiness 與既有 graceful cleanup contract 不變。
15. Race、unit、vet、lint、policy validation、Kustomize render 與 diff checks 通過。

## 驗證需求

- Unit：config validation、safe errors、retry classification、backoff、canonical identity、result clone
- Concurrency：bulkhead、bounded queue、coalescing、waiter ref count、Vault re-auth generation、shutdown
- Application：authorization-before-fetch、Webhook generic response、Sync cancellation與既有 duration metric
- Infra：Vault／AWS nested retry disabled、typed classifier、raw error redaction
- Metrics：固定 label schema、series 數、counter／gauge balance、敏感 marker absence
- Static：`go vet ./...`、`make lint`、`go list ./...`
- Regression：`go test -race -count=1 ./...`、`make policy-validate`、base/prod Kustomize render
- 文件：defaults、調校方式、總預算、無 cache、metric taxonomy 與資料禁區一致

## 風險與假設

| 類型 | 內容 | 處理方式 |
|------|------|----------|
| 風險 | Retry 與 SDK retry 相乘 | 關閉 Vault／AWS 內層 retry，由單一 decorator 計數 |
| 風險 | Bulkhead blocking 仍累積無界 goroutine | Active 與 queue 分開設限；queue full fail fast |
| 風險 | 同一 flight 的 caller 取消互相影響 | Waiter ref count；只在最後 waiter 離開時取消 operation |
| 風險 | Shared result map 被 caller 修改 | 每個 caller 回傳獨立 clone，race test 固定 |
| 風險 | In-flight identity 保存敏感 path | 只保存 SHA-256 digest，完成即刪除，且禁止輸出 digest |
| 風險 | Vault 403 同時觸發 login storm | Token generation 與 single re-auth coordinator |
| 風險 | Timeout 預設不適合所有環境 | 預設 4 秒並提供設定；caller 較早 deadline 永遠優先 |
| 風險 | Webhook 與 Sync 共用 backend 容量互相飢餓 | 本階段接受；metrics 提供證據，優先級隔離另案規劃 |
| 假設 | Vault／AWS fetch 都是冪等 read | 只在 `SecretFetcher.FetchSecret` 套用 retry，不擴及 Secret update |
| 假設 | Backend credentials 是 replica process 內共用 | Coalescing 不改授權結果，因 workload authorization 已逐 caller 完成 |

## 摘要

- 關鍵決策：單一總預算、單層 bounded retry、per-backend bulkhead、bounded queue、reference-counted in-flight 合併、Vault re-auth 收斂
- 安全邊界：不做 cache，不略過 authorization，不輸出 Secret metadata、raw error 或 flight identity
- 待確認項目：無
- 下一步：依 production traffic 評估 Webhook／Sync 獨立容量池與 circuit breaker；failure drill 維持最後整合階段
