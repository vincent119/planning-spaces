# 需求文件：Webhook Startup Readiness

## 來源

- Draft: 無
- Type: BugFix
- Owner: Vincent
- Status: Complete

## 文件定位

本 spec 接續 `.specs/2026-08-04-14-39_BugFix-kubernetes-graceful-termination/` 留下的 startup readiness 項目，並支援 `.specs/2026-08-04-15-50_Feature-webhook-failure-policy-ha/` 的 fail-closed 高可用目標。前一個 graceful termination spec 已讓 `/readyz` 在 shutdown 與 server error 時轉為 `503`，但 readiness state 仍以 ready 起始，早於 Webhook TLS 與 operations listener 實際 bind。本 spec 只修正啟動階段的 readiness 契約，不重寫 Webhook handler、caller authentication、workload authorization、Sync Worker、graceful cleanup 或 Kubernetes probes。

參考來源：

- 既有 readiness：`cmd/vault-agent/main.go` 的 `newReadinessState()` 立即儲存 `true`
- 既有 server 啟動：兩個 goroutine 分別呼叫 `ListenAndServeTLS` 與 `ListenAndServe`
- 既有測試：`TestReadinessHandler_TransitionsToNotReady` 固定初始 `200`
- 既有部署：readiness probe 呼叫 operations port 的 `/readyz`
- 既有高可用契約：正式 Webhook 使用 `failurePolicy: Fail` 與 2 replicas

## 背景

目前 application 在建立 readiness state 時就宣告 ready。兩個 HTTP server 尚未 bind 前，或只有其中一個 server 能夠 bind 時，程式內部狀態仍是 ready。一般 Kubernetes probe 必須先連上 operations listener 才能觀察 `/readyz`，因此雙方都尚未 bind 時不會直接產生 ready endpoint；但既有狀態模型無法表示「operations listener 可用、Webhook listener 尚未完成」或「啟動必要條件尚未全部成立」。

在正式環境使用 `failurePolicy: Fail` 時，Pod 被加入 Service endpoints 代表 API Server 可能開始送出 AdmissionReview。readiness 必須只在 Webhook TLS listener 與 operations listener 都成功 bind，且 TLS server certificate 已成功載入後轉為 ready，避免尚未具備處理能力的 replica 被視為可用。

## 現有行為

- `newReadinessState()` 初始為 `true`，`/readyz` 預設回傳 `200`。
- server task 先啟動 Sync Worker，再以兩個 goroutine 呼叫 `ListenAndServeTLS` 與 `ListenAndServe`。
- `log.Info("servers listening")` 在 goroutine 啟動後立即寫入，沒有同步確認兩個 socket 已 bind。
- 任一 server 非預期返回 error 時，`waitForTermination` 將 readiness 切為 false並返回 error。
- 收到 shutdown context 時，readiness 切為 false，既有 HTTP、Worker、tracer cleanup 順序生效。
- TLS server certificate 由 `ListenAndServeTLS` 在 server goroutine 中載入。

## 新行為

- readiness state 初始為 not-ready，`/readyz` 回傳 `503`。
- 啟動流程同步載入並驗證 Webhook TLS certificate/key pair，失敗時保持 not-ready並返回具 cause 的英文 error。
- 啟動流程以明確 listener ownership 依序 bind Webhook 與 operations address。
- 第二個 listener bind 失敗時，關閉第一個 listener，保持 not-ready並返回具 cause 的英文 error。
- 兩個 listener 成功 bind後，server 使用既有 handler、timeouts 與 TLS config開始 Serve，接著 readiness 才切為 ready。
- ready 後 `/readyz` 回傳 `200`；shutdown 或任一 server error仍切回 `503`。
- `servers listening` 日誌只在啟動必要條件完成後寫入，不記錄 certificate內容或機密資料。

## 目標

1. readiness 初始狀態為 false。
2. Webhook TLS certificate/key pair 必須在 ready 前成功載入。
3. Webhook 與 operations listeners 必須都在 ready 前成功 bind。
4. partial bind failure 必須回收已取得的 listener，不留下半啟動 socket。
5. ready transition 必須使用既有 concurrency-safe atomic state。
6. shutdown 與 server error 的 not-ready 行為、error cause及 cleanup順序維持不變。
7. 以 deterministic unit tests 固定狀態轉換、啟動順序與 partial failure cleanup。

## 非目標

1. 不新增 startupProbe、修改 readinessProbe timing或變更 Deployment manifest。
2. 不新增 readiness config、port、endpoint或 response body。
3. 不檢查 Vault、AWS、Kubernetes API、policy內容或 Sync Worker首輪完成狀態。
4. 不把外部 backend availability納入 Pod readiness，避免暫時性外部故障移除所有 Webhook endpoints。
5. 不替換 `commons/graceful`、不導入 `errgroup` 或新 dependency。
6. 不修改 HTTP handlers、server timeouts、TLS最低版本、mTLS、bearer authentication或authorization。
7. 不修改 5 秒 preStop、30 秒 application cleanup、45 秒 termination budget。
8. 不執行目標叢集 rollout、failure drill或端對端 AdmissionReview；使用者將於最後階段統一執行。

## 使用情境

### 情境一：正常啟動

Process 建立初始 not-ready state，載入 TLS key pair並成功 bind Webhook與operations listeners。兩個 server開始 Serve後才標記 ready，Kubernetes readiness probe隨後取得 `200`。

### 情境二：TLS key pair無效

TLS certificate或private key不存在、不合法或不匹配。Process不得宣告 ready，也不得開始 listener；task返回保留底層cause的error並進入既有結束流程。

### 情境三：Webhook listener bind失敗

