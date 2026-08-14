# 任務文件：Webhook Failure Policy 與高可用

Status: InProgress

## Execution Context

- 意圖: 收斂Webhook攔截範圍，建立最低HA與safe rollout，再把opt-in Pod切為fail closed。
- 非目標: 不修改Go handler、authentication、authorization、Secret fetch、Sync Worker或外部backend HA。
- 已定決策: canonical admission.yaml；selector要求injection label；final Fail；replicas2；0 unavailable/1 surge；PDB1；hostname ScheduleAnyway spread＋zone preference；兩階段rollout。
- 邊界: admission、Deployment、PDB、prod replica override、unused manifest cleanup、manifest tests、繁中與既有英文對應文件、本spec。
- 關鍵檔案: `deployments/kustomize/base/admission.yaml`、`deployment.yaml`、`pdb.yaml`、prod overlay、deployment/annotation docs。
- 完成條件: local render與assertions、server-side dry-run、non-production drills、文件與regression gates完成；target cluster結果不得由本地測試代替。

### Protected Behavior

- AdmissionReview v1、Pod CREATE、timeout5秒與`/mutate`不變。
- caller authentication、contract validation、workload authorization與deny-before-fetch不變。
- namespace opt-in、system namespace排除與application annotation opt-in不變。
- readiness/graceful termination的5/30/45契約不變。
- Secret sync、RBAC、policy loading與validation CLI不變。

### 邊界

#### Allowed Changes

- `deployments/kustomize/base/admission.yaml`
- `deployments/kustomize/base/deployment.yaml`
- `deployments/kustomize/base/pdb.yaml`
- `deployments/kustomize/base/webhook.yaml`
- `deployments/kustomize/base/kustomization.yaml`
- `deployments/kustomize/overlays/prod/kustomization.yaml`
- `deployments/kustomize/overlays/prod-fail-open/kustomization.yaml`
- `README.md`
- `docs/README.zh-Hant.md`
- `docs/README.en.md`
- `docs/deploy.zh-Hant.md`
- `docs/deploy.en.md`
- `docs/annotations.zh-Hant.md`
- `docs/annotations.en.md`
- `.specs/2026-08-04-15-50_Feature-webhook-failure-policy-ha/`

#### Forbidden

- 修改`cmd/`、`internal/`、Go application behavior或go.mod
- 修改Webhook credential、TLS certificate、authorization policy或RBAC
- 新增secret literal、token或真實namespace/workload資料
- 在production直接執行failure drill
- 宣稱PDB防止node failure或兩replicas保證零中斷
- 納入`_workspace/`

## 任務依賴

| 任務 | Depends | 狀態 | 備註 |
|------|---------|------|------|
| T1 建立manifest contract tests | 無 | Complete | red evidence已確認 |
| T2 收斂canonical Webhook selector | T1 | Complete | 先縮blast radius |
| T3 建立Deployment HA與safe rollout | T1 | Complete | replicas/strategy/topology/PDB |
| T4 切換final failure policy與移除duplicate | T2、T3 | Complete | desired state Fail |
| T5 更新rollout、rollback與監控文件 | T2、T3、T4 | Complete | 兩階段程序 |
| V1 本地與回歸驗證 | T1至T5 | Complete | render、tests、diff已通過；既有lint載入問題另列 |
| V2 目標cluster dry-run與failure drills | V1 | Planned | 需外部環境 |

## 實作任務

- [x] T1 建立manifest contract tests
  - Status: Complete
  - Boundary:
    - Allowed Changes: 本spec的Implementation Notes，只記錄render與field-count red evidence
    - Forbidden: production manifests與application code
  - Depends: 無
  - Context: 不新增測試dependency或script；以既有`kubectl kustomize`輸出固定canonical count、selector label、Fail、timeout5、replicas2、0/1 rollout、topology、PDB1與prod不覆寫回1。
  - Verify:
    - red test應證明現況為Ignore、replica1、缺少injection label與strategy/topology
    - `git diff --check`

