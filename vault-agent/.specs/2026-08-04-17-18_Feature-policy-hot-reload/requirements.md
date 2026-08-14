# 需求文件：Policy Hot Reload 與 Generation 一致性

## 來源

- Draft: 無
- Type: Feature
- Owner: Vincent
- Status: Complete

## 文件定位

本 spec 接續 `.specs/2026-08-04-15-29_Feature-policy-validation` 與既有 policy directory deployment，讓已啟動的 vault-agent 在不重啟 Pod 的情況下套用有效 workload authorization policy。範圍只涵蓋本機 policy file/directory 的安全重載、程序內原子切換、觀測性與部署文件，不改動 policy schema、授權比對規則或 Vault/AWS server-side policy。

參考來源：

- 需求來源：使用者確認下一階段規劃 Policy hot reload 與 generation 一致性
- 既有文件：`docs/config.zh-Hant.md`、`docs/deploy.zh-Hant.md`
- 既有程式碼：`cmd/vault-agent/main.go`、`internal/configs/config.go`、`internal/syncer/domain/policy.go`、`internal/infra/metrics/metrics.go`
- Kubernetes ConfigMap 文件：<https://kubernetes.io/docs/concepts/configuration/configmap/>
- fsnotify 官方 FAQ：<https://github.com/fsnotify/fsnotify>
- Kubernetes projected volume AtomicWriter 問題紀錄：<https://github.com/kubernetes/kubernetes/issues/112677>

## 背景

目前 authorization 啟用時只在 application startup 呼叫一次 `domain.LoadPolicy`，之後 webhook 與 sync worker 共用固定的 `PolicyAuthorizer`。Kubernetes deployment 已將 ConfigMap 以目錄掛載至 `/app/policy`，且沒有使用 `subPath`，因此 kubelet 可在 ConfigMap 更新後逐步更新 volume；但應用程式不會重新載入內容。

Kubernetes projected volume 以 symlink 世代切換內容，直接監看單一檔案會面臨 watch 失效或只收到刪除事件。此功能採固定週期重新建立候選 snapshot，不增加 Kubernetes API 權限，也不依賴檔案監看事件的完整性。

## 問題陳述

Policy 變更目前必須重啟 Pod 才會生效，緊急撤銷授權的操作時間與 rollout 綁定。若只在背景逐檔覆寫既有 authorizer，並行請求可能看到混合世代；若無效或更新中的檔案直接取代現行 policy，則可能造成錯誤授權或服務中斷。

## 目標

1. Authorization 啟用時支援可設定週期的 policy hot reload。
2. 每次重載先取得來源的一致 snapshot，完成 strict decode、合併與 semantic validation 後才發布。
3. Webhook 與 sync worker 共用同一個 reloadable authorizer；每次授權決策只讀取一個完整 generation。
4. 無效、空目錄、讀取失敗或來源正在變更時保留最後一版有效 policy，不發布部分內容。
5. 相同內容不重複發布或增加 generation。
6. 提供低基數 metrics 與安全日誌，能辨識成功、未變更及失敗狀態。
7. 保持 startup fail-closed：初始 policy 無效時仍不得啟動服務。
8. 不增加 Kubernetes API `get/watch` 權限或第三方檔案監看 dependency。

## 非目標

1. 不提供跨 Pod 的同步 barrier 或 cluster-wide atomic generation。
2. 不保證 ConfigMap 更新後的固定總生效時間；總延遲仍包含 kubelet propagation delay 與 reload interval。
3. 不中止已通過授權並開始執行的 in-flight Vault/AWS request。
4. 不修改 policy YAML schema、rule matching 或 deny-by-default 語意。
5. 不修改 Vault/AWS server-side policy，也不取代外部服務的授權控制。
6. 不新增 Kubernetes informer、ConfigMap API watch 或額外 RBAC。
7. 不在本階段提供管理 API、手動 reload endpoint 或 signal-based reload。

## 已定決策

