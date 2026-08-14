# 需求文件：zlogger v1.1.0 升級與直接 zap 依賴移除

## 文件定位

本 Chore 接續 vault-agent 現有 logger adapter，將 `github.com/vincent119/zlogger` 從 `v1.0.5` 升至正式 tag `v1.1.0`，並採用新版可回報錯誤及明確 cleanup 的初始化契約。此變更不重寫既有結構化日誌內容、request context 欄位、安全稽核 taxonomy 或 graceful shutdown 主流程。

需求來源：

- zlogger `v1.1.0` tag：<https://github.com/vincent119/zlogger/tree/v1.1.0>
- `v1.0.5...v1.1.0`：<https://github.com/vincent119/zlogger/compare/v1.0.5...v1.1.0>
- 初始化生命週期：<https://github.com/vincent119/zlogger/blob/v1.1.0/docs/zh-TW/lifecycle.md>
- 設定契約：<https://github.com/vincent119/zlogger/blob/v1.1.0/docs/zh-TW/configuration.md>
- 安全契約：<https://github.com/vincent119/zlogger/blob/v1.1.0/docs/zh-TW/security.md>

## 背景

目前 vault-agent 固定 `zlogger v1.0.5`，`internal/infra/logger.InitLogger` 建立完整 `zlogger.Config` 後呼叫已棄用的 `zlogger.Init`，再回傳 `*zap.Logger`。該函式宣告 error，但實際上永遠回傳 nil error。

zlogger `v1.1.0` 新增 `ConfigPatch`、`Configure`、cleanup 與嚴格設定驗證。舊 `Init` 雖保留來源碼相容，但初始化失敗可能 panic，且無法由 vault-agent 管理新版持有的檔案資源。vault-agent 的 `main.go` 也直接使用 `zap.Error`、`zap.Bool` 與 `zap.String`，因此 `go.mod` 將 zap 列為直接依賴。

## 目前行為

1. `InitLogger` 呼叫 `zlogger.Init(*Config)`，不會把初始化錯誤回傳給 caller。
2. `InitLogger` 回傳 `*zap.Logger`，使 adapter API 暴露底層 implementation type。
3. `runServer` 只 defer `log.Sync()`，沒有持有或執行 zlogger `v1.1.0` cleanup。
4. `main.go` 直接 import zap，只為使用 logger type與 field helpers。
5. logger 設定缺少新版的 strict validation；不安全 file name、未知 output 或重複 output 可能在 library 升級後改變啟動結果。
6. logger 初始化前使用 zlogger global log API 無法保證訊息已初始化或正確結束程序。

## 目標行為

1. 依賴固定為 zlogger 正式 tag `v1.1.0`，不使用 `main` pseudo-version。
2. `InitLogger` 使用 `zlogger.Configure(*ConfigPatch)`，初始化錯誤由 caller 處理，不得 panic。
3. logger adapter 回傳 `*zlogger.Logger` 與可重複呼叫的 lifecycle cleanup，不直接 import zap。
4. lifecycle cleanup 固定先同步 logger，再執行 zlogger cleanup；即使 Sync 失敗也必須繼續 cleanup。
5. `main.go` 的 zap field helpers 全部改用等價 zlogger helpers，移除 Go source 對 zap 的直接 import。
6. `go.mod` 不再把 zap 視為 vault-agent 的直接程式碼依賴；zap 可因 zlogger 而保留為 indirect dependency。
7. 現有 dev／prod logger defaults、context fields、message、field key與 log level 行為保持不變。
8. 新版 strict validation 失敗時啟動 fail closed，回傳可診斷錯誤，不發布半初始化 global logger。
9. file output 沿用預設 `./logs`，新建目錄／檔案權限依 zlogger `v1.1.0` 收斂為 `0700`／`0600`。
10. logger 尚未初始化前的 config／logger bootstrap failure 使用明確 stderr／return contract，不依賴 zlogger global API。

## 目標

- 安全升級 zlogger 至 `v1.1.0`。
- 採用可回報 error、可 cleanup 的初始化生命週期。
- 移除 vault-agent source 對 zap 的直接耦合。
- 保持既有結構化日誌與 context correlation 契約。
- 讓新版 strict config與 file output security 行為有自動化測試與文件。

## 非目標

- 不追蹤 zlogger `main` 或使用 pseudo-version。
- 不改用 non-global `zlogger.Instance`；現有 application仍使用 zlogger global context API。
- 不啟用 `SplitOutput`、`NewSplitCore`、timberjack或 log rotation。
- 不新增 `WithDirPerm`、`WithFilePerm` 的外部設定。
- 不批次改寫既有 log message、field key、level或安全 taxonomy。
- 不使用 `Redacted` 取代既有 category-only logging；敏感資料收斂另案處理。
- 不修改 metrics、tracing、authorization、Admission、Secret fetch 或 Kubernetes manifests。
- 不把 logger cleanup 加入會提前關閉 logger 的 graceful cleaner順序；logger必須最後仍可記錄 shutdown結果。

## 驗收情境

### 場景一：dev logger 初始化

- 測試：`TestToZloggerConfigPatch_DevelopmentDefaults`、`TestInitLogger_UsesConfigure`
- 假設：env為dev，logger設定使用專案預設值。
- 當：呼叫 `InitLogger`。
- 那麼：Configure只呼叫一次，format為console、development與color enabled為true、caller enabled為true，並回傳非nil logger與lifecycle。

