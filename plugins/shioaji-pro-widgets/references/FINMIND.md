# FinMind（免費版）

用途：**較長期的歷史面板資料**——月營收、財報、股利、法人買賣超、融資融券、
本益比。這些 Shioaji 不提供或只給短區間，是 FinMind 在本 plugin 的唯一位置。

## 怎麼呼叫

優先使用已安裝的 **FinMind MCP server tools**（`finmind-mcp` plugin），
不要自己組 HTTP 請求：

- `list_datasets` — 不確定用哪個 dataset 時先查
- `get_stock_info` — 公司名 ↔ 股票代號
- `query_dataset` — 主要查詢（`dataset` + `data_id` + `start_date` / `end_date`）

沒有安裝該 MCP server 時，才退回官方 HTTP API，並先向使用者確認 token 來源。
需要 `FINMIND_TOKEN`（環境變數）；401 就引導使用者去會員中心取得，
**永遠不要請使用者把 token 貼進對話**。

## Widget 常用 dataset

| 意圖 | dataset |
| --- | --- |
| 月營收 | `TaiwanStockMonthRevenue` |
| 三大法人買賣超 | `TaiwanStockInstitutionalInvestorsBuySell` |
| 融資融券餘額 | `TaiwanStockMarginPurchaseShortSale` |
| 股利／配息 | `TaiwanStockDividend` |
| 本益比、股價淨值比 | `TaiwanStockPER` |
| 還原股價（除權息調整） | `TaiwanStockPriceAdj` |
| 財報／EPS | `TaiwanStockFinancialStatements` |

完整清單與欄位以 `list_datasets` 的回傳為準，不要憑記憶列欄位名。

## 免費版限制（已查證，來源 <https://finmind.github.io/llms-full.txt>）

| 項目 | 事實 |
| --- | --- |
| 未帶 token | 300 requests / hour |
| 免費會員帶 token | 600 requests / hour |
| 超過額度 | HTTP **402**，`{'msg': 'Requests reach the upper limit. https://finmindtrade.com/', 'status': 402}` |
| 查目前用量 | `GET https://api.web.finmindtrade.com/v2/user_info`，看 `user_count` 與 `api_request_limit` |
| Base URL | `https://api.finmindtrade.com/api/v4` |
| 取得 token | <https://finmindtrade.com/analysis/#/account/user> |

**免費版最重要的那條界線**：多數個股 dataset 標的是
`Free (with data_id) / Backer,Sponsor (all stocks by start_date only)`——
**帶股票代號查單一檔是免費的；不帶代號想一次撈全市場當日資料要付費等級。**
`TaiwanStockPrice`、`TaiwanStockInstitutionalInvestorsBuySell`、
`TaiwanStockMarginPurchaseShortSale` 都是這個模式。

付費才有的（免費版直接放棄，不要嘗試）：分點資料
`TaiwanStockTradingDailyReport`（Sponsor）、逐筆 `TaiwanStockPriceTick`
（Backer/Sponsor）、`TaiwanStock10Year`（Backer/Sponsor）。
各 dataset 的等級以官方文件的 `Tier:` 標註為準。

## 更新時間（台灣時間，平日）

| Dataset | 更新 |
| --- | --- |
| `TaiwanStockPrice` | 17:30 |
| `TaiwanStockInstitutionalInvestorsBuySell` | 20:00 |
| `TaiwanStockMarginPurchaseShortSale` | 21:00 |

**盤中不要用 FinMind 回答「現在多少」**——它最快也要收盤後一個半小時才有今天的資料。

## 實務守則

- **先收斂再查明細。** 免費版不能撈全市場，所以掃描一定要先用 Shioaji 的 scanner
  或使用者自選股收到 3–5 檔，再逐檔帶 `data_id` 去 FinMind。
- **只取需要的日期區間。** 沒指定時預設近三個月，不要一次拉數年。
- **402 就是額度用完**，明說取不到並回傳已取得的部分，不要改用其他來源硬湊同一張表。
