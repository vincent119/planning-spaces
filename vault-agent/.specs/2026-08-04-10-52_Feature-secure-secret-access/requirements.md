# 安全機密存取需求

## 文件定位

- 類型：Feature
- 狀態：Planned
- 來源：`_workspace/05_review_summary.md` 的 P0 修正項目
- 接續模組：既有 Mutating Admission Webhook、SecretFetcher、Sync Worker 與 Kubernetes deployment manifests
- 本 spec 不重寫 Vault/AWS client 的取值實作，也不處理機密快取、同步刪鍵語意、OTLP TLS 或 Worker graceful shutdown

## 背景

目前 `/mutate` 僅檢查 HTTP method，任何可連線的叢集內工作負載都能提交偽造 AdmissionReview。請求中的 annotation 會直接決定 backend、path 與 keys，系統未依 namespace、Pod ServiceAccount 或 workload policy 授權，回應又包含明文 secret patch。

同一個 vault-agent ServiceAccount 目前透過 ClusterRoleBinding 取得全叢集 Secret 與 ConfigMap 的 `get/list/create/update/patch/delete` 權限，並建立未使用的長效 ServiceAccount token。這些權限超過現有程式所需範圍，放大端點或供應鏈遭入侵後的影響。

## 現有行為

1. `/mutate` 接受任何可連線來源送出的 POST。
2. Handler 未驗證 AdmissionReview 的 kind、resource、operation、namespace 與 request 是否存在。
3. `MutateUseCase` 直接信任 annotation 內的 backend、path 與 keys。
4. Sync Worker 直接信任受選 label Secret 的 namespace 與 annotation。
5. Base RBAC 使用 ClusterRoleBinding，授予全叢集 Secret/ConfigMap 廣泛權限。
6. Webhook、healthz 與 metrics 共用同一 listener。

## 新行為

1. production 啟動時若未配置可用的 Webhook 呼叫端認證，機密讀取路徑不得啟用。
2. `/mutate` 在解析或讀取任何外部機密前，必須先完成來源認證與 AdmissionReview 契約驗證；啟用 workload authorization 時，還必須先完成 policy 授權。
3. 新增 `authorization.enabled` 開關，預設為 `true`。啟用時至少使用 namespace、Pod ServiceAccount、backend 與 delimiter-aware path 規則，並採預設拒絕。
4. 啟用 authorization 時，Sync Worker 必須套用獨立但同源的 namespace/backend/path policy；未授權時不得呼叫 Vault/AWS。
5. Kubernetes RBAC 只允許 vault-agent 在明確受管 namespace 對 Secret 執行實際需要的 `list` 與 `update`。
6. 移除 ConfigMap 權限、未使用 verbs、全叢集 ClusterRoleBinding、非必要 `system:auth-delegator` 綁定及長效 ServiceAccount token Secret。
7. `/mutate` 與維運端點分離，避免 health probe 或 metrics 因 Webhook client authentication 契約而失效。

## 目標

1. 阻止未認證網路來源觸發外部機密讀取。
2. 阻止已認證來源為未授權 workload 讀取外部機密。
3. 將 Kubernetes 權限縮小至明確 namespace 與必要 verbs。
4. 建立可自動驗證的 deny-before-fetch 保證。
5. 提供從現有 deployment 遷移至安全模式的操作文件。

## 非目標

1. 不在本 spec 導入 Secret Store CSI Driver、sidecar 或 init container。
2. 不設計 Vault/AWS 端的完整 IAM/policy 自動化；只定義應遵循最小權限與 workload 分割。
3. 不處理 Admission fetch 的快取、singleflight、bulkhead 或壓力最佳化。
4. 不修改 `failurePolicy: Ignore`；其風險另案評估。
5. 不將 AdmissionRequest.UserInfo 當成呼叫端網路身分驗證的替代品。
6. 不提供允許匿名機密讀取的相容模式。

## 使用情境

### UC-1：API Server 呼叫 Webhook

API Server 使用已配置的 client certificate 或 bearer token 呼叫 `/mutate`。vault-agent 驗證來源、AdmissionReview 契約及 workload policy 後，才允許 SecretFetcher 取值。

### UC-2：叢集內 Pod 偽造請求

一般 Pod 即使能連到 Service，若無有效憑證仍須在解析 secret reference 與呼叫 SecretFetcher 前收到拒絕回應。

### UC-3：已認證但 workload 越權

