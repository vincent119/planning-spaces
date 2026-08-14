# 設計文件：zlogger v1.1.0 升級與直接 zap 依賴移除

## 文件定位

本設計只處理 vault-agent 的 logger dependency boundary：依賴升級、設定轉換、global logger初始化、資源關閉與底層zap去耦。既有各層呼叫 `zlogger.InfoContext` 等global API的方式不變，也不重構application logging taxonomy。

## 已知契約狀態

### 依賴來源

- vault-agent目前使用`zlogger v1.0.5`與direct `zap v1.27.0`。
- zlogger最新正式tag為`v1.1.0`；tag後main只有changelog文件commit，無程式碼變更。
- zlogger `v1.1.0`最低Go版本為1.25，並依賴`zap v1.28.0`。
- vault-agent目前Go版本為1.26.5，符合最低版本。

### zlogger API契約

- `Configure(*ConfigPatch)`回傳`cleanup func() error`與error。
- Configure第一次成功後，同一process不可再次成功Configure；cleanup不重設資格。
- Configure失敗不發布半成品global logger，修正設定後可重試。
- `Init(*Config)`已deprecated，初始化失敗可能panic。
- ConfigPatch pointer欄位可區分未提供與明確零值；Resolve套用defaults、normalize並validate。
- cleanup關閉instance擁有的resources，但呼叫端仍應先Sync。
- `Logger`與`Field`是zap型別別名，vault-agent可只import zlogger而保持method／field相容。

### 設定資料契約

- vault-agent `LogConfig`維持現有九個欄位與`LOG_*`環境變數。
- dev defaults：level info、format console、outputs console、add caller、development與color enabled。
- prod override：format json、development false、color enabled false。
- 空log path目前由zlogger default落到`./logs`；升級後必須保留。
- file name不新增路徑設定語意，必須符合新版safe leaf規則。

### 現有實作

- `InitLogger`是單一logger adapter入口，只有`runServer`呼叫。
- `main.go`直接import zap，只使用`Error`、`Bool`、`String` helpers。
- delivery／application／infra已直接使用zlogger global與context API。
- shutdown結束後仍會輸出`server exited`，logger cleanup不得在此前執行。

## Bounded Context

### 包含

- zlogger與zap module版本關係。
- `LogConfig`到`zlogger.ConfigPatch`的轉換。
- global Configure error與cleanup ownership。
- 本地logger lifecycle的Sync／cleanup順序。
- main bootstrap失敗邊界及field helper去zap。
- logger升級測試與繁中文件。

### 不包含

- log message與field taxonomy重設計。
- logger backend、rotation、split sinks或non-global Instance導入。
- metrics、tracing exporter、security audit或secret redaction策略變更。
- Kubernetes manifests、ConfigMap、RBAC或環境變數新增。
- zlogger repository本身的修改。

## 設計原則

1. Tag pinning：只使用可重現的`v1.1.0`。
2. Dependency inversion：vault-agent source只認識zlogger公開alias與helpers。
3. Explicit lifecycle：初始化成功必須產生單一owner，負責Sync與cleanup。
4. Fail closed：strict config失敗即停止啟動，不fallback到未驗證logger。
5. Preserve behavior：project defaults與prod overrides優先於library default變動。
6. Test without global pollution：大部分測試使用轉換與注入seam，不重複Configure全域狀態。
7. Cleanup despite errors：Sync失敗不能阻止resource cleanup。
8. Safe bootstrap：global logger尚未建立時不呼叫zlogger global Fatal。

## 目標結構

```text
configs.LogConfig
       │
       ▼
toZloggerConfigPatch(env, config)
       │
       ▼
zlogger.Configure(patch)
       │
       ├── error ──► bootstrap failure，停止server啟動
       │
       └── cleanup
              │
              ▼
LoggerLifecycle
  ├── Logger *zlogger.Logger
  └── Close()
       ├── Logger.Sync()
       └── cleanup()
```

## Logger adapter設計

### 回傳模型

新增package-private或exported internal lifecycle type：

```go
type Lifecycle struct {
	Logger *zlogger.Logger
	// 內部持有cleanup、sync.Once與close error
}

func InitLogger(env string, cfg *configs.LogConfig) (*Lifecycle, error)
func (l *Lifecycle) Close() error
```

`Lifecycle.Close`契約：

1. nil receiver可安全返回nil。
2. 第一次呼叫先執行`Logger.Sync()`。
3. 無論Sync是否失敗都執行Configure cleanup。
4. 使用`errors.Join`保留兩者錯誤。
5. 使用`sync.Once`確保重複／並行Close不重複操作，並回傳相同結果。

不將cleanup暴露為獨立裸函式，避免caller遺漏Sync或錯置順序。

### 測試seam

Configure在process只能成功一次，不適合每個unit test直接操作global state。將核心建立拆成可注入helper：

