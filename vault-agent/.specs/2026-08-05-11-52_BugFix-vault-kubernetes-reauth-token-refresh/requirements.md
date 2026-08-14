# 需求文件：Vault Kubernetes Re-auth ServiceAccount Token 輪替支援

## 定位

修正 Vault Kubernetes authentication 在重新認證時重用舊 ServiceAccount token 的安全與可用性缺口。當 Kubernetes 輪替 projected ServiceAccount token 後，下一次 Vault 重新認證必須重新讀取 token 檔案並建立新的 `KubernetesAuth`，不得繼續使用啟動時快取的 JWT。

## 背景

目前 `VaultClient` 在啟動時建立一次 `KubernetesAuth`，之後 `reAuthenticate()` 只再次呼叫同一物件的 `Login()`。專案固定使用的 HashiCorp Kubernetes auth 套件會在 `NewKubernetesAuth()` 時讀取 ServiceAccount token，並將內容保存在物件內；`Login()` 不會重新讀檔。

因此 Pod 長時間執行且 ServiceAccount token 已輪替時，Vault 403 觸發的重新認證仍會提交舊 JWT。既有併發鎖定雖能把同一 generation 的多個 403 收斂成一次登入，卻會收斂到一次使用過期 token 的登入。

## 目前行為

1. 啟動時建立 `KubernetesAuth` 並讀取一次 ServiceAccount token。
2. 初次 Vault Kubernetes login 成功後，`VaultClient` 長期保留同一個 authenticator。
3. Secret read 回傳 403 時，由單一 re-auth leader 呼叫既有 authenticator 的 `Login()`。
4. authenticator 使用啟動時保存的 JWT，不會讀取已輪替的 token 檔案。
5. 同 generation 的其他請求等待 leader 結果，不會重複登入。

## 目標行為

1. 初次登入與每次實際重新認證都透過同一個 auth factory 建立全新的 `KubernetesAuth`。
2. 每次建立 authenticator 時都從 ServiceAccount token 檔案取得當下內容。
3. `VaultClient` 不長期保存含 JWT 的 `KubernetesAuth`；只保存建立 authenticator 所需的非機密設定與 factory。
4. 同 generation 的並行 403 仍只有一個 leader 讀取 token、建立 authenticator 並登入 Vault。
5. token 讀取、authenticator 建立、登入或 response validation 任一步驟失敗時 fail closed，不得回退使用舊 JWT。
6. 失敗時不得更新 Vault client token 或 token generation；下一個獨立重新認證週期可以重新讀檔再嘗試。
7. static Vault token 模式完全不讀取 ServiceAccount token，也不執行 Kubernetes re-auth。

## 目標

- 支援 Kubernetes projected ServiceAccount token 的正常輪替。
- 保留既有 single-flight re-auth、generation 收斂與 waiter cancellation 行為。
- 縮短 JWT 在長生命週期物件中的保留時間。
- 對 token 讀取與登入失敗維持安全、固定且不洩漏機密的錯誤邊界。
- 以自動化測試證明輪替後送往 Vault 的是新 token。

## 非目標

- 不新增自訂 ServiceAccount token path 設定。
- 不新增背景 token watcher、主動更新或定時 Vault login。
- 不改變由 Vault Secret read 403 觸發一次 auth recovery 的既有條件。
- 不改變 Vault token renewal、lease renewal或 Vault Agent sidecar 行為。
- 不改變 static Vault token、AWS Secrets Manager 或其他 backend。
- 不修改 external fetch retry、bulkhead、queue、in-flight 合併或 metrics taxonomy。
- 不新增 Vault policy、Kubernetes RBAC、volume、Deployment 或 Dockerfile 設定。
- 不保證 Go 字串中的 JWT 可被可靠清零；只避免將 authenticator 長期保存在 client。

## 使用情境

### 情境一：ServiceAccount token 已輪替

- Given：Pod 啟動時以 token A 完成 Vault Kubernetes login。
- And：Kubernetes 已將 token 檔案內容更新為 token B。
- When：Secret read 回傳 403 並觸發重新認證。
- Then：系統重新讀取 token 檔案並建立新的 `KubernetesAuth`。
- And：Vault login payload 使用 token B，登入成功後重讀 Secret 一次。

