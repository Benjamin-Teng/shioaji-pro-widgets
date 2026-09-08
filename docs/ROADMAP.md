# Widget Backlog

每個 widget 一個痛點。狀態：`ready`（已可用）／`next`（工具名已確定，可直接寫）／
`blocked`（缺工具名，見下方「共同阻塞點」）。

| Widget | 痛點 | 需要能力 | 狀態 |
| --- | --- | --- | --- |
| [`workspace-conductor`](#1-workspace-conductor-版面指揮官) | 每天手動排版面 | `ui.control` | ready |
| [`indicator-from-words`](#2-indicator-from-words-說一句就有指標) | 有想法但不會寫指標 | `ui.control` | next |
| [`backtest-doctor`](#3-backtest-doctor-回測體檢) | 回測漂亮、實盤打臉 | `ui.control` | next |
| [`morning-brief`](#4-morning-brief-盤前三分鐘) | 開盤前不知道該盯什麼 | `account.read` + `market.read` | blocked |
| [`order-preflight`](#5-order-preflight-下單前健檢) | 掛單撞漲跌停或掛進流動性沙漠 | `trade.preview` + `market.read` | blocked |
| [`position-attribution`](#6-position-attribution-今天賺賠是誰害的) | 帳戶紅綠一片說不出原因 | `account.read` + `market.read` | blocked |
| [`settlement-alarm`](#7-settlement-alarm-交割款鬧鐘) | T+2 忘了補錢 | `account.read` | blocked |
| [`credit-sentinel`](#8-credit-sentinel-券資哨兵) | 融券被軋、券源斷掉才知道 | `market.read` | blocked |
| [`whos-weird-today`](#9-whos-weird-today-今天誰不對勁) | 排行榜看不完 | `market.read` | blocked |
| [`closing-journal`](#10-closing-journal-收盤複盤日記) | 從不寫交易日誌 | `account.read` | blocked |

## 共同阻塞點

`blocked` 的六個全部卡在同一件事：**`market.*` 與 `account.*` 家族的 v1 工具名
不在任何 repo 裡**，必須對連線中的 Shioaji Pro MCP server 讀一次 advertised
schema 才能定案。解掉之後，這些 widget 大多是半小時內能寫完的尺寸。
在那之前**不要照猜工具名**——見 `references/APP_TOOLS.md`。

下面每一節的「用到什麼」欄位中，**粗體**的是已在 `MCP_TOOLS.md` 有明文的 v1
工具名；非粗體的是意圖描述，實作前要對 schema 確認。

---

## 1. `workspace-conductor` 版面指揮官

**痛點**：每天開盤前手動拖面板、換標的、重排版面，同樣的動作重複一輩子。

**產出**：一句話把工作區排成這次要盯的樣子，回報主標的、面板清單、哪些被釘住。

**用到什麼**：**`get_app_state`**、**`list_panels`**、**`apply_layout`**、
**`select_contract`**、**`add_panel`**／**`remove_panel`**、**`set_panel_pin`**。

**DEMO 亮點**：全場最炸的一個——畫面自己動起來，觀眾當場看到版面重排。

**風險**：版面變更必須序列化，不能並行；使用者說「我平常那個版面」時要列出
可用版面請他指認，不要猜。

---

## 2. `indicator-from-words` 說一句就有指標

**痛點**：腦中有「量增價漲」這種想法，卡在不會寫指標語法，於是永遠沒做出來。

**產出**：自然語言描述 → App 原生自訂指標，存檔後直接出現在圖上。

**用到什麼**：**`list_custom_indicators`**（先讀再改）、**`save_custom_indicator`**。
每次存檔給一把新的 `idempotency_key`。

**DEMO 亮點**：第二炸——講一句話，圖上長出一條新的線。

**風險**：這裡指的是 **App 原生內容，不是 Pine Script、也不是專案裡的檔案**。
驗證失敗是可行動的回饋，修正後用同一個使用者意圖再送一次；完成的定義是
「拿到成功回執 **且** 重新 list 讀得到」。

---

## 3. `backtest-doctor` 回測體檢

**痛點**：回測曲線很漂亮，實盤一上就打臉，但說不出到底哪裡不對。

**產出**：三行體檢結論——樣本數夠不夠、最大回撤與集中度、有沒有過度擬合的跡象。

**用到什麼**：**`get_backtest_result`** →（需要逐檔比較才呼叫）
**`list_backtest_symbol_results`** →（需要看個別交易才呼叫）**`get_backtest_trades`**。
**一定要照這個順序漸進讀取**，不要把整包結果倒進回應。

**DEMO 亮點**：講人話——「這條曲線靠 3 筆交易撐起來」比任何指標都有說服力。

**風險**：狀態要如實回報 `empty`／`running`／`failed`／`completed`，
不要從局部指標推斷完成。`multi` 是多檔獨立單標的測試的 Batch Run，
**不是投資組合**，不可推論跨標的資金配置或組合風險。Phase 0 快照是記憶體內、
會消失、不可重現的，影響結論時要講明。分頁上限：symbol results 預設 20 最大 100；
trades 預設 20 最大 100，且每檔只保留最新 500 筆。

---

## 4. `morning-brief` 盤前三分鐘

**痛點**：開盤前十分鐘打開軟體，不知道今天第一眼該看哪裡。

**產出**：一段晨報——我持有什麼、昨天收在哪、今天哪幾檔需要注意、為什麼。

**用到什麼**：`account.read` 讀持倉 + `market.read` 讀快照與昨日量能。

**DEMO 亮點**：負責收尾講故事——「這就是我每天早上第一件事」。

**風險**：這是最容易寫成落落長報告的一個，**硬性限制在三行加一張小表**。
盤前沒有即時成交價，要說清楚數字的時間基準是昨收還是試撮。

---

## 5. `order-preflight` 下單前健檢

**痛點**：掛單掛在流動性沙漠、或價格撞到漲跌停，成交不了才發現。

**產出**：這張單送出去會發生什麼——距漲跌停多遠、檔位對不對、對手方厚不厚。

**用到什麼**：**`preview_order`** + `market.read` 讀快照與五檔。

**DEMO 亮點**：有「下單」的戲劇性但**零成交風險**，現場演最安全。

**風險**：**停在 preview，絕不 execute**。`place_order` 僅對已驗證的模擬伺服器
開放，正式環境下單在目前版本仍是人工終端流程。preview 的 `idempotency_key`
不可被後續不同 payload 重用。

---

## 6. `position-attribution` 今天賺賠是誰害的

**痛點**：帳戶一片紅綠，知道賠了多少，說不出是被哪一檔、哪個時段拖下去的。

**產出**：今日損益歸因——貢獻最大的正負各兩檔，各自發生在哪個時段。

**用到什麼**：`account.read` 讀持倉與未實現損益 + `market.read` 讀當日 K 棒。

**DEMO 亮點**：把「我今天賠 3 萬」變成「你今天賠的 3 萬有 2.4 萬來自這一檔的早盤」。

**風險**：**帳務只能走 Shioaji Pro 原生工具**，任何情況都不可用公開資料推測持倉。
歸因是計算，回報時要與「觀察到的事實」區分開。

---

## 7. `settlement-alarm` 交割款鬧鐘

**痛點**：T+2 交割款忘了轉進去，最糟會變成違約交割。

**產出**：未來幾個交割日各要準備多少錢、哪一天有缺口。

**用到什麼**：`account.read` 的交割資訊與保證金查詢。

**DEMO 亮點**：冷門但痛到爆，而且幾乎沒有工具在做這件事。

**風險**：交割資訊有新舊兩種來源，欄位不同，要確認 App 回的是哪一種。
金額是提醒，不是理財建議——不要延伸成「你應該賣掉什麼來補」。

---

## 8. `credit-sentinel` 券資哨兵

**痛點**：融券被軋、或券源突然斷掉，通常是出事後才知道。

**產出**：手上（或觀察中）標的的資券餘額變化與券源狀況，異常時才出聲。

**用到什麼**：`market.read` 的資券餘額與券源查詢。上櫃標的可用
TPEx `/tpex_mainboard_margin_balance` 交叉驗證，**但那是盤後日更**，
且**只有上櫃沒有上市**。

**DEMO 亮點**：冷門但很專業，會讓懂的人眼睛一亮。

**風險**：跨來源時要標明時間基準，不可把盤中即時值與盤後日結值放進同一欄比較。

---

## 9. `whos-weird-today` 今天誰不對勁

**痛點**：排行榜一次幾百檔，看不完也看不出所以然。

**產出**：**只回三檔**——與使用者自選股／持倉有交集、且今天行為異常的標的。

**用到什麼**：`market.read` 的排行掃描（漲跌幅、成交量、成交金額、振幅）
與自選股，取交集後再收斂。需要基本面佐證時才逐檔問 FinMind。

**DEMO 亮點**：輸出乾淨——別人給你兩百檔，它給你三檔。

**風險**：**FinMind 免費版不能撈全市場**（帶 `data_id` 查單檔才免費），
所以「先用排行收斂到 3–5 檔、再逐檔查明細」不是效能優化，是硬性順序。
見 `references/FINMIND.md`。

---

## 10. `closing-journal` 收盤複盤日記

**痛點**：都知道該寫交易日誌，沒有人真的寫。

**產出**：當日已實現損益 + 進出場點位標在 K 棒上，指出可觀察的行為模式
（例如進場價接近當日高點、出場價接近當日低點）。

**用到什麼**：`account.read` 的損益查詢與損益明細 + `market.read` 的當日 K 棒。

**DEMO 亮點**：自動生成的日誌，且會指出「追高賣低」這類自己不想承認的模式。

**風險**：**描述行為，不評價人**。陳述「這 3 筆的進場價位於當日區間前 10%」，
不要寫成「你在追高」或任何買賣建議。
