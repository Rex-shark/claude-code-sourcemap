# CLAUDE.md

> 語言規格：回覆一律使用**繁體中文（台灣用語）**，專有名詞保持原文（如 API、CLI、IDE、sourcemap）。
> 台灣慣用譯名：套件、函式、陣列、字串、迴圈、變數、型別、介面、列舉等。

## 專案概述

這是一個**非官方**的 Claude Code 原始碼還原專案，透過 npm 套件 `@anthropic-ai/claude-code@2.1.88` 中的 source map（`cli.js.map`）還原 TypeScript 原始碼，**僅供研究用途**。

- 還原版本：`2.1.88`
- 還原檔案數：約 4756 個（含 1884 個 `.ts`/`.tsx` 原始檔）
- 原始碼版權歸 [Anthropic](https://www.anthropic.com) 所有

## 目錄結構

```
├── package/                 # 原始 npm 發佈套件內容
│   ├── cli.js               # 打包後的 CLI 入口（bundled）
│   ├── cli.js.map           # Source map（還原依據）
│   └── package.json         # 套件設定
├── restored-src/            # 從 source map 還原的原始碼
│   └── src/                 # TypeScript 原始碼根目錄
│       ├── main.tsx         # CLI 入口
│       ├── tools/           # 工具實作（Bash、FileEdit、Grep、MCP 等 30+ 個）
│       ├── commands/        # 指令實作（commit、review、config 等 40+ 個）
│       ├── services/        # API、MCP、分析等服務
│       ├── utils/           # 工具函式（git、model、auth、env 等）
│       ├── coordinator/     # 多 Agent 協調模式
│       ├── assistant/       # 助手模式（KAIROS）
│       ├── plugins/         # 外掛系統
│       ├── skills/          # 技能系統
│       ├── voice/           # 語音互動
│       └── vim/             # Vim 模式
├── extract-sources.js       # Source map 提取腳本
├── docs/                    # 分析文件
└── examples/                # 範例
```

## 常用指令

```bash
# 還原原始碼（需先有 package/cli.js.map）
node extract-sources.js
```

## 研究導覽

核心原始碼位於 `restored-src/src/`，以下為主要模組：

| 模組 | 路徑 | 說明 |
|------|------|------|
| CLI 入口 | `src/main.tsx` | 程式進入點 |
| 工具系統 | `src/tools/` | 各種內建工具的實作 |
| 指令系統 | `src/commands/` | slash commands 與 CLI 指令 |
| 服務層 | `src/services/` | API 呼叫、MCP 協定等 |
| 協調器 | `src/coordinator/` | 多 Agent 協調邏輯 |
| 外掛系統 | `src/plugins/` | 外掛載入與管理 |
| 技能系統 | `src/skills/` | 技能定義與執行 |

## 注意事項

- 此為還原碼，**無法直接編譯或執行**，僅供閱讀與研究
- 原始碼結構為推測還原，不代表 Anthropic 內部實際架構
- 請勿用於商業用途