### 場景二：prod override

- 測試：`TestToZloggerConfigPatch_ProductionOverrides`
- 假設：env為prod，外部設定要求console、development與color enabled。
- 當：建立 zlogger `ConfigPatch` 並 resolve。
- 那麼：format固定為json、development與color enabled固定為false，其餘合法設定保持不變。

### 場景三：明確 false 不被預設覆蓋

- 測試：`TestToZloggerConfigPatch_PreservesExplicitFalse`
- 假設：`LogConfig` 明確提供 `add_caller=false`、`add_stacktrace=false`、`development=false`、`color_enabled=false`。
- 當：轉換並 resolve設定。
- 那麼：四個 bool仍為false，不因 zlogger defaults改回true。

### 場景四：嚴格設定驗證失敗

- 測試：`TestInitLogger_RejectsInvalidConfiguration`
- 假設：level、format、output、重複 output、file name或 file output組合不符合新版契約。
- 當：初始化 logger。
- 那麼：回傳可由 `errors.Is` 分類的設定錯誤，不 panic、不發布 logger、不遺留資源。

### 場景五：file output使用安全預設

- 測試：`TestToZloggerConfigPatch_FileOutputUsesDefaultPath`、`TestInitLogger_RejectsUnsafeFileName`
- 假設：outputs包含file，未提供log path，file name為空。
- 當：resolve並初始化設定。
- 那麼：log path為`./logs`，空file name使用日期命名；若file name含路徑語意則拒絕。

### 場景六：logger關閉順序

- 測試：`TestLoggerLifecycle_CloseSyncsBeforeCleanup`、`TestLoggerLifecycle_CloseIsIdempotent`
- 假設：logger已成功初始化。
- 當：lifecycle被關閉一次或多次。
- 那麼：第一次固定先Sync再cleanup；Sync失敗仍執行cleanup；後續呼叫不重複同步或關閉資源，並回傳相同結果。

### 場景七：移除直接 zap 使用

- 測試：`go list -deps ./...`與 source scan
- 假設：所有 `zap.Error/Bool/String` 已有 zlogger等價 helper。
- 當：完成升級並執行搜尋。
- 那麼：vault-agent Go source不再import `go.uber.org/zap`；`go.mod`不把zap列為direct dependency，編譯與日誌欄位型別仍相容。

### 場景八：bootstrap failure

- 測試：`TestRunServer_LoggerInitializationFailure`或等價可注入邊界測試
- 假設：設定載入或 `zlogger.Configure` 失敗，global logger尚未發布。
- 當：server啟動流程處理該錯誤。
- 那麼：不呼叫未初始化的 zlogger global Fatal、不繼續啟動backend或listeners，並由程序入口得到明確失敗結果。

### 場景九：既有 context與安全觀測相容

- 測試：既有 delivery／application tests及全專案race test
- 假設：Webhook與Sync建立request_id、component、operation及安全決策fields。
- 當：升級zlogger後執行既有流程。
- 那麼：欄位名稱、decision taxonomy與category-only logging維持不變，context slice defensive copy不造成資料競爭或欄位遺失。

## 驗收條件

- [ ] `go.mod`固定`github.com/vincent119/zlogger v1.1.0`。
- [ ] 不使用zlogger `main` pseudo-version。
- [ ] vault-agent Go source沒有`go.uber.org/zap` import或`zap.*`呼叫。
- [ ] logger adapter公開給專案內部的型別為`*zlogger.Logger`，不是`*zap.Logger`。
- [ ] `zlogger.Init`不再使用；初始化改用`Configure`。
- [ ] 初始化錯誤可回傳且不panic。
- [ ] 成功初始化會保存並執行cleanup。
- [ ] lifecycle固定Sync後cleanup，且idempotent。
- [ ] dev與prod既有override語意保持不變。
- [ ] bool明確false可被保留。
- [ ] 未提供log path的file output沿用`./logs`。
- [ ] unsafe file name、未知值與重複output會fail closed。
- [ ] 新建file output權限契約記錄為`0700`／`0600`。
- [ ] logger初始化前的bootstrap failure不使用zlogger global API。
- [ ] 既有log message、field key、context與安全觀測契約沒有非預期變更。
- [ ] `go mod tidy`後module graph一致且zap至多為indirect dependency。
- [ ] 全專案race、vet、lint、package loading與既有policy／Kustomize gate通過。

## 驗證需求

- Logger adapter以table-driven tests固定defaults、prod overrides、bool零值與strict validation。
- Lifecycle使用可注入Sync／cleanup seam驗證順序、error join與idempotency，不依賴重複呼叫global Configure。
- Main bootstrap使用可注入初始化函式或小型helper測試失敗short-circuit。
- Source scan驗證沒有直接zap import、`zap.*`與`zlogger.Init`。
- `go mod graph`與`go list -m all`確認zlogger及zap版本來源。
- 執行全專案race test、vet、固定版本lint、policy validation與base/prod Kustomize render。

## 影響範圍

- Go module依賴版本與checksums。
- `internal/infra/logger`的初始化、設定轉換與lifecycle。
- `cmd/vault-agent`的bootstrap、logger helper與shutdown cleanup wiring。
- logger相關單元測試與繁中文件。
- 不影響外部API、annotation、Kubernetes設定schema或deployment manifest。
