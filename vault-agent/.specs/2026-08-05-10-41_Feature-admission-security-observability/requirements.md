# 需求文件：Caller Authentication 與 Admission Contract 安全觀測

## 文件定位

本 spec 接續 `.specs/2026-08-04-10-52_Feature-secure-secret-access` 與 `.specs/2026-08-04-18-00_Feature-authorization-decision-observability`。前者已建立 caller authentication、AdmissionReview contract validation 與 workload authorization 的安全順序；後者只觀測 workload authorization，並明確將 delivery security surface 留待另案。本 spec 補上 caller authentication 與 Admission contract 的低基數 metric及安全拒絕日誌，不重寫 authentication、contract validation、mutation、workload authorization或backend fetch。

## 背景

Webhook目前會先限制HTTP method，再執行`RequestAuthenticator`；認證成功後才讀取、decode並驗證AdmissionReview。只有通過contract的請求才會呼叫MutateUseCase，因此未認證或無效contract不會觸發外部機密讀取。

現有`vault_agent_mutate_requests_total`只記錄進入MutateUseCase後的成功或錯誤，`vault_agent_authorization_decisions_total`只記錄workload policy allow／deny／bypass。Caller authentication拒絕、authenticator未設定、disabled access、AdmissionReview decode失敗與contract欄位拒絕目前無獨立metric。

Admission contract拒絕目前會記錄raw validation error；caller authentication拒絕則沒有安全分類日誌。若直接把auth error、principal、certificate、Authorization header或AdmissionReview內容加入觀測，可能洩漏credential、呼叫端身分或workload metadata，並造成Prometheus高基數series。

## 問題陳述

1. 無法區分請求是在caller authentication或Admission contract階段被拒絕。
2. 無法判斷認證拒絕是一般驗證失敗、authenticator未設定或secret access明確停用。
3. Admission contract拒絕只有HTTP 400與raw operational error，缺少固定安全分類。
4. Caller principal、certificate subject／URI、token、UID、namespace與request body都不適合作為metric label。
5. Security gate觀測不得與workload authorization的surface／decision／reason taxonomy混用。

## 目標

1. 新增`vault_agent_admission_security_decisions_total` counter。
2. 使用固定`gate`、`decision`、`reason`三個labels，預先建立且只允許14組series。
3. Caller authentication對每個到達該gate的POST請求恰記一次allow或deny。
4. Admission contract對每個已通過caller authentication且到達decode／validation的請求恰記一次allow或deny。
5. Authentication deny與contract deny都在後續gate、mutation、authorization與fetch前完成記錄並停止流程。
6. Operational reject log的security-specific fields只輸出固定category，不輸出raw error或request-derived metadata。
7. 保持既有HTTP status、response body、validation順序與MutateUseCase行為。
8. 建立metric taxonomy、order、exactly-once、redaction與回歸測試。

## 非目標

1. 不修改bearer constant-time comparison、mTLS chain validation或principal allowlist語意。
2. 不新增authentication mode、principal、certificate issuer／subject／URI或token fingerprint metric。
3. 不將AdmissionRequest.UserInfo視為caller authentication。
4. 不修改AdmissionReview schema、接受的resource／kind／operation或namespace一致性規則。
5. 不觀測HTTP method gate、TLS handshake failure、Kubernetes API Server端的webhook transport error或metrics endpoint authentication。
6. 不修改workload authorization counter、audit設定或policy decision。
7. 不新增dashboard、PrometheusRule、alert、trace、SIEM exporter或dependency。
8. 不執行target cluster failure drill。

## 功能需求

### R1：固定安全觀測counter

Metric名稱固定為：

```text
vault_agent_admission_security_decisions_total
```

Labels固定為：

```text
gate, decision, reason
```

合法組合固定如下：

| gate | decision | reason |
|------|----------|--------|
| caller_authentication | allow | authenticated |
| caller_authentication | deny | authentication_failed |
| caller_authentication | deny | not_configured |
| caller_authentication | deny | access_disabled |
| admission_contract | allow | valid |
| admission_contract | deny | decode_failed |
| admission_contract | deny | invalid_type_meta |
| admission_contract | deny | missing_request_fields |
| admission_contract | deny | invalid_resource |
| admission_contract | deny | invalid_kind |
| admission_contract | deny | invalid_operation |
| admission_contract | deny | missing_namespace |
| admission_contract | deny | invalid_object |
| admission_contract | deny | namespace_mismatch |

