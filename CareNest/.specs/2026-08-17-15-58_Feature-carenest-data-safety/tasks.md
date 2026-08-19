# 任務文件：CareNest 資料安全

Status: Planned

## Execution Context

- 意圖: 提供使用者主動控制的加密備份、還原與完整刪除
- 非目標: 雲端、帳號、同步、共享與 Backend Server
- 已定決策: 先驗證再寫入、取代前安全備份、禁止明文交付
- 邊界: backup、database、files、settings、安全決策與測試
- 關鍵檔案: `lib/core/backup/**`、`lib/core/database/**`、`lib/core/files/**`、`lib/features/settings/**`
- 完成條件: 安全決策確認、requirements 三個情境與所有前置回歸通過

### Protected Behavior

- 不建立明文備份或未確認取代
- 不上傳資料或加入帳號
- 安全決策未確認前不開始正式加密實作

### 邊界

#### Allowed Changes

- 安全決策、備份、還原、刪除、必要平台設定與測試

#### Forbidden

- Backend Server、雲端、自動外傳、明文敏感資料、同步

## 任務依賴

| 任務 | Depends | 狀態 | 備註 |
|------|---------|------|------|
| T1 安全決策 | 核心、照護細節 | Planned | 先決條件 |
| T2 備份與驗證 | T1 | Planned | 不含還原寫入 |
| T3 還原與刪除 | T2 | Planned | 需 rollback |

## 實作任務

- [ ] T1：完成安全決策紀錄
  - Status: Planned
  - Boundary:
    - Allowed Changes: 私有 spec、安全決策與技術驗證原型
    - Forbidden: 正式備份、還原與刪除實作
  - Depends: 核心 MVP、照護細節完成
  - Context: 確認加密、金鑰、格式版本、檔案選取與平台限制
  - Verify:
    - 決策經審查並由使用者確認
    - 威脅與失敗模式均有處理

- [ ] T2：實作備份與驗證
  - Status: Planned
  - Boundary:
    - Allowed Changes: backup、加密、封存、摘要與測試
    - Forbidden: 雲端、自動外傳、明文敏感資料
  - Depends: T1
  - Context: 備份包含資料與私有照片，具格式版本與完整性驗證
  - Verify:
    - 正常備份、損毀、錯誤金鑰、遺失照片與敏感日誌測試
    - 前置規格回歸

- [ ] T3：實作還原、刪除與失敗回復
  - Status: Planned
  - Boundary:
    - Allowed Changes: restore、交易、檔案清理、settings UI 與整合測試
    - Forbidden: 未確認取代、不可回復中途狀態、同步
  - Depends: T2
  - Context: 所有還原先驗證，取代前建立安全備份
  - Verify:
    - requirements 無效備份與 rollback 情境
    - 合併衝突、取消、全部刪除、照片清理與所有回歸

## 驗證任務

- [ ] 驗收情境覆蓋
  - Verify: requirements 三個情境均有測試
- [ ] 回歸驗證
  - Verify: 核心與照護細節全部測試通過
- [ ] 品質檢查清單
  - 無明文敏感備份與日誌
  - 還原失敗不改動資料
  - 文件與安全決策一致
  - `git diff --stat` 已檢查
  - `git diff --check` 已通過

## 實作中斷恢復

恢復時依序讀取 Execution Context、目前任務、Protected Behavior、Implementation Notes；不得掃描整個 `.specs`。

## Implementation Notes

- 2026-08-17：由過大 MVP 規格拆分，等待前置規格與安全決策。
- 2026-08-17：依 SDD skill 樣本重整文件，尚未實作。

## 驗證結果摘要

- 新行為驗證: 未執行
- 回歸驗證: 未執行
- 文件一致性: 已依樣本重整
- 剩餘風險: 加密、金鑰、格式與衝突契約未確認

## 後續改善

- [ ] 前置規格完成後建立正式安全決策紀錄
