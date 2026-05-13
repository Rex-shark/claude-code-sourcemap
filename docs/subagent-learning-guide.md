# 全端開發者學習指南：打造 Code Writer + Code Reviewer Subagent 系統

> **目標**：從 Claude Code 原始碼中，學習如何打造「AI 設計工作流程，自動完成與修正任務」的 subagent 系統  
> **適合對象**：熟悉 TypeScript / JS 全端開發，目標是建立雙 Agent（寫 code + 審查 code）協作系統

---

## 一、你的目標系統架構（先理解設計）

```
使用者需求
    ↓
[Coordinator]          ← 理解需求、設計任務、分派工作
    ├── [Writer Agent]  ← 負責寫 code（有完整工具存取權）
    └── [Reviewer Agent] ← 負責審查 code（唯讀，挑毛病，輸出 VERDICT）
         ↓
    如果 FAIL → Coordinator 把問題回傳給 Writer，重新執行
    如果 PASS → 任務完成，回報使用者
```

這個架構在 Claude Code 原始碼中已有完整範本可以參考：
- `Coordinator` → `coordinatorMode.ts`
- `Writer` → `generalPurposeAgent.ts`（改造）
- `Reviewer` → `verificationAgent.ts`（幾乎可直接借用）

---

## 二、最值得學習的原始碼（依優先順序）

### ⭐⭐⭐ 最高優先 — 直接可抄的 Agent 結構

#### 1. `verificationAgent.ts` → 你的 **Reviewer Agent 藍本**

**路徑**：`restored-src/src/tools/AgentTool/built-in/verificationAgent.ts`

這個 Agent 的設計是全部裡面最精彩的。幾個值得偷學的技術：

**① 明確列出「AI 會犯的錯誤」並要求對抗它：**
```
You have two documented failure patterns:
1. Verification avoidance: you find reasons not to run checks
2. Being seduced by the first 80%: you see a polished result and skip edge cases
```
→ **學習重點**：Reviewer 的 system prompt 要明確描述「reviewer 的常見失誤」。
光是「請認真審查」不夠，要告訴 AI 它的心理偏誤是什麼。

**② 強制輸出格式（structured output）：**
```
### Check: [what you're verifying]
**Command run:** [exact command]
**Output observed:** [copy-paste output]
**Result: PASS / FAIL**

End with: VERDICT: PASS / VERDICT: FAIL / VERDICT: PARTIAL
```
→ **學習重點**：Coordinator 要能解析 VERDICT 關鍵字，決定是否重試。
這是讓自動化工作流程成立的關鍵。

**③ `criticalSystemReminder_EXPERIMENTAL` 欄位：**
```typescript
criticalSystemReminder_EXPERIMENTAL:
  'CRITICAL: You CANNOT edit files. You MUST end with VERDICT: PASS/FAIL/PARTIAL.'
```
→ **學習重點**：這個欄位的內容會在**每個 user turn 都被重新注入**，確保 AI 不會忘記關鍵限制。

---

#### 2. `coordinatorMode.ts` → 你的 **Coordinator 藍本**

**路徑**：`restored-src/src/coordinator/coordinatorMode.ts`

最重要的設計原則（完整引用）：

```
Workers can't see your conversation. Every prompt must be self-contained.

Never write "based on your findings" — this delegates understanding to the worker.
You must synthesize findings yourself.

GOOD: "Fix the null pointer in src/auth/validate.ts:42. The user field on Session 
is undefined when sessions expire... Add a null check before user.id access — 
if null, return 401 with 'Session expired'. Commit and report the hash."

BAD: "Based on your findings, fix the auth bug"
```

→ **學習重點**：Coordinator 在呼叫 Writer 之前，必須自己整合所有上下文，寫出**完全自給自足的 prompt**。

**並行策略（Parallelism is your superpower）：**
```
Launch independent workers concurrently whenever possible.
To launch workers in parallel, make multiple tool calls in a single message.
```
→ **學習重點**：如果有多個獨立子任務，Coordinator 應同時啟動多個 Writer，不要序列化。

---

#### 3. `generalPurposeAgent.ts` → 你的 **Writer Agent 藍本**

**路徑**：`restored-src/src/tools/AgentTool/built-in/generalPurposeAgent.ts`

