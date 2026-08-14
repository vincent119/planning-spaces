# 需求文件：Webhook Failure Policy 與高可用

## 來源

- Draft: 無
- Type: Feature
- Owner: Vincent
- Status: InProgress

## 文件定位

本 spec 接續 `.specs/2026-08-04-10-52_Feature-secure-secret-access` 中刻意延後的 `failurePolicy: Ignore` 風險，以及 `.specs/2026-08-04-14-39_BugFix-kubernetes-graceful-termination` 已完成的 readiness 與終止時間契約。本 spec 只處理 Mutating Admission Webhook 的 selector、failure policy、Deployment 高可用、RollingUpdate、PDB、拓撲分散、rollout、rollback 與驗證，不重寫 caller authentication、AdmissionReview validation、workload authorization、Secret fetch 或 Sync Worker。

參考來源：

- 需求來源: 使用者要求規劃「Webhook failure policy 與高可用設計」
- 既有文件: `docs/deploy.zh-Hant.md`、`docs/annotations.zh-Hant.md`、README
- 既有 manifests: `deployments/kustomize/base/admission.yaml`、`deployment.yaml`、`pdb.yaml`、prod overlay

## 背景

實際 Kustomize 使用的 `admission.yaml` 設定 `failurePolicy: Ignore`。API Server 無法呼叫 Webhook 時，符合 selector 的 Pod 仍可建立，可能在沒有注入機密的狀態下啟動。Deployment 與 prod overlay目前都只有一個 replica，雖有 PDB `minAvailable: 1`，但單一 Pod、節點故障或 Rolling 更新仍會造成沒有可用 endpoint。

現有 `admission.yaml` 的 `objectSelector` 只排除 `app=vault-agent`，沒有實作文件宣稱的 `inject-vault-agent=true` 過濾；因此受管 namespace 內多數 Pod CREATE 都會呼叫 Webhook。若直接切為 `Fail`，Webhook 故障會阻擋所有被 selector 命中的 Pod，而不只 opt-in workload。

此外，repository 還保留未被 Kustomize 引用的 `webhook.yaml`，其 selector 與實際 `admission.yaml` 不一致，文件又指向未使用檔案，形成部署契約漂移。

## 問題陳述

目前 fail-open 行為可能讓需要機密注入的 Pod 在 Webhook 故障時繼續建立；直接 fail-closed 又會因單 replica 與過寬 selector 放大成 namespace 級部署中斷。必須先建立最低可用性，再以同一個Webhook object的原子更新同時縮小攔截範圍並切換failure policy。

## 目標

1. Webhook 只攔截 namespace opt-in 且 Pod label `inject-vault-agent=true` 的 CREATE request。
2. 最終 desired state 使用 `failurePolicy: Fail`，使 opt-in Pod 在 Webhook 不可用或拒絕時 fail closed。
3. Deployment 與 prod overlay 至少使用 2 replicas。
4. RollingUpdate 使用 `maxUnavailable: 0`、`maxSurge: 1`，避免部署主動移除最後可用 replica。
5. 使用 hostname `ScheduleAnyway` topology spread分散node，並以zone preferred pod anti-affinity偏好跨可用區，不因缺少zone label阻止排程。
6. PDB 維持 `minAvailable: 1`，保護 voluntary disruption 時至少一個 replica。
7. 定義兩階段 rollout、故障演練、rollback、監控與目標 cluster 驗證。
8. 移除或收斂未被 Kustomize 使用的重複 Webhook manifest，讓文件只引用 canonical manifest。

## 非目標

1. 不宣稱兩個 replicas 可防止整個 cluster、control plane、網路或憑證系統故障。
2. 不導入 service mesh、外部 load balancer、Blue-Green 或 Canary controller。
3. 不修改 Webhook handler、authentication、authorization、SecretFetcher 或注入 patch。
4. 不在 repository 假造雲端控制平面的 admission metrics 名稱或告警 API。
5. 不自動建立 production credential、namespace label 或 workload label。
6. 不建立數值型 SLA/SLO；availability SLO 需由服務 owner 另行確認。

## 已定決策

