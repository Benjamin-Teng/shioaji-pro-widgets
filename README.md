# Shioaji Pro Widgets

一組小而專一的 skill，掛在 [Shioaji Pro](https://github.com/Sinotrade/shioaji-pro-app)
的原生語意 MCP 工具上。

**一個 widget 解決一個投資人一天會遇到的痛點。** 輸出是三行結論加一張小表，
不是報告；說不出「一句話解決什麼」，就不值得成為一個 widget。

## 安裝

```bash
/plugin marketplace add Benjamin-Teng/shioaji-pro-widgets
/plugin install shioaji-pro-widgets@shioaji-pro-widgets
```

本機開發時把第一行換成本地路徑即可，例如
`/plugin marketplace add D:\projects\shioaji-pro-widgets`。

裝好後在 Shioaji Pro 的 Agent 面板用自然語言呼叫，例如
「幫我把版面排成盯 2330 的樣子」。

## 現有 widget

| Widget | 解決什麼 | 需要能力 |
| --- | --- | --- |
| `workspace-conductor` | 一句話把工作區排成這次要盯的樣子 | `ui.control` |

規劃中的 10 個 widget、各自的痛點、用到什麼、風險，見
[docs/ROADMAP.md](docs/ROADMAP.md)。

## 設計原則

- **小**：`SKILL.md` 不超過 60 行，最多一個自己的 reference。
- **專一**：一個 widget 一個痛點。
- **不下單**：預設只到 `trade.preview`。`place_order` 僅在已驗證的模擬環境、
  且使用者當下明確要求時才碰；正式環境下單在目前版本仍是人工終端流程。
- **不給投資建議**：陳述事實與計算，不做買賣結論、不預測價格。
- **工具名不猜**：`market.*` 與 `account.*` 家族的 v1 工具名不在本 repo，
  呼叫前讀連線中 MCP server 的 advertised schema。schema 沒有的工具就是不存在。

## 資料來源怎麼選

這是本 plugin 最容易出錯的地方。**依序取第一個能回答問題的來源就停**，
不要跨來源混算同一個數字。

| 順位 | 來源 | 用途 |
| --- | --- | --- |
| 1 | Shioaji Pro 原生 MCP tools | 使用者的帳務、持倉、下單、版面、指標、回測 |
| 2 | Shioaji API | 即時與歷史行情、資券餘額、券源、市場訊號 |
| 3 | TWSE Open API | 上市市場公開盤後統計、月營收、財報 |
| 4 | FinMind | 三大法人、基本面；Sponsor 另有券商分點與分K |
| 5 | TPEx Open API | 只在標的確定為上櫃時 |

完整規則與每個來源的坑見
[DATA_SOURCES.md](plugins/shioaji-pro-widgets/references/DATA_SOURCES.md)。

## 已經查證過做不到的事

寫在這裡，是為了讓下一個人不用再查一次：

- **券商分點進出（免費管道）**：官方只有帶驗證碼的互動式網頁、無 API。
  FinMind 的 `TaiwanStockTradingDailyReport` 需要 **Sponsor 等級**——
  有訂閱就能做，Free 等級不成立。
- **基金完整持股明細**：SITCA 自民國 104 年 6 月起停更，只剩月前十大與季佔淨值
  1% 以上。想看投信調節請改用三大法人的投信買賣超。
- **三大法人不在 TWSE OpenAPI**：143 個端點全文掃描沒有這個主題，要走 FinMind。
- **Shioaji 沒有基本面**：財報、月營收、股利、除權息、美股港股一律沒有。
- **條件單／停損單沒有原生支援**：是本地輪詢自建，斷線或重啟就失效。
- **訂單級冪等不存在**：逾時要先 `update_status()` 對帳，不能重送。

## 寫一個新 widget

1. 讀 [docs/WIDGET_TEMPLATE.md](docs/WIDGET_TEMPLATE.md)——尺寸上限、`SKILL.md` 骨架、
   完成的定義。
2. 讀 [APP_TOOLS.md](plugins/shioaji-pro-widgets/references/APP_TOOLS.md)——能力層級與
   v1 語意工具名。
3. 讀 [SHIOAJI_LIMITS.md](plugins/shioaji-pro-widgets/references/SHIOAJI_LIMITS.md)——
   Shioaji API 的硬性限制與空白區（基準 1.7.4）。
4. 在真的連上 Shioaji Pro 的 session 跑過一次，輸出貼進 commit 訊息。
5. `markdownlint-cli2`，0 error。

## 結構

```text
.claude-plugin/marketplace.json      # 讓這個 repo 可以直接被 marketplace add
plugins/shioaji-pro-widgets/
├── .claude-plugin/plugin.json       # Claude Code
├── .codex-plugin/plugin.json        # Codex
├── references/                      # 所有 widget 共用的知識
│   ├── APP_TOOLS.md                 # 能力層級與 v1 語意工具名
│   ├── SHIOAJI_LIMITS.md            # Shioaji API 能力邊界（基準 1.7.4）
│   ├── DATA_SOURCES.md              # 來源優先序與已排除清單
│   ├── TWSE_OPENAPI.md              # 上市市場公開資料，主要來源
│   ├── FINMIND.md                   # 三大法人與歷史面板資料
│   └── TPEX_OPENAPI.md              # 上櫃／興櫃，補充用
└── skills/<widget-name>/SKILL.md
```

`references/` 裡的外部 API 事實都查證自官方一手來源並標註查證日期。
發現過期請直接更正，不要沿用。

## License

MIT
