# TWSE Open API（證交所）

**上市市場公開資料的主要來源。** 免驗證、免註冊、授權明確，比 TPEx 穩定，
且涵蓋一般人真正在看的股票（2330、2317 都在這裡）。
以下事實實測自官方 swagger 與 HTTP 回應（2026-09-08）。

- 文件頁：<https://openapi.twse.com.tw/>
- 規格檔：`https://openapi.twse.com.tw/v1/swagger.json`（**Swagger 2.0**，143 個端點，全部 GET）
- **Base URL**：`https://openapi.twse.com.tw/v1`
  （規格是 Swagger 2.0，用 `host` + `basePath` + `schemes`，**沒有 `servers` 欄位**）
- **免 API key、免註冊**：`securityDefinitions` 與 `security` 皆不存在，實測匿名 GET 回 200。
- **授權明確**：政府資料開放授權條款第 1 版（<https://data.gov.tw/license>），
  允許商業使用、無頻率限制，義務是**顯名標示資料來源機關**，未標示視為自始未取得授權。

## 六個一定要先知道的限制

1. **沒有任何查詢參數。** 143 個端點的 `parameters` 掃描結果為 0——不吃股票代號、
   不吃日期。每次回傳全市場整包（`STOCK_DAY_ALL` 約 320 KB），自己在本地過濾。
2. **沒有分頁。** 單一 JSON 陣列，沒有 `page` / `cursor` / `next`。
3. **盤後批次更新，不是即時。** 實測 `Last-Modified` 落在台北時間上午 9 點多，
   內容是**前一交易日**。盤中不要用它回答「現在多少」。
4. **資料端點沒有 CORS。** swagger.json 本身有 `Access-Control-Allow-Origin: *`，
   **但資料端點完全沒有任何 `Access-Control-*` 標頭**——很容易誤以為整個平台對前端開放。
   瀏覽器跨源直連會被擋，只能從伺服器端呼叫。
5. **沒有三大法人買賣超。** 對全部 143 個端點的 path/summary/description/tags 做關鍵字掃描，
   「三大法人」「外資」「投信」「自營商」「買賣超」**一次都沒有出現**。
   這與許多教學文章的說法不同。法人資料請走 FinMind，見 `FINMIND.md`。
6. 只支援 HTTPS（`http://` 會 301 轉址）；不需要特定 User-Agent。

## 常用端點

| 用途 | 路徑 | key 語言 |
| --- | --- | --- |
| 個股日成交資訊 | `/exchangeReport/STOCK_DAY_ALL` | 英 |
| 大盤每日市場成交 | `/exchangeReport/FMTQIK` | 英 |
| 大盤收盤指數 | `/exchangeReport/MI_INDEX` | **中** |
| 大盤加權指數歷史 | `/indicesReport/MI_5MINS_HIST` | 英 |
| 融資融券餘額 | `/exchangeReport/MI_MARGN` | **中** |
| 當日沖銷交易標的 | `/exchangeReport/TWTB4U` | 英 |
| 當日可借券賣出股數 | `/SBL/TWT96U` | 英 |
| 暫停交易證券 | `/exchangeReport/TWTAWU` | 英 |
| 注意股票 | `/announcement/notice` | 英 |
| 處置股票 | `/announcement/punish` | 英 |
| 除權除息預告 | `/exchangeReport/TWT48U_ALL` | 英 |
| 月營收（MOPS 資料） | `/opendata/t187ap05_L` | **中** |
| 資產負債表 | `/opendata/t187ap07_X_ci`、`t187ap07_X_mim` | **中** |
| 綜合損益表 | `/opendata/t187ap06_X_*`（依業別分） | **中** |

## 資料格式的坑（每一條都會咬人）

**日期格式至少三種並存，跨端點絕不能共用同一個 parser：**

| 形式 | 例 | 出現在 |
| --- | --- | --- |
| 民國年無分隔 `yyyMMdd` | `"1150907"` = 2026-09-07 | 多數 `exchangeReport` |
| **西元**年無分隔 `yyyyMMdd` | `"19620209"` = 1962-02-09 | `t187ap03_L` 的上市／成立日期 |
| 民國年帶斜線 `yyy/MM/dd` | `"115/09/01"` | `company/suspendListingCsvAndHtml` |

另外 `t187ap05_L` 的 `資料年月` 是 5 碼 `"11507"`（民國 115 年 07 月）。

**其他坑：**

- **全部欄位型別都是 string**，包含金額、股數、指數。缺值是空字串 `""`，不是 `null` 或 `0`。
- **中英文 key 混用**。`MI_INDEX`、`MI_MARGN`、`opendata/*` 用中文 key，
  還含全形括號與百分比符號，例如 `"營業收入-上月比較增減(%)"`。
- **同語意欄位跨端點結構不同**：漲跌在 `MI_INDEX` 拆成 `"漲跌":"+"` 與
  `"漲跌點數":"860.80"` 兩欄；在 `FMTQIK` 卻是 `"Change":"-784.00"` 符號內嵌。
- **命名與內容不符**：`/company/suspendListingCsvAndHtml` 名字說 CSV+HTML，
  **實際回傳 JSON**。
- **欄位名字義不符預期**：`TWTB4U` 只有 `Suspension`（暫停現股當沖註記），
  **沒有沖銷量或沖銷比率**；`SBL/TWT96U` 給的是「可借券額度上限」，
  **不是借券賣出成交量**。
- **舊系統遺留字串**：`t187ap03_L` 的 `普通股每股面額` 是
  `"新台幣                 10.0000元"` 這種固定寬度字串，數值要用正規表示式挖，
  不能直接當數字解析；`外國企業註冊地國` 可能是全形破折號 `"－ "`。
- 百分比欄位不四捨五入，例如 `"2.70047776585692"`。
- 回應帶 `Content-Disposition: attachment`，直接用瀏覽器網址列開會觸發下載對話框
  （fetch / 後端呼叫不受影響）。

## 快取與節制

回應**沒有 `Cache-Control`**，但有 `ETag` 與 `Last-Modified`。
既然是每日批次更新，同一次對話對同一端點只打一次；需要重複使用就在記憶體內留著。
使用條款未載明任何明文頻率限制數字，實測連續 20 次未觸發 429——
**這不代表沒有隱性限制**，仍要節制。

引用資料時記得依授權條款標示來源機關（臺灣證券交易所）。
