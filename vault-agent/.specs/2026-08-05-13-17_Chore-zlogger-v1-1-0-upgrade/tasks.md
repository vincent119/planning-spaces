# 任務文件：zlogger v1.1.0 升級與直接 zap 依賴移除

Status: InProgress

## Execution Context

- 意圖：將zlogger從v1.0.5升至正式v1.1.0，採用Configure／ConfigPatch／cleanup契約，並移除vault-agent source對zap的直接依賴。
- 非目標：不導入Instance、split sinks、rotation、permission config、log taxonomy重寫或Kubernetes manifest變更。
- 已定決策：pin v1.1.0；global Configure；Lifecycle封裝Sync後cleanup；`*zlogger.Logger`與zlogger field helpers；strict config fail closed；空log path沿用`./logs`。
- 邊界：module metadata、logger adapter、main bootstrap／wiring、對應tests、四份繁中文件與本spec。
- 關鍵檔案：`internal/infra/logger/logger.go`、`cmd/vault-agent/main.go`、`go.mod`、`go.sum`。
- 完成條件：版本、ConfigPatch、error、cleanup、idempotency、prod defaults、explicit false、strict validation、no direct zap、bootstrap short-circuit與全套品質gate通過。

### Protected Behavior

- 現有log message、level、field key與field value保持不變；只替換等價helper。
- Webhook request_id、component、operation及authorization／admission安全taxonomy不變。
- Runtime、error、log與Go測試失敗訊息使用英文；註解與文件使用繁體中文。
- Dev維持console/development/color，prod維持json/non-development/no-color。
- 空log path在file output時沿用zlogger預設`./logs`。
- Logger在`server exited`等最後shutdown logs完成後才Sync與cleanup。
- Sync失敗不得阻止cleanup；Close必須idempotent且race-safe。
- Logger初始化失敗不得啟動backend、worker或HTTP listeners。
- 不得新增global reset或允許第二次Configure的production後門。
- zap可作為zlogger的indirect dependency存在，但vault-agent source不得直接import。
- `_workspace/`不得修改或納入提交。

### 邊界

#### Allowed Changes

- `go.mod`
- `go.sum`
- `internal/infra/logger/logger.go`
- 新增 `internal/infra/logger/logger_test.go`
- `cmd/vault-agent/main.go`
- `cmd/vault-agent/main_test.go`
- `docs/README.zh-Hant.md`
- `docs/config.zh-Hant.md`
- `docs/observability.zh-Hant.md`
- `docs/deploy.zh-Hant.md`
- `.specs/2026-08-05-13-17_Chore-zlogger-v1-1-0-upgrade/`

#### Forbidden

- 使用zlogger main pseudo-version或未tag commit
- 保留`zlogger.Init`或新增recover包住初始化panic
- 使用non-globalInstance取代現有global zlogger calls
- 新增loggerglobal reset、第二次Configure或test-only production backdoor
- 改log message、level、field key、安全taxonomy或加入敏感值
- 新增split output、rotation、timberjack、custom sinks或permission env/config
- 修改`internal/configs`schema/default/env binding、config YAML或deployment manifests
- 修改delivery、application、syncer adapters、metrics、tracing、authorization或Secret fetch
- 修改英文文件
- 修改`_workspace/`

## 任務依賴

| 任務 | Depends | 狀態 | 備註 |
|------|---------|------|------|
| T1 固定zlogger v1.1.0 dependency baseline | 無 | Complete | tag pin、module graph、Go相容性 |
| T2 建立ConfigPatch與Lifecycle red tests | T1 | Complete | defaults、strict validation、cleanup order |
| T3 實作logger Configure與Lifecycle | T2 | Complete | no panic、Sync後cleanup、idempotent |
| T4 移除main直接zap依賴並串接bootstrap | T3 | Complete | zlogger helpers、short-circuit、最後cleanup |
| T5 更新繁中文件 | T4 | Complete | strict config、lifecycle、file security |
| V1 驗收與回歸驗證 | T1至T5 | InProgress | lint package loading未通過，其餘gate通過 |

## 實作任務

- [x] T1 固定zlogger v1.1.0 dependency baseline
  - Status: Complete
  - Boundary:
    - Allowed Changes: `go.mod`、`go.sum`、本spec Implementation Notes
    - Forbidden: Go source、tests、docs、使用pseudo-version
  - Depends: 無
  - Context: 使用`go get github.com/vincent119/zlogger@v1.1.0`，確認Go 1.26.5符合minimum；zap解析至v1.28.0。此階段只建立可編譯dependency baseline，不採用deprecated API作最終方案。
  - Verify:
    - `go list -m -json github.com/vincent119/zlogger`顯示`v1.1.0`
    - `go list -m -json go.uber.org/zap`顯示`v1.28.0`
    - `go mod graph`不存在zlogger pseudo-version
    - `go mod verify`
    - `git diff --check`