```
Complete the task fully — don't gold-plate, but don't leave it half-done.

Guidelines:
- NEVER create files unless absolutely necessary.
- NEVER proactively create documentation files.
```

→ **學習重點**：Writer 要有明確的「完成」定義，避免多做或少做。

---

### ⭐⭐ 高優先 — 理解 Agent 定義格式

#### 4. `loadAgentsDir.ts` → Agent 的 **Schema 和欄位定義**

**路徑**：`restored-src/src/tools/AgentTool/loadAgentsDir.ts`

這個檔案定義了一個 Agent 可以有哪些欄位，是你設計自己 Agent 時的規格書：

```typescript
type BaseAgentDefinition = {
  agentType: string          // agent 的識別名稱
  whenToUse: string          // Coordinator 看到這個，決定何時呼叫這個 agent
  tools?: string[]           // 允許使用的工具白名單（'*' 表示全部）
  disallowedTools?: string[] // 禁止使用的工具黑名單
  model?: string             // 'inherit' | 'haiku' | 'sonnet' | 完整 model ID
  permissionMode?: PermissionMode  // 'default' | 'acceptEdits' | 'bypassPermissions'
  maxTurns?: number          // 最多幾輪對話
  background?: boolean       // 是否在背景非同步執行
  criticalSystemReminder_EXPERIMENTAL?: string  // 每輪都重新注入的關鍵提醒
  omitClaudeMd?: boolean     // 唯讀 agent 可跳過 CLAUDE.md，省 token
  getSystemPrompt: () => string  // 或傳入 toolUseContext 版本
}
```

**Agent 定義的兩種格式：**

**格式一：Markdown 檔案**（放在 `.claude/agents/` 目錄）

```markdown
---
name: code-reviewer
description: >
  Use this agent to review code changes before committing.
  Pass the diff or list of changed files.
model: sonnet
permissionMode: default
disallowedTools: Write, Edit, Bash
maxTurns: 20
---

You are a code reviewer...（system prompt 正文）
```

**格式二：TypeScript BuiltInAgentDefinition**（適合程式化整合）

```typescript
export const MY_AGENT: BuiltInAgentDefinition = {
  agentType: 'code-reviewer',
  whenToUse: 'Use when you need to review code changes...',
  model: 'sonnet',
  disallowedTools: ['Write', 'Edit'],
  getSystemPrompt: () => `You are a code reviewer...`,
}
```

---

#### 5. `runAgent.ts` → 理解 Agent **執行機制**

**路徑**：`restored-src/src/tools/AgentTool/runAgent.ts`

幾個重要的執行機制概念：

**① 同步 vs 非同步 Agent：**
```typescript
isAsync: boolean
// isAsync=false → 同步，分享父 agent 的 abortController
// isAsync=true  → 非同步，獨立的 AbortController，不能顯示 permission 提示
```

**② Agent 工具隔離（tool scoping）：**
```typescript
// 子 agent 只能使用 allowedTools 裡的工具，不繼承父 agent 的工具批准
if (allowedTools !== undefined) {
  toolPermissionContext = {
    alwaysAllowRules: { session: [...allowedTools] }
  }
}
```
→ **學習重點**：Reviewer Agent 的工具黑名單（Write, Edit）在這裡執行，不是靠 AI 自律，而是靠底層強制過濾。

**③ 上下文傳遞：**
```typescript
// 唯讀 agent 不需要 CLAUDE.md
const shouldOmitClaudeMd = agentDefinition.omitClaudeMd && ...
// 唯讀 agent 不需要 git status（可能 40KB）
const resolvedSystemContext = 
  agentType === 'Explore' || agentType === 'Plan'
    ? systemContextNoGit : baseSystemContext
```
→ **學習重點**：不同 agent 可以精確控制收到哪些上下文，節省 token。

---

### ⭐ 中優先 — 提示詞工程參考

#### 6. `constants/prompts.ts` → **主 System Prompt 的組裝方式**

**路徑**：`restored-src/src/constants/prompts.ts`

主要學習點：

- **靜態/動態分層**：靜態部分（角色、規則）放在 `__SYSTEM_PROMPT_DYNAMIC_BOUNDARY__` 前，可被快取。動態部分（環境、記憶）每次重新計算。
- **Section 化組裝**：每個區塊是獨立的 `get...Section()` 函式，組合時按順序串接。
- **環境資訊注入**：`enhanceSystemPromptWithEnvDetails()` 動態追加工作目錄、平台、模型資訊。

