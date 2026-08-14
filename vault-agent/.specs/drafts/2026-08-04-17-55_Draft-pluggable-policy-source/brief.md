# Draft：Pluggable Policy Source 與外部 Policy 安全載入

Status: Draft

## 來源

- 日期：2026-08-04
- Owner：Vincent
- 使用者決策：先記錄，暫時不實作
- 接續 spec：`.specs/2026-08-04-17-18_Feature-policy-hot-reload`

## 文件定位

本 draft 記錄 workload authorization policy 未來可能來自本機檔案、目錄、Vault 或 AWS 加密來源時的設計邊界。它不修改已完成的本機 policy hot reload，也不建立 implementation tasks；待來源、bootstrap authentication 與 revision 契約確認後，再 promotion 為正式 spec。

## 背景

現有實作只支援：

```yaml
authorization:
  policy_file: ""
  policy_dir: /app/policy
  reload_interval_seconds: 30
```

Policy 由本機 file/directory 讀取，完成 strict decode、semantic validation 與來源前後 fingerprint 一致性檢查後，以 atomic generation 發布。若未來 policy 由 Vault 或 AWS 來源提供，檔案系統 snapshot、ConfigMap propagation 與 Git revision 都不能再作為通用來源契約。

## 問題陳述

1. Policy source 可能來自不同 backend，載入、版本、認證、逾時與重試語意不一致。
2. Vault 或 AWS policy source 需要先取得 backend credential，但該 credential 不能依賴尚未載入的 workload policy，否則形成循環依賴。
3. AWS KMS 負責加解密，不是一般 policy 文件儲存服務；若採 KMS，必須另外定義 ciphertext blob 的實際儲存位置與版本來源。
4. Vault KV version、AWS Secrets Manager VersionId、S3 object version 與本機 fingerprint 不能直接作為跨來源、跨 replica 的共同 identity。
5. 外部來源錯誤不得使無效或部分 policy 取代 last-known-good generation，也不得在 log、metric 或 error 洩漏 policy 內容。

## 未來目標

1. 建立 source-agnostic `PolicySource` contract，讓 startup 與 hot reload 不直接依賴 file/directory。
2. 支援本機 file、directory 與經核准的外部 source adapter。
3. 明確區分 policy bootstrap authentication 與 workload secret authorization。
4. 保持 validate-before-publish、immutable snapshot、atomic generation、startup fail-closed 與 background last-known-good 行為。
5. 為遠端呼叫定義 timeout、retry、backoff、cancellation、size limit 與安全 error category。
6. 在完成來源抽象後，再定義 source-independent revision 與跨 replica 收斂觀測。

## 暫定架構方向

```go
type PolicySource interface {
    Load(ctx context.Context) (*PolicyPayload, error)
}

type PolicyPayload struct {
    Content       []byte
    SourceVersion string
}
```

以上只表示責任邊界，不是已確認的公開 API：

- `Content` 必須有大小上限，且不得出現在 log、metric label 或 error。
- `SourceVersion` 是來源提供的診斷 metadata，不得直接視為跨來源 policy identity。
- Decode、merge、semantic validation、內容 identity 與 atomic publish 仍由共用 domain flow 負責。
- Source adapter 只負責安全取得完整 payload，不得自行改變授權語意。

## 候選來源

| Source | 取得方式 | 必要安全邊界 | 尚待確認 |
|--------|----------|--------------|----------|
| File | 讀取單一 YAML | 路徑限制、穩定 snapshot | 與現行行為相容 |
| Directory | 排序合併 YAML | symlink escape、完整 generation | 與現行行為相容 |
| Vault KV | 由 vault-agent bootstrap identity 讀取 | 最小權限 Vault policy、timeout、token renewal | mount、path、auth method、KV version |
| AWS Secrets Manager | 由 workload runtime identity 讀取 | 最小 IAM、VersionId、timeout | secret ARN、stage/version 契約 |
| KMS encrypted blob | 從指定 storage 取 ciphertext，再呼叫 KMS decrypt | storage read 與 `kms:Decrypt` 分離授權、encryption context | S3／Secrets Manager／其他 storage、key ARN、blob 格式 |

## Bootstrap authentication

- 讀取 authorization policy 的身分屬於 vault-agent control-plane identity。
- 它不得使用待載入 policy 進行授權，也不得沿用 Pod caller 的 namespace 或 ServiceAccount 決策。
- Vault adapter 應使用 vault-agent 自身 Kubernetes auth role、AppRole 或明確核准的 machine identity。
- AWS adapter 應使用 vault-agent Pod 的 IAM role 或其他 workload runtime identity。
- Backend policy／IAM 僅允許讀取指定 policy object 與必要 decrypt action，不得擴大至一般 workload secrets。
- Credential 不得寫入 policy YAML、repository、log 或 metric。

