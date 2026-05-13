# Claude Code 原始碼架構分析

> **版本**：`@anthropic-ai/claude-code v2.1.88`  
> **還原方式**：從 npm 包的 `cli.js.map` source map 中提取 `sourcesContent`  
> **還原檔案數**：4756 個（含 1884 個 `.ts`/`.tsx` 源文件）

---

## 一、整體專案概覽

```
claude-code-sourcemap/
├── claude-code-2.1.88.tgz     # 原始 npm 包壓縮檔
├── extract-sources.js          # 提取 sourcemap 的腳本
├── package/                    # 解壓後的 npm 發布包
│   ├── cli.js                  # 打包後的單檔執行擋（~13MB）
│   ├── cli.js.map              # Source Map（~60MB，從此還原源碼）
│   ├── package.json            # 套件定義
│   └── sdk-tools.d.ts          # SDK 型別宣告
└── restored-src/               # 從 source map 還原的原始碼
    ├── src/                    # 主要原始碼
    ├── vendor/                 # 自製 native 模組
    └── node_modules/           # 相依套件原始碼
```

**技術棧**：
- **Runtime**：Node.js ≥ 18，以 ES Module 形式運行
- **語言**：TypeScript + TSX（React）
- **UI 框架**：Ink（Terminal React 渲染框架）
- **打包**：單檔 Bundle（cli.js），僅 `@img/sharp` 為 optional 原生相依

---

## 二、核心架構分層

```
┌──────────────────────────────────────────────────┐
│                  CLI 入口層                        │
│   entrypoints/cli.tsx   main.tsx                  │
└────────────────────────┬─────────────────────────┘
                         │
┌────────────────────────▼─────────────────────────┐
│               命令與互動層 (Commands)               │
│   ~86 個子命令目錄 + 15 個頂層 .ts 檔               │
│   (commit, review, config, mcp, plan, voice...)   │
└────────────────────────┬─────────────────────────┘
                         │
┌────────────────────────▼─────────────────────────┐
│              查詢引擎 (Query Engine)                │
│   QueryEngine.ts  query.ts  query/                │
│   負責 Agent 推理循環、工具呼叫、回應處理             │
└────────────────────────┬─────────────────────────┘
                         │
┌────────────────────────▼─────────────────────────┐
│                工具系統 (Tools)                    │
│   42 個工具目錄，各有獨立實作                        │
└──────────┬─────────────────┬──────────────────────┘
           │                 │
┌──────────▼──────┐ ┌───────▼───────────────────────┐
│  服務層(Services) │ │     工具函式層(Utils)            │
│  API、MCP、LSP、 │ │  298+ 工具函式，涵蓋檔案、Git、  │
│  Auth、Analytics│ │  Auth、Sessions、Shell、AI 等   │
└─────────────────┘ └───────────────────────────────┘
```

---

## 三、目錄結構詳解

### 3.1 入口點（`entrypoints/`）

| 檔案 | 說明 |
|------|------|
| `cli.tsx` | 主 CLI 入口，解析參數、初始化 Ink 渲染 |
| `init.ts` | 初始化流程（首次使用設定） |
| `mcp.ts` | 作為 MCP Server 啟動的入口 |
| `agentSdkTypes.ts` | Agent SDK 的型別定義 |
| `sandboxTypes.ts` | 沙箱模式型別定義 |
| `sdk/` | SDK 模式入口（非互動式使用） |

### 3.2 命令系統（`commands/`）—— 86+ 個子命令

#### 核心操作命令
| 命令 | 說明 |
|------|------|
| `commit.ts` | Git commit（含 AI 生成 commit message） |
| `commit-push-pr.ts` | Commit + Push + 建立 PR |
| `review.ts` / `review/` | 程式碼審查 |
| `init.ts` | 專案初始化 |
| `install.tsx` | 安裝/設定 |