- [x] T2 建立ConfigPatch與Lifecycle red tests
  - Status: Complete
  - Boundary:
    - Allowed Changes: 新`internal/infra/logger/logger_test.go`、必要`main_test.go`、本spec Implementation Notes
    - Forbidden: production source、docs、其他package
  - Depends: T1
  - Context: 先固定dev/prod、explicit false、empty path default、invalid enum／duplicate output／unsafe leaf、Configure error、Sync-cleanup順序、error join、idempotent與bootstrap short-circuit。使用fake configure/getter，避免污染global Configure一次性狀態。
  - Verify:
    - `go test ./internal/infra/logger ./cmd/vault-agent -run 'Test(ToZloggerConfigPatch|InitLogger|LoggerLifecycle|InitializeServerLogger)'`應先因新contract不存在而失敗
    - Tests不得直接重設zlogger global狀態
    - 所有新test failure messages使用英文
    - `git diff --check`

- [x] T3 實作logger Configure與Lifecycle
  - Status: Complete
  - Boundary:
    - Allowed Changes: `internal/infra/logger/logger.go`、`logger_test.go`
    - Forbidden: main、module metadata、docs、其他logger call sites
  - Depends: T2
  - Context: 建立`toZloggerConfigPatch`、Configure/getter seam與Lifecycle。Optional空string／slice保持未提供；bool以pointer保留false；prod最後覆寫。Init失敗不panic、不回傳half-initialized lifecycle。Close以sync.Once固定Sync後cleanup及errors.Join。
  - Verify:
    - `go test -race ./internal/infra/logger -run 'Test(ToZloggerConfigPatch|InitLogger|LoggerLifecycle)'`
    - Event recorder證明Sync先於cleanup且Sync error不跳過cleanup
    - 100次並行Close只有一次Sync及cleanup
    - Invalid config可由`errors.Is`分類且Configure failure不呼叫getter
    - `rg -n 'zlogger\.Init|go\.uber\.org/zap' internal/infra/logger`無結果

- [x] T4 移除main直接zap依賴並串接bootstrap
  - Status: Complete
  - Boundary:
    - Allowed Changes: `cmd/vault-agent/main.go`、`main_test.go`、`go.mod`、`go.sum`
    - Forbidden: 改既有message/key/level、component初始化順序、graceful cleaners或其他packages
  - Depends: T3
  - Context: 以`zlogger.Err/Bool/String`逐一等價替換zap helpers；runServer持有Lifecycle且在最後log後defer Close。設定／logger初始化失敗透過可測return boundary停止，不呼叫未初始化zlogger global Fatal。最後`go mod tidy`移除direct zap requirement。
  - Verify:
    - `go test -race ./cmd/vault-agent -run 'Test.*(Logger|Bootstrap|InitializeServerLogger)'`
    - `rg -n 'go\.uber\.org/zap|zap\.' --glob '*.go' --glob '!_workspace/**' .`無結果
    - `rg -n 'zlogger\.Init' --glob '*.go' --glob '!_workspace/**' .`無結果
    - `go mod why -m go.uber.org/zap`只顯示經zlogger或其他dependency的indirect路徑
    - Existing log call的message、level、keys diff保持語意等價
    - Logger init failure不進入backend或listener construction

- [x] T5 更新繁中文件
  - Status: Complete
  - Boundary:
    - Allowed Changes: 四份繁中文件與本spec Implementation Notes
    - Forbidden: 英文文件、config YAML、deployment manifests
  - Depends: T4
  - Context: 記錄zlogger v1.1.0、strict values、duplicate output拒絕、safe leaf、default permissions、JSON no ANSI、Configure一次性與shutdown cleanup。不得宣稱支援未導入的split output／rotation／permission config。
  - Verify:
    - `rg -n 'zlogger|ConfigPatch|Configure|0700|0600|file_name|ANSI|cleanup' docs/README.zh-Hant.md docs/config.zh-Hant.md docs/observability.zh-Hant.md docs/deploy.zh-Hant.md`
    - 文件設定值與zlogger v1.1.0 contract一致
    - `git diff --check`

## 驗證任務

- [x] V1 驗收情境覆蓋
  - Status: Complete
  - Depends: T1至T5
  - Verify: requirements中的version pin、dev/prod、explicit false、strict invalid、file default/safe leaf、cleanup order/idempotency、no direct zap、bootstrap failure及context regression均有automated evidence。

- [ ] V1 回歸驗證
  - Status: InProgress
  - Depends: T1至T5
  - Verify:
    - `go test -race -count=1 ./...`
    - `go vet ./...`
    - `go list ./...`
    - `go mod verify`
    - `make lint`
    - `make policy-validate`
    - `kubectl kustomize deployments/kustomize/base`
    - `kubectl kustomize deployments/kustomize/overlays/prod`
    - Rendered manifests與升級前一致
    - `git diff --stat`
    - `git diff --check`

