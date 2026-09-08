# Widget Backlog

## 這個 repo 在做什麼

面向**初次接觸 Shioaji Pro 的大眾使用者**的體驗品——讓人快速感受「原來這東西可以這樣擴充」。
不是進階玩家的自用裝備。這條定位會覆寫「痛點多痛」的直覺排序：新手在感受到價值之前就被
設定手續勸退的話，痛點再痛也沒有用。

## 收錄門檻（三條都要過）

1. **一個 widget 一個痛點。**
2. **裝得下 MCP schema 表達不了的東西**——呼叫順序、語意陷阱、生命週期、輸出紀律。
   只是把 schema 已經講清楚的事再講一次，不算 widget（見文末「已評估後捨棄」）。
3. **對新手可達**：不要求註冊外部帳號或付費等級、不預設使用者有持倉、不需要綁憑證。
   過不了這條的不是被砍，是往後排。

## 梯隊

| 梯隊 | 判準 |
| --- | --- |
| A | 動到 Shioaji Pro 本身。展示的是 Pro 的能力，最貼題 |
| B | 零設定的公開資料。免 API key、免註冊，輸入一個代號就有輸出 |
| C | 有門檻（缺工具名、或要外部帳號），但仍是好的展示題 |
| D | 帳務類。技術上可行後仍有「新手空手＝空畫面」的問題，不該當展示主線 |

