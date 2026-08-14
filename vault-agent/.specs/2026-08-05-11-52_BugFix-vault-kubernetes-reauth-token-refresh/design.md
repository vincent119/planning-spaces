# 設計文件：Vault Kubernetes Re-auth ServiceAccount Token 輪替支援

## 定位

本設計以最小變更修正 Vault Kubernetes re-auth 的 credential freshness。核心是把長生命週期 authenticator 改為可重建的 auth factory，同時保持既有 `authMu`、token generation 與共享 re-auth state 的併發契約。

## 已知契約

- `kubernetes.NewKubernetesAuth()` 在建立時讀取 ServiceAccount token。
- `KubernetesAuth.Login()` 使用物件內已保存的 token，不會重新讀檔。
- `VaultClient.FetchSecret()` 只在 Kubernetes auth 模式且 Secret read 回傳 403 時嘗試 auth recovery。
- 每個 logical fetch 最多執行一次 re-auth 與一次驗證重讀。
- `ensureReauthenticated()` 以 generation 判斷是否已有其他請求完成更新，並以鎖定狀態合併並行重新認證。
- waiter context cancellation 不得取消共享 leader。
- 上層只取得固定 `ErrSecretFetchFailed` 類型，不得收到 raw credential 或 backend detail。

## Bounded Context

本修正只位於 Vault infrastructure adapter 的 Kubernetes authentication 邊界：

```text
VaultClient
  ├─ static token mode
  └─ Kubernetes auth mode
       ├─ auth factory
       │    ├─ ServiceAccount token file
       │    └─ NewKubernetesAuth(role, mount path, current token)
       └─ generation-aware re-auth coordinator
```

Application、delivery、authorization、resilient fetch decorator、AWS adapter 與 Kubernetes deployment 不參與本次設計。

## 設計原則

1. Credential freshness：每次實際登入都從 token source 建立新 authenticator。
2. Fail closed：讀取或建立失敗不得使用舊 credential。
3. Single leader：credential refresh 放在既有 leader critical path 內，不讓 waiter 各自讀檔。
4. Minimal retention：client 只保存 factory，不保存 raw JWT 或可長期持有 JWT 的 authenticator。
5. Single construction path：initial login 與 re-auth 使用相同 factory。
6. Stable external contract：不新增設定、不改 domain interface、不改 retry 與 HTTP 行為。
7. Safe diagnostics：錯誤與 log 只表達固定階段分類，不輸出 credential 或 Vault metadata。

## 目標流程

### 初次 Kubernetes login

```text
NewVaultClient
  → 建立 production auth factory(role, mount path, default token path)
  → factory.New(ctx)
      → 讀取目前 ServiceAccount token
      → 建立新的 KubernetesAuth
  → authenticator.Login(ctx, vault client)
  → 驗證 auth response
  → SetToken(Vault client token)
  → VaultClient 保存 factory，不保存 authenticator
```

### 403 重新認證

```text
Secret read 403
  → ensureReauthenticated(request generation)
      → generation 已更新：直接返回
      → 已有 leader：成為 waiter
      → 無 leader：建立共享 re-auth state，啟動單一 leader
          → factory.New(shared context)
              → 重新讀取目前 ServiceAccount token
              → 建立新的 KubernetesAuth
          → Login(shared context, vault client)
          → 驗證 response
          → SetToken(new Vault token)
          → generation + 1
      → caller 等待共享結果或自己的 context 取消
  → 成功後只驗證重讀 Secret 一次
```

## 元件設計

### Auth factory

在 `internal/syncer/infra` 定義最小內部介面：

```go
type vaultKubernetesAuthFactory interface {
	New(context.Context) (vaultKubernetesAuthenticator, error)
}
```

Production implementation 保存：

- Vault Kubernetes role
- auth mount path
- ServiceAccount token path
- 必要且可測試的 token reader 或 SDK constructor seam

`New(ctx)` 每次都建立獨立 authenticator。實作可以使用 HashiCorp 套件的 `WithServiceAccountTokenPath()`，但不得在 factory 建立時預讀並快取 token。若使用自有 reader，讀出的 token 只傳入當次 SDK constructor，不得寫入 `VaultClient` 欄位。

### VaultClient 欄位

以 auth factory 取代長生命週期 `k8sAuth`：

```text
k8sAuth vaultKubernetesAuthenticator
    ↓
k8sAuthFactory vaultKubernetesAuthFactory
```

factory 為 `nil` 表示 static token 模式。既有 `authMu`、`tokenGeneration`、`vaultReauth`、waiter counter 與 lifecycle context 保持原責任。

### Context 與檔案讀取

- `New(ctx)` 在讀檔前後檢查 context，避免已取消操作繼續登入。
- 一般本機檔案讀取無法在進行中由 context 強制中斷；其範圍僅限單一小型 projected token 檔案。
- Vault login 使用既有共享 re-auth context 與總 fetch deadline。
- waiter cancellation 只停止該 waiter 等待，不取消 leader。

