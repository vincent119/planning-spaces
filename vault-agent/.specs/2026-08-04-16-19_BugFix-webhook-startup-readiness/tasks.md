# 任務文件：Webhook Startup Readiness

Status: Complete

## Execution Context

- 意圖: 讓readiness初始為false，只有在TLS key pair有效且Webhook與operations listeners都成功bind後才轉為true。
- 非目標: 不修改Kubernetes probes/manifests、Webhook handler/security、authorization、backend health、Worker語意或graceful cleanup。
- 已定決策: 初始503；同步TLS preparation；顯式`net.Listen`；Webhook先bind、operations後bind；partial failure關閉已取得listener；成功後才mark ready；目標cluster演練最後統一執行。
- 邊界: main startup/readiness orchestration、main tests、README、既有中英文deploy文件與本spec。
- 關鍵檔案: `cmd/vault-agent/main.go`、`cmd/vault-agent/main_test.go`、`README.md`、`docs/deploy.zh-Hant.md`、`docs/deploy.en.md`。
- 完成條件: 驗收情境A至G、main與全套race regression、vet/lint/format、manifest no-drift、文件與Boundary檢查通過；不以本機測試宣稱target cluster已驗證。

### Protected Behavior

- `/readyz`與`/healthz`路徑、operations port及status-only response contract不變。
- `/healthz`在process存活期間維持`200`。
- Webhook `/mutate`、AdmissionReview v1、TLS 1.2、mTLS/bearer authentication與authorization不變。
- Webhook與operations server timeouts及error filtering不變。
- signal path返回nil，server error path保留原cause並mark not-ready。
- Sync Worker context、cancel、done wait與HTTP、Worker、tracer cleanup順序不變。
- graceful shared timeout30秒、preStop5秒、termination45秒、replicas/PDB/rollout/failurePolicy不變。
- runtime log與error使用英文且不記錄secret、token、PEM、private key或完整AdmissionReview。

### 邊界

#### Allowed Changes

- `cmd/vault-agent/main.go`
- `cmd/vault-agent/main_test.go`
- `README.md`
- `docs/deploy.zh-Hant.md`
- `docs/deploy.en.md`
- `.specs/2026-08-04-16-19_BugFix-webhook-startup-readiness/`

#### Forbidden

- 修改`internal/`、go.mod、go.sum或新增dependency
- 修改`deployments/`、Dockerfile、RBAC、Service、PDB、Deployment或admission manifest
- 修改Webhook handler、authentication、authorization、Secret fetch/sync或policy behavior
- 修改ports、server timeout、graceful timeout、cleanup order或5/30/45契約
- 把Vault/AWS/Kubernetes API availability或Worker首輪納入readiness
- 使用固定sleep、port race、真實OS signal或live cluster作為unit test同步方式
- 執行target cluster rollout或failure drill
- 納入`_workspace/`

## 任務依賴

| 任務 | Depends | 狀態 | 備註 |
|------|---------|------|------|
| T1 建立startup readiness red tests | 無 | Complete | red evidence已確認 |
| T2 實作readiness與TLS prerequisite | T1 | Complete | 初始false、markReady、key pair preparation |
| T3 實作listener ownership與startup gate | T2 | Complete | explicit bind、Serve、partial close |
| T4 更新功能與部署文件 | T3 | Complete | startup contract與排障界線 |
| V1 驗收情境與main race驗證 | T1至T4 | Complete | requirements A至G通過 |
| V2 全專案回歸與品質檢查 | V1 | Complete | race/vet/format/render通過；既有lint載入問題已記錄 |

## 實作任務

- [x] T1 建立startup readiness red tests
  - Status: Complete
  - Boundary:
    - Allowed Changes: `cmd/vault-agent/main_test.go`、本spec Implementation Notes
    - Forbidden: 修改production code；真實固定port、OS signal、sleep同步或production certificate
  - Depends: 無
  - Context: 將既有readiness test改為初始503、markReady後200、markNotReady後503；新增TLS invalid key pair、兩次bind成功、第一個bind失敗、第二個bind失敗且close第一個listener測試。使用injected listen function、fake listener、sentinel error與`errors.Is`。
  - Verify:
    - `go test ./cmd/vault-agent -run 'Test(ReadinessHandler_StartsNotReadyAndTransitions|PrepareWebhookTLS_InvalidKeyPair|StartServerListeners_)'`
    - T2/T3前應因初始狀態或helper尚未實作而按預期失敗，red evidence記錄至Implementation Notes。
    - `gofmt -l cmd/vault-agent/main_test.go`無輸出。
    - `git diff --stat`、`git diff --check`。