| 梯隊 | Widget | 痛點 | 需要什麼 | 狀態 |
| --- | --- | --- | --- | --- |
| A | [`workspace-conductor`](#1-workspace-conductor-版面指揮官) | 每天手動排版面 | `ui.control` | ready |
| A | [`backtest-doctor`](#2-backtest-doctor-回測體檢) | 回測漂亮、實盤打臉 | `ui.control` | 已寫，待實機驗證 |
| B | [`dividend-calendar`](#3-dividend-calendar-除權息行事曆) | 哪天除權息、配多少 | TWSE + TPEx 公開 | ready |
| B | [`monthly-revenue-digest`](#4-monthly-revenue-digest-月營收速讀) | 每月營收看不完 | TWSE + TPEx 公開 | ready |
| C | [`order-preflight`](#5-order-preflight-下單前健檢) | 掛單撞漲跌停或流動性沙漠 | `trade.preview` + `market.read` | blocked |
| C | [`whos-weird-today`](#6-whos-weird-today-今天誰不對勁) | 排行榜看不完 | `market.read` | blocked |
| C | [`credit-sentinel`](#7-credit-sentinel-券資哨兵) | 融券被軋、券源斷掉才知道 | `market.read` | blocked |
| C | [`institutional-flow`](#8-institutional-flow-法人動向) | 法人在買什麼賣什麼 | FinMind token + TPEx | 降級 |
| D | [`morning-brief`](#9-morning-brief-盤前三分鐘) | 開盤前不知道該盯什麼 | `account.read` + `market.read` | blocked |
| D | [`position-attribution`](#10-position-attribution-今天賺賠是誰害的) | 帳戶紅綠說不出原因 | `account.read` + `market.read` | blocked |
| D | [`settlement-alarm`](#11-settlement-alarm-交割款鬧鐘) | T+2 忘了補錢 | `account.read` | blocked |
| D | [`closing-journal`](#12-closing-journal-收盤複盤日記) | 從不寫交易日誌 | `account.read` | blocked |

## 共同阻塞點

C 與 D 梯隊的 `blocked` 全部卡在同一件事：**`market.*` 與 `account.*` 家族的 v1 工具名
不在任何 repo 裡**，必須對連線中的 Shioaji Pro MCP server 讀一次 advertised schema
才能定案。在那之前**不要照猜工具名**——見 `references/APP_TOOLS.md`。

**A 與 B 梯隊不受這件事影響**，隨時可以動手。

Shioaji API 本身能做到哪、做不到哪，見 `references/SHIOAJI_LIMITS.md`（基準 1.7.4）。
**三件對設計影響最大的事**：條件單／停損單**沒有原生支援**（是本地輪詢，斷線就沒了）、
訂單級**冪等不存在**（逾時要對帳不能重送）、財報／法人／分點／除權息 Shioaji **一律沒有**。

## 公開資料的共用護欄

B 梯隊與任何動到公開來源的 widget 都適用，細節見 `references/DATA_SOURCES.md`：

- **一個問題只用一個來源算數字。** 同時拿 Shioaji 盤中快照與 TWSE 盤後收盤價講「今天漲幅」
  是本 plugin 最常見的錯誤，會產生對不起來的數字。跨來源時標明各自的時間基準。
- **公開來源全是盤後日更。** 各家時點不同：FinMind 三大法人 20:00、融資券 21:00；
  TPEx 約 22:00；**TWSE 要隔日上午 9 點多才有前一交易日內容——TWSE 的「今天」其實是昨天。**
- **TWSE 與 TPEx 的端點完全不吃參數**，每次回傳全市場整包、沒有分頁、沒有 CORS。
  收斂到使用者關心的那幾檔是 widget 的工作。
- **帳務只能走 Shioaji Pro 原生工具**，任何情況都不可用公開資料推測持倉。

下面每一節的「用到什麼」欄位中，**粗體**的是已在 `references/APP_TOOLS.md` 有明文的 v1
工具名；非粗體的是意圖描述，實作前要對 schema 確認。

---

## 1. `workspace-conductor` 版面指揮官

**梯隊 A**。**痛點**：每天開盤前手動拖面板、換標的、重排版面，同樣的動作重複一輩子。

**產出**：一句話把工作區排成這次要盯的樣子，回報主標的、面板清單、哪些被釘住。

**用到什麼**：**`get_app_state`**、**`list_panels`**、**`apply_layout`**、
**`select_contract`**、**`add_panel`**／**`remove_panel`**、**`set_panel_pin`**。

**為什麼排第一**：畫面當場自己動起來，新手第一眼就懂發生了什麼事，而且展示的是 Pro
自己的能力，不是外部資料好用。體驗品的最佳開場。

**風險**：版面變更必須序列化，不能並行；使用者說「我平常那個版面」時要列出
可用版面請他指認，不要猜。

---

## 2. `backtest-doctor` 回測體檢

**梯隊 A**。**痛點**：回測曲線很漂亮，實盤一上就打臉，但說不出到底哪裡不對。

**產出**：三行體檢結論——樣本數夠不夠、最大回撤與集中度、有沒有過度擬合的跡象。

**用到什麼**：**`get_backtest_result`** →（需要逐檔比較才呼叫）
**`list_backtest_symbol_results`** →（需要看個別交易才呼叫）**`get_backtest_trades`**。
**一定要照這個順序漸進讀取**，不要把整包結果倒進回應。

**為什麼排第二**：只吃 `ui.control`，零外部依賴，現在就能寫。講人話——
「這條曲線靠 3 筆交易撐起來」比任何指標都有說服力。

**風險**：狀態要如實回報 `empty`／`running`／`failed`／`completed`，
不要從局部指標推斷完成。`multi` 是多檔獨立單標的測試的 Batch Run，
**不是投資組合**，不可推論跨標的資金配置或組合風險。Phase 0 快照是記憶體內、
會消失、不可重現的，影響結論時要講明。分頁上限：symbol results 預設 20 最大 100；
trades 預設 20 最大 100，且每檔只保留最新 500 筆。

**併入項**：原本獨立的「還原股價校正」併進這裡當一個檢查項——用未還原除權息的股價
算長期報酬是**沉默的錯誤**（不報錯、只給偏低的數字），對照 FinMind
`TaiwanStockPriceAdj` 與 `TaiwanStockPrice` 可以抓出來。

---

## 3. `dividend-calendar` 除權息行事曆

**梯隊 B**。**痛點**：手上或觀察中的股票哪天除權息、配多少、要不要參與，每年重查一次。

**產出**：給一組代號 → 回報各自的除權息日期與配發內容，標明資料時間基準。

**用到什麼**：TWSE `/exchangeReport/TWT48U_ALL`（除權除息預告）+
TPEx `tpex_exright_prepost` / `tpex_exright_daily`；配息細節補 FinMind
`TaiwanStockDividend`（免費，帶 `data_id`）。

**為什麼是 B 梯隊的第一個**：TWSE Open API **免 API key、免註冊、無頻率限制**，
使用者輸入一個代號就有輸出，門檻低到不能再低。不需要 Shioaji 連線，
也不需要 `account.read`，現在就能寫完並拿到真實輸出當證據。

**風險（全是 schema 說不出來的）**：

- **上市走 TWSE、上櫃走 TPEx，兩邊端點完全不同且互不涵蓋**，使用者不會知道自己那檔在哪邊。
  第一步必須先判斷市場別。
- 日期是**民國年字串**（`"1150908"` = 2026-09-08），當西元年解析會錯得很安靜。
- TPEx 有**三種官方拼字錯誤，而且各端點不一樣**（2026-09-09 實測）：預告表是
  Rights → `Rrights`；計算結果表同時有 Dividend → `Diviend` 與 Dividend → `Divdend`，
  且拼對與拼錯的股利欄位**並存**、精度不同。兩邊的代號欄位名也不同
  （TWSE 是 `Code`／`Name`，TPEx 是 `SecuritiesCompanyCode`／`CompanyName`），
  日期欄位在預告表與結果表之間也不一致。完整對照見該 widget 的 `references/FIELDS.md`。
- **TPEx 需要瀏覽器等級標頭**（完整 UA 加 `Referer`），否則會拿到 520 或 403。
- 只陳述日期與金額，**不要延伸成「值不值得參與」**——那是投資建議。

---

## 4. `monthly-revenue-digest` 月營收速讀

**梯隊 B**。**痛點**：每月 10 號一堆公司公布營收，看不完也不知道哪些值得看。

**產出**：給一組代號 → 各自最新月營收與變化，只講數字。

**用到什麼**：上市 TWSE `/opendata/t187ap05_L`；**上櫃不在 TWSE 上**，要走 TPEx
`/mopsfin_t187ap05_O`（興櫃 `/t187ap05_R`）——2026-09-09 實測 TWSE 沒有 `t187ap05_O`。
兩邊同源於 MOPS，**欄位結構完全相同**，一套解析邏輯即可。

**風險**：TWSE 端點**完全不吃參數**，一次回傳全市場整包、沒有分頁，收斂是 widget 的工作。
key 是**中文**（TWSE 各端點中英文 key 混雜、彼此不一致）。月營收與財報端點的更新頻率
官方文件**未逐條標註**，只能歸在「盤後批次」，回報時不要編造更新時點。

**最大風險**：這是全部點子裡最容易滑進投資建議的一個。護欄寫死：**只陳述數字與變化，
不解釋成因、不推論股價**。「營收年增 32%」可以，「營收動能強勁值得留意」不行。

---

## 5. `order-preflight` 下單前健檢

**梯隊 C**。**痛點**：掛單掛在流動性沙漠、或價格撞到漲跌停，成交不了才發現。

**產出**：這張單送出去會發生什麼——距漲跌停多遠、檔位對不對、對手方厚不厚。

**用到什麼**：**`preview_order`** + `market.read` 讀快照與五檔。
Contract V2 的處置股欄位（`disposition_level`、`disposition_match_interval_min`、
`disposition_max_lots_single_order`、`disposition_prepay_ratio`、`attention_flag`）
可以直接判斷「這檔今天是不是分盤交易、單筆有沒有張數上限」，不必自己查公告。

**半解鎖**：`preview_order` 工具名已知，只差 `market.read`。schema 撈回來後
這個大概是 C 梯隊最快出成果的。有「下單」的戲劇性但**零成交風險**，現場演最安全。

**風險**：**停在 preview，絕不 execute**。`place_order` 僅對已驗證的模擬伺服器
開放，正式環境下單在目前版本仍是人工終端流程。preview 的 `idempotency_key`
不可被後續不同 payload 重用。

---

## 6. `whos-weird-today` 今天誰不對勁

**梯隊 C**。**痛點**：排行榜一次幾百檔，看不完也看不出所以然。

**產出**：**只回三檔**——與使用者自選股有交集、且今天行為異常的標的。

**用到什麼**：**優先用 Shioaji 1.7.4 的 market signals**——`LimitScanner`（觸及／
接近漲跌停、漲停打開）、`PriceMoveScanner`（1 秒內漲跌 >1% 且 ≥3 檔的急拉急殺）、
`VolumeScanner.burst()`（單筆成交金額爆量）。這比自己從排行榜推斷「異常」精準得多，
而且是即時推送。

**風險**：signals 的**門檻寫死不可調**（>1%、≥3 檔、1 秒冷卻都是文件定死的），
所以 widget 只能選訂閱哪些規則，不能調靈敏度。範圍限台股股票
（`TSE`／`OTC`），指數與期權沒有。所有訊號共用同一個全域 callback，要自行分派。
新手可能沒有自選股，要能在只給一組代號的情況下運作。

---

## 7. `credit-sentinel` 券資哨兵

**梯隊 C**。**痛點**：融券被軋、或券源突然斷掉，通常是出事後才知道。

**產出**：觀察中標的的資券餘額變化與券源狀況，異常時才出聲。

**用到什麼**：`market.read` 的資券餘額與券源查詢。**Shioaji 1.7.4 的 Contract V2
新增了現成的券賣資格欄位**——`StockInfo` 的 `margin_shortable`／`sbl_shortable`／
`below_ref_shortable`（來源是官方每日名單），不必自己從餘額推斷「還能不能券賣」。
上市可用 TWSE `/exchangeReport/MI_MARGN` 交叉驗證，上櫃用 TPEx
`tpex_mainboard_margin_balance`，**但兩者都是盤後日更**。

**風險**：跨來源時要標明時間基準，不可把盤中即時值與盤後日結值放進同一欄比較。
對新手偏進階，排在 C 梯隊末段。

---

## 8. `institutional-flow` 法人動向

**梯隊 C，狀態降級**。**痛點**：想知道三大法人這幾天在買什麼賣什麼。

**為什麼降級**：**上市的三大法人只能走 FinMind，而 FinMind 要 token**
（不帶 token 只有 300/hr 且 dataset 受限）。這道註冊手續對體驗品是致命的，
而新手手上九成是上市股。技術上完全可行，只是不該當展示主線。

**用到什麼**：**上市** → FinMind `TaiwanStockInstitutionalInvestorsBuySell`
（免費但**必須帶 `data_id` 逐檔查**；不帶代號想撈全市場要 Backer 以上。
吃 `start_date`/`end_date`，一次呼叫可同時拿多日）。
**上櫃** → TPEx `tpex_3insti_daily_trading`（免費、整包全市場）。

**關鍵不對稱**：**TWSE Open API 143 個端點裡完全沒有三大法人**（「外資」「投信」
「自營商」「買賣超」關鍵字掃過皆無）。不寫下這條，第一直覺一定是去 TWSE 找端點然後亂猜。

**風險**：TPEx 該端點的 key **開頭帶空白**——
`" Foreign Investors include Mainland Area Investors (Foreign Dealers excluded)-Total Sell"`，
另有字中多空白的 `"Dealers -TotalSell"`。資料**平日 20:00 才更新**，盤中跑「今天」不存在。
免費版 600 次/小時，逐檔查很容易燒掉，要限制一次問幾檔。
**單日差額雜訊很大**，改看連續多日方向或相對近期均量的倍數會穩得多——
但那是計算，回報時要與觀察到的事實分開，且不可延伸成買賣建議。

---

## 9. `morning-brief` 盤前三分鐘

**梯隊 D**。**痛點**：開盤前十分鐘打開軟體，不知道今天第一眼該看哪裡。

**產出**：一段晨報——我持有什麼、昨天收在哪、今天哪幾檔需要注意、為什麼。

**用到什麼**：`account.read` 讀持倉 + `market.read` 讀快照與昨日量能。

**風險**：這是最容易寫成落落長報告的一個，**硬性限制在三行加一張小表**。
盤前沒有即時成交價，要說清楚數字的時間基準是昨收還是試撮。

**待決**：價值幾乎全押在「輸出紀律」上，取材本身沒有 schema 表達不了的難處。
如果三行限制可以靠 `WIDGET_TEMPLATE.md` 的共用規則達成，這個 widget 就沒有存在理由。

---

## 10. `position-attribution` 今天賺賠是誰害的

**梯隊 D**。**痛點**：帳戶一片紅綠，知道賠了多少，說不出是被哪一檔、哪個時段拖下去的。

**產出**：今日損益歸因——貢獻最大的正負各兩檔，各自發生在哪個時段。

**用到什麼**：`account.read` 讀持倉與未實現損益 + `market.read` 讀當日 K 棒。

**風險**：**帳務只能走 Shioaji Pro 原生工具**，任何情況都不可用公開資料推測持倉。
歸因是計算，回報時要與「觀察到的事實」區分開。

---

## 11. `settlement-alarm` 交割款鬧鐘

**梯隊 D**。**痛點**：T+2 交割款忘了轉進去，最糟會變成違約交割。

**產出**：未來幾個交割日各要準備多少錢、哪一天有缺口。

**用到什麼**：`account.read` 的交割資訊與保證金查詢。

**風險**：交割資訊有新舊兩種來源，欄位不同，要確認 App 回的是哪一種。
金額是提醒，不是理財建議——不要延伸成「你應該賣掉什麼來補」。

---

## 12. `closing-journal` 收盤複盤日記

**梯隊 D**。**痛點**：都知道該寫交易日誌，沒有人真的寫。

**產出**：當日已實現損益 + 進出場點位標在 K 棒上，指出可觀察的行為模式。

**用到什麼**：`account.read` 的損益查詢與損益明細 + `market.read` 的當日 K 棒。

**風險**：**描述行為，不評價人**。陳述「這 3 筆的進場價位於當日區間前 10%」，
不要寫成「你在追高」或任何買賣建議。

---

## 已評估後捨棄

留著是為了避免同一個點子再被撿回來一次。

### `indicator-from-words` 說一句就有指標

**構想**：自然語言描述（「量增價漲」）→ App 原生自訂指標，存檔後出現在圖上。

**為什麼砍**：核心動作是「把想法翻成 Shioaji Pro 指標語法」，而語法規則就在
`save_custom_indicator` 的 advertised schema 裡——參數、型別、`enum`、必填欄位，
模型讀完 schema 自然就寫得出來。**重複 baseline，不是補 baseline 的缺口。**

**殘留物**：「完成的定義是拿到成功回執 **且** 重新 `list` 讀得到」——真實失效模式，
但撐不起一個 widget，應併入 `WIDGET_TEMPLATE.md` 的共用護欄。

### 當沖標的／借券資格查詢

**構想**：用 TWSE `/exchangeReport/TWTB4U`（當日沖銷標的）與 `/SBL/TWT96U`
（當日可借券賣出股數）判斷這檔能不能當沖或放空。

**為什麼砍**：Contract V2 的 `margin_shortable`／`sbl_shortable`／`below_ref_shortable`
已經直接給了答案，來源同樣是官方每日名單。**重複 baseline。**

### 注意股／處置股雷達

**構想**：TWSE `/announcement/notice`、`/announcement/punish`、`/exchangeReport/TWTAWU`
與 TPEx `tpex_trading_warning_information`、`tpex_disposal_information` 做預警。

**為什麼砍**：Contract V2 已有 `attention_flag`、`disposition_level`、
`disposition_match_interval_min` 等欄位，重疊度高。唯一沒被涵蓋的是「為什麼被列入」
與「何時解除」，撐不起獨立 widget——當作 §5 `order-preflight` 的補充資料，不另開。

### 本益比位階

**構想**：用 FinMind `TaiwanStockPER` 講「現在 PER 位於過去 N 年第幾百分位」。

**為什麼砍**：技術上可行，但產出幾乎必然被讀成買賣訊號，離「不給投資建議」太近。

### 大單偵測 + 大型分點進出（移出主線，非砍）

**構想**：檢查持倉各檔在大型分點與三大法人相較昨日的進出，以及市場上有無單筆 ≥100 張的大單。

**為什麼移出**：這是**老手需求**，不符「面向初次使用者」的定位。三件子任務可行性差很多：

- **大型分點：免費版做不到，且沒有繞法。** 官方一手來源 TWSE `bsr.twse.com.tw/bshtm/`
  與 TPEx `brokerBS.php` 都是互動式網頁、都有驗證碼、都沒有 API，且 TWSE 使用條款
  第 6 條明文禁止未經同意的自動化擷取。FinMind `TaiwanStockTradingDailyReport` 是
  **Sponsor 專屬**，還須走專屬端點 `GET /taiwan_stock_trading_daily_report`
  （走 `/data?dataset=...` 回 422）且**單次只能查一天**。
  規則是：**Free 等級就明說做不到，不要改用爬蟲。**
- **三大法人日對日**：可行，已獨立成 §8 `institutional-flow`。
- **大單 ≥100 張：定義先有問題。** 台股 2020 年 3 月起逐筆撮合，一筆大額委託會與多個
  對手方成交、在 tick 上呈現為多筆較小紀錄；開收盤集合競價又會合併。
  **tick 上的「單筆成交量」≠「有人下了一張大單」**——此為機制推論，**實作前須拿真實
  tick 驗證**，不可寫死。免費公開來源也拿不到 tick（`TaiwanStockPriceTick` 是
  Backer/Sponsor）。Shioaji 的 `VolumeScanner.burst()` 是即時的，但**門檻寫死不可調**，
  設不了「100 張」。真要自訂門檻只能自己拉 tick 算，成本高一個量級。

### 判準怎麼套用在新點子上

兩個問題連續問：**①這個 widget 裝的東西，JSON Schema 表達得出來嗎？** 表達得出來
（參數怎麼填、合法值有哪些）就砍。**②新手要先做什麼才能用到它？** 需要註冊、
綁憑證、開帳務權限、或先有持倉的，往後排，不放展示主線。
