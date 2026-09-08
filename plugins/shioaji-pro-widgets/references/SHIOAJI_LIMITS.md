# Shioaji API 能力邊界

基準版本 **1.7.4**（本機官方 skill references 逐字查證，2026-09-08）。
設計 widget 前先讀這份，可以省下大量「試了才知道做不到」的時間。

## 硬性數字（1.5.7 → 1.7.4 全部未變）

這批是永豐後端與交易所側的規則，**不隨套件版本改變**，不要期待升級解決。

| 項目 | 值 |
| --- | --- |
| 每人連線數 | **5 條**（一個 `login()` 行程＝一格；daemon 模式所有 client 共用一格） |
| 每日登入次數 | 1000 次 |
| 行情查詢頻率 | 50 次 / 5 秒（`snapshots`／`ticks`／`kbars`／`credit_enquires`／`short_stock_sources` 共用） |
| 帳務查詢頻率 | 25 次 / 5 秒 |
| 委託操作頻率 | 250 次 / 10 秒 |
| 盤中 `ticks` 查詢 | **10 次** |
| 盤中 `kbars` 查詢 | **270 次** |
| `kbars` 單次區間 | **30 個 calendar days**（超過回 `400`；實作建議切 29 天） |
| `snapshots` 單次 | **500 檔合約** |
| 歷史起始日 | 指數／股票 **2020-03-02**；期貨 **2020-03-22** |
| 每日流量額度 | 500MB–10GB（依交易量），每開盤日 **08:00 重置**；耗盡時 `ticks`／`kbars`／`snapshots` 靜默回空 |
| SSE 心跳 | 30 秒 |

**即時行情訂閱不消耗每日流量額度。** realtime KBar／enriched／signals 訂閱依此通用規則
推論同樣不消耗，但**文件未逐字明載**，別當成保證。

## 拿不到什麼（1.7.4 重新確認，沒有鬆動）

對全部 references（含四個新檔）逐一 grep 確認，以下 **Shioaji 一概不提供**：

- 財報、月營收、股利、本益比等**基本面**
- **三大法人買賣超**（外資／投信／自營商）
- **券商分點進出**
- **除權息**資訊
- 新聞、公司重大訊息公告（`regulatory_punish`／`regulatory_notice` 只有處置股與注意股）
- **美股／港股**（`SKILL.md` frontmatter 明文排除）

需要以上任一項，請走 `DATA_SOURCES.md` 的順位 3／4。

**特別澄清**：1.7.4 新增的 enriched index data **不是**法人籌碼資料。
它是從指數與成分股行情**運算而來**的市場結構衍生值（貢獻度、權重、排行），
完全沒有法人身分維度。不要拿它當「三大法人」的替代。

## 1.7.4 新能力（對 widget 有用的部分）

### Market Signals

**這是掃描類 widget 的最佳來源**，純 Shioaji、即時、不用碰公開資料。

- 範圍封閉：只支援**台股股票**（`region=TW, security_type=Stock, exchange=TSE|OTC`），
  三參數皆必填。
- 規則是**寫死的封閉集合，門檻不可調**：
  - `LimitScanner`：買方接近／觸及漲停、漲停打開，及跌停對應共 6 種
  - `PriceMoveScanner`：`trade_surge`／`trade_drop`（1 秒內漲跌 **>1% 且 ≥3 檔**，
    1 秒冷卻）、`bid_surge`／`ask_drop`
  - `VolumeScanner.burst()`：單筆成交金額超過**當日門檻**（伺服器每日重算，
    `extra.threshold` 只能讀不能設），5 秒冷卻
- `StreamScanner.Simtrade` 是**狀態過濾**不是規則：推播處於模擬撮合的個股報價，
  **不限開收盤集合競價**——處置股分盤、穩定措施期間的試撮也會推。
  `StreamScanner.Suspend` 推暫停交易個股。這兩者回 `ScannerQuoteEvent`（無 `extra`），
  規則類回 `ScannerSignalEvent`（有 `extra`）。
- 所有訊號**共用同一個全域 callback**，要自行依事件類別分派；未設 callback 會直接印出。
- **唯一有斷線補報的串流頻道**：`ScannerGapEvent` 帶 `dropped_count`／`first_time`／
  `last_time`。

### Enriched Index Data

- **只支援兩個指數**：加權 `IX0001`（TSE）、櫃買 `IX0043`（OTC）。其他指數在送出前就被拒。
- 四種能力：自算指數（每秒多次）、指數貢獻、產業貢獻（各 1 秒）、
  成分股排行與產業群組投影（市場／群組層 1 秒，群組內排行 5 秒）。
- `api.index_components(index)` 是**一次性權威快照**（全部成分股＋產業群組），
  **明文禁止持續輪詢**；`429` 代表當日資料用量額度耗盡（額度數字文件未載明）。
