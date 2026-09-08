---
name: backtest-doctor
description: |
  Use when the user wants a sanity check on a backtest they already ran in
  Shioaji Pro — "這個回測可以信嗎", "幫我看一下回測結果", "為什麼回測很漂亮實盤卻虧",
  "這條曲線是不是過度擬合", "review my backtest result". Reads an existing backtest
  and reports whether its numbers are trustworthy; it does not run or design strategies.
  Trigger keywords: 回測, 回測績效, 過度擬合, 最大回撤, 勝率, 樣本數, backtest, drawdown,
  overfitting, equity curve.
---

# 回測體檢

**痛點**：回測曲線很漂亮，實盤一上就打臉，但說不出到底哪裡不對。
**回答**：不是再給一次績效數字，而是回答「這份結果的數字站不站得住腳」。

## 前置

需要 `ui.control`。沒有就直說缺這個能力，不要改用其他方式取得回測結果。
**使用者必須先在 Shioaji Pro 跑過回測**；讀到空的就說沒有回測結果可讀，
不要給空表，也不要提議自己造一份資料。

## 流程

**一定要照這個順序漸進讀取，有需要才往下一層**，不要把整包結果倒進回應。

1. `get_backtest_result` — 先讀總覽。**先確認狀態**：如實回報
   `empty` / `running` / `failed` / `completed`，**不要從局部指標有值就推斷跑完了**。
2. `list_backtest_symbol_results` — 只在需要逐檔比較集中度時才呼叫。
   分頁預設 20、最大 100。
3. `get_backtest_trades` — 只在需要看個別交易時才呼叫。分頁預設 20、最大 100，
   且每檔只保留最新 500 筆，**筆數被截斷時要講明，不要當成全部交易**。

## 四項體檢

1. **樣本數**：交易筆數太少時，漂亮曲線是運氣不是策略。講出實際筆數。
2. **集中度**：把貢獻最大的幾筆拿掉後還剩多少。最有說服力的一句話是
   「這條曲線靠 3 筆交易撐起來」。**但交易總數超過可讀上限（每檔最新 500 筆）時，
   集中度一律標「無法判定」**——不可拿被截斷的樣本算出一個看起來完整的結論。
   總覽若已提供完整的分布指標，改用那個，並說明來自哪一層。
3. **最大回撤**：與總報酬並列，不要只講報酬。
4. **還原股價**：長期報酬若用未還原除權息的價格會**偏低且不報錯**。需要佐證時用
   FinMind `TaiwanStockPriceAdj` 對照 `TaiwanStockPrice`；沒有 token 就說這項查不了。

## 兩個一定會踩的語意陷阱

- **`multi` 不是投資組合。** 它是多檔各自獨立的單標的測試（Batch Run），
  每檔都假設用了全部資金。**不可推論跨標的資金配置、分散效果或組合風險**——
  那個數字看起來合理但完全沒有意義。
- **Phase 0 快照是記憶體內的、會消失、不可重現。** 結論若建立在它上面要明講。

## 輸出

三行結論（狀態與樣本數、集中度、回撤）加一張小表列關鍵指標。
每個數字說得出來自哪一層讀取。體檢不通過的項目要指名，不要含糊帶過。

## 不做

不下單、不讀帳務、不改策略或指標、不建議參數怎麼調。
**不預測這個策略未來會不會賺**，也不說「可以上實盤了」——只講這份結果的可信度。
