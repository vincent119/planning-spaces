# Review Notes

Status: Draft

## 前一版看起來怪的原因

前一版把舊 spec 幾乎一對一轉成 `Draft-*` 目錄，問題是：

1. 太像檔案整理，不像新版 SDD 的重新規劃。
2. 沒有先說明「為什麼要 Draft」，也沒有說 promote 後正式 spec 的樣子。
3. `DateField` 被拆成兩個 Draft，但實際上它們可能應該是同一個元件契約。
4. 近期 debug fix 被放在同一層，但沒有清楚說明是 bugfix trace。
5. 每個 brief 太薄，讀起來像索引，不像可審閱的規劃。

## 本版修正

本版改成審閱導向：

- 先說明重新規劃方法。
- 再列出每個功能的 Draft 對應。
- 再描述 Draft promote 後正式 spec 的文件形狀。
- 把看起來應該合併的功能先合併成 Draft。
- 把 bugfix trace 與大型 feature 分開看。

## 待確認

- `DateField` 是否同意合併成一個 `Draft-datefield`。
- `structured-access-logging` 與 `session-idle-timeout` 是否要納入舊設計重新規劃，還是只保留在近期修正文件。
- 已接近完成的舊功能，例如 Import Ticket Tools，是否只補 `Docs` spec，不重新建立完整 Feature spec。
- 權限類是否要優先 promote 成正式 `BugFix` spec，作為後續 SLA、Dashboard、Ticket metadata 的共同依據。

