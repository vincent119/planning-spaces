# 需求文件：Authorization Decision Observability 與安全稽核

## 來源

- Draft：無
- Type：Feature
- Owner：Vincent
- Status：Planned

## 文件定位

本 spec 接續 `.specs/2026-08-04-10-52_Feature-secure-secret-access` 與 `.specs/2026-08-04-17-18_Feature-policy-hot-reload`，補上 workload authorization 已執行但不可獨立觀測的缺口。範圍涵蓋 Webhook／Sync 授權結果 metric、選配的結構化 audit log，以及相鄰 operational error 的機密資料 redaction；不重寫 caller authentication、AdmissionReview validation、policy matching、hot reload 或 backend fetch。

參考來源：

- 需求來源：使用者確認先建立 requirements、design、tasks，暫不實作
- 既有程式碼：`internal/syncer/application/authorization.go`、`mutate_usecase.go`、`sync_worker_usecase.go`、`internal/syncer/domain/policy.go`、`internal/infra/metrics/metrics.go`
- 既有文件：`docs/config.zh-Hant.md`、`docs/deploy.zh-Hant.md`、`docs/annotations.zh-Hant.md`
- Prometheus metric naming：<https://prometheus.io/docs/practices/naming/>
- Prometheus instrumentation 與 cardinality：<https://prometheus.io/docs/practices/instrumentation/>

## 背景

Webhook 與 Sync Worker 都會在外部 Vault／AWS fetch 前執行 workload authorization。現有 metrics 可觀察 mutate request、sync error、backend fetch duration、authorization enforcement switch 與 policy reload，但不能區分授權 allow、deny、disabled bypass 或 authorizer 未初始化。

`domain.Decision` 已提供 `Allowed`、`RuleID` 與 `Reason`。目前 allow reason 為 `allowed`，deny reason 為 `no_matching_rule`；application layer 另外處理 authorization disabled 與 authorizer nil。`RuleID`、namespace、ServiceAccount、secret path 與 keys 都不適合成為 Prometheus label。

Discovery 同時確認既有 fetch error 可能包含 `path` 或 `key`，Webhook／Sync 的 operational log 會記錄 raw error。若直接增加 audit log 而不收斂這條路徑，仍可能將機密 metadata 寫入日誌。

## 問題陳述

1. 無法量化 Webhook 與 Sync 的 allow、deny、bypass 趨勢。
2. 無法區分未匹配 policy、authorization 未初始化或明確停用。
3. 無法驗證 deny 是否確實發生在外部 fetch 前。
4. 直接把 workload identity、RuleID 或 annotation 值放入 metrics 會造成高 cardinality 與資訊洩漏。
5. 直接記錄 raw authorization／fetch error 可能暴露 secret path、key、backend resource identifier 或 policy metadata。
6. Audit log 若沒有明確開關與 identity opt-in，會在 production 無意間增加大量 workload metadata。

## 目標

1. 新增單一 workload authorization decision counter，涵蓋 Webhook 與 Sync。
2. 記錄 `allow`、`deny`、`bypass` 與固定 reason taxonomy。
3. 所有 metric label 都使用固定 allowlist，總 series 上限可計算且不得隨 workload 數量增加。
4. 每個有效 SecretRef 恰好記錄一次 authorization observation，且 deny／not initialized 必須在 external fetch 前記錄並停止。
5. Authorization disabled 時記錄 `bypass/disabled`，不把 bypass 誤計為 allow。
6. 提供預設關閉的結構化 audit log，並將 workload identity 設為第二層明確 opt-in。
7. Audit 與 operational log 不輸出 secret path、keys、RuleID、policy fingerprint、完整 policy、credential 或 raw backend error。
8. 收斂既有 Vault／AWS fetch error 與 Webhook／Sync error logging，改用固定安全 category。
9. 不讓 metrics 或 audit recorder failure 改變授權結果、fetch 行為或 admission response。

## 非目標