### 狀態更新

只有下列條件全部成功後才允許狀態更新：

1. factory 建立新 authenticator。
2. Vault login 成功。
3. response、Auth 與 ClientToken 均有效。

成功後在既有鎖定規則下呼叫 `SetToken()` 並增加 generation。任何失敗都不得更新 token 或 generation，且不得保留失敗 authenticator供下次使用。

### 錯誤與日誌

- Runtime error 與 log message 使用固定英文分類字串，以符合既有程式碼契約。
- 不 wrap 或記錄 token file path、JWT、Vault address、auth mount path、role 或 raw SDK error。
- 上層 fetch 結果維持既有安全 domain error。
- 初次 auth response 為空時的相鄰 log 移除 address、auth path 與 role fields，只保留固定事件分類。
- 測試使用明確虛構 marker，並驗證 marker 不出現在 error 或 captured log。

## 併發不變量

- 同一 generation 同時收到 N 個 403，只能有一個 active re-auth state。
- 只有 leader 呼叫 factory；waiter 不讀 token 檔、不建立 authenticator、不呼叫 Login。
- leader 成功後 generation 只增加一次。
- leader 失敗後所有 waiter 共用該次失敗；state 完成且 waiter 清理後才移除。
- 後續獨立 403 可以建立新 state，再次讀取最新 token。
- 任一 waiter 取消不影響 leader、其他 waiter或 generation。
- shutdown 仍可透過既有 lifecycle context 終止登入與等待。

## 測試設計

### Factory 與 token rotation

- 暫存 token 檔先寫固定 token A，執行初次登入。
- 將同一檔案更新為固定 token B，觸發 403 re-auth。
- 攔截 Vault Kubernetes login JSON payload，驗證兩次 JWT 依序為 A、B。
- 驗證 client 不保留 authenticator或 raw token 欄位。

### 併發收斂

- N 個不同 Secret read 使用相同 generation 並同時回傳 403。
- fake factory、token reader 與 authenticator 分別計數。
- 驗證 re-auth 階段的 read、create、login 均恰為一次，所有 caller 成功或共用失敗。

### Fail-closed

- token 檔案不存在或 reader 回錯。
- factory constructor 回錯。
- Login 回錯。
- response nil、Auth nil 或 ClientToken empty。
- 驗證每種情況不 fallback、不更新 token/generation、不重複 re-auth，error/log 不含 marker。

### Regression

- static token 403 不呼叫 factory。
- generation 已更新的 caller 不再 re-auth。
- re-auth 後重讀仍為 403 時直接失敗。
- waiter cancellation 維持獨立。
- lifecycle shutdown、總 fetch budget 與 resilient retry contract不變。

## 受影響檔案

- `internal/syncer/infra/vault_client.go`
- `internal/syncer/infra/vault_client_test.go`
- 可新增 `internal/syncer/infra/vault_kubernetes_auth.go`
- 可新增 `internal/syncer/infra/vault_kubernetes_auth_test.go`
- `docs/README.zh-Hant.md`
- `docs/deploy.zh-Hant.md`
- `docs/architecture-diagrams.zh-Hant.md`
- 本 spec 目錄

## 明確不變更

- `internal/configs/` 與 `configs/config.sample.yaml`
- `cmd/vault-agent/`
- `deployments/`、Dockerfile、RBAC與 volume mounts
- `internal/syncer/infra/resilient_fetcher.go`
- AWS adapter、authorization、Admission contract與 webhook handler
- 公開 interface、annotation與 metrics

## 風險與控制

| 風險 | 控制 |
|------|------|
| factory 被每個 waiter 呼叫，造成登入風暴 | factory invocation 只能位於既有 single leader path，並以計數測試固定 |
| 初次登入與 re-auth 使用不同讀檔邏輯 | 兩者共用同一 factory |
| 讀檔失敗後回退舊 JWT | 不保存長生命週期 authenticator；失敗直接 fail closed |
| generation 在失敗時被錯誤增加 | 只在新 Vault token 已設定後更新 generation，加入狀態測試 |
| token marker 出現在錯誤或 log | 固定錯誤分類、禁止 raw wrap、captured log marker test |
| token path 成為新設定負擔 | 維持 Kubernetes 預設 projected token path，不新增 config |
| context 已取消仍開始登入 | factory 讀檔前後檢查 context，Login 使用既有 context |
| 修正影響 static token 模式 | factory nil 明確代表 static mode，加入零讀檔測試 |

## 決策摘要

- 採用每次登入建立新 authenticator，不在既有 authenticator 上增加 refresh method。
- auth factory 是內部測試 seam，不成為公開 API 或 application dependency。
- 保留 403-driven recovery，不引入背景 watcher。
- 不新增使用者設定或 deployment 變更。
- token 無法讀取時 fail closed，不以 availability 為由重用舊 credential。
