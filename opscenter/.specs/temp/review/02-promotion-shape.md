# Promotion Shape

Status: Draft

## 文件定位

本文件描述每個 Draft 被確認後，正式 spec 應該如何建立。這是新版 SDD 的目標形狀。

## Draft 到正式 spec 的轉換

Draft 仍在審閱時，只需要說清楚問題、邊界與待確認事項。

Promote 後必須建立：

```text
.kiro/specs/{YYYY-MM-DD-HH-mm}_{Type}-{kebab-case-name}/requirements.md
.kiro/specs/{YYYY-MM-DD-HH-mm}_{Type}-{kebab-case-name}/design.md
.kiro/specs/{YYYY-MM-DD-HH-mm}_{Type}-{kebab-case-name}/tasks.md
```

## `requirements.md` 應包含

- 文件定位
- 來源 Draft 與來源舊 spec
- 背景
- 目標
- 非目標
- 現有行為
- 新行為
- 使用情境
- 驗收情境
- 驗收條件
- 驗證需求

驗收情境格式：

```text
場景
測試
假設
當
那麼
```

## `design.md` 應包含

- 文件定位
- 已知契約狀態
- Bounded Context
- 設計原則
- 流程或架構圖
- 受影響檔案計畫
- 關鍵行為
- 風險與處理方式

已知契約狀態至少要列：

- API contract
- Data contract
- Permission contract
- UI contract
- 不可假造或不可變更的欄位、狀態、角色、權限

## `tasks.md` 應包含

- `Status: Planned`
- `Execution Context`
- `Protected Behavior`
- 實作任務
- 驗證任務
- 品質檢查清單
- `Implementation Notes`

每個 task 必須有：

```text
Boundary:
Allowed Changes:
Forbidden:
Depends:
Context:
Verify:
```

## 範例：群組讀取權限 Draft promote 後

建議正式目錄：

```text
.kiro/specs/{timestamp}_BugFix-group-based-read-permission/
```

`requirements.md` 重點：

- op_member 讀取事件類型、SLA、Dashboard、ticket metadata options 時，應以群組設定的表單 read 權限為主要依據。
- project membership 不應成為所有 read API 的隱性必要條件。
- 寫入、管理與刪除權限不在本次 bugfix 範圍。

`design.md` 重點：

- middleware policy 如何判斷 form read。
- 哪些 API 只確認 project exists。
- 哪些 API 仍需要 project membership。
- log 中如何呈現 permission rejected 的原因。

`tasks.md` 重點：

- 每個 endpoint 一個 task 或一組同類 task。
- 每個 task 都要有 allow/deny 測試。
- 驗證需包含 `go test` 與 `git diff --check`。

## 範例：DateField Draft promote 後

建議正式目錄：

```text
.kiro/specs/{timestamp}_Feature-datefield-calendar-and-year-selector/
```

`requirements.md` 重點：

- 年份選擇與日曆顯示是同一個 DateField 使用者體驗。
- mobile 與 desktop 的互動需分別驗收。
- 日期輸入輸出格式不可影響後端 API contract。

`design.md` 重點：

- DateField component contract。
- 原生日曆與自製日曆的選擇條件。
- 鍵盤操作、可及性、錯誤狀態與表單整合。

`tasks.md` 重點：

- 元件 contract 先改。
- 再遷移使用頁面。
- 最後補 build、互動測試與視覺檢查。