- canonical manifest 為 `deployments/kustomize/base/admission.yaml`。
- 最終 `failurePolicy` 為 `Fail`；不得以停用 caller authentication 或放寬 authorization 取代 rollback。
- `objectSelector` 必須要求 Pod label `inject-vault-agent=true`。
- replicas 為 2，RollingUpdate 為 `maxUnavailable: 0`、`maxSurge: 1`。
- hostname topology spread採`ScheduleAnyway`；zone只使用preferred pod anti-affinity，避免缺少zone label的node被排除。
- PDB 使用 `minAvailable: 1`；它只保護 voluntary eviction，不視為 node failure 保證。
- rollout必須先套用`prod-fail-open`過渡overlay，以相同selector與HA設定在`Ignore`下驗證2個ready endpoints，再套用正式prod overlay切為`Fail`。
- rollback 第一動作是把 failure policy 暫時切回 `Ignore`，解除 admission 阻塞；後續才 rollback Deployment。

## 待確認項目

- 目標 cluster 是否至少有兩個可排程 node，以及是否提供 zone topology label。
- 平台是否能取得 API Server admission webhook failure、rejection 與 latency telemetry；不能取得時需使用 audit event、synthetic admission 與 Pod availability 補足。
- production availability SLO 與告警通知窗口由服務 owner 另案確認。

## 現有行為

- base 與 prod overlay皆為 `replicas: 1`。
- Deployment 沒有明確 RollingUpdate surge/unavailable contract。
- PDB 為 `minAvailable: 1`，單 replica 下會保護 voluntary eviction，但不提供備援 endpoint。
- 沒有 topology spread 或 pod anti-affinity。
- canonical `admission.yaml` 使用 `failurePolicy: Ignore`、`timeoutSeconds: 5`。
- canonical `objectSelector` 沒有要求 injection label。
- 未引用的 `webhook.yaml` 有不同 selector，文件錯誤引用該檔案。
- readiness 與 graceful termination 已存在，但尚未做 fail-closed outage 演練。

## 新行為

- 只有同時符合 namespace label 與 Pod injection label 的 Pod CREATE 會進入 Webhook。
- opt-in Pod 在 Webhook service 無可用 endpoint、TLS/caller authentication 失敗、timeout 或明確拒絕時不得建立。
- 非 opt-in Pod 不觸發 Webhook，即使 Webhook 不可用也不受其 failure policy 影響。
- 正常狀態至少有兩個 desired replicas，Service 只路由至 ready endpoints。
- Rolling 更新先建立 surge Pod並等它 ready，再終止舊 Pod。
- voluntary disruption 期間 PDB 至少保留一個 available replica。
- replicas優先跨hostname分散；zone label存在時再偏好跨zone，拓撲不足時允許排程並由preflight/監控揭露降級狀態。
- rollout 與 rollback 使用文件化兩階段程序，不依賴 `kubectl apply` 資源順序碰運氣。

## 影響範圍

- 使用者: 受管 namespace 的 workload deployer、平台管理者、on-call
- 功能: Pod admission、Webhook availability、Rolling update 與 disruption behavior
- API / CLI: Kubernetes admission 行為改為 opt-in workload fail closed
- Data / Storage: 無
- 文件 / 安裝 / 發布: Kustomize manifests、部署文件、annotations 文件、README 與 release checklist

## 使用情境

- 作為平台管理者，我想在 Webhook 故障時阻止 opt-in Pod 缺少機密仍繼續建立，同時不影響未使用機密注入的 Pod。
- 作為 on-call，我想能快速把 failure policy 暫時回復為 `Ignore`，解除 admission 阻塞後再修復 Webhook。

## 驗收情境

### 情境：只攔截 opt-in Pod

- 場景：同一受管 namespace 分別建立有與沒有 injection label 的 Pod
- 測試：Kustomize manifest assertion；目標 cluster synthetic admission test
- 假設：namespace 有 `vault-agent.io/admission-webhooks=enabled`
- 當：建立兩種 Pod
- 那麼：只有 label `inject-vault-agent=true` 的 Pod會呼叫 Webhook

### 情境：Webhook unavailable 時 opt-in Pod fail closed

- 場景：在非 production 驗證 namespace 暫時使 Webhook 無可用 endpoint
- 測試：目標 cluster failure drill
- 假設：`failurePolicy=Fail` 且測試具備可復原程序
- 當：建立 opt-in Pod
- 那麼：API Server 拒絕或逾時該 CREATE，Pod 不得建立

### 情境：Webhook unavailable 時非 opt-in Pod不受影響