```go
type configureFunc func(*zlogger.ConfigPatch) (func() error, error)
type loggerGetter func() *zlogger.Logger
```

Production傳入`zlogger.Configure`與`zlogger.GetLogger`；unit tests傳入fake configure及`zlogger.NewNop`。不得為測試增加production global reset能力。

## ConfigPatch轉換

`toZloggerConfigPatch`保留現有default／override順序：

1. 建立vault-agent dev defaults。
2. 若`LogConfig`非nil，套用現有非空string／非空outputs規則及明確bool值。
3. env為prod時固定覆寫format、development、color。
4. 轉成pointer fields。

特殊規則：

- 空`LogPath`與`FileName`以nil pointer表示未提供，使zlogger套用`./logs`及日期檔名。
- 非空`Outputs`複製後放入pointer，避免caller後續修改slice。
- bool一定使用non-nil pointer，保留明確false。
- level／format／outputs交由zlogger normalize及validate，不在adapter建立第二套enum parser。
- file output加上unsafe leaf與空path default tests。

## Main wiring

```text
load config
  → InitLogger
      → failure: 回傳bootstrap error，不呼叫zlogger.Fatal
      → success: log := lifecycle.Logger
  → 啟動既有components
  → graceful完成
  → 記錄server exited
  → defer lifecycle.Close
```

`main.go`所有`zap.Error/Bool/String`改為`zlogger.Err/Bool/String`。訊息、key、value與level保持相同。

為了讓bootstrap error可測試，`runServer`應回傳error或將load/init拆入可注入helper，由`main`在尚未初始化logger時寫入stderr並回傳非零exit。不得以panic或未初始化的zlogger global API處理。

## Dependency graph

目標source dependency：

```text
vault-agent ──direct──► zlogger v1.1.0 ──direct──► zap v1.28.0
```

vault-agent Go source不得import zap。`go.mod`經tidy後可能保留`go.uber.org/zap v1.28.0 // indirect`，這不視為違反去耦；禁止的是vault-agent的direct import與direct require語意。

## 關鍵行為

### 初始化成功

- Configure恰一次。
- GetLogger回傳非nil。
- caller持有唯一Lifecycle。
- global zlogger calls與raw logger method共用同一底層logger。

### 初始化失敗

- 原始Configure error可由adapter caller判斷。
- 不回傳Lifecycle。
- 不啟動Vault、AWS、Kubernetes clients或HTTP listeners。
- bootstrap輸出不得依賴尚未建立的global logger。

### 關閉

- shutdown最後一筆正常log完成後才Close。
- Sync error與cleanup error都保留。
- cleanup即使在Sync error時仍執行。
- 重複或並行Close不造成double-close。

### 相容性

- zlogger context API呼叫不變。
- `zlogger.Field`與`zlogger.Logger` alias維持field helper與method相容。
- JSON level不再有ANSI是預期修正。
- file output新建permissions收緊是預期安全變更。

## 受影響檔案計畫

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
- 本spec目錄

## 明確不變更

- `internal/configs` schema、env binding與defaults。
- `configs/config.yaml`與`configs/config.sample.yaml`。
- delivery、application與syncer infra既有log call sites。
- Deployment、Kustomize、Dockerfile、RBAC、policy與metrics。
- 英文文件。

## 風險與處理

| 風險 | 處理方式 |
|------|----------|
| `Init`在新版可能panic | 完全移除`zlogger.Init`，只用可回錯的Configure |
| Configure只能成功一次造成tests互相污染 | 注入configure/getter seam，不新增global reset |
| cleanup在Sync前執行導致buffer遺失 | Lifecycle集中固定Sync後cleanup並測試事件順序 |
| Sync失敗跳過cleanup造成file handle leak | 無條件執行cleanup，以errors.Join保留錯誤 |
| 空log path被當成明確空值而拒絕file output | 空optional string轉nil，沿用zlogger default `./logs` |
| bool false被defaults覆蓋 | 所有有效bool以non-nil pointer傳入並做table test |
| strict validation造成既有不合法部署無法啟動 | 文件列出合法值、duplicate與safe leaf限制，fail closed |
| 直接zap依賴殘留 | source scan加`go mod tidy`及module graph驗證 |
| logger過早cleanup使shutdown log遺失 | Lifecycle以defer放在runServer最外層，最後log後才執行 |
| bootstrap error使用未初始化logger | 程序入口改用return/stderr contract |

## 決策摘要

- 升`v1.1.0` tag，不追main。
- 使用global `Configure`，不切換Instance模式。
- 以Lifecycle封裝logger與cleanup，不回傳裸cleanup tuple。
- source完全移除zap import，field helpers統一由zlogger提供。
- 不新增logger設定欄位；file permission採library安全預設。
- strict validation失敗即停止啟動。