1. 不觀測 caller authentication 或 AdmissionReview contract validation；兩者另屬 delivery security surface。
2. 不修改 policy schema、matching、RuleID 或 deny-by-default 行為。
3. 不新增 policy source、跨 replica revision、手動 reload trigger 或 management API。
4. 不建立 PrometheusRule、Grafana dashboard、log backend、SIEM pipeline 或 retention policy。
5. 不將 namespace、ServiceAccount、Pod name、Secret name、path、keys、RuleID、request UID 或 error text放入 metrics。
6. 不把 audit log 當作不可否認性或法規合規證明；本功能只提供 application-level structured event。
7. 不新增 tracing span attributes 或 OpenTelemetry event。
8. 不執行 Kubernetes failure/recovery 演練。

## 已定決策

- Metric 名稱固定為 `vault_agent_authorization_decisions_total`。
- Labels 固定為：
  - `surface="webhook|sync"`
  - `decision="allow|deny|bypass"`
  - `reason="matched|no_matching_rule|not_initialized|disabled"`
  - `backend="vault|aws|unknown"`
- 合法 decision/reason pair 只有：`allow/matched`、`deny/no_matching_rule`、`deny/not_initialized`、`bypass/disabled`。
- 固定 series 上限為 `2 surfaces × 4 pairs × 3 backends = 24`；`InitMetrics` 預先初始化這 24 組。
- Backend annotation 是不受信任輸入，除 `vault`、`aws` 外一律正規化為 `unknown`，不得直接作為 label 或 audit field。
- 不直接使用 `domain.Decision.Reason`、`RuleID` 或 error string 當 label；application recorder 只接受固定 enum／constant。
- 未 opt-in、沒有 SecretRef、annotation parse failure 與 caller／Admission contract failure不算 authorization decision。
- Authorization disabled 且存在有效 SecretRef 時記錄 `bypass/disabled`。
- Audit config：
  - `authorization.audit.enabled` 預設 `false`
  - `authorization.audit.include_identity` 預設 `false`
  - `include_identity=true` 但 `enabled=false` 視為設定錯誤
- Audit 啟用但未包含 identity 時，只記錄 surface、decision、reason、normalized backend 與既有 context correlation fields。
- Identity opt-in 時，Webhook 可增加 namespace、ServiceAccount；Sync 只增加 namespace，不捏造 ServiceAccount。
- Audit 永遠不記錄 path、keys、RuleID、Pod／Secret name、policy generation/fingerprint、credential 或 raw error。
- Runtime log 與 error message 維持英文；文件使用繁體中文。

## 待確認項目

- 無。

## 現有行為

- Webhook authorization 在 `MutateUseCase.Execute` 解析 SecretRef 後、選擇 fetcher 前執行。
- Sync authorization 在 `SyncWorkerUseCase.syncSecret` 解析 SecretRef 後、選擇 fetcher 前執行。
- Authorization disabled 時直接略過 policy decision，沒有觀測記錄。
- Authorizer nil 時 fail closed，但沒有獨立 metric。
- Deny error 包含固定 `Decision.Reason`；Webhook handler與 Sync worker會將上層 error寫入 operational log。
- Vault／AWS not-found error 可能包含 path／key，Webhook use case 也會在 fetch error wrapper放入 path。
- Metrics 沒有 namespace／ServiceAccount／path 等高 cardinality labels。

## 新行為

- Application use cases 將固定型別的 authorization observation 傳給注入的 recorder。
- Production recorder 永遠增加固定 label counter；audit 開關只控制 structured log，不控制 metric。
- Allow、deny、bypass、not initialized 各自在外部 fetch 前記錄一次。
- Deny 與 not initialized 記錄後立即返回，fetcher call count 維持 0。
- Audit enabled 時輸出固定英文訊息 `workload authorization decision`；identity fields 依第二層開關加入。
- Backend、decision、reason、surface 先正規化再進 metric/log。
- Webhook 與 Sync operational errors 只記錄固定 error category；backend adapter error 不再組入 path／key。

## 影響範圍