- 新增 `authorization.reload_interval_seconds`。
- `reload_interval_seconds` 預設為 `30`；值大於 `0` 時啟用定期重載，值為 `0` 時維持 startup-only 行為，負數視為設定錯誤。
- Reload 採 polling，不使用 fsnotify 或 Kubernetes API watch。
- 每輪候選來源先建立內容 snapshot 與內部 fingerprint；來源在驗證期間有變化時放棄該輪，下一輪重試。
- Fingerprint 只用於內容相等與一致性判斷，不寫入 log、metric label 或 error。
- 發布以單一 atomic pointer swap 完成；policy 發布後視為 immutable。
- 初始載入失敗仍終止啟動；背景重載失敗保留 last-known-good policy，服務 readiness 不因此轉為失敗。
- 刪除全部 YAML 或空目錄視為重載失敗。撤銷全部權限必須部署一份 schema 有效且規則集合為空的 policy。
- Runtime log 與 error message 維持英文；文件使用繁體中文。

## 待確認項目

- 無。

## 現有行為

- `authorization.enabled=true` 時，startup 載入一次 `policy_file` 或 `policy_dir`。
- `PolicyAuthorizer` 保存固定 `*Policy`，不支援並行安全替換。
- Webhook 與 sync worker 共用同一個固定 authorizer instance。
- 無 reload generation、結果計數或最後成功時間 metrics。
- `/app/policy` 以 ConfigMap directory mount 提供，未使用 `subPath`。

## 新行為

- Startup 先同步載入並驗證 generation 1，成功後才建立 server 與背景 reloader。
- Reloader 每隔設定週期讀取 policy source，建立一致候選 snapshot 並驗證。
- 候選內容與現行 generation 相同時記錄 unchanged，不交換 pointer。
- 有效且已變更的候選透過 atomic swap 同時成為 webhook 與 sync worker 的新 generation。
- 候選無效、來源不穩定、讀取失敗或沒有 YAML 時只記錄安全錯誤與 failure metric，現行 generation 不變。
- Shutdown context 取消時 reloader 停止，graceful shutdown 等待其離開。

## 影響範圍

- 使用者：平台工程師、policy 維護者、on-call 人員
- 功能：policy snapshot loader、reloadable authorizer、background lifecycle
- API / CLI：既有 `policy validate` contract 不變
- Data / Storage：policy schema 不變；新增 runtime generation state
- Config：新增 `authorization.reload_interval_seconds`
- Observability：新增 reload result、generation 與最後成功時間 metrics
- 文件 / 安裝 / 發布：sample config、繁中設定與部署文件；既有 ConfigMap directory mount 不變

## 使用情境

- 作為平台工程師，我想更新 ConfigMap 後讓各 Pod 自動套用新 policy，以便不經 rollout 撤銷或增加 workload 權限。
- 作為 on-call 人員，我想知道各 Pod 是否成功載入新 generation，以便辨識內容錯誤或副本間暫時不一致。

## 驗收情境

### 情境：有效 policy 原子生效

- 場景：背景 reloader 讀到一份與現行內容不同且驗證有效的 policy
- 測試：`TestPolicyReloader_PublishesValidPolicyAtomically`
- 假設：現行 generation 允許舊 path，新 policy 撤銷舊 path 並允許新 path
- 當：候選 snapshot 完成驗證並發布
- 那麼：後續授權只看到完整新 generation，不會看到混合規則集合

### 情境：Webhook 與 sync worker 共用 generation

- 場景：同一 reloadable authorizer 注入兩個 use case
- 測試：`TestBuildUseCases_ShareReloadableAuthorizer`
- 假設：兩者在 reload 前後分別執行授權
- 當：有效候選完成 atomic swap
- 那麼：兩者後續決策都使用相同 generation

### 情境：無效更新保留最後有效 policy

- 場景：候選 policy 有 YAML、schema 或 semantic error
- 測試：`TestPolicyReloader_InvalidCandidateKeepsLastKnownGood`
- 假設：startup 已成功發布 generation 1
- 當：背景 reload 驗證失敗
- 那麼：generation 維持 1、既有授權結果不變、readiness 保持成功，且 failure metric 增加

### 情境：更新期間不發布混合世代

- 場景：policy directory 在 snapshot 擷取或驗證期間切換內容
- 測試：`TestPolicyReloader_SourceChangesDuringLoadDoesNotPublish`
- 假設：測試 loader 可控制前後 source fingerprint
- 當：前後 fingerprint 不一致
- 那麼：該輪候選被放棄，現行 generation 不變且下一輪可重試

### 情境：相同內容不建立新 generation

- 場景：檔案 metadata 或 ConfigMap symlink 改變，但 policy 內容相同
- 測試：`TestPolicyReloader_UnchangedContentDoesNotAdvanceGeneration`
- 假設：候選 fingerprint 與現行內容 fingerprint 相同
- 當：poll 執行
- 那麼：不交換 authorizer、generation 不變，結果記為 unchanged