除以上14組外不得建立其他series。不得直接使用error string、CallerIdentity、HTTP header或AdmissionReview欄位作label。

### R2：Caller authentication觀測

- 非POST請求維持既有405，因未到達caller authentication gate，不記錄本counter。
- Authenticator為nil時記錄`deny/not_configured`，維持503。
- `DisabledAuthenticator`拒絕時記錄`deny/access_disabled`，維持401。
- Bearer、mTLS或其他authenticator拒絕時統一記錄`deny/authentication_failed`，不區分credential不存在或錯誤。
- 成功時記錄`allow/authenticated`，不得記錄`CallerIdentity.Method`或`Principal`。
- Deny記錄後不得decode body、呼叫mutator或進入任何機密讀取路徑。

### R3：Admission contract觀測

- 只有caller authentication成功後才會進入Admission contract gate。
- JSON decode、body過大或body格式錯誤統一記錄`deny/decode_failed`。
- `validateAdmissionReview`依既有檢查順序回傳固定reason，不使用error text推斷分類。
- Contract有效時記錄`allow/valid`，再呼叫MutateUseCase。
- Contract deny記錄後不得呼叫MutateUseCase、workload authorizer或backend fetch。

### R4：Exactly-once與順序

- 一個有效POST請求會產生一筆caller authentication allow與一筆Admission contract allow。
- Auth deny只產生一筆caller authentication deny，不產生Admission contract observation。
- Contract deny會先有caller authentication allow，再有一筆Admission contract deny。
- Recorder必須無error回傳；metric或log觀測不得改變安全控制流程。

### R5：安全operational log

- Authentication deny記錄固定message與`gate`、`decision`、`reason`。
- Admission contract deny記錄固定message與`gate`、`decision`、`reason`。
- Allow只由metric觀測，避免每個合法request產生額外security log。
- Log不得包含Authorization header、token、principal、certificate、request body、UID、namespace、Pod name、path、keys、RuleID、raw error或credential。
- Decode與validation error不得透過`zlogger.Err`寫入reject log。
- Logger既有timestamp、level、caller、component、operation與程式產生的request ID等標準欄位可保留；不得以Admission UID或外部identity取代request ID。

### R6：既有契約不變

- HTTP method不符：405與`method not allowed`。
- Authenticator未設定：503與`caller authentication is not configured`。
- Caller authentication失敗：401與`caller authentication failed`。
- Decode失敗：400與`bad request`。
- Contract validation失敗：400與`invalid AdmissionReview`。
- Mutator error仍回傳AdmissionResponse `Allowed=false`與generic `secret injection failed`。
- `vault_agent_mutate_requests_total`與`vault_agent_authorization_decisions_total`語意不變。

## 現有行為

1. `WebhookHandler.ServeHTTP`依序執行method、authenticator、body limit、decode、`validateAdmissionReview`與mutator。
2. `RequestAuthenticator.Authenticate`回傳`CallerIdentity`與error；handler目前丟棄identity。
3. Bearer與mTLS一般失敗共用`errAuthenticationFailed`；disabled使用獨立固定error text。
4. `validateAdmissionReview`回傳一般error，handler會用`zlogger.Err`記錄。
5. Metrics目前沒有delivery security gate counter。

## 新行為

1. Delivery layer使用固定enum建立security observation，不從raw error或外部欄位生成label。
2. Authenticator error以sentinel／typed category區分一般失敗與access disabled。
3. Admission validator以固定typed reason表示第一個失敗的既有contract條件。
4. Handler在每個gate出口恰記一次，並維持原本return位置。
5. Production wiring注入Prometheus recorder；測試使用fake recorder驗證事件與順序。

## 影響範圍

- `internal/infra/metrics`：counter與14組series初始化
- `internal/syncer/delivery`：security observation、auth error分類、contract reason與handler integration
- `cmd/vault-agent`：production recorder wiring
- 繁體中文README／部署文件：metric taxonomy、查詢與資料禁區
- 對應unit、integration與regression tests

## 使用情境

- 作為維運人員，我想區分authentication reject與contract reject，以判斷錯誤發生在哪個安全gate。
- 作為資安人員，我想觀察固定分類的拒絕趨勢，又不取得token、principal或AdmissionReview內容。
- 作為開發人員，我想確保新增observability不改變fail-closed順序與HTTP contract。

## 驗收情境

### 情境：固定14組series

