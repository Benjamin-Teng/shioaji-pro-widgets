# Shioaji Pro Widgets

一組小而專一的 skill，掛在 [Shioaji Pro](https://github.com/Sinotrade/shioaji-pro-app)
的原生語意 MCP 工具上。**一個 widget 解決一個投資人一天會遇到的痛點**，
輸出是三行結論加一張小表，不是報告。

## 安裝

```bash
# 本機開發／DEMO
/plugin marketplace add D:\projects\shioaji-pro-widgets
/plugin install shioaji-pro-widgets@shioaji-pro-widgets
```

裝好後在 Shioaji Pro 的 Agent 面板直接用自然語言呼叫，例如
「幫我把版面排成盯 2330 的樣子」。

## 現有 widget

| Widget | 解決什麼 |
| --- | --- |
| `workspace-conductor` | 一句話把工作區排成這次要盯的樣子 |

規劃中的清單見 [docs/ROADMAP.md](docs/ROADMAP.md)。

## 設計原則

- **小**：`SKILL.md` 不超過 60 行，最多一個自己的 reference。
- **專一**：說不出「一句話解決什麼」就不要寫。
- **不下單**：預設只到 `trade.preview`；`place_order` 僅在已驗證的模擬環境、
  且使用者當下明確要求時才碰。
- **不給投資建議**：陳述事實與計算，不做買賣結論、不預測價格。
- **選對來源**：Shioaji Pro 原生工具 → Shioaji API → TWSE Open API → FinMind → TPEx，
  依序取第一個能回答的。規則與「已排除」清單見
  [references/DATA_SOURCES.md](plugins/shioaji-pro-widgets/references/DATA_SOURCES.md)。

寫新 widget 前先讀 [docs/WIDGET_TEMPLATE.md](docs/WIDGET_TEMPLATE.md)。

## 結構

```text
.claude-plugin/marketplace.json      # 讓這個 repo 可以直接被 marketplace add
plugins/shioaji-pro-widgets/
├── .claude-plugin/plugin.json
├── .codex-plugin/plugin.json
├── references/                      # 所有 widget 共用的知識
│   ├── APP_TOOLS.md                 # 能力層級與 v1 語意工具名
│   ├── DATA_SOURCES.md              # 來源優先序（最容易出錯的地方）
│   ├── TWSE_OPENAPI.md              # 上市市場公開資料，主要來源
│   ├── FINMIND.md                   # 三大法人與歷史面板資料
│   └── TPEX_OPENAPI.md              # 上櫃／興櫃，補充用
└── skills/<widget-name>/SKILL.md
```

## License

MIT
