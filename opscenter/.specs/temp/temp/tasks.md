# OnCall Ticket System Core Tasks

Status: Planned

## Execution Context

意圖：把舊大型 OnCall Ticket System spec 收斂成新版 SDD 可執行形狀，核心範圍限於平台基礎與 Ticket MVP。

非目標：不在本 spec 實作 SLA、通知、報表、Jira、完整排班、SSO、API Key、K8s。

已定決策：

- frontend/backend 不再分拆成兩份 task 文件。
- 新功能不得追加到舊已完成 task 並維持完成狀態。
- 一般工作區 read API 以表單 read 權限為第一層授權來源。

邊界：

- 文件來源限 `.kiro/specs/2026-06-01_10-22_oncall-ticket-system`。
- 本次輸出位置限 `.kiro/specs/temp/temp`。

關鍵檔案：

- `requirements.md`
- `design.md`
- `tasks.md`
- `roadmap.md`

完成條件：

- 核心需求、設計與 task 能追溯到舊 spec。
- Phase 2 / Phase 3 已拆出 roadmap。
- 每個 task 有 Boundary、Depends、Context、Verify。

## Protected Behavior

- 不修改原始 `.kiro/specs/2026-06-01_10-22_oncall-ticket-system`。
- 不改程式碼。
- 不把後續功能標成核心 MVP。
- 不用舊 task 勾選狀態直接宣稱新 spec 已完成。

## 實作任務

- [ ] 1. 收斂核心需求
  - Boundary:
  - Allowed Changes: `requirements.md`
  - Forbidden: 新增 SLA、報表、Jira、SSO、K8s 的正式需求細節
  - Depends: 無
  - Context: 舊 `requirements.md` 同時包含 MVP、Phase 2、Phase 3，需要只留下核心平台與 Ticket MVP
  - Verify: `rg --no-ignore -n "Status: Draft|## 目標|## 非目標|## 驗收情境" .kiro/specs/temp/temp/requirements.md`

- [ ] 2. 收斂跨層設計
  - Boundary:
  - Allowed Changes: `design.md`
  - Forbidden: 展開已拆出功能的細部資料表或 UI 流程
  - Depends: 1
  - Context: 舊設計分成 general/backend/frontend，本次需合成同一份 contract-first 設計
  - Verify: `rg --no-ignore -n "已知契約狀態|Bounded Context|Permission Contract|受影響檔案計畫" .kiro/specs/temp/temp/design.md`

- [ ] 3. 建立新版 task 邊界
  - Boundary:
  - Allowed Changes: `tasks.md`
  - Forbidden: 將舊已完成 task 直接搬成完成狀態
  - Depends: 1, 2
  - Context: 新版 SDD 要求每個 task 具備 Boundary、Depends、Context、Verify
  - Verify: `rg --no-ignore -n "Boundary:|Depends:|Context:|Verify:" .kiro/specs/temp/temp/tasks.md`

- [ ] 4. 拆出後續 roadmap
  - Boundary:
  - Allowed Changes: `roadmap.md`
  - Forbidden: 在核心 requirements 中保留 Phase 2/3 的實作承諾
  - Depends: 1
  - Context: SLA、通知、報表、Jira、排班、SSO 等已有後續 spec 或 deferred task
  - Verify: `rg --no-ignore -n "外部 spec|Deferred|Promote" .kiro/specs/temp/temp/roadmap.md`

## 驗證任務

- [ ] 5. 文件結構驗證
  - Boundary:
  - Allowed Changes: 無
  - Forbidden: 修改原始舊 spec
  - Depends: 1, 2, 3, 4
  - Context: 確認新版 SDD 文件存在且狀態清楚
  - Verify: `find .kiro/specs/temp/temp -maxdepth 1 -type f | sort`

- [ ] 6. 格式驗證
  - Boundary:
  - Allowed Changes: 無
  - Forbidden: 執行程式碼測試或修改程式碼
  - Depends: 5
  - Context: 本次只產文件，使用 diff check 驗證基本格式
  - Verify: `git diff --check -- .kiro/specs/temp/temp`

## 品質檢查清單

- [ ] `requirements.md` 有文件定位、背景、目標、非目標、現有行為、新行為、使用情境、驗收情境、驗收條件與驗證需求。
- [ ] `design.md` 有已知契約狀態、Bounded Context、設計原則、流程、受影響檔案計畫、關鍵行為、風險。
- [ ] `tasks.md` 有 `Status`、`Execution Context`、`Protected Behavior`、任務邊界、驗證任務與 `Implementation Notes`。
- [ ] `roadmap.md` 有拆出功能與 promote 建議。
- [ ] 沒有修改原始舊 spec。

## Implementation Notes

- 本文件為收斂稿，不代表核心功能全部已完成。
- 舊 backend task 的多數勾選只能代表歷史狀態，不能取代新版驗收。
- 若使用者確認採用，下一步應建立正式 spec 目錄並從本收斂稿複製內容，再依實際目標調整 task 狀態。

