# 設計文件：Policy Validation CLI

## 設計摘要

在既有 binary 增加輕量 command dispatcher，只有無 arguments 時才進入 server path；`policy validate` 使用標準庫 `flag` 解析互斥來源並直接呼叫 `domain.LoadPolicy`。PolicyRule 只新增不參與 YAML 的來源 basename，`Validate` 以 group name 與 index 組成安全位置。Makefile 封裝兩個 smoke commands，GitHub Actions 在 image build 前執行。

## 文件定位

本設計實現同目錄 `requirements.md`，接續 policy directory deployment。它不改動 policy schema、authorization decision、application config 或 Kubernetes manifest，只新增部署前 validation surface。

## 已知契約狀態

- 需求來源: `requirements.md` 的 CLI、exit code、錯誤安全與 CI gate
- API / CLI / Hook contract: 現有 binary 無 CLI；新增 `policy validate (--file <path> | --dir <path>)`
- Data contract: `Policy`、`PolicyRule`、`ResourceRule` 的 YAML schema 維持 v1
- 既有實作: `domain.LoadPolicy` 已提供 strict decode、目錄排序、版本合併與 semantic validation
- 不可假造: CLI 不證明 Vault/AWS resource 存在或 server-side authorization 正確

## Bounded Context

包含：

- binary arguments dispatch
- policy validate flag parsing與 exit codes
- policy error 的 basename 與 rule index
- CLI unit tests與 domain error tests
- Makefile、GitHub Actions 與繁中使用文件

不包含：

- policy schema 擴充或 migration
- live Vault/AWS/Kubernetes connectivity validation
- policy hot reload、format、generate 或 auto-fix
- 新的第三方 CLI dependency
- deployment environment variables 或 volume mount 變更

## 設計原則

- loader 是 validation 的 single source of truth。
- command path 不初始化 server dependencies。
- error 只提供定位所需 metadata，不回顯 resource values。
- 使用標準庫維持 dependency surface 不變。
- CI command 可在本機以同一 Make target 重現。

## 需求對應

| 需求 / 驗收情境 | 設計處理方式 | 驗證方式 |
|-----------------|--------------|----------|
| 有效 file/dir | `runCLI` 解析後呼叫 `LoadPolicy` | CLI table tests 與 smoke commands |
| 使用方式錯誤 | dispatcher 與 `flag.FlagSet` 回傳 exit code `2` | usage tests |
| policy 無效 | loader error 映射 exit code `1` | invalid policy test |
| 安全定位 | unexported source basename 加 group/index | marker 不洩漏 test |
| CI gate | `make policy-validate` 先於 build step | workflow diff 與 Make target execution |

## 受影響檔案計畫

| 檔案 | 預期變更 | 原因 | 風險 |
|------|----------|------|------|
| `cmd/vault-agent/main.go` | 無參數 server、有參數 CLI dispatch | 單一 binary contract | server path 回歸 |
| `cmd/vault-agent/policy_command.go` | CLI parser 與 exit codes | 隔離 command 邏輯 | usage 不一致 |
| `cmd/vault-agent/policy_command_test.go` | CLI success/error/security tests | 固定公開行為 | 無 |
| `internal/syncer/domain/policy.go` | 規則來源 basename 與位置化錯誤 | 分檔排錯 | error text 變更 |
| `internal/syncer/domain/policy_test.go` | location 與不洩漏測試 | 保護安全契約 | 無 |
| `Makefile` | 新增 `policy-validate` | 本機/CI 共用 | 範例漂移 |
| `.github/workflows/docker-publish.yml` | build 前執行 validation | 阻止無效 policy | CI 增加少量時間 |
| `docs/config.zh-Hant.md`、`docs/deploy.zh-Hant.md` | CLI 與 CI 使用方式 | 可操作性 | 無 |

## 目標結構或流程

```text
vault-agent
  無 arguments
    -> runServer
  policy validate
    -> parse --file / --dir
    -> domain.LoadPolicy
      -> strict YAML decode
      -> stable directory merge
      -> Policy.Validate
    -> exit 0 / 1
  其他 arguments
    -> usage
    -> exit 2
```

## Mermaid Diagrams

不需要；CLI 分支與 loader 流程已由上方結構完整表示。

## 介面與資料契約

### API / CLI / Hook

- Input: `policy validate --file <path>` 或 `policy validate --dir <path>`
- Output: 成功訊息寫入 stdout；usage 與 validation error 寫入 stderr
- Exit code: 成功 `0`、policy 無效 `1`、CLI usage error `2`
- Error: 英文摘要包含 basename、規則 group/index 與 rule ID，不包含 path prefixes、templates、keys 或 YAML content

### Data / Config

- 新增資料: `PolicyRule` 的 unexported source basename，只供 error location
- 既有資料相容性: unexported field 不參與 YAML decode、encode 或 public schema

## 關鍵行為

- `main` 不帶參數時完全沿用 server path。
- CLI validation 不讀取 `config.yaml` 或 environment authorization source。
- `--file` 與 `--dir` 同時缺漏或同時存在都屬 usage error。
- loader errors不做字串解析；CLI只加固定前綴。
- directory merge 仍依檔名排序，規則索引以合併後 policy 的全域位置計算；duplicate ID error 顯示目前與首次定義位置。

## 前後端或跨模組設計

CLI delivery layer 只負責 arguments、streams 與 exit codes；domain layer負責 policy decode、merge 與 semantic validation；Makefile 與 GitHub Actions 呼叫相同 binary contract。

## Protected Behavior

- 無參數啟動 server 的 config、logger、client、worker、HTTP 與 graceful lifecycle 不變。
- `domain.LoadPolicy` 的 file/dir 互斥、stable sorting、strict YAML 與 fail-closed behavior 不變。
- runtime error/log 保持英文且不含機密值。
- Dockerfile、Kubernetes RBAC、volume mount 與 authorization switch 不變。

## 替代方案

| 方案 | 優點 | 缺點 | 結論 |
|------|------|------|------|
| 獨立 validator binary | server main 無 dispatch | 多一個 artifact 與 Docker packaging contract | 不採用 |
| CI 直接寫 Go test | 變更小 | 使用者本機與實際 binary 無同一 contract | 不採用 |
| 第三方 CLI framework | 擴充方便 | 為單一子命令增加 dependency | 不採用 |
| 標準庫 dispatcher＋flag | 輕量、可測、同一 binary | 需手動固定 usage | 採用 |

## 風險與處理方式

| 風險 | 影響 | 處理方式 | 驗證 |
|------|------|----------|------|
| dispatcher 進錯分支 | server 無法啟動 | 只有 len(args)==0 進 server | main regression tests |
| semantic error 洩漏 path | 機密 metadata 暴露於 CI log | 不格式化 resource values；marker assertion | CLI/domain security tests |
| error 缺乏檔案定位 | 多檔排錯困難 | basename＋group/index＋rule ID | exact substring tests |
| CI 與本機命令不同 | 無法重現 failure | workflow 只呼叫 Make target | workflow review |
| example directory 無 YAML | CI 固定失敗 | repository 保留合法 sample | `make policy-validate` |

## 實作注意事項

- 先建立 CLI red tests，再新增 dispatcher 與 command implementation。
- 將原 `main` server body移至 `runServer`，不在重構中改變既有行為。
- source 只保存 `filepath.Base(path)`，避免 error 洩漏 host absolute path。
- 每完成 task 更新 `tasks.md` 的 Status 與 Implementation Notes。
- `_workspace/` 不屬於本 spec，不納入提交。