請求來源已認證，但 Pod namespace、ServiceAccount、backend 或 path 不符合 policy。系統回傳 deny，且外部機密後端呼叫次數維持零。

### UC-4：背景 Secret 同步

Worker 只在已授權 namespace 內列出 Secret，且每筆 annotation 必須通過 sync policy 才能呼叫 Vault/AWS 與更新 Kubernetes Secret。

### UC-5：managed control plane 無法提供 Webhook credential

若平台無法設定 Admission Webhook client credential，也沒有可驗證來源身分的可信代理，production 必須停用 Webhook 機密注入，不得退回匿名模式。

## 功能需求

### FR-1：來源認證

1. 支援 `mtls` 與 `bearer` 認證模式。
2. `mtls` 必須驗證 client certificate chain，並限制允許的 subject 或 URI SAN。
3. `bearer` 必須從 Secret 掛載或等價 secret source 載入，以 constant-time comparison 驗證，不得寫入日誌。
4. `disabled` 代表 `/mutate` 不提供機密讀取，回傳 503；不得代表匿名允許。
5. production 不得在認證設定缺漏或無效時啟動 Webhook listener。

### FR-2：AdmissionReview 契約驗證

在授權與取值前驗證：

1. `apiVersion=admission.k8s.io/v1` 與 `kind=AdmissionReview`。
2. `request`、UID、Object.Raw 必須存在。
3. resource/kind 必須是 core/v1 Pod。
4. operation 僅允許 `CREATE`；若保留 `UPDATE`，必須另行定義不會重複注入的行為後才可開啟。
5. request namespace 必須與 Pod metadata namespace 一致；空 namespace 必須使用 request namespace 正規化。

### FR-3：Workload 授權

1. Policy 採 default deny。
2. Webhook policy subject 至少包含 namespace 與 Pod ServiceAccount。
3. Resource 至少包含 backend 與允許 path prefix；可選限制 keys。
4. path prefix 比對必須辨識分隔邊界，`team-a/` 不得匹配 `team-ab/`。
5. Vault 與 AWS path matcher 必須分開實作與測試，不得用未定義語意的單一字串 prefix 涵蓋所有 backend。
6. 無 policy、policy 格式錯誤或沒有匹配規則時一律拒絕。
7. 拒絕結果不得包含 secret value、credential 或完整敏感 path。
8. `authorization.enabled` 預設為 `true`；只有明確設定為 `false` 時才略過 Webhook 與 Sync workload policy。
9. `authorization.enabled=false` 不得停用 caller authentication、AdmissionReview validation 或 RBAC 最小權限。
10. 關閉 authorization 時，policy 來源不再是啟動必要條件，但必須輸出高可見度警告、記錄 enforcement disabled 指標，且文件必須說明 Vault/AWS policy 會成為唯一的外部 path 授權邊界。
11. Policy 載入來源同時支援 `authorization.policy_file` 與 `authorization.policy_dir`，兩者必須互斥。
12. `policy_file` 載入單一 YAML；`policy_dir` 只載入目錄第一層的 `.yaml` 與 `.yml`，依檔名排序後合併。
13. `authorization.enabled=true` 時必須設定一種 policy 來源；兩種來源皆未設定或同時設定時拒絕啟動。
14. `policy_dir` 中所有 rule ID 必須全域唯一，版本必須一致；任一檔案無效、衝突或無法讀取時整體 fail closed。
15. Policy 規則同時支援精確 namespace 清單與 `namespace_globs`。Glob 採完整名稱匹配，只支援受控的 `*`、`?` 語意，不接受任意正規表示式。
16. Resource 規則同時支援精確 `path_prefixes` 與 `path_prefix_templates`；模板第一版只允許 `{{namespace}}` 變數。
17. 展開後的 path template 必須再通過對應 backend matcher 的正規化與邊界驗證，不能直接作未驗證字串 prefix 比對。

### FR-4：背景同步授權

1. `authorization.enabled=true` 時，Sync policy 至少包含 Secret namespace、backend 與 path 規則。
2. 未授權 Secret 必須略過 fetch 與 update，記錄不含敏感值的稽核事件並增加拒絕指標。
3. Webhook policy 與 sync policy 使用同一份設定格式，但規則用途必須明確區分。

### FR-5：RBAC 最小權限