- 場景：程序尚未處理任何Webhook request
- 測試：`TestInitMetrics_InitializesFixedAdmissionSecuritySeries`
- 假設：執行`metrics.InitMetrics()`
- 當：收集Prometheus metric family
- 那麼：counter恰有14組series，labels只有gate、decision、reason

### 情境：Authentication成功後才進入contract

- 場景：合法caller提交合法AdmissionReview
- 測試：`TestWebhookHandler_SecurityObservationOrder`
- 假設：fake authenticator成功且fake recorder可記錄順序
- 當：handler處理POST request
- 那麼：依序記錄authentication allow、contract allow，再呼叫mutator

### 情境：Authentication拒絕停止後續流程

- 場景：錯誤bearer或未允許mTLS principal
- 測試：`TestWebhookHandler_AuthenticationDeniedObservation`
- 假設：body與mutator都能偵測是否被存取
- 當：authenticator回傳一般authentication error
- 那麼：只記一筆`caller_authentication/deny/authentication_failed`，body未decode且mutator未呼叫

### 情境：Disabled與未設定分類固定

- 場景：secret access disabled或handler缺少authenticator
- 測試：`TestWebhookHandler_AuthenticationControlStateObservation`
- 假設：兩個情境分別執行
- 當：request到達authentication gate
- 那麼：分別記錄`access_disabled`與`not_configured`，HTTP status維持401與503

### 情境：Contract拒絕使用固定reason

- 場景：已認證request包含不同無效AdmissionReview contract
- 測試：`TestAdmissionContractReasonTaxonomy`
- 假設：table涵蓋decode與八種validator失敗
- 當：handler依既有順序驗證
- 那麼：每個case只記一筆對應contract deny，mutator未呼叫

### 情境：安全reject log不洩漏marker

- 場景：auth error、principal、body與UID含敏感marker
- 測試：`TestSecurityRejectLogFields_RedactSensitiveMarkers`
- 假設：可擷取結構化log fields
- 當：authentication或contract拒絕
- 那麼：security-specific fields只有固定gate／decision／reason，不含marker或raw error

### 情境：Workload authorization觀測不受影響

- 場景：request通過delivery security gates並進入既有authorization
- 測試：`TestInitMetrics_KeepsSecurityAndAuthorizationTaxonomiesSeparate`
- 假設：既有authorization recorder與新security recorder都可觀測
- 當：workload policy作出決策
- 那麼：security counter不加入backend／namespace，authorization counter不加入security gate

## 驗收條件

1. 新counter名稱與三個labels固定。
2. 預初始化series恰為14組，無runtime動態reason。
3. Authentication與contract observation符合exactly-once與既有執行順序。
4. Authentication deny與contract deny均在mutator／authorization／fetch前停止。
5. Disabled是deny/access_disabled，不是allow或bypass。
6. Metric與log不含principal、token、certificate、body、UID、namespace或secret metadata。
7. HTTP status、response body、contract規則與workload authorization行為不變。
8. Race tests、vet、lint、policy validation與Kustomize render通過。

## 驗證需求

- Unit：taxonomy、14 series、auth error分類、contract reason、safe fields
- Delivery：exactly-once、order、deny短路與HTTP response回歸
- Main：production recorder已注入
- Regression：既有authentication、AdmissionReview、authorization與mutation tests
- Static：敏感marker、動態label、`zlogger.Err` reject path與Boundary檢查
- Quality：`go test -race -count=1 ./...`、`go vet ./...`、`make lint`、`make policy-validate`、base/prod Kustomize render與`git diff --check`

## 風險與假設

| 類型 | 內容 | 處理方式 |
|------|------|----------|
| 風險 | 每個有效request增加兩次counter更新 | 固定CounterVec且無I/O；以測試確認不改control flow |
| 風險 | Validator重構改變first-failure語意 | 保持檢查順序並建立table regression |
| 風險 | Authenticator實作回傳任意error造成動態分類 | 未識別error一律正規化為authentication_failed |
| 風險 | Security與authorization taxonomy混淆 | 使用獨立metric與recorder，不共享labels |
| 假設 | 非POST request不屬於本次兩個security gates | 維持405但不記本counter |
| 假設 | Allow request不需要逐筆security log | Allow只記metric，deny才記固定category log |

## 參考

- `.specs/2026-08-04-10-52_Feature-secure-secret-access/`
- `.specs/2026-08-04-18-00_Feature-authorization-decision-observability/`
- `internal/syncer/delivery/authenticator.go`
- `internal/syncer/delivery/webhook_handler.go`
- `internal/infra/metrics/metrics.go`