- 使用者：平台工程師、資安人員、on-call 人員
- 功能：Webhook／Sync authorization observation、audit config、safe error category
- API / CLI：HTTP 與 `policy validate` contract 不變
- Data / Storage：不新增 persistence；只增加 metric series 與選配 log event
- Config：新增 `authorization.audit.enabled`、`authorization.audit.include_identity`
- Observability：新增 counter 與安全 audit event
- Deployment：新增兩個明確 audit env，預設均為 false
- 文件：設定、部署、metric taxonomy、cardinality 與資料分類

## 使用情境

- 作為平台工程師，我想比較 Webhook／Sync 的 allow、deny 與 bypass rate，以便發現 policy rollout 或設定異常。
- 作為資安人員，我想在明確啟用時取得安全的授權 audit event，以便調查拒絕趨勢，又不將 secret path 寫入日誌。
- 作為 on-call 人員，我想辨識 `not_initialized` 與 `no_matching_rule`，以便區分 wiring failure 與正常 policy deny。

## 驗收情境

### 情境：允許決策在 fetch 前記錄一次

- 場景：Webhook 或 Sync policy 匹配有效 SecretRef
- 測試：`TestAuthorizationObservation_AllowRecordedBeforeFetch`
- 假設：Recorder 與 fetcher 都能記錄呼叫順序
- 當：use case 執行授權與 fetch
- 那麼：先記錄 `allow/matched`，再呼叫 fetcher，且只記錄一次

### 情境：拒絕決策阻止外部 fetch

- 場景：Policy 沒有匹配 workload 或 resource
- 測試：`TestAuthorizationObservation_DenyRecordedWithoutFetch`
- 假設：Authorization enabled 且 authorizer 回傳 deny
- 當：Webhook 或 Sync 處理 SecretRef
- 那麼：記錄 `deny/no_matching_rule` 一次，fetcher、Kubernetes get/update 都不執行

### 情境：未初始化時 fail closed 且可觀測

- 場景：Authorization enabled 但 authorizer nil
- 測試：`TestAuthorizationObservation_NotInitializedDenied`
- 假設：Recorder 已注入
- 當：Use case 到達授權點
- 那麼：記錄 `deny/not_initialized` 並回傳安全英文錯誤，外部 fetch 不執行

### 情境：停用 authorization 記錄 bypass

- 場景：Authorization disabled 且 SecretRef 有效
- 測試：`TestAuthorizationObservation_DisabledRecordsBypass`
- 假設：其他 caller authentication、contract validation 與 backend behavior 不變
- 當：Use case 略過 policy
- 那麼：記錄 `bypass/disabled`，不得記成 allow

### 情境：不受信任 backend 不增加 cardinality

- 場景：Annotation backend 使用任意敏感 marker
- 測試：`TestAuthorizationRecorder_UnknownBackendIsNormalized`
- 假設：Marker 不是 `vault` 或 `aws`
- 當：Recorder 接收 observation
- 那麼：Metric 與 audit field 都使用 `unknown`，marker 不出現在輸出

### 情境：Metrics 預先初始化固定 series

- 場景：服務尚未收到 SecretRef
- 測試：`TestInitMetrics_InitializesAuthorizationDecisionSeries`
- 假設：呼叫 `metrics.InitMetrics`
- 當：Gather metric family
- 那麼：`vault_agent_authorization_decisions_total` 恰有 24 組合法 labels

### 情境：Audit 預設關閉且不含 identity

- 場景：使用預設 config 或只設定 `audit.enabled=true`
- 測試：`TestAuthorizationAudit_DefaultDisabledAndIdentityOptIn`
- 假設：可擷取 structured log fields
- 當：記錄 authorization observation
- 那麼：預設不產生 audit event；啟用後產生 event，但沒有 namespace／ServiceAccount field

### 情境：Identity 明確 opt-in

- 場景：Audit 與 include identity 都啟用
- 測試：`TestAuthorizationAudit_IdentityRequiresExplicitOptIn`
- 假設：Webhook observation 有 namespace、ServiceAccount；Sync 只有 namespace
- 當：Recorder 寫入 audit event
- 那麼：Webhook 包含兩個 identity fields，Sync 不包含虛構 ServiceAccount，且仍不含 path、keys、RuleID