- 場景：同一故障期間建立沒有 injection label 的 Pod
- 測試：目標 cluster failure drill
- 假設：canonical objectSelector 已生效
- 當：建立非 opt-in Pod
- 那麼：API Server 不呼叫本 Webhook，Pod 可依其他 admission policy 正常處理

### 情境：單一 replica 故障仍可服務

- 場景：兩個 ready replicas 中一個被刪除或失去 readiness
- 測試：目標 cluster availability drill
- 假設：另一個 replica 與 Service endpoint 正常，外部 backend 可用
- 當：建立合法 opt-in Pod
- 那麼：Webhook request 由剩餘 ready replica 處理，Pod admission 正常完成

### 情境：Rolling update 不主動降為零 endpoint

- 場景：更新 Deployment image或 template
- 測試：manifest assertions 與目標 cluster rollout observation
- 假設：新 Pod 可在 readiness deadline 內成功啟動
- 當：Deployment 執行 RollingUpdate
- 那麼：`maxUnavailable=0`，新 replica ready 前舊 replica不被主動移除

### 情境：安全 rollback

- 場景：切換 `Fail` 後發生持續 admission outage
- 測試：runbook演練
- 假設：on-call 具有更新 MutatingWebhookConfiguration 與 Deployment 的權限
- 當：執行 rollback
- 那麼：先暫時切回 `Ignore` 解除阻塞，再回復前一 Deployment revision；不得停用 caller authentication、authorization 或擴大 RBAC

## 驗收條件

1. canonical render 只有一個 MutatingWebhookConfiguration，`objectSelector` 要求 injection label。
2. 最終 base/prod render 的 `failurePolicy` 為 `Fail`，`timeoutSeconds` 維持 5。
3. base/prod render 的 replicas 為 2，RollingUpdate 為 0 unavailable、1 surge。
4. Pod template具有hostname `ScheduleAnyway` topology spread與zone preferred pod anti-affinity，selectors對齊`app=vault-agent`。
5. PDB `minAvailable` 維持 1 且 selector 對齊 Deployment。
6. 重複未引用的 Webhook manifest 已移除或明確收斂，不再與 canonical manifest 漂移。
7. `prod-fail-open`只覆寫failure policy，其他selector、HA、image與設定和prod render一致。
8. 本地 Kustomize、server-side dry-run 與 manifest field assertions 通過。
9. 非 production failure drill 覆蓋 opt-in fail closed、非 opt-in unaffected、單 replica failure 與 rollback。
10. 文件明確說明兩階段 rollout、rollback、cluster preconditions、PDB限制與監控需求。

## 驗證需求

- Static: base/prod Kustomize render 與結構化 manifest assertions
- Server-side: `kubectl apply --dry-run=server -k deployments/kustomize/overlays/prod`
- Runtime: 兩個 ready endpoints、跨 node分布、RollingUpdate、單 replica failure 與 admission outage drill
- Regression: `go test -race -count=1 ./...`、`go vet ./...`，確認 manifest-only 變更未破壞程式
- 文件檢查: failure policy、selector、兩階段 rollout、rollback、監控與限制一致

## 風險與假設

| 類型 | 內容 | 處理方式 |
|------|------|----------|
| 風險 | `Fail` 在零 ready endpoint 時阻擋 opt-in Pod | 2 replicas、safe RollingUpdate、PDB、preflight、監控與 rollback |
| 風險 | selector 過寬導致 namespace blast radius | canonical Webhook update原子套用Pod injection label與`Fail` |
| 風險 | 兩 replicas落在同一 node或zone | hostname spread＋zone preference＋runtime distribution check；承認best-effort可能降級 |
| 風險 | PDB 被誤認為防止 node failure | 文件明確限制並執行單 replica failure drill |
| 風險 | 直接 apply 最終 manifest 的資源順序不可控 | 首次切換使用兩階段 runbook，不依賴 YAML 順序 |
| 假設 | 至少一個 Webhook replica、TLS、auth、policy 與 backend 可用 | rollout gate與 synthetic admission 驗證 |

## 摘要

- 關鍵決策: 先擴到2個ready replicas，再原子套用selector＋Fail；0/1 RollingUpdate、PDB1與best-effort拓撲分散
- 待確認項目: cluster node/zone能力、control plane telemetry、production SLO
- 風險: fail-closed 的 admission outage blast radius與拓撲降級
- 下一步: 確認本 spec 後依 tasks 先完成 manifest tests，再修改 canonical manifests與操作文件
