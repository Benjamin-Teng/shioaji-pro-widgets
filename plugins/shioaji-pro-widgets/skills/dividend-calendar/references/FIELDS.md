# 除權息端點的真實欄位

**全部欄位名都是 2026-09-09 實際打端點取回的第一筆紀錄，不是照文件轉述。**
官方隨時可能改，對不上時以當下回應為準，不要沿用本檔硬拼。

## 三個必踩的坑

1. **日期一律是民國年字串**，例如 `"1150907"` = 2026-09-07、`"1150831"` = 2026-08-31。
   當成西元年解析不會報錯，只會安靜地錯七十幾年。格式是 `yyyMMdd`（年 3 碼）。
2. **TPEx 有三種不同的官方拼字錯誤，而且各端點不一樣。** 照正確英文拼字取欄位
   會拿到 `undefined` / 空值，不會拋錯：
   - `tpex_exright_prepost`：Rights → **`Rrights`**（`ExRrightsExDividendDate`、
     `ExRrightsExDividend`）。此端點的 `CashDividend` 拼字是**正確**的。
   - `tpex_exright_daily`：Dividend → **`Diviend`**（`ExRightsDiviend`、
     `ExRightsDiviendQuote`、`ClosePriceBeforeExRightsDiviend`），
     另有 Dividend → **`Divdend`**（`CashDivdend`、`StockDivdendThousandShares`）。
3. **`tpex_exright_daily` 同時有拼對與拼錯的兩組股利欄位**，數值相同但精度不同
   （`CashDividend` = `"3.200000"`、`CashDivdend` = `"3.20000000"`）。
   固定用拼對的那組，另一組只當交叉檢查，不要兩組混著用。

## TWSE 上市除權除息預告表

`GET https://openapi.twse.com.tw/v1/exchangeReport/TWT48U_ALL`（免驗證、免 key）

| 欄位 | 意義 | 實例 |
| --- | --- | --- |
| `Date` | 除權息交易日（民國年） | `"1150907"` |
| `Code` | 證券代號 | `"00400A"` |
| `Name` | 證券名稱 | `"主動國泰動能高息"` |
| `Exdividend` | 除權/除息別（注意 d 小寫） | `"息"` |
| `CashDividend` | 現金股利 | `"0.120000"` |
| `StockDividendRatio` | 股票股利配股率 | `""` |
| `SubscriptionRatio` | 現增認股率 | `""` |
| `SubscriptionPricePerShare` | 現增認購價 | `""` |

另有 `SharesOffered`、`SharesEmpOwner`、`SharesholderOwner`、`StockHoldingRatio`。
**空字串代表沒有該項配發，不是缺資料**，不要顯示成 0 以外的推測值。

## TPEx 上櫃除權除息預告表

`GET https://www.tpex.org.tw/openapi/v1/tpex_exright_prepost`

| 欄位 | 意義 | 實例 |
| --- | --- | --- |
| `ExRrightsExDividendDate` | 除權息日（民國年，Rrights 拼錯） | `"1150831"` |
| `SecuritiesCompanyCode` | 證券代號（**不叫 Code**） | `"2751"` |
| `CompanyName` | 公司名稱（**不叫 Name**） | `"王座"` |
| `ExRrightsExDividend` | 除權/除息別 | `"除息"` |
| `CashDividend` | 現金股利（此端點拼字正確） | `"1.99664894"` |
| `StockDividendRatio` | 配股率 | `"0.00000000"` |

另有 `SubscriptionRatioToNewSharesIssued`、`SubscriptionPricePerShare`、
`AllocatedForPublicUnderwriting`、`SubscribedByEmployees`、
`SubscribedByExistingShareholders`、`SubscribedProRataInThousandShares`。

## TPEx 上櫃除權除息計算結果表

`GET https://www.tpex.org.tw/openapi/v1/tpex_exright_daily`

只有已經除權息的標的，筆數很少（實測 10 筆）。**日期欄位叫 `Date`，
不是 `ExRrightsExDividendDate`**——與預告表不一致。

| 欄位 | 意義 | 實例 |
| --- | --- | --- |
| `Date` | 除權息日（民國年） | `"1150908"` |
| `SecuritiesCompanyCode` / `CompanyName` | 代號 / 名稱 | `"2743"` / `"山富"` |
| `ExRightsDiviend` | 除權/除息別（Diviend 拼錯） | `"除息"` |
| `ClosePriceBeforeExRightsDiviend` | 除權息前收盤價 | `"59.30"` |
| `ExRightsDiviendQuote` | 除權息參考價 | `"56.10"` |
| `OpeningReferencePrice` | 開盤競價基準 | `"56.10"` |
| `LimitUp` / `LimitDown` | 當日漲/跌停價 | `"61.70"` / `"50.50"` |
| `CashDividend` | 現金股利（拼字正確，優先用這個） | `"3.200000"` |
| `StockDividend` | 股票股利 | `"0.000000"` |
| `StockDividendPlusCashDividend` | 合計 | `"3.200000"` |

另有拼錯的 `CashDivdend`、`StockDivdendThousandShares`，以及
`DividendDeductedQuote`、`CashCapitalIncreaseShares`、`SubscriptionPricePerShare`
與三個認購欄位。

## 資料本身的兩個坑（2026-09-09 實測發現）

1. **預告表不是只有未來的排程。** 實測當天取回的上櫃預告表仍含
   `1150831`（2026-08-31，已過去）的紀錄。以今天為基準分類，
   不要假設預告表＝未來。
2. **金額字串帶浮點雜訊。** 實測台積電 `CashDividend` 回 `"7.000001"`，
   實際配息是 7 元；上櫃也有 `"1.99664894"` 這種長尾數。顯示前四捨五入，
   但不要改寫來源值，也不要說成「精確配息」。

## 呼叫注意

- **TPEx 的必要標頭見共用檔 `references/TPEX_OPENAPI.md`**，不在這裡重複一份。
- 兩邊都**沒有 CORS**，前端不能直連，只能由 widget 側取。
- 回應是 JSON 陣列，可能帶 UTF-8 BOM，用 `utf-8-sig` 解。
- 用回應標頭的 `Last-Modified` 當時間基準：實測 TWSE 為台北時間隔日上午
  （**TWSE 的「今天」其實是昨天**），TPEx 約當日深夜。
- 端點無參數、無分頁，整包回傳（實測 TWSE 79 筆、TPEx 預告 108 筆）。