### 情境二：多個請求同時收到 403

- Given：多個請求使用相同 token generation。
- When：它們同時收到 403。
- Then：只有一個 leader 重新讀取 ServiceAccount token、建立新的 authenticator 並執行一次 Vault login。
- And：其他 waiter 共用相同結果。

### 情境三：token 檔案無法讀取

- Given：Secret read 已回傳 403，且 ServiceAccount token 檔案暫時不存在或不可讀。
- When：leader 執行重新認證。
- Then：不呼叫 Vault login、不使用舊 JWT，也不更新 Vault client token 或 generation。
- And：caller 收到既有安全 Secret fetch failure。

### 情境四：新 token 無法登入

- Given：系統已讀取輪替後 token，且 Vault Kubernetes login 拒絕該 token。
- When：重新認證結束。
- Then：不回退使用先前 token，也不執行無界登入循環。
- And：所有 waiter 收到同一安全失敗結果。

### 情境五：waiter 被取消

- Given：leader 正在使用最新 token 執行 Vault login。
- And：其中一個 waiter 的 context 被取消，其他 waiter 仍在等待。
- Then：被取消的 waiter 立即返回，但不取消共享的重新認證。
- And：其他 waiter 仍可取得 leader 結果。

### 情境六：static Vault token 模式

- Given：client 使用 static Vault token。
- When：Secret read 回傳 403。
- Then：不讀取 ServiceAccount token 檔案，也不建立 `KubernetesAuth`。
- And：沿用既有 terminal failure 行為。

## 驗收準則

- [ ] Production factory 每次被呼叫都建立新的 `KubernetesAuth`，並在該次建立期間讀取目前的 ServiceAccount token。
- [ ] 初次登入與重新認證共用相同 factory，避免兩套 token 讀取邏輯。
- [ ] `VaultClient` 不保存長生命週期 `KubernetesAuth` 或 raw ServiceAccount token。
- [ ] token A 輪替為 token B 後，下一次 login request 可由測試觀察到 token B，且不再送出 token A。
- [ ] 同 generation 的 N 個並行 403 只造成一次 token read、一次 authenticator creation 與一次 Vault login。
- [ ] token read failure 時 Vault login 次數為零，Vault client token 與 generation 不變。
- [ ] authenticator creation、login 或空 auth response 失敗時不回退舊 JWT，且不執行第二次 re-auth。
- [ ] re-auth 成功後的驗證讀取若仍為 403，維持每個 logical fetch 最多一次 auth recovery。
- [ ] waiter cancellation 不取消 leader 或其他 waiter。
- [ ] static token 模式不讀取 ServiceAccount token。
- [ ] error、log 與測試輸出不包含 JWT、token path、Vault address、auth mount path、role 或 raw backend error。
- [ ] 初次 auth response 為空的相鄰 log 改為固定分類，不輸出 Vault address、auth path 或 role。
- [ ] 不新增 config、環境變數、Kubernetes manifest或 runtime dependency。

## 驗證方式

- 使用可計數 auth factory 驗證每個 re-auth generation 只建立一次 authenticator。
- 使用暫存 token 檔與攔截 transport 驗證 initial login 與 rotated login payload 分別使用不同固定測試 token。
- 使用並行測試與 race detector 驗證 single leader、waiter cancellation、generation 與共享結果。
- 使用失敗注入驗證 token read、factory、login、空 response 與重讀 403 都 fail closed。
- 執行 Vault adapter 測試、全專案 race test、vet、固定版本 lint 與既有 policy／Kustomize regression checks。

## 影響範圍

- 主要影響 `internal/syncer/infra/vault_client.go` 與其測試。
- 可新增同 package 的 Kubernetes auth factory 檔案與測試，以隔離 token source 與 SDK construction。
- 更新繁體中文架構與部署文件，說明 token rotation 與 re-auth 收斂語意。
- 不影響公開 annotation、設定 schema、domain interface、Kubernetes manifest或其他 backend。
