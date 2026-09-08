# Widget Backlog

每個 widget 一個痛點。狀態：`ready`（已可用）／`next`（下一批）／`blocked`（缺資訊）。

| Widget | 痛點 | 需要能力 | 狀態 |
| --- | --- | --- | --- |
| `workspace-conductor` | 每天手動排版面 | `ui.control` | ready |
| `indicator-from-words` | 有想法但不會寫指標 | `ui.control` | next |
| `morning-brief` | 開盤前不知道該盯什麼 | `account.read` + `market.read` | blocked |
| `backtest-doctor` | 回測漂亮、實盤打臉 | `ui.control` | next |
| `order-preflight` | 掛單撞漲跌停或掛進流動性沙漠 | `trade.preview` + `market.read` | blocked |
| `position-attribution` | 帳戶紅綠一片說不出原因 | `account.read` + `market.read` | blocked |
| `settlement-alarm` | T+2 忘了補錢 | `account.read` | blocked |
| `credit-sentinel` | 融券被軋、券源斷掉才知道 | `market.read` | blocked |
| `whos-weird-today` | 排行榜看不完 | `market.read` | blocked |
| `closing-journal` | 從不寫交易日誌 | `account.read` | blocked |

`blocked` 的共同原因：`market.*` 與 `account.*` 家族的 v1 工具名不在本 repo，
需要對連線中的 Shioaji Pro MCP server 讀一次 advertised schema 才能定案。
解掉這個之後，這些 widget 大多是半小時內能寫完的尺寸。