---

## 三、你要撰寫的兩個 Agent 之 System Prompt 設計

### 3.1 Writer Agent System Prompt

```markdown
---
name: code-writer
description: >
  Software implementation agent. Use this when you need to write, modify, or 
  create code. Pass a fully self-contained spec: what to build, where the files 
  are, what existing patterns to follow, and what the expected behavior is.
  Returns a summary of changes made and files modified.
model: sonnet
tools: Read, Write, Edit, Bash, Glob, Grep, TodoWrite
permissionMode: acceptEdits
maxTurns: 50
---

You are a software implementation specialist. Your job is to write correct, 
clean, and complete code.

## Core Principles
- Complete the task fully — don't gold-plate, but don't leave it half-done.
- Read files before modifying them. Never propose changes to code you haven't read.
- Follow existing patterns in the codebase. Don't introduce new conventions unless asked.
- NEVER add features, refactor, or "improve" beyond what was asked.
- Three similar lines of code is better than a premature abstraction.

## When Finished
Report:
1. Files created/modified
2. What was implemented
3. Any known limitations or follow-up needed

Do NOT run tests or verify your own work — that is the reviewer's job.
```

---

### 3.2 Reviewer Agent System Prompt

```markdown
---
name: code-reviewer
description: >
  Code review agent. Use this AFTER the code-writer completes a task. Pass the 
  original task description, list of files changed, and approach taken. Returns 
  VERDICT: PASS, FAIL, or PARTIAL.
model: sonnet
disallowedTools: Write, Edit, NotebookEdit
permissionMode: default  
maxTurns: 30
criticalSystemReminder: "CRITICAL: You CANNOT edit files. You MUST end with VERDICT: PASS, VERDICT: FAIL, or VERDICT: PARTIAL."
---

You are a code review specialist. Your job is NOT to confirm the implementation 
works — it's to try to find problems with it.

## Your Documented Failure Modes (Fight These)
1. **Review avoidance**: You read code, think it looks fine, and write PASS 
   without running anything. Reading is NOT review. RUN the code.
2. **Happy path bias**: You test the main flow, it works, you write PASS. 
   But you didn't test edge cases, error paths, or concurrent access.

## Review Checklist
1. Read the task requirement — understand what "correct" means
2. Read the changed files
3. Run the code / tests
4. Test edge cases and error paths
5. Check for security issues (injection, XSS, auth bypass)
6. Check for regressions in related code

## Output Format (Required)
Every check MUST include a command that was actually run:

### Check: [what you're verifying]
**Command run:** [exact command]
**Output observed:** [actual output, not paraphrased]
**Result: PASS** (or **FAIL** — with what you expected vs what you got)

## Verdict
End your response with EXACTLY one of:
VERDICT: PASS
VERDICT: FAIL  
VERDICT: PARTIAL
```

---

## 四、Coordinator 的工作流程設計

這是讓系統能「自動修正」的關鍵。Coordinator 需要能解析 VERDICT 並決定是否重試：

```typescript
// 偽代碼：Coordinator 的任務迴圈
async function runTaskWithAutoFix(task: string, maxRetries = 3) {
  let attempt = 0
  
  while (attempt < maxRetries) {
    // 1. 呼叫 Writer Agent
    const writerResult = await runAgent('code-writer', {
      prompt: buildWriterPrompt(task, previousFeedback)
    })
    
    // 2. 呼叫 Reviewer Agent
    const reviewerResult = await runAgent('code-reviewer', {
      prompt: buildReviewerPrompt(task, writerResult.filesChanged, writerResult.summary)
    })
    
    // 3. 解析 VERDICT
    const verdict = parseVerdict(reviewerResult.output)
    
    if (verdict === 'PASS') {
      return { success: true, result: writerResult }
    }
    
    if (verdict === 'PARTIAL') {
      // 環境問題，提示使用者手動確認
      return { success: false, needsManualVerification: true }
    }
    
    // FAIL → 把 reviewer 的意見整合回 Writer 的下一次 prompt
    previousFeedback = extractFailureDetails(reviewerResult.output)
    attempt++
  }
  
  return { success: false, error: 'Max retries exceeded' }
}

// 關鍵：Reviewer Prompt 要自給自足
function buildReviewerPrompt(task, filesChanged, writerSummary) {
  return `