- [x] T2 收斂canonical Webhook selector
  - Status: Complete
  - Boundary:
    - Allowed Changes: `deployments/kustomize/base/admission.yaml`
    - Forbidden: 先切Fail、修改namespaceSelector或handler
  - Depends: T1
  - Context: objectSelector新增`inject-vault-agent In ["true"]`，可保留app self-exclusion；selector與Fail在Phase2的同一個Webhook object更新中原子生效。
  - Verify:
    - base/prod render只有label opt-in Pod命中
    - synthetic labeled/unlabeled admission測試由V2執行

- [x] T3 建立Deployment HA與safe rollout
  - Status: Complete
  - Boundary:
    - Allowed Changes: base deployment/PDB與prod replica override
    - Forbidden: readiness、preStop、termination、resources或application config漂移
  - Depends: T1
  - Context: replicas2；RollingUpdate 0/1；hostname maxSkew1 ScheduleAnyway；zone使用preferred pod anti-affinity；PDB1；selectors對齊app label。
  - Verify:
    - base/prod structured assertions通過
    - render仍保留5/30/45中的manifest contract與既有probes

- [x] T4 切換final failure policy與移除duplicate
  - Status: Complete
  - Boundary:
    - Allowed Changes: canonical admission、unused webhook manifest、kustomization必要cleanup
    - Forbidden: 改timeout、rules、scope、ports或certificate contract
  - Depends: T2、T3
  - Context: final desired state為Fail；repository只保留一個canonical MutatingWebhookConfiguration source；新增繼承prod且只覆寫Ignore的migration/rollback overlay。
  - Verify:
    - base/prod render各只有一個Webhook且failurePolicy Fail、timeout5
    - prod-fail-open與prod除failurePolicy外的selector、replicas、strategy與image一致
    - `rg`確認文件不再引用deleted duplicate path

- [x] T5 更新rollout、rollback與監控文件
  - Status: Complete
  - Boundary:
    - Allowed Changes: tasks列出的README與deploy/annotations文件
    - Forbidden: 新增未確認SLO、假造provider metrics、記錄secret/path/AdmissionReview body
  - Depends: T2、T3、T4
  - Context: 文件必須說明Phase1套用prod-fail-open並驗證2 endpoints、Phase2套用prod、rollback重套fail-open overlay、cluster preconditions、PDB與ScheduleAnyway限制、監控與drill。
  - Verify:
    - 繁中與既有英文對應段落一致
    - canonical manifest path、selector與failure policy無舊描述

## 驗證任務

- [x] V1 本地與回歸驗證
  - Status: Complete
  - Boundary:
    - Allowed Changes: 本spec tasks狀態與Implementation Notes
    - Forbidden: 將本地render描述為target cluster runtime證據
  - Depends: T1至T5
  - Context: 執行manifest assertions、base/prod render、Go regression、格式、Boundary與diff checks。
  - Verify:
    - `kubectl kustomize deployments/kustomize/base`
    - `kubectl kustomize deployments/kustomize/overlays/prod`
    - `go test -race -count=1 ./...`
    - `go vet ./...`
    - `git diff --check`

- [ ] V2 目標cluster dry-run與failure drills
  - Status: Planned
  - Boundary:
    - Allowed Changes: 本spec驗證紀錄；外部non-production測試需使用者另行明確授權
    - Forbidden: production outage drill、未記錄restore值的scale/patch、修改RBAC或停用auth
  - Depends: V1
  - Context: V2不得因一般`run task`自動執行；需使用者另行明確授權。執行前確認至少兩個schedulable nodes、記錄原replicas/failurePolicy、限制測試namespace並準備立即rollback。
  - Verify:
    - server-side dry-run通過
    - 兩個ready endpoints與實際node/zone分布已記錄
    - labeled fail closed、unlabeled unaffected、delete-one continuity、RollingUpdate與rollback drill通過