- 排行組合是**封閉矩陣**，不可自訂排序或筆數。例如 `Contribution`／`PctChange` 只有
  `Desc/10`、`AbsDesc/10`、`PositiveDesc/25`、`NegativeAsc/25` 四種；
  `Weight`／`Amount` 只有 `Desc/10`；**`AmountShare` 完全不推播**。
  不符合矩陣的組合直接 `ValueError`（Python）或 `400`（HTTP）。
- 每個事件是該 (index, projection) 的**完整替換**——沒有 delta、沒有序號、沒有斷線重播。
  重連或有 gap 時要重呼叫 `index_components()` 取權威狀態整個覆蓋，不要等補播。

### Realtime KBar

- **伺服器推送，不是本地聚合**：`quote_type=sj.QuoteType.KBar`，每分鐘收線推一根完整 K 棒。
- **只有 1 分鐘**一種粒度；**只有股票**（指數／期貨／選擇權尚不支援）；
  不支援盤中零股與 `version` 參數；CLI 不支援（只能 Python 或 HTTP/SSE）。
- **`kbar.time` 是起始時間（左標）**，與歷史 `api.kbars()` 的**右標**語意相反。混用必錯。
- 走**專屬路由** `POST /api/v1/stream/subscribe/kbars`（body `{"stocks":[...]}`，
  回應**沒有** `subscription` 欄位），不是通用的 `/stream/subscribe`。

### Contract V2

**破壞性變更**：`login()` **移除了** `fetch_contract`／`contracts_timeout`／`contracts_cb`
三個參數，登入不再下載合約。改為 `api.contracts.get()`／`.info()`／`.list()` 等按需
lazy 載入，第一次存取才下載該型別或 shard。

- 「啟動要等合約下載完」這個老問題**被架構性解除**，不是調參數解決。
- **沒有** preload、reload 或 readiness 端點；查詢會自行等待所需的那一小塊資料。
- 合約變更事件由 SDK 內部自動處理（標記 dirty、下次存取自動刷新），
  **應用程式不應自行安裝 contract callback 或手動 reload**。
  HTTP/CLI 端的 `contract_event` 只是**變更訊號不是資料本體**，收到後要重打 GET。
- 舊索引方式失效：**`TSE001` 已無效，加權指數一律用 `IX0001`**；
  不可用 `api.Contracts.Stocks.TSE.TSE2330` 這種帶前綴的存取。
  舊 facade 仍可用但會發 `DeprecationWarning`。
- **新欄位對 widget 很有用**：`StockInfo` 新增券賣資格
  `margin_shortable`／`sbl_shortable`／`below_ref_shortable`（官方每日名單，
  `TSE`／`OTC` 才有效，`OES` 一律 `False`），以及處置股相關
  `disposition_level`／`disposition_match_interval_min`／
  `disposition_max_lots_single_order`／`disposition_max_lots_total_orders`／
  `disposition_prepay_ratio`、`attention_flag`、`etf_constituent`。

## 交易面邊界

### 條件單／停損單：沒有原生支援

`ADVANCED.md` 的 Stop Orders 是**本地輪詢自建**——訂 tick、在 `on_tick` 裡逐筆比價、
觸發才呼叫 `place_order()`。1.7.4 與 1.5.7 逐字相同，沒有伺服器端觸價 API，
`StockOrder`／`FuturesOrder` 也沒有 `stop_price` 類欄位。

**意涵**：任何「守價」邏輯的狀態都活在自己的行程記憶體裡，**斷線或重啟就消失**。
文件並建議自行維護 `executed=True` 之類的旗標防止連續 tick 重複送單。

### 冪等：訂單級仍不存在

`place_order` 沒有 `client_order_id` 或 `idempotency_key` 參數。
新增的 Agent Harness 是**授權層不是去重層**——受保護操作需帶一次性 capability
token 綁定精確 body bytes、伺服器原子性消耗 nonce，但官方在「Current scope boundary」
**明文承認尚未綁定 idempotency key**。且預設 `off`。

**唯一安全規則**：逾時或「處理中」時**先 `update_status()` 對帳，絕不盲目重送**。
重複下單比漏單更糟。

### 委託狀態與非終局訊號

狀態值：`PendingSubmit`／`PreSubmitted`／`Submitted`／`Failed`／`Cancelled`／
`Filled`／`PartFilled`。

- **`PendingSubmit` 是正常中間狀態，不是交易所最終確認**，不可據此宣稱已送出或失敗。
- **空列表不是失敗訊號**（可能是無資料、模擬模式、或帳號選錯）。
- 成交回報可能**比委託回報更早到達**，不可假設順序。
- 優先等 `order_deal_event` 回報；`update_status()` 只在回報遺漏、斷線重連或對帳時用。
- 非阻塞模式（`timeout=0`）立即回傳的 `Trade` 是 **placeholder**，
  `order.id`／`seqno`／`ordno` 可能是空字串，不可拿來做刪單或風控。