1. 移除 ConfigMap 權限。
2. 移除 Secret 的 `get/create/patch/delete`，除非實作證據與測試證明必要。
3. 保留 ClusterRole 作為可重用規則時，只能透過受管 namespace 內的 RoleBinding 綁定，不得使用全叢集 ClusterRoleBinding。
4. 每個受管 namespace 必須顯式建立 RoleBinding，subject 指向 `vault-agent/vault-agent` ServiceAccount。
5. 移除未使用的長效 `kubernetes.io/service-account-token` Secret。
6. 驗證 `system:auth-delegator` 是否由本程式實際需要；若沒有 TokenReview 呼叫路徑則移除。

### FR-6：端點與部署隔離

1. Webhook listener 與 health/metrics listener 使用不同 port。
2. Webhook listener 僅提供 `/mutate`；維運 listener 不得提供 `/mutate`。
3. Webhook Kubernetes Service 僅暴露 Webhook port。
4. readiness 必須反映認證是否有效；authorization 啟用時還必須反映 policy 是否有效載入。
5. NetworkPolicy 作為 defense in depth 由各環境 overlay 提供；不得將其視為唯一認證機制。

### FR-7：可觀測性與稽核

1. 至少提供 `authentication_denied`、`authorization_denied`、`policy_load_error` 計數，以及 `authorization_enforcement_enabled` 狀態指標。
2. 稽核日誌包含 request ID、namespace、ServiceAccount、backend 與 rule ID。
3. 預設不得記錄 bearer token、client certificate 原文、secret value 或完整 secret path。

## 非功能需求

1. 所有拒絕判斷必須在 `SecretFetcher.FetchSecret` 之前完成。
2. 認證錯誤永遠必須 fail closed；authorization 啟用時，授權元件錯誤也必須 fail closed。
3. Policy 在啟動時完成 schema 與語意驗證；本階段不要求 hot reload。
4. 新增程式必須通過 `go test -race ./...`、`go vet ./...`、`gofmt` 與既有 lint。
5. production secret 不得出現在 ConfigMap、Kustomize literal、command line 或版本控制檔案中。

## 驗收情境

### AC-1：匿名來源不得觸發取值

場景：一般 Pod 直接 POST 合法 AdmissionReview

測試：待建立 `TestWebhookHandler_UnauthenticatedRequestDoesNotFetch`

假設：Webhook 認證模式為 `mtls` 或 `bearer`

當：請求未提供有效 credential

那麼：回應為 401 或 TLS handshake 失敗，且 mock SecretFetcher 呼叫次數為零

### AC-2：畸形 AdmissionReview 不得進入 use case

場景：已認證來源送出缺少 request 或錯誤 resource 的 JSON

測試：待建立 `TestWebhookHandler_InvalidAdmissionReviewDoesNotFetch`

假設：來源認證成功

當：AdmissionReview 契約驗證失敗

那麼：回傳受控拒絕，程序不 panic，SecretFetcher 呼叫次數為零

### AC-3：授權 workload 可以讀取允許路徑

場景：允許的 namespace 與 ServiceAccount 建立 Pod

測試：待建立 `TestMutateUseCase_AuthorizedWorkloadFetchesSecret`

假設：policy 允許 `team-a/app` 使用 Vault `teams/team-a/` prefix

當：Pod 引用 `teams/team-a/database`

那麼：只呼叫一次 Vault fetcher 並回傳預期 patch

### AC-4：越權 workload 不得讀取

場景：已認證請求引用未授權路徑

測試：待建立 `TestMutateUseCase_UnauthorizedPathDoesNotFetch`

假設：policy 只允許 `teams/team-a/`

當：Pod 引用 `teams/team-b/database`

那麼：AdmissionResponse 為 denied，SecretFetcher 呼叫次數為零

### AC-5：prefix 不得跨邊界匹配

場景：攻擊者使用相似字首繞過 policy

測試：待建立 `TestVaultPathMatcher_DelimiterBoundary`

假設：policy 允許 `teams/team-a/`

當：請求 path 為 `teams/team-ab/database` 或包含非法 traversal segment

那麼：授權失敗

### AC-6：Sync Worker 套用相同安全邊界

場景：受管 namespace 中的 Secret annotation 引用未授權 path

測試：待建立 `TestSyncWorker_UnauthorizedReferenceDoesNotFetchOrUpdate`

假設：Secret 可被 Worker 列出但不符合 sync policy

當：執行一輪同步

那麼：fetch 與 Kubernetes update 呼叫次數皆為零，拒絕指標增加

### AC-7：production 缺少認證設定時拒絕啟動

場景：production deployment 未掛載 client CA 或 bearer token