#### 互動功能命令
| 命令 | 說明 |
|------|------|
| `compact/` | 壓縮對話脈絡 |
| `plan/` | 計畫模式（Ultra Plan）|
| `ultraplan.tsx` | 進階計畫功能 |
| `thinkback/` | 回顧思考過程 |
| `rewind/` | 回溯到上一個狀態 |

#### 系統/設定命令
| 命令 | 說明 |
|------|------|
| `config/` | 設定管理（讀取/寫入） |
| `mcp/` | MCP 伺服器管理 |
| `model/` | AI 模型選擇 |
| `memory/` | 記憶管理 |
| `plugin/` | 外掛管理 |
| `hooks/` | Hooks 設定 |
| `permissions/` | 權限管理 |
| `login/` / `logout/` | 認證 |

#### 開發/分析命令
| 命令 | 說明 |
|------|------|
| `stats/` | 使用統計 |
| `cost/` | 費用分析 |
| `doctor/` | 診斷工具 |
| `debug-tool-call/` | 工具呼叫除錯 |
| `insights.ts` | 深度分析（115KB！）|
| `security-review.ts` | 安全審查 |

#### 整合命令
| 命令 | 說明 |
|------|------|
| `ide/` | IDE 整合 |
| `desktop/` | 桌面應用整合 |
| `mobile/` | 行動裝置整合 |
| `chrome/` | Chrome 整合 |
| `voice/` | 語音整合 |
| `teleport/` | Teleport（跨環境工作） |
| `install-github-app/` | GitHub App 安裝 |
| `install-slack-app/` | Slack App 安裝 |

### 3.3 工具系統（`tools/`）—— 42 個工具

#### 核心 Agent 工具
| 工具 | 說明 |
|------|------|
| `AgentTool/` | 子 Agent 發起（Spawning） |
| `BashTool/` | 執行 Shell 命令 |
| `FileReadTool/` | 讀取檔案 |
| `FileWriteTool/` | 寫入檔案 |
| `FileEditTool/` | 編輯檔案（精準替換） |
| `GrepTool/` | 正規表達式搜尋 |
| `GlobTool/` | 檔案路徑模式比對 |
| `WebFetchTool/` | 網頁抓取 |
| `WebSearchTool/` | 網路搜尋 |

#### 協作/任務工具
| 工具 | 說明 |
|------|------|
| `TaskCreateTool/` | 建立非同步任務 |
| `TaskGetTool/` | 取得任務狀態 |
| `TaskListTool/` | 列出所有任務 |
| `TaskUpdateTool/` | 更新任務 |
| `TaskStopTool/` | 停止任務 |
| `TaskOutputTool/` | 取得任務輸出 |
| `SendMessageTool/` | Agent 間傳訊 |
| `TeamCreateTool/` | 建立 Agent 團隊 |
| `TeamDeleteTool/` | 解散 Agent 團隊 |

#### 整合工具
| 工具 | 說明 |
|------|------|
| `MCPTool/` | MCP 協定工具 |
| `McpAuthTool/` | MCP 認證 |
| `ListMcpResourcesTool/` | 列出 MCP 資源 |
| `ReadMcpResourceTool/` | 讀取 MCP 資源 |
| `LSPTool/` | 語言伺服器整合 |
| `NotebookEditTool/` | Jupyter 編輯 |
| `PowerShellTool/` | Windows PowerShell |
| `REPLTool/` | 互動式 REPL |

#### 系統工具
| 工具 | 說明 |
|------|------|
| `ConfigTool/` | 設定管理 |
| `SkillTool/` | Skills 系統 |
| `TodoWriteTool/` | TODO 管理 |
| `AskUserQuestionTool/` | 向使用者提問 |
| `BriefTool/` | 摘要工具 |
| `SleepTool/` | 等待/延遲 |
| `ToolSearchTool/` | 搜尋可用工具 |
| `EnterPlanModeTool/` | 進入計畫模式 |
| `ExitPlanModeTool/` | 退出計畫模式 |
| `EnterWorktreeTool/` | 進入 Git Worktree |
| `ExitWorktreeTool/` | 退出 Worktree |
| `ScheduleCronTool/` | 排程任務 |
| `RemoteTriggerTool/` | 遠端觸發 |
| `SyntheticOutputTool/` | 合成輸出（測試用） |