- [x] T2 實作readiness與TLS prerequisite
  - Status: Complete
  - Boundary:
    - Allowed Changes: `cmd/vault-agent/main.go`、必要的`main_test.go`輔助調整、本spec Implementation Notes
    - Forbidden: listener/startup orchestration、server timeout、handler、security mode、Worker或cleanup變更
  - Depends: T1
  - Context: constructor保持atomic zero-value false，新增`markReady()`；保留`markNotReady()`。新增同步TLS key pair preparation，在ready與bind前驗證cert/key，保留既有TLS config的MinVersion、ClientCAs、ClientAuth等欄位，只設定server Certificates。
  - Verify:
    - `go test ./cmd/vault-agent -run 'Test(ReadinessHandler_StartsNotReadyAndTransitions|PrepareWebhookTLS_)'`
    - 測試確認invalid/mismatched key pair返回error且`errors.Is`或error chain可診斷。
    - code inspection確認error/log不輸出PEM、private key或secret值。
    - `go test -race -count=1 ./cmd/vault-agent`。
    - `git diff --stat`、`git diff --check`。

- [x] T3 實作listener ownership與startup gate
  - Status: Complete
  - Boundary:
    - Allowed Changes: `cmd/vault-agent/main.go`、`cmd/vault-agent/main_test.go`、本spec Implementation Notes
    - Forbidden: config schema、ports、server timeout、error filter、Worker lifecycle helper、graceful options或manifest變更
  - Depends: T2
  - Context: 新增package-private listen function與listener pair helper；先Webhook後operations。第一次失敗不做第二次；第二次失敗close第一個listener，close failure以`errors.Join`保留。成功後以`ServeTLS(listener, "", "")`與`Serve(listener)`啟動，然後mark ready。Sync Worker維持task內先啟動，確保startup failure時既有cleanup已有done channel可等待。
  - Verify:
    - `go test ./cmd/vault-agent -run 'TestStartServerListeners_'`
    - 成功測試確認兩次address順序與第二次成功前readiness為false。
    - failure測試確認call count、close count與`errors.Is`保留原cause。
    - 既有`TestWaitForTermination_*`、`TestStartSyncWorker_CancelAndWait`、`TestWaitForSyncWorker_DeadlineExceeded`通過。
    - `go test -race -count=1 ./cmd/vault-agent`。
    - `git diff --stat`、`git diff --check`。

- [x] T4 更新功能與部署文件
  - Status: Complete
  - Boundary:
    - Allowed Changes: `README.md`、`docs/deploy.zh-Hant.md`、`docs/deploy.en.md`、本spec Implementation Notes
    - Forbidden: 新增未實作config、宣稱backend健康、零中斷或target cluster已驗證
  - Depends: T3
  - Context: README說明startup初始503、TLS與兩個listeners完成後200；deploy文件記錄ready範圍、TLS/bind failure排障與外部backend不屬於gate。繁中與既有英文對應段落語意一致。
  - Verify:
    - `rg -n 'readyz|startup|listener|TLS|503|200' README.md docs/deploy.zh-Hant.md docs/deploy.en.md`
    - 人工比對文件與main實作，不得聲稱真實TLS handshake、AdmissionReview或backend health已完成。
    - `git diff --stat`、`git diff --check`。

## 驗證任務

- [x] V1 驗收情境與main race驗證
  - Status: Complete
  - Boundary:
    - Allowed Changes: task邊界內production code、tests、文件與Implementation Notes
    - Forbidden: 以sleep、固定port或live cluster取代deterministic tests
  - Depends: T1至T4
  - Context: 對照requirements場景A至G，確認initial false、ready transition、TLS prerequisite、兩個bind ordering、partial cleanup及既有shutdown/Worker lifecycle。
  - Verify:
    - `go test -race -count=1 ./cmd/vault-agent`
    - `go test ./cmd/vault-agent -run 'Test(ReadinessHandler_|PrepareWebhookTLS_|StartServerListeners_|WaitForTermination_|StartSyncWorker_|WaitForSyncWorker_)'`
    - 測試不得依賴wall-clock sleep或外部網路。

- [x] V2 全專案回歸與品質檢查
  - Status: Complete
  - Boundary:
    - Allowed Changes: 僅修正本BugFix直接造成的邊界內失敗，並先記錄Implementation Notes
    - Forbidden: 為通過檢查擴張至internal、manifest、dependency或外部cluster
  - Depends: V1
  - Context: 執行全套race、vet、lint、format、Kustomize no-drift、Boundary與diff檢查。既有lint若重現package loading問題，依狀態／根本原因／建議修復方式記錄，不得宣稱通過。
  - Verify:
    - `go test -race -count=1 ./...`
    - `go vet ./...`
    - `golangci-lint run ./...`
    - `gofmt -l cmd/vault-agent`無輸出。
    - `kubectl kustomize deployments/kustomize/base`與prod overlay成功，且`git diff -- deployments/`無輸出。
    - `git diff --stat`、`git diff --check`。
    - `git status --short`確認`_workspace/`未納入。

