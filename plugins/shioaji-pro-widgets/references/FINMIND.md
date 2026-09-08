# FinMind

用途：**Shioaji 拿不到的東西**——三大法人、券商分點、基本面（月營收、財報、股利、
本益比），以及較長期的歷史面板資料。

## 怎麼呼叫

優先使用已安裝的 **FinMind MCP server tools**（`finmind-mcp` plugin），
不要自己組 HTTP 請求：

- `list_datasets` — 不確定用哪個 dataset 時先查
- `get_stock_info` — 公司名 ↔ 股票代號
- `query_dataset` — 主要查詢（`dataset` + `data_id` + `start_date` / `end_date`）
- `query_trading_daily_report` — 券商分點（Sponsor 等級，必填 `data_id` + 單日 `date`）

需要 `FINMIND_TOKEN`（環境變數）；401 就引導使用者去
<https://finmindtrade.com/analysis/#/account/user> 取得，
**永遠不要請使用者把 token 貼進對話**。

沒有安裝該 MCP server 時才退回官方 HTTP API：
base URL `https://api.finmindtrade.com/api/v4`。

## 等級決定能做什麼——先確認等級再規劃

**呼叫 `GET https://api.web.finmindtrade.com/v2/user_info` 就知道當前帳號的等級與額度**，
回傳 `level`、`level_title`、`api_request_limit_hour`、`api_request_limit_day`、
`user_count`，以及 `SponsorInfo` / `SponsorProInfo` 的訂閱區間與額度。
**不要假設等級，當場查。**

| 等級 | 每小時額度 | 官方描述 |
| --- | --- | --- |
| 未帶 token | 300 | — |
| Free（帶 token） | 600 | Basic datasets |
| Backer | 文件未公布數字 | More datasets, higher limits |
| Sponsor | **文件未公布數字**（實測某 SponsorYear 帳號為 **6000／hour、無每日上限**） | Full access to all datasets |
| SponsorPro | 文件未公布 | 文件中唯一明確歸屬它的能力是 `storage_objects` 批量下載 |

官方「Membership Tiers」只給 Free 的 600 這個數字，Backer 與 Sponsor 都只有定性描述。
SponsorPro **沒有**被列入 Membership Tiers 章節，只作為 `storage_objects` 的權限標籤出現。

## Free 等級的關鍵界線

多數個股 dataset 標為
`Free (with data_id) / Backer,Sponsor (all stocks by start_date only)`——
**帶股票代號查單檔免費；不帶代號想撈全市場當日資料要 Backer 以上。**
`TaiwanStockPrice`、`TaiwanStockInstitutionalInvestorsBuySell`、
`TaiwanStockMarginPurchaseShortSale`，以及月營收與財報都是這個模式。

## 常用 dataset

| 意圖 | dataset |
| --- | --- |
| 三大法人買賣超 | `TaiwanStockInstitutionalInvestorsBuySell` |
| 融資融券餘額 | `TaiwanStockMarginPurchaseShortSale` |
| 月營收 | `TaiwanStockMonthRevenue` |
| 財報／EPS | `TaiwanStockFinancialStatements` |
| 股利／配息 | `TaiwanStockDividend` |
| 本益比、股價淨值比 | `TaiwanStockPER` |
| 還原股價（除權息調整） | `TaiwanStockPriceAdj` |

完整清單與欄位以 `list_datasets` 的回傳為準，不要憑記憶列欄位名。

## Sponsor 才有的 dataset

**籌碼與分點**——這是 Sponsor 最有價值的部分，也是 Shioaji 與 TWSE 都給不了的：

| Dataset | 用途 | 更新 |
| --- | --- | --- |
| `TaiwanStockTradingDailyReport` | 券商分點 | 平日 21:00 |
| `TaiwanStockTradingDailyReportSecIdAgg` | 當日券商分點統計 | 平日 21:00 |
| `TaiwanStockWarrantTradingDailyReport` | 權證分點 | — |
| `TaiwanStockGovernmentBankBuySell` | 八大行庫買賣 | 平日 23:30 |
| `TaiwanStockBlockTradingDailyReport` | 鉅額交易買賣日報 | 平日 21:00 |
| `TaiwanStockMarginMaintenance` | 個股融資維持率 | 週一至週六 22:30 |

**分鐘與即時**：`TaiwanStockKBar`（台股分K，平日 15:50）、`TaiwanFuturesKBar`（平日 16:30）、
`TaiwanFuturesSpreadTick`，以及即時報價三件組
`taiwan_stock_tick_snapshot`／`taiwan_futures_snapshot`／`taiwan_options_snapshot`。