- [ ] 品質檢查清單
  - [x] canonical selector只攔截opt-in Pod
  - [x] final failurePolicy為Fail且timeout維持5
  - [x] base/prod replicas為2
  - [x] RollingUpdate為0 unavailable、1 surge
  - [x] hostname topology spread為ScheduleAnyway，zone為preferred pod anti-affinity
  - [x] PDB minAvailable為1且selectors一致
  - [x] duplicate manifest與舊文件引用已清理
  - [x] 既有readiness與5/30/45 termination contract不變
  - [x] 本地render與Go regression通過
  - [ ] target cluster dry-run與non-production drills通過
  - [x] rollout、rollback、monitoring與限制已文件化
  - [x] runtime訊息與監控不含機密值
  - [x] `git diff --stat`已檢查
  - [x] `git diff --check`已通過

## 實作中斷恢復

恢復時優先讀取：

1. 本文件的`Execution Context`
2. 目前未完成task
3. `Protected Behavior`
4. `Implementation Notes`

不得預設掃描整個`.specs`目錄。若文件很大，先用標題與關鍵字定位：

```bash
rg -n "^#|^##|^###|Boundary:|Depends:|Implementation Notes|Status:" .specs/2026-08-04-15-50_Feature-webhook-failure-policy-ha
```

## Implementation Notes

- 2026-08-04: discovery確認Kustomize canonical source為`admission.yaml`，未引用的`webhook.yaml`與文件描述不一致。
- 2026-08-04: canonical objectSelector目前未要求`inject-vault-agent=true`，若直接切Fail會擴大受管namespace的admission blast radius。
- 2026-08-04: base/prod均為replica1、沒有明確RollingUpdate與topology spread；PDB1在單replica下不提供備援endpoint。
- 2026-08-04: 已建立requirements/design/tasks，尚未修改tests、production manifests或user-facing docs。
- 2026-08-04: T1 base/prod red evidence皆為replica1、PDB1、failurePolicy Ignore、timeout5，render中沒有injection selector、RollingUpdate或topology spread欄位。
- 2026-08-04: Kubernetes官方文件確認缺少任一topology key的node會被多重spread constraints略過；設計修正為hostname spread＋zone preferred anti-affinity，避免無zone label環境被排除。
- 2026-08-04: T2 canonical objectSelector已要求`inject-vault-agent=true`，namespace與self-exclusion條件維持不變。
- 2026-08-04: T3 base/prod已render為replicas2、RollingUpdate 0/1、hostname spread、zone preference與PDB1；既有probe、preStop5與termination45保持不變。
- 2026-08-04: T4 final base/prod皆為Fail且timeout5；unused`webhook.yaml`已移除。新增`prod-fail-open`過渡overlay，其render與prod只有failurePolicy一行差異。
- 2026-08-04: T5 README、deploy與annotations雙語既有文件已改指向canonical`admission.yaml`，並記錄兩階段rollout、rollback、PDB/topology限制與監控需求。
- 2026-08-04: V1已通過`go test -race -count=1 ./...`、`go vet ./...`、三個Kustomize render、結構化manifest assertions、stale reference搜尋與`git diff --check`。
- 2026-08-04: `golangci-lint run ./...`仍因既有`no go files to analyze`載入問題失敗；此問題與本次僅修改manifest及文件的範圍無關，未宣稱lint通過。
- 2026-08-04: V2未執行；server-side dry-run、實際endpoint/node/zone分布與failure drills需使用者另行授權non-production目標叢集操作。
- `devops-deployment-strategies`影響設計：採Rolling而非Blue-Green/Canary，並把readiness、PDB、rollback與failure drill納入同一交付。
- `_workspace/`不屬於本spec，不納入提交。

## 驗證結果摘要

- 新行為驗證: 本地render與結構化manifest契約驗證通過；prod-fail-open與prod只有failurePolicy差異。
- 回歸驗證: Go race tests與vet通過；既有golangci-lint專案載入問題仍存在。
- 文件一致性: canonical manifest、selector、兩階段rollout、rollback與限制已同步至繁中及既有英文文件。
- 剩餘風險: target cluster server-side dry-run、實際拓撲、control-plane telemetry與non-production failure drills尚未驗證，V2維持Planned。

## 後續改善

- [ ] 由service owner定義Webhook availability SLO與error budget。
- [ ] 評估replicas3或跨cluster disaster recovery是否符合未來SLO。
- [ ] 評估startup readiness在兩個listeners bind後才轉true。