- [x] 品質檢查清單
  - [x] readiness初始為false，`/readyz`回傳503
  - [x] TLS key pair準備完成前不得bind或mark ready
  - [x] 兩個listeners成功前不得mark ready
  - [x] ready後`/readyz`回傳200
  - [x] 第一個bind失敗不嘗試第二個bind
  - [x] 第二個bind失敗關閉第一個listener且保留error cause
  - [x] TLS config保留既有security fields
  - [x] shutdown與server error切回not-ready
  - [x] healthz維持200
  - [x] Worker cancel/wait與cleanup順序不變
  - [x] server addresses、handlers、timeouts與error filter不變
  - [x] 5/30/45、replicas、PDB、rollout與failurePolicy無manifest drift
  - [x] runtime error/log為英文且不含機密值
  - [x] deterministic tests不使用sleep、固定port或live cluster
  - [x] README與中英文deploy文件一致
  - [x] 全套race與vet通過
  - [x] lint結果如實記錄
  - [x] `git diff --stat`已檢查
  - [x] `git diff --check`已通過
  - [x] `_workspace/`未納入

## 實作中斷恢復

恢復時優先讀取：

1. 本文件的`Execution Context`
2. 目前未完成task
3. `Protected Behavior`
4. `Implementation Notes`

不得預設掃描整個`.specs`目錄。若文件很大，先用標題與關鍵字定位：

```bash
rg -n "^#|^##|^###|Boundary:|Depends:|Implementation Notes|Status:" .specs/2026-08-04-16-19_BugFix-webhook-startup-readiness
```

## Implementation Notes

- 2026-08-04: discovery確認`newReadinessState()`目前立即儲存true，現有test固定初始200。
- 2026-08-04: discovery確認兩個servers以`ListenAndServeTLS`／`ListenAndServe`各自在goroutine內bind，`servers listening`日誌沒有同步bind證據。
- 2026-08-04: `go doc net/http.Server.ServeTLS`確認可傳入預先建立的`net.Listener`；TLSConfig已含Certificates時不要求再次提供certificate/key檔案。
- 2026-08-04: 已定設計為TLS prerequisite、Webhook bind、operations bind、Serve goroutines、mark ready的順序；第二個bind失敗必須close第一個listener。
- 2026-08-04: 使用者要求本次只建立requirements/design/tasks，尚未修改production code、tests、manifest或user-facing docs。
- 2026-08-04: 目標cluster rollout與failure drills延後到最後整合驗證，不納入一般`run task`的自動外部操作。
- 2026-08-04: T1 red test如預期在compile階段失敗，缺少`markReady`、`prepareWebhookTLS`與`bindServerListeners`；production code修改前已固定startup state、TLS與listener ownership契約。
- 2026-08-04: T1 tests同時引用T2與T3 helper，因此T2無法在T3 helper缺少時單獨完成package compile；依既定dependency連續實作T2/T3，再以各自selector分開驗證。
- 2026-08-04: 實作檢查發現若TLS/bind先於Worker，startup failure會讓預先註冊的Worker cleanup等待nil done channel；設計修正為保留既有Worker先啟動順序，startup failure由defer cancel並由cleanup等待done，不影響readiness gate。
- 2026-08-04: T2 readiness初始false、markReady與TLS config clone/key pair preparation已完成；security fields preservation與main race test通過。
- 2026-08-04: T3顯式listener ownership、兩階段bind、partial close、joined error causes與ServeTLS/Serve startup gate已完成；listener tests、既有termination及Worker tests通過。
- 2026-08-04: T4 README與中英文deploy文件已說明startup 503/200 gate、ready範圍、TLS/bind排障與機密紀錄限制；未修改manifest或宣稱backend/cluster已驗證。
- 2026-08-04: V1 requirements A至G與main package race test通過；tests使用httptest、injected listen function、fake listener與test-only certificate，未使用固定port、sleep或外部網路。
- 2026-08-04: V2全專案race tests、`go vet ./...`、gofmt、base/prod Kustomize render、manifest no-drift、Boundary與`git diff --check`通過。
- 2026-08-04: `golangci-lint run ./...`仍因既有`no go files to analyze`package loading問題失敗；未執行超出Boundary的`go mod tidy`，lint未宣稱通過。
- `_workspace/`不屬於本spec，不納入提交。

## 驗證結果摘要

- 新行為驗證: initial 503、ready 200、TLS prerequisite、bind ordering、partial close與joined causes測試通過。
- 回歸驗證: main與全專案race tests、vet通過；既有golangci-lint package loading問題仍存在。
- 文件一致性: README、中英文deploy文件與requirements/design/tasks均描述相同startup gate與限制。
- 剩餘風險: socket bind不等同真實TLS handshake或AdmissionReview；target cluster rollout、EndpointSlice時間線與failure drills依使用者決策延後至最後整合驗證。

## 後續改善

- [ ] 最後整合階段以non-production rollout觀察Pod readiness與Service EndpointSlice時間線。
- [ ] 最後整合階段執行Webhook fail-closed、單replica刪除與startup failure drills。
- [ ] 依實際telemetry評估startup duration metric；禁止使用namespace、Pod或secret path等高基數／機密labels。