### 組合單

- **恰好 2 腿**，`combo_type` 6 個值（`PriceSpread`／`TimeSpread`／`Straddle`／
  `Strangle`／`ConversionReversal`／`WeeklyTimeSpread`）。
- **模擬環境完全不支援**下單與刪單。
- TAIFEX 標準選擇權組合單**不支援 ROD**（`LMT+ROD` 被 `9927` 退），
  連續交易時段須 `LMT+IOC` 或 `LMT+FOK`；開盤前不接受組合單／跨月價差／FOK／範圍市價單。
- 1.7.4 新增 Managed 模式（`api.contracts.combo(legs=...)` 自動推導腿序與 `combo_type`）
  與**組合合約本身的行情訂閱**；與舊 Directed／`ComboBase` 模式**不可混用**。

### 其他委託規則

- 期貨／選擇權 **MKT 不可配 ROD**（`op_code 9938`），只能 `IOC`／`FOK`。
- 改單**只能減量不能增量**；**盤中零股只能減量不能改價**。
- `custom_field` 備註**最多 6 字元**。

## 錯誤判讀

錯誤分四種形式：例外／`trade.status.msg`／`op_code`／**靜默失敗**。

**最容易踩的是靜默失敗**：流量額度耗盡時 `ticks`／`kbars`／`snapshots`
**回空資料、不拋例外、沒有錯誤碼**。查空值時第一個要懷疑的是額度，
先看 `api.usage()` 或 `GET /api/v1/auth/usage`。

| 情況 | 碼 | 該怎麼辦 |
| --- | --- | --- |
| 查詢區間過長 | `400` | **改參數**，縮小區間 |
| 查無資料 | `404` | 多半是**合法查無**，先確認日期／代碼 |
| 連線數超限 | `451 Too Many Connections` | **自行處理**：`logout()` 或關掉殘留行程 |
| 頻率超限／被限流 | `503`（`操作異常，請1分鐘後再重新登入`） | **等待重試**，停止重試迴圈等 1 分鐘 |
| 系統忙線 | `503 Service is unavailable` | **等待重試** |
| 下單中台逾時 | `408 STS Request Timeout` | **狀態不確定**：先 `update_status()` 對帳，**不要重送** |
| 非開放時段查交易額度 | `4xx` | 預期行為，等 08:30–15:00 |

**`op_code` 自成一套代碼空間**——`"00"` 成功、其餘失敗看 `op_msg`。
文件明文警告**不要與其他章節交叉查表**；TAIFEX 的 `9938`／`9927` 屬於另一套，
不在錯誤碼對照表內。

本地 `shioaji server` 的 `401`（`Missing Authorization header` 等）是
client↔daemon 的 bearer 認證失敗，**與後端永豐 Token 無關**，不要查登入錯誤表。

## 模擬 vs 正式

**行情面完全相同**：`subscribe`／`snapshots`／`ticks`／`kbars`／`credit_enquires`／
`scanners`／`Contracts` 在模擬環境完全可用，是真實市場行情。

**帳務面在模擬環境回空值或預設值**（不代表真實資金）：`account_balance`、`margin`、
`settlements`、`trading_limits`、`list_position_detail`、`list_profit_loss_detail`、
`list_profit_loss_summary`、`order_deal_records`、預收相關。
但 `list_positions()` 與 `list_profit_loss()` 走模擬端點、**會回真實模擬數據**。

**組合單在模擬環境完全不可用**（會報錯）。

**模式綁在行程上**：模擬／正式在啟動或 `login()` 時決定，daemon 整個生命週期不可切換，
要換必須重啟。

## 時間與資料正確性的坑

- **行情資料的 `ts` 解出來就是台灣牆鐘時間，不可再 +8 小時**（會把 09:01 錯移到 17:01）。
  此規則**只適用行情**，不可外推到委託／成交／帳務的 timestamp。
- **期貨交易日歸屬**：`ticks(contract, date=D)` 涵蓋「前一交易日 15:00 夜盤」到
  「D 13:45 日盤收盤」，**夜盤掛在下一個交易日**。
- **歷史 `kbars` 是右標**（`[ts-1分鐘, ts)`），**realtime KBar 是左標**。兩者混用必錯。
- 用 tick 自聚合時，精確等於收盤時刻的 tick 要**內縮 1 微秒**才會歸進收盤那根。
- `kbars` 可能含**零成交量的 carry-forward**，純 tick 聚合無法重現。
- **Python 屬性名 ≠ HTTP 欄位名**：Python 用 `ts`，HTTP 用 `datetime`。
  SSE 的價格與金額欄位序列化為**字串**（伺服器用 Decimal），成交量類才是數字。
