# Widget 撰寫規範

一個 widget = 一個投資人一天會遇到的痛點。做不到「一句話說完它解決什麼」，
就是還沒想清楚，不要開始寫。

## 尺寸上限

- `SKILL.md` **不超過 60 行**。
- 最多一個自己的 `references/` 檔；共用知識一律連到
  `plugins/shioaji-pro-widgets/references/` 下的共用檔，不要複製貼上。
- 輸出以「三行結論 + 一張小表」為目標。widget 不是報告產生器。

## 目錄結構

```text
plugins/shioaji-pro-widgets/skills/<widget-name>/
├── SKILL.md
└── references/          # 選用，最多一個檔
```

`<widget-name>` 用 kebab-case，名字講「解決什麼」而非「用什麼技術」。

## SKILL.md 骨架

```markdown
---
name: <widget-name>
description: |
  Use when <使用者會怎麼開口>. <這個 widget 回答哪一個問題>.
  Trigger keywords: <中英文觸發詞>.
---

# <中文標題>

**痛點**：一句話。
**回答**：這個 widget 產出什麼。

## 前置

需要的能力層級（`market.read` / `account.read` / `ui.control` / `trade.preview`）。
缺少時直說缺什麼，不要繞路。

## 流程

1. 讀 …（最窄的語意工具）
2. 算 …
3. 回報 …

## 輸出

固定格式，含使用者一眼要看的三個數字。

## 不做

明列邊界，特別是不下單、不給個人化投資建議、不預測價格。
```

## 每個 widget 都要遵守

- **選對資料來源**：讀 `references/DATA_SOURCES.md` 的順位規則。
- **能力邊界**：讀 `references/APP_TOOLS.md`。工具名以連線中 server 的 schema 為準。
- **不下單**：預設只到 `trade.preview`。
- **不給投資建議**：陳述事實與計算，不做「該買該賣」的結論。
- **數字要有出處**：每個數字說得出來自哪個工具、哪個時間基準。
- **失敗要明說**：取不到就講取不到，不要用推估補值。

## 完成的定義

1. 在真的連上 Shioaji Pro 的 session 裡跑過一次，輸出貼在 PR 或 commit 訊息裡。
2. 缺少能力時的分支也跑過一次（例如未授予 `account.read`）。
3. `SKILL.md` 通過 `markdownlint-cli2`，0 error。