**其他**：`TaiwanStockActiveETFHolding`（主動式 ETF 每日持股明細）與
`TaiwanStockActiveETFHoldingChange`（持股異動）、
`TaiwanStockIndustryChainMoneyFlow`（產業鏈資金流向）、
`TaiwanStockLoanCollateralBalance`、`TaiwanStockInfoWithWarrantSummary`、
`TaiwanStockBlockTrade`。

**注意**：逐筆 `TaiwanStockPriceTick`、`TaiwanFuturesTick`、`TaiwanOptionTick`
與 `TaiwanStock10Year` 是 **Backer/Sponsor**，不是 Sponsor 專屬。

**即時報價請優先用 Shioaji**（來源順位 2）。Shioaji 的即時行情不消耗 FinMind 額度、
延遲更低，而且有 market signals 這種現成的異常偵測。

## Sponsor 也擋不掉的限制

**這些與等級無關。**

**單次只能查一天**（用 `date`，不吃 `start_date`／`end_date`）：
`TaiwanStockTradingDailyReport`、`TaiwanStockKBar`、`TaiwanStockPriceTick`、
`TaiwanFuturesTick`、`TaiwanOptionTick`、`TaiwanFuturesSpreadTick`、
`TaiwanStockEvery5SecondsIndex`、`TaiwanStockNews`、
`TaiwanStockIndustryChainMoneyFlow`（且**不可帶 `end_date`**）。

**實務意涵**：回補一年的分點資料就是 250 次以上呼叫，必須寫成分批迴圈。
6000／hour 的額度夠用，但不可能靠一次請求拿到。

**兩個 dataset 有專屬端點，走 `/data?dataset=...` 會回 `422`**：

- `TaiwanStockTradingDailyReport` → `GET /taiwan_stock_trading_daily_report`
- `TaiwanStockTradingDailyReportSecIdAgg` → `GET /taiwan_stock_trading_daily_report_secid_agg`

用 MCP 的 `query_trading_daily_report` 就不會踩到這個坑。

**單次回傳筆數上限**：官方文件未載明，也沒有分頁機制。

## `storage_objects` 批量下載需要 SponsorPro

`GET /api/v4/storage_objects`（參數 `dataset` + `date`，忽略 `data_id`）
回傳整日 parquet 的簽名 URL，支援 6 個 dataset：`TaiwanStockPriceTick`、
`TaiwanStockTradingDailyReport`、`TaiwanStockWarrantTradingDailyReport`、
`TaiwanStockKBar`、`TaiwanFuturesTick`、`TaiwanOptionTick`。

官方文件一律標註 **sponsorpro-tier**。**Sponsor 不等於 SponsorPro**——
用 `user_info` 的 `SponsorProInfo.api_request_limit` 確認，為 `0` 就是沒有，
只能走一般 API 逐日拉。

## 錯誤

| 情況 | 回應 |
| --- | --- |
| 額度用盡 | **HTTP 402**，`{'msg': 'Requests reach the upper limit. https://finmindtrade.com/', 'status': 402}` |
| 走錯端點（該用專屬端點卻用 `/data`） | **HTTP 422**（訊息內容文件未載明） |
| 等級不足 | 官方文件**未載明**專屬錯誤碼或訊息 |

## 更新時間（台灣時間，平日）

| Dataset | 更新 |
| --- | --- |
| `TaiwanStockPriceTick`（逐筆） | 15:30 |
| `TaiwanStockKBar`（分K） | 15:50 |
| `TaiwanStockPrice`（日成交） | 17:30 |
| `TaiwanStockInstitutionalInvestorsBuySell`（三大法人） | 20:00 |
| `TaiwanStockMarginPurchaseShortSale`（融資融券） | 21:00 |
| `TaiwanStockTradingDailyReport`（分點） | 21:00 |
| `TaiwanStockGovernmentBankBuySell`（八大行庫） | 23:30 |
| 月營收、財報 | 官方文件**無 `Update:` 標註** |

**盤中不要用 FinMind 回答「現在多少」**——除了 Sponsor 的即時 snapshot 之外，
其餘最快也要收盤後一小時以上。而即時的部分應該用 Shioaji。

## 實務守則

- **先確認等級再規劃流程。** 同一個 widget 在 Free 與 Sponsor 下的可行路徑完全不同，
  分點類 widget 在 Free 等級根本不成立。
- **先收斂再查明細。** 即使 Sponsor 額度充足，掃描仍應先用 Shioaji 的 market signals
  或使用者自選股收到 3–5 檔，再逐檔查 FinMind。
- **只取需要的日期區間。** 沒指定時預設近三個月，不要一次拉數年。
- **402 就是額度用完**，明說取不到並回傳已取得的部分，不要改用其他來源硬湊同一張表。

事實查證日期：2026-09-08，來源 <https://finmind.github.io/llms-full.txt>
與 `GET /v2/user_info` 實測。
