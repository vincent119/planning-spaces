# Op Admin Cross Project Read Access Tasks

## 1. 文件與授權規則

- [x] 1.1 建立 `op_admin` 跨專案讀取規格
  - 限定為專案 API 的 `read` 類操作
  - 保留 `/admin/*` 與跨專案寫入限制
  - _Requirements: 1.1-1.6_

- [x] 1.2 調整後端 `EnforceProject`
  - `admin` 維持既有全域放行
  - `op_admin` 僅 `read` 略過 project role
  - `member` 維持既有專案角色檢查
  - _Requirements: 1.1-1.4_

- [x] 1.3 補授權單元測試
  - 覆蓋 `op_admin` read 放行
  - 覆蓋 `op_admin` update 不放行
  - 覆蓋 `member` read 不放行
  - _Requirements: 2.1-2.3_

## 2. 使用者選項 API

- [x] 2.1 建立後端使用者選項 API
  - 新增非 `/admin/*` 路由
  - 允許 `admin` / `op_admin` 讀取
  - 回傳最小選人欄位
  - 排除 `global_role = admin` 的使用者
  - _Requirements: 3.1-3.5, 3.7_

- [x] 2.2 前端人員班別頁改用使用者選項 API
  - 移除 `listAdminUsers()` 依賴
  - 新增 `listUserOptions()` client
  - 下拉選單只使用 `UserOption`
  - _Requirements: 3.3-3.6_

- [x] 2.3 補使用者選項 API 測試
  - `op_admin` 可讀
  - `member` 被拒絕
  - response 不含管理欄位
  - response 排除 `global_role = admin`
  - _Requirements: 3.1-3.7_

## 3. 驗證

- [x] 3.1 執行後端 auth 測試
  - `go test ./internal/auth`
  - _Requirements: 2.1-2.3_

- [x] 3.2 執行使用者選項 API 測試
  - 依實作 package 執行對應 `go test`
  - _Requirements: 3.1-3.7_

- [x] 3.3 執行前端型別檢查
  - `npm run typecheck`
  - _Requirements: 3.3-3.6_

- [x] 3.4 檢查格式與差異
  - `gofmt`
  - `git diff --check`
  - _Requirements: 1.1-3.7_