- [ ] V1 品質檢查清單
  - [x] zlogger固定v1.1.0，無pseudo-version
  - [x] Go 1.26.5符合dependency最低版本
  - [x] zap解析為v1.28.0且vault-agent source無direct import
  - [x] zlogger.Init已完全移除
  - [x] InitLogger使用Configure並可回傳error
  - [x] Logger adapter只暴露zlogger alias
  - [x] ConfigPatch保留dev／prod既有語意
  - [x] Explicit false未被defaults覆蓋
  - [x] Empty log path在file output時使用`./logs`
  - [x] Invalid enum、duplicate output與unsafe leaf fail closed
  - [x] Configure failure不發布或回傳half-initialized lifecycle
  - [x] Lifecycle固定Sync後cleanup
  - [x] Sync failure仍執行cleanup並保留兩個errors
  - [x] Lifecycle Close可重複及並行呼叫
  - [x] Bootstrap failure不使用未初始化global logger
  - [x] Logger cleanup晚於最後shutdown log
  - [x] 既有message、level、field key與security taxonomy未變
  - [x] Context correlation tests無race或field regression
  - [x] File output預設權限與safe leaf文件正確
  - [x] Config schema、sample、deployment manifests未變
  - [ ] 全專案race、vet、lint、package loading與module verify全部通過；目前只有lint package loading失敗
  - [x] Policy validation與base/prod Kustomize render通過
  - [x] 繁中文件一致
  - [x] `_workspace/`未納入
  - [x] `git diff --stat`已檢查
  - [x] `git diff --check`已通過

## 實作中斷恢復

恢復時優先讀取：

1. 本文件的`Execution Context`
2. 目前未完成task
3. `Protected Behavior`
4. `Implementation Notes`

不得預設掃描整個`.specs`目錄。需要定位時使用：

```bash
rg -n "^#|^##|^###|Boundary:|Depends:|Implementation Notes|Status:" .specs/2026-08-05-13-17_Chore-zlogger-v1-1-0-upgrade
```

## Implementation Notes

- 2026-08-05：Discovery確認vault-agent固定zlogger v1.0.5與direct zap v1.27.0，Go source只有`logger.go`型別及`main.go`field helpers直接使用zap。
- 2026-08-05：zlogger最新正式tag為v1.1.0；main比tag多兩個changelog-only commits，因此選擇tag，不使用pseudo-version。
- 2026-08-05：v1.1.0新增Configure／ConfigPatch／cleanup、strict validation、safe leaf、0700／0600 defaults及context field defensive copy；Init已deprecated且可能panic。
- 2026-08-05：設計以Lifecycle集中Sync與cleanup，避免裸cleanup被遺漏或順序錯置；global Configure一次性測試透過注入seam，不新增reset。
- 2026-08-05：本階段只建立requirements、design與tasks，尚未修改Go source、tests、dependencies或user-facing docs，所有task維持Planned。
- 2026-08-05：收到`run task`命令，開始T1正式tag dependency baseline。
- 2026-08-05：T1已固定zlogger v1.1.0與zap v1.28.0；兩者module metadata、Go版本與`go mod verify`通過。
- 2026-08-05：T2紅燈確認ConfigPatch轉換、Configure seam與Lifecycle contract不存在；T3完成strict resolve、explicit false、Sync後cleanup、error join及100 callers並行idempotency。
- 2026-08-05：T4以可注入bootstrap helper固定load/init failure short-circuit，main改持有Lifecycle並以zlogger helpers取代全部zap helpers；定向race tests通過。
- 2026-08-05：`go mod tidy`後vault-agent source不再direct import zap，zap v1.28.0只保留為indirect dependency；既有test直接import的prometheus/client_model被正確列為direct。
- 2026-08-05：T5完成繁中README、config、observability與deploy文件，記錄strict validation、safe leaf、JSON no ANSI、0700／0600及Sync後cleanup；未修改config schema或manifests。
- 2026-08-05：V1通過全專案race test、vet、`go list ./...`、module verify、policy validation與base/prod Kustomize render；實際zlogger Configure lifecycle整合測試亦通過。
- 2026-08-05：固定v2.12.2的golangci-lint仍在載入階段回報`no go files to analyze`，未產生任何本次變更lint finding；module已tidy且`go list ./...`成功，因此V1維持InProgress。
- `_workspace/`不屬於本spec，不納入未來implementation commit。

## 驗證結果摘要

- 新行為驗證：dev／prod、explicit false、file default、strict invalid、Configure integration、Sync-cleanup順序、joined errors、100 callers並行Close及bootstrap short-circuit tests通過。
- Dependency驗證：zlogger v1.1.0、zap v1.28.0 indirect、無pseudo-version、module checksum與module graph正確。
- 去耦驗證：vault-agent Go source沒有zap import、`zap.*`或`zlogger.Init`。
- 回歸驗證：全專案race、vet、package loading、policy validation及base/prod Kustomize render通過。
- 未通過項目：`golangci-lint v2.12.2`仍在context loading階段回報沒有可分析的Go檔案。

## 後續改善

- [ ] 另案評估是否全面採用non-global `zlogger.Instance`以提升多instance測試隔離。
- [ ] 另案評估`zlogger.Redacted`在明確需要保留欄位名稱的安全log情境；不得以自動遮罩取代allowlist。
- [ ] 只有在有分級檔案或容量rotation需求時，另案評估`NewSplitCore`與caller-owned sinks。
