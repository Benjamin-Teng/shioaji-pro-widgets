# Shioaji Pro 原生能力邊界

Widget 透過 App 的 **semantic MCP tools** 操作 Shioaji Pro，
不用 shell、不用 UI 座標、不模擬鍵盤。連線中的 MCP server 的
advertised schema 是工具名稱、版本與可用性的唯一真相；本檔只描述意圖與邊界。

## 能力層級（各自獨立、預設拒絕）

| 層級 | 涵蓋 |
| --- | --- |
| `market.read` | 健康狀態、合約、報價、訂閱、市場脈絡 |
| `account.read` | 帳戶、餘額、持倉、委託、交割 |
| `ui.control` | 選標的、面板、連動、版面、自訂指標、策略、回測讀取 |
| `trade.preview` | 驗證一筆確切委託或變更，不執行 |
| `trade.execute` | 執行已核准的操作或收斂其結果 |

**廣的層級不會蘊含窄的。** 安裝 skill 本身不授予任何能力。
權限一律讀 App 回傳的 capability 狀態，不要從「上次成功過」或對話內容推論。

## v1 語意工具名

- App 狀態：`get_app_state`、`list_panels`
- 版面變更：`select_contract`、`add_panel`、`remove_panel`、`set_panel_pin`、`apply_layout`
- 原生內容：`list_custom_indicators`、`save_custom_indicator`、`list_strategies`、`save_strategy`
- 回測讀取：`get_backtest_result`、`list_backtest_symbol_results`、`get_backtest_trades`
- 交易：`preview_order`、`place_order`、`cancel_order`、`reconcile_order`

`market` 與 `account` 家族的完整 v1 工具名不在本 repo 內，**呼叫前先讀連線中
server 的 advertised schema**，不要照猜。schema 沒有的工具就是不存在——
不要自創 task、audit、approval-token 或 raw HTTP 工具。

## 交易安全

- 本 plugin 的 widget **預設只做 `trade.preview`，不做 `trade.execute`**。
  需要執行時必須是使用者當下明確要求。
- `place_order` / `cancel_order` 僅對已驗證的**模擬**伺服器開放；正式環境下單
  在目前版本仍是人工終端流程。
- 每個變更帶呼叫端自產的穩定 `idempotency_key`；**同一把 key 絕不用於不同 payload**。
- 結果不確定時呼叫 `reconcile_order` 收斂，**不要重試**。

## 變更後的驗證

任何變更完成後，用一次新的語意讀取確認狀態，再回報。
回報時區分：觀察到的事實 / 計算 / preview / 待核准 / 已執行。