### 情境：Operational error 不洩漏機密 metadata

- 場景：Backend error 與 annotation 含 path、key、ARN／URL marker
- 測試：`TestAuthorizationObservability_OperationalLogsRedactSensitiveMetadata`
- 假設：Webhook／Sync logger 可擷取，backend 回傳包裝 error
- 當：Fetch 或授權失敗
- 那麼：Log 只有固定 category，不含 marker、raw error、path、key、RuleID 或 credential

### 情境：既有安全流程不被破壞

- 場景：執行 caller authentication、Admission validation、authorization、fetch 與 sync retry 回歸測試
- 測試：`go test -race -count=1 ./...`
- 假設：Audit 維持預設關閉
- 當：執行全套測試
- 那麼：既有 allow／deny、fetch-before-deny、hot reload、readiness 與 graceful behavior 維持不變

## 驗收條件

1. Counter 名稱、四個 label、固定值與 24 series 由 metrics tests 固定。
2. Webhook／Sync allow、deny、not initialized、disabled 各有 observation tests。
3. 每個有效 SecretRef 恰好記錄一次；deny 仍在 external fetch 前停止。
4. Raw backend 值只能正規化為 `vault`、`aws` 或 `unknown`。
5. Metrics 不含 workload identity、Secret metadata、RuleID、fingerprint 或 error text。
6. Audit 預設關閉；identity 必須第二層 opt-in，錯誤 config fail fast。
7. Audit 與 operational log marker tests 證明不洩漏 path、keys、RuleID、raw error 或 credential。
8. Recorder 不回傳 error，不能改變 authorization 或 fetch control flow。
9. Vault／AWS errors 保留 `errors.Is` 語意，但不把 path／key放入 error text。
10. Config、Deployment 與繁中維運文件對齊。
11. Race test、vet、固定版本 lint、policy validation、Kustomize render 與 diff checks 通過。

## 驗證需求

- Unit：metric taxonomy、backend normalization、config、audit switch、identity fields、安全 error category
- Application：Webhook／Sync allow、deny、nil、disabled、exactly-once 與 order tests
- Infra：Vault／AWS not-found／fetch failure `errors.Is` 與敏感 marker redaction
- Delivery：Admission response 維持 generic，captured log 不含 raw use case error
- Concurrency：`go test -race -count=1 ./...`
- Static：`go vet ./...`、`make lint`
- Regression：`make policy-validate`、base/prod Kustomize render
- 文件：metric labels、24 series、audit defaults、identity opt-in 與資料禁區

## 風險與假設

| 類型 | 內容 | 處理方式 |
|------|------|----------|
| 風險 | Backend annotation 形成無界 label | 僅允許 vault／aws，其他固定為 unknown |
| 風險 | Identity audit 增加 workload metadata exposure | 預設關閉且第二層 opt-in；禁止進 metrics |
| 風險 | Allow audit 造成高 log volume | Audit 預設關閉；文件要求依容量與 retention 評估 |
| 風險 | Raw backend error 洩漏 URL、ARN、path 或 key | Operational log只記固定 category；adapter error移除 path/key |
| 風險 | Observation 重複計數 | 每個 use case 只有單一授權 observation point與順序測試 |
| 風險 | Metrics 與授權邏輯耦合後影響安全決策 | 注入 no-error recorder；decision control flow仍由 authorizer決定 |
| 假設 | Surface、decision、reason、backend taxonomy 是封閉集合 | constants、normalization 與 exact-series tests 保護 |
| 假設 | Namespace／ServiceAccount 屬需明確授權的 operational metadata | 只允許 identity audit opt-in，不進 metrics |

## 摘要

- 關鍵決策：單一 24-series counter、固定 taxonomy、backend normalization、audit 雙層開關、安全 error category
- 安全邊界：不記錄 path、keys、RuleID、policy fingerprint、credential 或 raw backend error
- 待確認項目：無
- 下一步：等待使用者確認後，依 `tasks.md` 執行；本階段不實作
