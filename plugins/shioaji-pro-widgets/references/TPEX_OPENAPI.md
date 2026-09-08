# TPEx Open API（櫃買中心）

用途：**上櫃／興櫃市場的公開盤後統計**，且 Shioaji Pro 原生工具與 Shioaji API
都拿不到時才用。以下事實實測自官方 swagger 與 HTTP 回應（2026-09-08）。

- 文件頁：<https://www.tpex.org.tw/openapi/>
- 規格檔：`https://www.tpex.org.tw/openapi/swagger.json`（OpenAPI 3.0.0，225 個端點，全部 GET）
- **Base URL**：`https://www.tpex.org.tw/openapi/v1`
- **免驗證**：規格內沒有 `securitySchemes`，端點也沒有任何認證參數，實測直接呼叫回 200。

## 五個一定要先知道的限制

1. **只有上櫃／興櫃。** 台積電 2330、鴻海 2317 這些**上市股票不在這裡**。
   使用者問上市個股，這個來源直接不適用，不要硬找。
2. **沒有任何查詢參數。** 225 個端點全都不吃股票代號或日期，
   每次呼叫回傳當日**全市場一整包**（`tpex_mainboard_quotes` 單次約 357 KB），
   要哪一檔請在自己這端過濾。
3. **盤後日更。** 實測 `Last-Modified` 落在台北時間當天 22:00 前後。
   **盤中不要用它回答「現在多少」。**
4. **沒有 CORS。** 回應不含任何 `Access-Control-*` 標頭，`OPTIONS` 回 405。
   瀏覽器端跨網域 fetch 會被擋；只能從伺服器端呼叫。
5. **瀏覽器等級標頭是必要的，資料端點也一樣**（2026-09-09 實測）：帶完整
   `User-Agent`（含 `Mozilla/5.0 ... Chrome/...`）並加
   `Referer: https://www.tpex.org.tw/`，否則會拿到 `403` 或 `520`。
   只給 `-A "Mozilla/5.0"` 這種簡短 UA 實測仍可能 520。抓 `swagger.json` 同樣適用。

## 常用端點

| 用途 | 路徑 |
| --- | --- |
| 上櫃收盤行情 | `/tpex_mainboard_quotes` |
| 上櫃行情（多均價、次日參考價／漲跌停） | `/tpex_mainboard_daily_close_quotes` |
| 融資融券餘額 | `/tpex_mainboard_margin_balance` |
| 三大法人個股買賣明細 | `/tpex_3insti_daily_trading` |
| 三大法人全市場彙總 | `/tpex_3insti_summary` |
| 注意股票 | `/tpex_trading_warning_information` |
| 處置有價證券 | `/tpex_disposal_information` |
| 除權息計算結果 | `/tpex_exright_daily` |
| 除權息預告 | `/tpex_exright_prepost` |
| 大盤日成交量值與指數 | `/tpex_daily_trading_index` |

命名與直覺相反：`tpex_mainboard_daily_close_quotes` 欄位反而比
`tpex_mainboard_quotes` 多（含 `Average`、`NextReferencePrice`、
`NextLimitUp/Down`）。用之前先跑一次確認欄位。

## 欄位名的坑（官方原文如此，不是筆誤）

**一律用 swagger 定義的確切字串取值，不要 `.strip()` 後猜 key。**

- `tpex_3insti_daily_trading` 有 key **開頭帶空白**
  （`" Foreign Investors include Mainland Area Investors (Foreign Dealers excluded)-Total Sell"`），
  也有 key **字中多空白**（`"Dealers -TotalSell"`，代表自營商避險賣出）。
- `tpex_exright_daily` / `tpex_exright_prepost` 官方把 Rights 拼成 `Rrights`、
  Dividend 拼成 `Divdend`：`ExRrightsExDividendDate`、`CashDivdend`。
- 日期欄位是**民國年字串**，例如 `"1150908"` = 2026-09-08。轉換時不要當西元。

## 格式

同一端點同時支援 `application/json` 與 `text/csv`，用 `Accept` 標頭選擇；
CSV 的表頭是中文。兩者都帶 `Content-Disposition: attachment`。
**沒有分頁**，規格未定義 `limit`／`offset`／`page`／`cursor`。

## 使用條款

官方站台通用使用條款第五條禁止未經同意的自動化擷取，第七條說明已授權
政府資料開放平臺（<http://data.gov.tw>）提供公眾使用者不在此限。
**沒有查到任何明文的呼叫頻率上限數字。** 因此 widget 一律節制：
同一次對話對同一端點只打一次，取回後在記憶體內過濾重用，不要逐檔重打。