## Reload 與失敗語意

- Startup 沒有任何有效 policy 時維持 fail-closed。
- Background reload timeout、認證失敗、backend unavailable、decrypt failure、payload 過大或 validation failure 時保留記憶體中的 last-known-good generation。
- Context cancellation 必須中止外部請求並納入 graceful shutdown。
- Retry 必須使用 bounded exponential backoff 與 jitter，避免多 replica 同時重試造成 backend 壓力。
- 是否需要跨 process 的持久化 last-known-good cache 尚未決定；若未設計完整的 encryption、integrity、expiry 與 ownership，不得落地明文 policy cache。
- 多來源 fallback 不在第一階段預設啟用，避免來源優先級與 stale policy 語意不明。

## Revision 與跨 replica 觀測

- 本機 generation number 仍只代表單一 process 的成功發布次數。
- Git SHA、ConfigMap resourceVersion、Vault KV version 或 AWS VersionId 都只適用個別來源，不能作為通用 identity。
- 通用 revision 應由完成解密、canonicalization 與 validation 後的 policy 內容產生。
- 直接公開 policy digest 可能形成可猜測 metadata；候選方案是使用 cluster-shared key 執行 HMAC-SHA256，只暴露受控長度的 opaque revision。
- HMAC key 的產生、配送、rotation、跨 cluster scope 與遺失處理尚未決定，因此 revision 設計不得先行實作。
- Prometheus label 的 revision cardinality 與 stale series retention 必須在正式設計中評估。

## 非目標

1. 本 draft 不新增 Vault、AWS Secrets Manager、S3 或 KMS client。
2. 不修改目前 `policy_file`、`policy_dir`、hot reload 或 CLI validation。
3. 不新增管理 API、手動 reload trigger 或跨 replica barrier。
4. 不將 Vault／AWS workload secret fetcher 直接重用為 policy source；兩者的身分與權限邊界不同。
5. 不決定 durable cache、source fallback 或 HMAC key management。
6. 不執行 Kubernetes failure/recovery 演練。

## 待確認項目

1. Vault policy 的 mount、path、KV version 與 authentication method。
2. AWS 實際來源是 Secrets Manager，或由 S3／其他 storage 保存並交由 KMS decrypt 的 ciphertext blob。
3. 是否一次只允許一種 policy source；若支援 fallback，來源優先級與 freshness 如何定義。
4. Policy payload 最大容量、poll interval、request timeout 與 retry budget。
5. 是否需要跨 restart 的 encrypted last-known-good cache，以及 cache expiry。
6. Source-independent revision 是否採 HMAC，以及 key 的 scope、rotation 與管理來源。
7. 外部來源 credential 的 deployment contract、RBAC／IAM／Vault policy 與稽核需求。

## 未來 promotion 順序

1. 確認外部 backend 與 bootstrap authentication contract。
2. 建立正式 `Feature-pluggable-policy-source` requirements、design、tasks。
3. 先抽象現有 file/directory source，保持行為完全相容。
4. 每個外部 adapter 以獨立 task 或獨立 spec 實作與安全審查。
5. 完成 source abstraction 後，再規劃 source-independent revision 與跨 replica 收斂監控。
6. Policy validation JSON output 與 failure/recovery 演練維持後續順位。

## 風險摘要

| 風險 | 影響 | 未來處理方向 |
|------|------|--------------|
| Bootstrap identity 權限過大 | 可讀取非必要 policy 或 workload secrets | backend 最小權限與獨立稽核 |
| 外部 backend outage | 新 policy 無法載入 | startup fail-closed、background last-known-good、bounded retry |
| KMS 與 storage 責任混淆 | 無法定義版本與一致讀取 | 明確指定 ciphertext storage 與 object version |
| Digest 暴露 policy metadata | 可能被猜測或關聯 | 評估 HMAC opaque revision，不直接輸出 raw digest |
| Durable cache 保存明文 | policy metadata 落地洩漏 | 未完成加密與 integrity 設計前禁止持久化 |
| 多來源 fallback | stale 或錯誤來源覆蓋新 policy | 第一階段只允許單一來源 |

## 下一步

目前不進入實作。等外部來源類型與 bootstrap authentication 決策明確後，再將本 draft promotion 為正式 spec。
