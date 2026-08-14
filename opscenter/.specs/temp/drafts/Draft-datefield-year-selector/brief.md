# Draft: DateField Year Selector

Status: Draft

## Source

- `.kiro/specs/2026-07-23_10-00_datefield-year-selector`

## Intent

重新規劃 DateField 年份選擇、日期輸入格式、鍵盤操作、瀏覽器相容與表單整合。

## Bounded Context

包含：

- DateField component
- Year selector UX
- Input/output value contract
- Existing form migration

不包含：

- 所有日期時間元件
- 後端日期儲存格式重寫

## Promotion Notes

Promote 時應與 Native DateField Calendar Draft 合併判斷，避免同一元件有兩套互斥規劃。