Original task: ${task}

The code-writer made the following changes:
${writerSummary}

Files modified:
${filesChanged.join('\n')}

Please review these changes and verify they correctly implement the task.
`
}
```

---

## 五、實作路徑建議（給全端開發者）

### 方案 A：使用 Claude Code 的 `.claude/agents/` 機制（最快）

這是 Claude Code v2.1.88 已支援的功能，不需要寫任何 TypeScript：

```
你的專案/
├── .claude/
│   └── agents/
│       ├── code-writer.md    ← Writer Agent 定義
│       └── code-reviewer.md  ← Reviewer Agent 定義
└── CLAUDE.md                 ← Coordinator 的工作流程說明
```

在 `CLAUDE.md` 裡告訴 Coordinator 何時使用這些 agents：
```markdown
# 工作流程

當收到程式碼實作任務時：
1. 使用 code-writer agent 實作功能
2. 使用 code-reviewer agent 審查結果
3. 若 VERDICT: FAIL，將 reviewer 的意見整合後，再次呼叫 code-writer
4. 重複直到 VERDICT: PASS 或達到 3 次上限
```

---

### 方案 B：使用 Claude Agent SDK（有完整程式控制）

Anthropic 提供了官方 SDK（Node.js/TypeScript/Python），可以程式化地呼叫 agent：

```typescript
import Anthropic from '@anthropic-ai/sdk'

const client = new Anthropic()

// 呼叫 Writer
const writerResponse = await client.messages.create({
  model: 'claude-sonnet-4-5',
  max_tokens: 8096,
  system: WRITER_SYSTEM_PROMPT,
  messages: [{ role: 'user', content: writerTask }]
})

// 呼叫 Reviewer
const reviewerResponse = await client.messages.create({
  model: 'claude-sonnet-4-5', 
  max_tokens: 4096,
  system: REVIEWER_SYSTEM_PROMPT,
  messages: [{ role: 'user', content: reviewerTask }]
})

// 解析 VERDICT
const verdict = reviewerResponse.content
  .find(b => b.type === 'text')?.text
  .match(/VERDICT: (PASS|FAIL|PARTIAL)/)?.[1]
```

---

### 方案 C：Fork Claude Code 原始碼（最深度客製化）

如果你想要修改 Agent 執行機制本身：

1. 在 `src/tools/AgentTool/built-in/` 下新增你的 Agent TypeScript 檔案
2. 在 `src/tools/AgentTool/builtInAgents.ts` 中註冊
3. 修改 `src/coordinator/coordinatorMode.ts` 加入重試邏輯

---

## 六、最容易踩到的坑（從原始碼中發現的）

| 問題 | 原始碼中的解法 |
|------|--------------|
| Reviewer 不執行命令，只看程式碼就給 PASS | 在 prompt 中列出「你會犯的藉口」，要求對抗 |
| Reviewer 修改了程式碼 | 用 `disallowedTools` 在**底層強制禁止**，不靠 AI 自律 |
| Writer 做太多「改善」 | 明確寫「不要加功能、不要重構超出範圍」 |
| Coordinator 寫 lazy prompt 給 Worker | 明確禁止「based on your findings」這類語句 |
| Context 太長浪費 token | 唯讀 agent 用 `omitClaudeMd: true` 跳過 CLAUDE.md |
| AI 忘記關鍵限制 | 使用 `criticalSystemReminder_EXPERIMENTAL` 每輪重新注入 |
| Reviewer 環境問題無法驗證 | 設立 VERDICT: PARTIAL 讓 Coordinator 知道需人工確認 |

---

## 七、學習重點整理

1. **結構化輸出是自動化的基礎**：`VERDICT: PASS/FAIL/PARTIAL` 讓 Coordinator 能程式化解析結果
2. **工具禁止要用底層機制**：`disallowedTools` 在執行層強制，不靠 prompt 約束
3. **Prompt 要自給自足**：每個 Agent 的 prompt 必須包含所有它需要的資訊
4. **明確列出 AI 的失敗模式**：Verification Agent 的設計哲學——預設 AI 會逃避驗證，明確告訴它會有什麼藉口
5. **快取優化**：靜態 prompt 和動態 context 分開，降低每次呼叫的成本