### 3.4 服務層（`services/`）

| 服務 | 說明 |
|------|------|
| `api/` | Anthropic API 呼叫封裝 |
| `mcp/` | MCP 協定伺服器實作 |
| `lsp/` | Language Server Protocol |
| `oauth/` | OAuth 認證流程 |
| `analytics/` | 使用情況分析 |
| `compact/` | 對話壓縮 |
| `extractMemories/` | 記憶提取 |
| `plugins/` | 外掛載入管理 |
| `settingsSync/` | 設定同步 |
| `teamMemorySync/` | 團隊記憶同步 |
| `AgentSummary/` | Agent 摘要生成 |
| `MagicDocs/` | 自動文件生成 |
| `PromptSuggestion/` | 提示詞建議 |
| `SessionMemory/` | 會話記憶 |
| `autoDream/` | Auto Dream 功能 |
| `policyLimits/` | 政策限制 |
| `remoteManagedSettings/` | 遠端管理設定 |
| `toolUseSummary/` | 工具使用摘要 |
| `voice.ts` | 語音服務（~17KB）|
| `voiceStreamSTT.ts` | 串流語音轉文字（~21KB）|

### 3.5 UI 元件（`components/`）—— 113+ 個元件

使用 **Ink**（Terminal React）渲染，主要元件：

```
重量級元件（檔案較大）：
- LogSelector.tsx     (200KB) - 日誌選擇器
- ScrollKeybindingHandler.tsx (149KB) - 滾動鍵盤綁定
- Messages.tsx        (147KB) - 訊息列表
- Stats.tsx           (152KB) - 統計顯示
- MessageSelector.tsx (115KB) - 訊息選擇
- ConsoleOAuthFlow.tsx (79KB) - OAuth 流程
- Message.tsx         (79KB)  - 單條訊息

主要功能元件：
- FullscreenLayout.tsx  - 全螢幕佈局
- PromptInput/          - 提示詞輸入框
- ModelPicker.tsx       - 模型選擇器
- ThemePicker.tsx       - 主題選擇器
- StatusLine.tsx        - 狀態列
- Feedback.tsx          - 回饋功能
- Messages.tsx          - 訊息顯示
- BridgeDialog.tsx      - IDE 橋接對話框
- CoordinatorAgentStatus.tsx - 多 Agent 狀態
- ContextVisualization.tsx   - 上下文視覺化
```

### 3.6 狀態管理（`state/`）

```
state/
├── AppState.tsx        # React 狀態樹型別定義
├── AppStateStore.ts    # Zustand/自製狀態 Store
├── onChangeAppState.ts # 狀態變更監聽
├── selectors.ts        # 狀態選擇器
├── store.ts            # Store 建立
└── teammateViewHelpers.ts # 多 Agent 視圖輔助
```

### 3.7 Vendor 模組（`restored-src/vendor/`）

自製的原生 Node.js Addon 模組：

| 模組 | 說明 |
|------|------|
| `audio-capture-src/` | 麥克風音訊擷取（語音輸入） |
| `image-processor-src/` | 圖片處理（截圖、剪貼板） |
| `modifiers-napi-src/` | 按鍵修飾鍵偵測（原生快捷鍵） |
| `url-handler-src/` | 系統 URL 處理（Deep Link） |

---

## 四、特殊功能模組

### 4.1 多 Agent 架構（Coordinator）

```
coordinator/coordinatorMode.ts   # 多 Agent 協調中樞
tools/AgentTool/                 # 子 Agent 派生工具
tools/SendMessageTool/           # Agent 間訊息傳遞
tools/TeamCreateTool/            # 動態建立 Agent 團隊
utils/forkedAgent.ts             # 分叉 Agent 實作
utils/swarm/                     # Swarm 模式（Agent 群）
```