### 情境：明確撤銷全部授權

- 場景：有效 policy 保留 `version: v1`，但 webhook 與 sync 規則集合皆為空
- 測試：`TestPolicyReloader_EmptyRuleSetsPublishDenyAll`
- 假設：候選通過 schema 與 semantic validation
- 當：候選發布
- 那麼：後續 workload secret access 全部 deny；此行為不得與空目錄混淆

### 情境：停用 hot reload

- 場景：`reload_interval_seconds=0`
- 測試：`TestBuildPolicyReloader_DisabledForZeroInterval`
- 假設：startup policy 有效
- 當：application 啟動並修改來源
- 那麼：不啟動 reloader，authorizer 維持 startup generation

### 情境：安全觀測重載失敗

- 場景：policy 內含敏感 path marker 且 validation 失敗
- 測試：`TestPolicyReloader_ErrorDoesNotExposePolicyContent`
- 假設：logger 與 metrics 可由測試擷取
- 當：reload 失敗
- 那麼：log 與 metric labels 不含 policy path、resource path、keys、rule ID、fingerprint 或 YAML 內容

## 驗收條件

1. `reload_interval_seconds` 的預設、零值與負數驗證由 config tests 固定。
2. Startup 初始載入仍為同步且失敗即停止；background failure 保留 last-known-good。
3. Snapshot 前後一致性檢查阻止來源變更期間的候選發布。
4. 每次授權決策只載入一次 immutable policy pointer，並通過 race test。
5. Webhook 與 sync worker 共用同一 reloadable authorizer instance。
6. 相同內容不增加 generation；有效空規則 policy 可發布並 deny all；空目錄不可發布。
7. Reloader 隨 application context 啟停，graceful shutdown 不遺留 goroutine。
8. Metrics 使用固定低基數 label，log/error 不洩漏 policy 內容與 fingerprint。
9. 既有 CLI validation、授權、sync retry、readiness 與 Kubernetes deployment 測試維持通過。
10. 不增加 Kubernetes RBAC，且 deployment 保持 directory mount、不得改用 `subPath`。

## 驗證需求

- Unit：config、snapshot loader、reloadable authorizer、reloader state transition 與 metrics
- Concurrency：`go test -race -count=1 ./...`
- Integration：main wiring、webhook/sync 共用 generation、context cancellation
- CLI 回歸：`make policy-validate`
- Static：`go vet ./...`、`make lint`
- Deployment：base/prod Kustomize render，確認 `/app/policy` directory mount 與 config key
- 文件檢查：設定預設值、停用方式、收斂延遲、last-known-good 風險與 HA generation skew

## 風險與假設

| 類型 | 內容 | 處理方式 |
|------|------|----------|
| 風險 | 無效的撤銷更新不會取代 last-known-good，舊權限會暫時保留 | 部署前執行 `policy validate`；failure metric 與告警必須可見 |
| 風險 | 多副本各自 reload，短時間內可能使用不同 generation | 文件明示 per-process atomicity；以 Pod 維度觀察 generation 與最後成功時間 |
| 風險 | ConfigMap propagation delay 加上 polling interval 延長撤銷收斂時間 | 文件說明延遲模型；預設 30 秒並允許降低 interval |
| 風險 | 逐檔讀取遇到 symlink generation 切換而混合內容 | 來源 snapshot 前後 fingerprint 一致才允許發布 |
| 風險 | background goroutine 未隨 shutdown 結束 | context cancellation 與 wait tests |
| 假設 | Policy 在發布後保持 immutable | 所有 reload 都建立全新 `Policy`，禁止原地修改 |
| 假設 | 既有 ConfigMap 使用 directory mount 且未使用 `subPath` | Kustomize render 測試保護此契約 |

## 摘要

- 關鍵決策：30 秒預設定期重讀、零值停用、穩定 snapshot、atomic pointer、last-known-good、低基數觀測
- 一致性邊界：保證單一程序內每次授權決策只看到一個 generation；不提供跨 Pod 原子切換
- 安全邊界：初始載入 fail-closed；背景失敗不發布候選且不得洩漏 policy 內容
- 待確認項目：無
- 下一步：確認本 spec 後，依 `tasks.md` 逐項實作與驗證