測試：待建立 `TestConfig_ProductionWebhookAuthRequired`

假設：`APP_ENV=prod`

當：載入設定並建立 Webhook server

那麼：回傳明確設定錯誤，Webhook listener 不啟動

### AC-8：RBAC 權限符合最小集合

場景：部署至測試叢集後檢查 ServiceAccount 權限

測試：待建立腳本或 CI manifest 測試

假設：`team-a` 是受管 namespace，`team-b` 未受管

當：以 `kubectl auth can-i --as=system:serviceaccount:vault-agent:vault-agent` 檢查權限

那麼：只允許在 `team-a` list/update secrets；不得在 `team-b` 操作 Secret，也不得操作 ConfigMap 或 delete Secret

### AC-9：不再建立長效 ServiceAccount token

場景：渲染 base 與 production overlay

測試：待建立 Kustomize manifest assertion

假設：使用更新後 manifests

當：檢查渲染結果

那麼：不存在 `type: kubernetes.io/service-account-token` 的 Secret

### AC-10：明確關閉 workload authorization

場景：維運者決定只依賴 Vault/AWS policy 管理 path 權限

測試：待建立 `TestMutateUseCase_AuthorizationDisabledBypassesPolicyOnly`

假設：`authorization.enabled=false`，caller credential 有效，AdmissionReview 合法，但未提供 policy file

當：Webhook 處理啟用注入的 Pod

那麼：系統略過 workload policy 並允許 fetch，同時保留 caller authentication 與 AdmissionReview validation，啟動日誌明確警告且狀態指標為 0

### AC-11：單檔與目錄 policy 產生相同決策

場景：相同規則分別由單一檔案及多檔目錄提供

測試：待建立 `TestPolicyLoader_FileAndDirectoryProduceEquivalentPolicy`

假設：`policy_file` 包含完整規則，`policy_dir` 將相同規則拆成多個檔案

當：分別載入並執行相同授權案例

那麼：兩種來源產生相同 allow/deny 決策，且目錄檔案依名稱排序載入

### AC-12：namespace template 維持租戶隔離

場景：一條模板規則服務多個命名一致的 namespace

測試：待建立 `TestAuthorizer_NamespaceGlobAndPathTemplateIsolation`

假設：規則允許 `namespace_globs: [team-*]`，path template 為 `teams/{{namespace}}/`

當：`team-a` workload 分別要求 `teams/team-a/database` 與 `teams/team-b/database`

那麼：第一個請求允許，第二個請求拒絕

### AC-13：Policy 來源或合併衝突時拒絕啟動

場景：設定同時提供單檔與目錄，或目錄內存在重複 rule ID

測試：待建立 `TestPolicyLoader_ConflictingSourcesAndDuplicateRuleIDsFailClosed`

假設：authorization 已啟用

當：載入衝突的來源設定或多檔規則

那麼：回傳明確設定錯誤，Webhook listener 不啟動，且不會選擇其中一個來源繼續執行

## 驗證需求

1. 單元測試必須以 spy/mock 證明所有拒絕路徑的 fetch 次數為零。
2. Handler 測試涵蓋 mTLS 或 bearer 成功、缺漏、錯誤與 credential 不進日誌。
3. Policy 測試涵蓋預設拒絕、namespace、namespace glob、ServiceAccount、backend、path 邊界、path template、AWS ARN/name、keys、單檔/目錄載入與 authorization enabled/disabled。
4. 整合測試以 `httptest` 建立 TLS server/client，驗證 client certificate chain 與 subject 限制。
5. Manifest 測試驗證 Service、ports、RoleBinding、verbs、無 ClusterRoleBinding 與無長效 token。
6. 若有可用測試叢集，執行匿名 curl、合法 API Server 呼叫與 `kubectl auth can-i` 驗證。

## 上線閘門與待確認項目

1. 目標 Kubernetes 平台是否允許設定 Admission controller kubeconfig 的 client credential。
2. 若不允許，是否已有能提供可驗證來源身分的 control-plane proxy；若沒有，本功能只能保持 disabled。
3. 首批受管 namespace、Pod ServiceAccount、Vault path prefix 與 AWS SecretId/ARN 規則清單。
4. 背景同步是否仍需跨 namespace；若需要，需為每個 namespace 建立 RoleBinding 與 sync policy。
5. `UPDATE` operation 是否確實必要；在重複注入語意完成前，本 spec 預設只允許 `CREATE`。