Webhook port已被占用或無法 bind。readiness維持 `503`，operations listener不應建立，task返回 bind error。

### 情境四：operations listener bind失敗

Webhook listener已成功取得，但operations port bind失敗。啟動流程關閉Webhook listener，readiness維持 `503`，task返回 bind error。

### 情境五：啟動後 server失敗或收到終止訊號

兩個 server已正常啟動且readiness為 `200`。任一 server回傳非 `http.ErrServerClosed` error或task context取消時，既有wait helper先將readiness切回 `503`，再進入graceful cleanup。

## 驗收情境

### 場景 A：readiness 初始為 not-ready

- 測試: `TestReadinessHandler_StartsNotReadyAndTransitions`
- 假設: 建立新的 readiness state
- 當: ready前與 `markReady()` 後分別呼叫 `/readyz`
- 那麼: HTTP status依序為 `503` 與 `200`

### 場景 B：shutdown transition保持單向安全

- 測試: `TestReadinessHandler_StartsNotReadyAndTransitions`、既有兩個 `TestWaitForTermination_*`
- 假設: readiness已由startup flow標記ready
- 當: 呼叫 `markNotReady()`、取消context或收到server error
- 那麼: `/readyz`回傳 `503`，context path返回nil，server error path保留原cause

### 場景 C：兩個 listener完成後才 ready

- 測試: `TestStartServerListeners_MarksReadyAfterBothBind`
- 假設: 可控制的listen function依序回傳兩個listener
- 當: startup helper完成TLS prerequisite與兩次bind
- 那麼: 第二次bind成功前readiness一直為false，完成後才能轉為true

### 場景 D：第一個 bind失敗

- 測試: `TestStartServerListeners_WebhookBindFailure`
- 假設: listen function第一次呼叫即返回sentinel error
- 當: startup helper執行
- 那麼: readiness維持false、只呼叫一次listen、返回error可由`errors.Is`找到sentinel cause

### 場景 E：第二個 bind失敗並回收資源

- 測試: `TestStartServerListeners_OperationsBindFailureClosesWebhookListener`
- 假設: 第一個listener成功，第二次listen返回sentinel error
- 當: startup helper執行
- 那麼: 第一個listener只被關閉一次、readiness維持false、返回error保留sentinel cause

### 場景 F：TLS key pair失敗不得進入bind

- 測試: `TestPrepareWebhookTLS_InvalidKeyPair`
- 假設: certificate/key測試資料無效或不匹配
- 當: startup TLS preparation執行
- 那麼: 返回具cause的error，listen function未被呼叫，readiness維持false

### 場景 G：既有 lifecycle不回歸

- 測試: `TestWaitForTermination_ContextMarksNotReady`、`TestWaitForTermination_ServerErrorMarksNotReady`、`TestStartSyncWorker_CancelAndWait`、`TestWaitForSyncWorker_DeadlineExceeded`
- 假設: server已進入ready或Worker已啟動
- 當: process進入termination或Worker cancellation流程
- 那麼: not-ready、return error semantics、Worker done wait與cleanup budget維持既有契約

## 驗收條件

- [x] `/readyz` 初始回傳 `503`。
- [x] 只有在TLS key pair有效且兩個listeners都成功bind後才能mark ready。
- [x] ready後 `/readyz` 回傳 `200`。
- [x] 第一個bind失敗時不嘗試第二個bind。
- [x] 第二個bind失敗時關閉第一個listener，且不得mark ready。
- [x] startup error使用英文且保留底層cause，不包含certificate內容、token或機密path。
- [x] shutdown與server error都將ready狀態切回false。
- [x] `/healthz`仍固定回傳 `200`。
- [x] Webhook與operations handler、address、timeouts及TLS security contract不變。
- [x] Sync Worker cancellation、HTTP shutdown、tracer cleanup與30秒shared budget不變。
- [x] Deployment的readiness probe、5/30/45 termination contract及HA manifests不變。
- [x] main package與全專案race tests通過。

## 影響範圍

- `cmd/vault-agent/main.go`
- `cmd/vault-agent/main_test.go`
- `README.md`
- `docs/deploy.zh-Hant.md`
- `docs/deploy.en.md`
- 本 spec 文件

## 驗證需求

- readiness state測試使用`httptest`，不啟動真實socket。
- listener sequencing與partial cleanup使用fake/injected listen function，不依賴固定port、sleep或外部網路。
- TLS preparation使用測試certificate fixture或暫存檔，fixture不得包含production key；測試完成後由testing lifecycle回收。
- 執行`go test -race -count=1 ./cmd/vault-agent`與`go test -race -count=1 ./...`。
- 執行`go vet ./...`、專案既有lint、`gofmt -l cmd/vault-agent`與`git diff --check`。
- 使用`kubectl kustomize`確認本次沒有產生manifest drift。
- 目標叢集startup與failure drill延後至使用者最後的整合驗證階段，不作為本次本機實作的假證據。

## 已知風險

- socket成功bind代表OS已接受監聽，但不保證後續每個TLS handshake或handler request成功；server runtime error仍由既有error channel切回not-ready。
- listener ownership從`ListenAndServe*`內部移到startup orchestration，partial failure與graceful close若處理錯誤可能造成socket洩漏或double close。
- 同步TLS key pair載入會使憑證問題更早暴露；錯誤訊息不得輸出PEM、private key或不必要的絕對機密路徑。
- readiness不檢查外部backend；這是刻意隔離startup能力與依賴健康度，避免backend故障造成所有Webhook endpoints同時被移除。

## 待確認項目

- 無。採「TLS key pair有效＋兩個listener成功bind」作為startup ready gate；目標叢集演練延後至最後整合階段。