### 4.2 Skills 系統

```
skills/
├── bundledSkills.ts      # 內建 Skills 定義
├── loadSkillsDir.ts      # 從目錄載入 Skills
├── mcpSkillBuilders.ts   # MCP 型 Skill 建構器
└── bundled/              # 實際內建 Skill 實作
```

### 4.3 Teleport（跨環境工作）

- `utils/teleport.tsx`（175KB！）：核心 Teleport 邏輯
- `commands/teleport/`：Teleport 命令
- `components/TeleportProgress.tsx`、`TeleportError.tsx`：Teleport UI

### 4.4 Buddy（AI 伴侶）

```
buddy/
├── CompanionSprite.tsx   # 像素風格 AI 角色動畫（~46KB）
├── companion.ts          # 伴侶互動邏輯
├── sprites.ts            # Sprite 動畫幀
└── useBuddyNotification.tsx  # 通知系統
```

### 4.5 Worktree 模式

- `tools/EnterWorktreeTool/`、`ExitWorktreeTool/`
- `utils/worktree.ts`（~50KB）
- `utils/getWorktreePaths.ts`
- 支援 git worktree 的多任務並行工作

### 4.6 Plan Mode（計畫模式）

- `tools/EnterPlanModeTool/`、`ExitPlanModeTool/`
- `commands/plan/`
- `commands/ultraplan.tsx`（66KB）
- `utils/plans.ts`

---

## 五、核心資料流

```
使用者輸入
    │
    ▼
entrypoints/cli.tsx
    │  解析 CLI 參數、處理斜線命令
    ▼
commands/ (對應子命令)
    │  或直接進入對話模式
    ▼
QueryEngine.ts           ←→  services/api/
    │  管理訊息佇列、推理循環        (Anthropic API)
    │
    ├─→ Tool 呼叫決策
    │       │
    │       ▼
    │   tools/ (各工具執行)
    │       │
    │       ├── BashTool → Shell 執行
    │       ├── FileEditTool → 檔案操作
    │       ├── AgentTool → 派生子 Agent
    │       ├── MCPTool → MCP 伺服器
    │       └── WebSearchTool → 網路搜尋
    │
    ▼
components/ (Ink React 渲染)
    │  Terminal UI 顯示
    ▼
使用者觀看輸出
```

---

## 六、值得注意的細節

### 巨大的 main.tsx（~804KB）

`src/main.tsx` 與 `src/query.ts`（68KB）是整個應用的核心，包含：
- 主要的 React 元件樹
- 完整的查詢循環邏輯
- 工具結果處理

### hooks.ts（~159KB）

`utils/hooks.ts` 包含大量 React Hooks，幾乎所有狀態邏輯都在這裡。

### messages.ts（~193KB）

`utils/messages.ts` 定義完整的 Anthropic API 訊息格式與轉換邏輯。

### 語音功能的完整實作

包含完整的 STT（語音轉文字）串流處理：
- `vendor/audio-capture-src/`：C++ 原生音訊擷取
- `services/voiceStreamSTT.ts`：串流 STT
- `services/voice.ts`：語音服務核心

---

## 七、架構特點總結

| 特點 | 說明 |
|------|------|
| **Terminal-First** | 使用 Ink 框架在 Terminal 渲染完整 React UI |
| **多 Agent 支援** | 內建 Coordinator、Team、Swarm 等多 Agent 協作模式 |
| **插件化設計** | Skills 系統、Plugin 系統、MCP 協定三層可擴充 |
| **跨平台** | 支援 macOS、Linux、Windows（PowerShell），含原生 Addon |
| **IDE 整合** | VS Code、JetBrains、桌面版等多種 IDE 橋接 |
| **語音功能** | 完整語音輸入 Pipeline（原生音訊擷取 + STT） |
| **離線 Bundle** | 完全打包成單一 cli.js，零外部相依（除 sharp） |
| **Enterprise 就緒** | 支援 OAuth、AWS Bedrock、Proxy、mTLS、政策控制 |
