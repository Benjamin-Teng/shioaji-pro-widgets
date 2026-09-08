# shioaji-pro-widgets

這個 repo 是一個 Claude Code / Codex plugin marketplace，裡面裝的是掛在
Shioaji Pro 原生語意 MCP 工具上的小 skill。**不是應用程式，沒有 build，
沒有執行期程式碼**——內容幾乎全是 Markdown。

## 動手前

- 寫或改 widget 前先讀 `docs/WIDGET_TEMPLATE.md`。
- 能力邊界與 v1 工具名讀 `plugins/shioaji-pro-widgets/references/APP_TOOLS.md`。
- 選資料來源讀 `plugins/shioaji-pro-widgets/references/DATA_SOURCES.md`。

## 硬規則

1. **一個 widget 一個痛點。** `SKILL.md` 不超過 60 行，最多一個自己的 reference。
   共用知識放 `references/`，不要複製貼上。
2. **工具名不准猜。** `market.*` 與 `account.*` 家族的 v1 工具名不在本 repo；
   呼叫前讀連線中 server 的 advertised schema。schema 沒有的工具就是不存在。
3. **預設不下單。** widget 只做到 `trade.preview`。`place_order` / `cancel_order`
   僅限已驗證的模擬伺服器，且必須是使用者當下明確要求。
4. **不給投資建議、不預測價格。** 只陳述事實與計算。
5. **改任何 `.md` 都要跑 `markdownlint-cli2`**，0 error 才算完成。
6. **完成宣稱要有證據**：在真的連上 Shioaji Pro 的 session 跑過一次，
   輸出貼進 commit 訊息或 PR。
7. **外部 API 的精確字串必須查證**（端點、參數、欄位名、額度）。查不到就在文件裡
   標「待查證」，不要填訓練記憶值。

## 版本

新增或修改 widget 時，同步更新三處版本號：
`.claude-plugin/marketplace.json`、`plugins/shioaji-pro-widgets/.claude-plugin/plugin.json`、
`plugins/shioaji-pro-widgets/.codex-plugin/plugin.json`。三者必須一致。

新增 widget 也要更新 `README.md` 的清單與 `docs/ROADMAP.md` 的狀態。
