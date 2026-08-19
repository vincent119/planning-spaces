# 任務文件：CareNest 本機提醒

Status: Planned

## Execution Context

- 意圖: 新增使用者可選的裝置本機提醒
- 非目標: 推播、醫療警報、用藥、同步與 Backend Server
- 已定決策: 使用者啟用時才請求權限，提醒失敗不影響核心功能
- 邊界: notifications、reminders、平台設定與測試
- 關鍵檔案: `lib/core/notifications/**`、`lib/features/reminders/**`、`android/**`、`ios/**`
- 完成條件: requirements 三個情境與核心回歸通過

### Protected Behavior

- 不在使用者啟用前請求權限
- 不修改核心紀錄或完成數
- 不新增遠端服務

### 邊界

#### Allowed Changes

- 本機通知、提醒設定、必要平台設定與測試

#### Forbidden

- Backend Server、遠端推播、用藥提醒、醫療警報、同步

## 任務依賴

| 任務 | Depends | 狀態 | 備註 |
|------|---------|------|------|
| T1 提醒設定與權限 | 核心 T6 | Planned | 等待核心 |
| T2 排程與重排程 | T1 | Planned | 不得重複 |

## 實作任務

- [ ] T1：提醒設定與權限
  - Status: Planned
  - Boundary:
    - Allowed Changes: reminder settings、permission gateway、平台設定與測試
    - Forbidden: 推播 token、遠端 API、用藥提醒
  - Depends: 核心 MVP T6
  - Context: 權限請求只能由使用者啟用提醒觸發
  - Verify:
    - requirements 首次權限情境
    - 權限拒絕與核心回歸

- [ ] T2：排程與重新排程
  - Status: Planned
  - Boundary:
    - Allowed Changes: 本機通知服務、排程狀態與測試
    - Forbidden: 背景同步與核心完成數修改
  - Depends: T1
  - Context: 設定或時間變更不得留下舊通知
  - Verify:
    - requirements 修改排程情境
    - 停用、App 重啟與排程失敗測試

## 驗證任務

- [ ] 驗收情境覆蓋
  - Verify: requirements 三個情境均有測試
- [ ] 回歸驗證
  - Verify: 核心 MVP 全部測試通過
- [ ] 品質檢查清單
  - 格式、分析與測試通過
  - 文件一致性已確認
  - Protected Behavior 通過
  - `git diff --stat` 已檢查
  - `git diff --check` 已通過

## 實作中斷恢復

恢復時依序讀取 Execution Context、目前任務、Protected Behavior、Implementation Notes；不得掃描整個 `.specs`。

## Implementation Notes

- 2026-08-17：由過大 MVP 規格拆分，等待核心 MVP。
- 2026-08-17：依 SDD skill 樣本重整文件，尚未實作。

## 驗證結果摘要

- 新行為驗證: 未執行
- 回歸驗證: 未執行
- 文件一致性: 已依樣本重整
- 剩餘風險: Android 與 iOS 排程限制待確認

## 後續改善

- [ ] 核心 MVP 使用者測試後重新評估提醒必要性
