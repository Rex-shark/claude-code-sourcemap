# Claude Code 指令 Prompt 彙整

> **版本**：`@anthropic-ai/claude-code v2.1.88`
> **來源**：`restored-src/src/commands/` 目錄下各指令檔案
> **用途**：記錄每個 slash command 背後實際送給模型的 Prompt 原文與分析

---

## 目錄

- [/init](#init)
- [/init-verifiers](#init-verifiers)
- [/commit](#commit)
- [/commit-push-pr](#commit-push-pr)
- [/review](#review)
- [/ultrareview](#ultrareview)
- [/security-review](#security-review)
- [/pr-comments](#pr-comments)
- [/insights](#insights)
- [/compact](#compact)
- [/ultraplan](#ultraplan)
- [/statusline](#statusline)
- [/brief](#brief)
- [/thinkback](#thinkback)
- [其他指令速查](#其他指令速查)

---

## /init

**來源檔案**：`restored-src/src/commands/init.ts`
**類型**：`prompt`（將 Prompt 文字注入對話，由模型執行）
**進度訊息**：`analyzing your codebase`

### 版本切換邏輯

```ts
feature('NEW_INIT') &&
  (process.env.USER_TYPE === 'ant' || isEnvTruthy(process.env.CLAUDE_CODE_NEW_INIT))
    ? NEW_INIT_PROMPT
    : OLD_INIT_PROMPT
```

- **舊版（OLD_INIT_PROMPT）**：所有一般使用者預設使用
- **新版（NEW_INIT_PROMPT）**：僅限 Anthropic 內部員工（`USER_TYPE === 'ant'`）或手動設定環境變數 `CLAUDE_CODE_NEW_INIT=true`

---

### OLD_INIT_PROMPT（舊版）

#### 原文

```
Please analyze this codebase and create a CLAUDE.md file, which will be given to future instances of Claude Code to operate in this repository.

What to add:
1. Commands that will be commonly used, such as how to build, lint, and run tests. Include the necessary commands to develop in this codebase, such as how to run a single test.
2. High-level code architecture and structure so that future instances can be productive more quickly. Focus on the "big picture" architecture that requires reading multiple files to understand.

Usage notes:
- If there's already a CLAUDE.md, suggest improvements to it.
- When you make the initial CLAUDE.md, do not repeat yourself and do not include obvious instructions like "Provide helpful error messages to users", "Write unit tests for all new utilities", "Never include sensitive information (API keys, tokens) in code or commits".
- Avoid listing every component or file structure that can be easily discovered.
- Don't include generic development practices.
- If there are Cursor rules (in .cursor/rules/ or .cursorrules) or Copilot rules (in .github/copilot-instructions.md), make sure to include the important parts.
- If there is a README.md, make sure to include the important parts.
- Do not make up information such as "Common Development Tasks", "Tips for Development", "Support and Documentation" unless this is expressly included in other files that you read.
- Be sure to prefix the file with the following text:

# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.
```

#### 重點摘要

| 項目 | 說明 |
|------|------|
| 目標 | 建立 `CLAUDE.md` 供後續 Claude Code 工作階段使用 |
| 包含內容 | 常用指令（build/lint/test）、高層架構 |
| 整合來源 | Cursor rules、Copilot rules、README.md |
| 禁止事項 | 不加泛用建議、不列檔案結構、不虛構章節 |

---

### NEW_INIT_PROMPT（新版 — 8 階段互動流程）

#### 總覽

新版 `/init` 是一個結構化的 8 階段互動流程，透過 `AskUserQuestion` 工具與使用者對話，產出量身定做的設定。

| 階段 | 名稱 | 說明 |
|------|------|------|
| Phase 1 | Ask what to set up | 詢問要建立哪些檔案（CLAUDE.md / CLAUDE.local.md / 兩者），以及是否設定 skills 和 hooks |
| Phase 2 | Explore the codebase | 用 subagent 掃描 manifest、README、CI、AI 設定檔、formatter 等 |
| Phase 3 | Fill in the gaps | 用 AskUserQuestion 補問程式碼無法回答的問題，並以 preview 呈現提案 |
| Phase 4 | Write CLAUDE.md | 撰寫專案級 CLAUDE.md（篩選原則：拿掉會讓 Claude 犯錯才保留） |
| Phase 5 | Write CLAUDE.local.md | 撰寫個人偏好檔，加入 .gitignore |
| Phase 6 | Suggest and create skills | 建立可重用的 skill 工作流程 |
| Phase 7 | Suggest additional optimizations | 建議 GitHub CLI、Linting、format-on-edit hook 等 |
| Phase 8 | Summary and next steps | 總結設定內容，提供後續優化建議 |

#### 原文

```
Set up a minimal CLAUDE.md (and optionally skills and hooks) for this repo. CLAUDE.md is loaded into every Claude Code session, so it must be concise — only include what Claude would get wrong without it.

## Phase 1: Ask what to set up

Use AskUserQuestion to find out what the user wants:

- "Which CLAUDE.md files should /init set up?"
  Options: "Project CLAUDE.md" | "Personal CLAUDE.local.md" | "Both project + personal"
  Description for project: "Team-shared instructions checked into source control — architecture, coding standards, common workflows."
  Description for personal: "Your private preferences for this project (gitignored, not shared) — your role, sandbox URLs, preferred test data, workflow quirks."

- "Also set up skills and hooks?"
  Options: "Skills + hooks" | "Skills only" | "Hooks only" | "Neither, just CLAUDE.md"
  Description for skills: "On-demand capabilities you or Claude invoke with `/skill-name` — good for repeatable workflows and reference knowledge."
  Description for hooks: "Deterministic shell commands that run on tool events (e.g., format after every edit). Claude can't skip them."

## Phase 2: Explore the codebase

Launch a subagent to survey the codebase, and ask it to read key files to understand the project: manifest files (package.json, Cargo.toml, pyproject.toml, go.mod, pom.xml, etc.), README, Makefile/build configs, CI config, existing CLAUDE.md, .claude/rules/, AGENTS.md, .cursor/rules or .cursorrules, .github/copilot-instructions.md, .windsurfrules, .clinerules, .mcp.json.

Detect:
- Build, test, and lint commands (especially non-standard ones)
- Languages, frameworks, and package manager
- Project structure (monorepo with workspaces, multi-module, or single project)
- Code style rules that differ from language defaults
- Non-obvious gotchas, required env vars, or workflow quirks
- Existing .claude/skills/ and .claude/rules/ directories
- Formatter configuration (prettier, biome, ruff, black, gofmt, rustfmt, or a unified format script like `npm run format` / `make fmt`)
- Git worktree usage: run `git worktree list` to check if this repo has multiple worktrees (only relevant if the user wants a personal CLAUDE.local.md)

Note what you could NOT figure out from code alone — these become interview questions.

## Phase 3: Fill in the gaps

Use AskUserQuestion to gather what you still need to write good CLAUDE.md files and skills. Ask only things the code can't answer.

If the user chose project CLAUDE.md or both: ask about codebase practices — non-obvious commands, gotchas, branch/PR conventions, required env setup, testing quirks. Skip things already in README or obvious from manifest files. Do not mark any options as "recommended" — this is about how their team works, not best practices.

If the user chose personal CLAUDE.local.md or both: ask about them, not the codebase. Do not mark any options as "recommended" — this is about their personal preferences, not best practices. Examples of questions:
  - What's their role on the team? (e.g., "backend engineer", "data scientist", "new hire onboarding")
  - How familiar are they with this codebase and its languages/frameworks? (so Claude can calibrate explanation depth)
  - Do they have personal sandbox URLs, test accounts, API key paths, or local setup details Claude should know?
  - Only if Phase 2 found multiple git worktrees: ask whether their worktrees are nested inside the main repo (e.g., `.claude/worktrees/<name>/`) or siblings/external (e.g., `../myrepo-feature/`). If nested, the upward file walk finds the main repo's CLAUDE.local.md automatically — no special handling needed. If sibling/external, the personal content should live in a home-directory file (e.g., `~/.claude/<project-name>-instructions.md`) and each worktree gets a one-line CLAUDE.local.md stub that imports it: `@~/.claude/<project-name>-instructions.md`. Never put this import in the project CLAUDE.md — that would check a personal reference into the team-shared file.
  - Any communication preferences? (e.g., "be terse", "always explain tradeoffs", "don't summarize at the end")

**Synthesize a proposal from Phase 2 findings** — e.g., format-on-edit if a formatter exists, a `/verify` skill if tests exist, a CLAUDE.md note for anything from the gap-fill answers that's a guideline rather than a workflow. For each, pick the artifact type that fits, **constrained by the Phase 1 skills+hooks choice**:

  - **Hook** (stricter) — deterministic shell command on a tool event; Claude can't skip it. Fits mechanical, fast, per-edit steps: formatting, linting, running a quick test on the changed file.
  - **Skill** (on-demand) — you or Claude invoke `/skill-name` when you want it. Fits workflows that don't belong on every edit: deep verification, session reports, deploys.
  - **CLAUDE.md note** (looser) — influences Claude's behavior but not enforced. Fits communication/thinking preferences: "plan before coding", "be terse", "explain tradeoffs".

  **Respect Phase 1's skills+hooks choice as a hard filter**: if the user picked "Skills only", downgrade any hook you'd suggest to a skill or a CLAUDE.md note. If "Hooks only", downgrade skills to hooks (where mechanically possible) or notes. If "Neither", everything becomes a CLAUDE.md note. Never propose an artifact type the user didn't opt into.

**Show the proposal via AskUserQuestion's `preview` field, not as a separate text message** — the dialog overlays your output, so preceding text is hidden. The `preview` field renders markdown in a side-panel (like plan mode); the `question` field is plain-text-only. Structure it as:

  - `question`: short and plain, e.g. "Does this proposal look right?"
  - Each option gets a `preview` with the full proposal as markdown. The "Looks good — proceed" option's preview shows everything; per-item-drop options' previews show what remains after that drop.
  - **Keep previews compact — the preview box truncates with no scrolling.** One line per item, no blank lines between items, no header. Example preview content:

    • **Format-on-edit hook** (automatic) — `ruff format <file>` via PostToolUse
    • **/verify skill** (on-demand) — `make lint && make typecheck && make test`
    • **CLAUDE.md note** (guideline) — "run lint/typecheck/test before marking done"

  - Option labels stay short ("Looks good", "Drop the hook", "Drop the skill") — the tool auto-adds an "Other" free-text option, so don't add your own catch-all.

**Build the preference queue** from the accepted proposal. Each entry: {type: hook|skill|note, description, target file, any Phase-2-sourced details like the actual test/format command}. Phases 4-7 consume this queue.

## Phase 4: Write CLAUDE.md (if user chose project or both)

Write a minimal CLAUDE.md at the project root. Every line must pass this test: "Would removing this cause Claude to make mistakes?" If no, cut it.

**Consume `note` entries from the Phase 3 preference queue whose target is CLAUDE.md** (team-level notes) — add each as a concise line in the most relevant section. These are the behaviors the user wants Claude to follow but didn't need guaranteed (e.g., "propose a plan before implementing", "explain the tradeoffs when refactoring"). Leave personal-targeted notes for Phase 5.

Include:
- Build/test/lint commands Claude can't guess (non-standard scripts, flags, or sequences)
- Code style rules that DIFFER from language defaults (e.g., "prefer type over interface")
- Testing instructions and quirks (e.g., "run single test with: pytest -k 'test_name'")
- Repo etiquette (branch naming, PR conventions, commit style)
- Required env vars or setup steps
- Non-obvious gotchas or architectural decisions
- Important parts from existing AI coding tool configs if they exist (AGENTS.md, .cursor/rules, .cursorrules, .github/copilot-instructions.md, .windsurfrules, .clinerules)

Exclude:
- File-by-file structure or component lists (Claude can discover these by reading the codebase)
- Standard language conventions Claude already knows
- Generic advice ("write clean code", "handle errors")
- Detailed API docs or long references — use `@path/to/import` syntax instead (e.g., `@docs/api-reference.md`) to inline content on demand without bloating CLAUDE.md
- Information that changes frequently — reference the source with `@path/to/import` so Claude always reads the current version
- Long tutorials or walkthroughs (move to a separate file and reference with `@path/to/import`, or put in a skill)
- Commands obvious from manifest files (e.g., standard "npm test", "cargo test", "pytest")

Be specific: "Use 2-space indentation in TypeScript" is better than "Format code properly."

Do not repeat yourself and do not make up sections like "Common Development Tasks" or "Tips for Development" — only include information expressly found in files you read.

Prefix the file with:

# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

If CLAUDE.md already exists: read it, propose specific changes as diffs, and explain why each change improves it. Do not silently overwrite.

For projects with multiple concerns, suggest organizing instructions into `.claude/rules/` as separate focused files (e.g., `code-style.md`, `testing.md`, `security.md`). These are loaded automatically alongside CLAUDE.md and can be scoped to specific file paths using `paths` frontmatter.

For projects with distinct subdirectories (monorepos, multi-module projects, etc.): mention that subdirectory CLAUDE.md files can be added for module-specific instructions (they're loaded automatically when Claude works in those directories). Offer to create them if the user wants.

## Phase 5: Write CLAUDE.local.md (if user chose personal or both)

Write a minimal CLAUDE.local.md at the project root. This file is automatically loaded alongside CLAUDE.md. After creating it, add `CLAUDE.local.md` to the project's .gitignore so it stays private.

**Consume `note` entries from the Phase 3 preference queue whose target is CLAUDE.local.md** (personal-level notes) — add each as a concise line. If the user chose personal-only in Phase 1, this is the sole consumer of note entries.

Include:
- The user's role and familiarity with the codebase (so Claude can calibrate explanations)
- Personal sandbox URLs, test accounts, or local setup details
- Personal workflow or communication preferences

Keep it short — only include what would make Claude's responses noticeably better for this user.

If Phase 2 found multiple git worktrees and the user confirmed they use sibling/external worktrees (not nested inside the main repo): the upward file walk won't find a single CLAUDE.local.md from all worktrees. Write the actual personal content to `~/.claude/<project-name>-instructions.md` and make CLAUDE.local.md a one-line stub that imports it: `@~/.claude/<project-name>-instructions.md`. The user can copy this one-line stub to each sibling worktree. Never put this import in the project CLAUDE.md. If worktrees are nested inside the main repo (e.g., `.claude/worktrees/`), no special handling is needed — the main repo's CLAUDE.local.md is found automatically.

If CLAUDE.local.md already exists: read it, propose specific additions, and do not silently overwrite.

## Phase 6: Suggest and create skills (if user chose "Skills + hooks" or "Skills only")

Skills add capabilities Claude can use on demand without bloating every session.

**First, consume `skill` entries from the Phase 3 preference queue.** Each queued skill preference becomes a SKILL.md tailored to what the user described. For each:
- Name it from the preference (e.g., "verify-deep", "session-report", "deploy-sandbox")
- Write the body using the user's own words from the interview plus whatever Phase 2 found (test commands, report format, deploy target). If the preference maps to an existing bundled skill (e.g., `/verify`), write a project skill that adds the user's specific constraints on top — tell the user the bundled one still exists and theirs is additive.
- Ask a quick follow-up if the preference is underspecified (e.g., "which test command should verify-deep run?")

**Then suggest additional skills** beyond the queue when you find:
- Reference knowledge for specific tasks (conventions, patterns, style guides for a subsystem)
- Repeatable workflows the user would want to trigger directly (deploy, fix an issue, release process, verify changes)

For each suggested skill, provide: name, one-line purpose, and why it fits this repo.

If `.claude/skills/` already exists with skills, review them first. Do not overwrite existing skills — only propose new ones that complement what is already there.

Create each skill at `.claude/skills/<skill-name>/SKILL.md`:

---
name: <skill-name>
description: <what the skill does and when to use it>
---

<Instructions for Claude>

Both the user (`/<skill-name>`) and Claude can invoke skills by default. For workflows with side effects (e.g., `/deploy`, `/fix-issue 123`), add `disable-model-invocation: true` so only the user can trigger it, and use `$ARGUMENTS` to accept input.

## Phase 7: Suggest additional optimizations

Tell the user you're going to suggest a few additional optimizations now that CLAUDE.md and skills (if chosen) are in place.

Check the environment and ask about each gap you find (use AskUserQuestion):

- **GitHub CLI**: Run `which gh` (or `where gh` on Windows). If it's missing AND the project uses GitHub (check `git remote -v` for github.com), ask the user if they want to install it. Explain that the GitHub CLI lets Claude help with commits, pull requests, issues, and code review directly.

- **Linting**: If Phase 2 found no lint config (no .eslintrc, ruff.toml, .golangci.yml, etc. for the project's language), ask the user if they want Claude to set up linting for this codebase. Explain that linting catches issues early and gives Claude fast feedback on its own edits.

- **Proposal-sourced hooks** (if user chose "Skills + hooks" or "Hooks only"): Consume `hook` entries from the Phase 3 preference queue. If Phase 2 found a formatter and the queue has no formatting hook, offer format-on-edit as a fallback. If the user chose "Neither" or "Skills only" in Phase 1, skip this bullet entirely.

  For each hook preference (from the queue or the formatter fallback):

  1. Target file: default based on the Phase 1 CLAUDE.md choice — project → `.claude/settings.json` (team-shared, committed); personal → `.claude/settings.local.json`. Only ask if the user chose "both" in Phase 1 or the preference is ambiguous. Ask once for all hooks, not per-hook.

  2. Pick the event and matcher from the preference:
     - "after every edit" → `PostToolUse` with matcher `Write|Edit`
     - "when Claude finishes" / "before I review" → `Stop` event (fires at the end of every turn — including read-only ones)
     - "before running bash" → `PreToolUse` with matcher `Bash`
     - "before committing" (literal git-commit gate) → **not a hooks.json hook.** Matchers can't filter Bash by command content, so there's no way to target only `git commit`. Route this to a git pre-commit hook (`.git/hooks/pre-commit`, husky, pre-commit framework) instead — offer to write one. If the user actually means "before I review and commit Claude's output", that's `Stop` — probe to disambiguate.
     Probe if the preference is ambiguous.

  3. **Load the hook reference** (once per `/init` run, before the first hook): invoke the Skill tool with `skill: 'update-config'` and args starting with `[hooks-only]` followed by a one-line summary of what you're building — e.g., `[hooks-only] Constructing a PostToolUse/Write|Edit format hook for .claude/settings.json using ruff`. This loads the hooks schema and verification flow into context. Subsequent hooks reuse it — don't re-invoke.

  4. Follow the skill's **"Constructing a Hook"** flow: dedup check → construct for THIS project → pipe-test raw → wrap → write JSON → `jq -e` validate → live-proof (for `Pre|PostToolUse` on triggerable matchers) → cleanup → handoff. Target file and event/matcher come from steps 1–2 above.

Act on each "yes" before moving on.

## Phase 8: Summary and next steps

Recap what was set up — which files were written and the key points included in each. Remind the user these files are a starting point: they should review and tweak them, and can run `/init` again anytime to re-scan.

Then tell the user that you'll be introducing a few more suggestions for optimizing their codebase and Claude Code setup based on what you found. Present these as a single, well-formatted to-do list where every item is relevant to this repo. Put the most impactful items first.

When building the list, work through these checks and include only what applies:
- If frontend code was detected (React, Vue, Svelte, etc.): `/plugin install frontend-design@claude-plugins-official` gives Claude design principles and component patterns so it produces polished UI; `/plugin install playwright@claude-plugins-official` lets Claude launch a real browser, screenshot what it built, and fix visual bugs itself.
- If you found gaps in Phase 7 (missing GitHub CLI, missing linting) and the user said no: list them here with a one-line reason why each helps.
- If tests are missing or sparse: suggest setting up a test framework so Claude can verify its own changes.
- To help you create skills and optimize existing skills using evals, Claude Code has an official skill-creator plugin you can install. Install it with `/plugin install skill-creator@claude-plugins-official`, then run `/skill-creator <skill-name>` to create new skills or refine any existing skill. (Always include this one.)
- Browse official plugins with `/plugin` — these bundle skills, agents, hooks, and MCP servers that you may find helpful. You can also create your own custom plugins to share them with others. (Always include this one.)
```

#### 各階段重點分析

**Phase 1 — 使用者選擇**
- 提供 CLAUDE.md（團隊共享）vs CLAUDE.local.md（個人私有）vs 兩者
- 提供 Skills + Hooks 的組合選項

**Phase 2 — Codebase 掃描**
掃描項目清單：
- manifest 檔案（package.json、Cargo.toml、pyproject.toml、go.mod、pom.xml 等）
- README、Makefile、CI 設定
- 既有 AI 工具設定：CLAUDE.md、`.claude/rules/`、AGENTS.md、`.cursor/rules`、`.cursorrules`、`.github/copilot-instructions.md`、`.windsurfrules`、`.clinerules`、`.mcp.json`
- Formatter 設定（prettier、biome、ruff、black、gofmt、rustfmt）
- Git worktree 狀態

**Phase 3 — 提案機制**
三種 artifact 類型的選擇邏輯：
| 類型 | 強制程度 | 適用場景 |
|------|----------|----------|
| Hook | 最嚴格，Claude 無法跳過 | 格式化、lint 等機械性操作 |
| Skill | 按需觸發 | 驗證、部署等工作流程 |
| CLAUDE.md note | 最寬鬆，僅影響行為 | 溝通偏好、思考風格 |

**Phase 4 — CLAUDE.md 撰寫原則**
核心篩選標準：**「拿掉這行會讓 Claude 犯錯嗎？」不會就刪。**

包含：非標準指令、與預設不同的 code style、測試 quirks、repo 規範、必要環境變數
排除：檔案結構、標準慣例、泛用建議、詳細 API 文件、經常變動的資訊

**Phase 5 — CLAUDE.local.md**
- 個人偏好，自動加入 .gitignore
- 支援 git worktree 多工作樹情境

**Phase 6 — Skills 建立**
- 消費 Phase 3 提案佇列中的 skill 項目
- 有副作用的 skill 加上 `disable-model-invocation: true`

**Phase 7 — 額外優化**
- GitHub CLI 檢查
- Linting 設定建議
- Hook 建立（使用 `update-config` skill 的 Constructing a Hook 流程）
- Hook 事件對應：`PostToolUse`（編輯後）、`Stop`（回合結束）、`PreToolUse`（執行前）

**Phase 8 — 總結與後續**
- 推薦 plugins：`frontend-design`、`playwright`、`skill-creator`
- 提醒可隨時重新執行 `/init`

---

## /init-verifiers

**來源檔案**：`restored-src/src/commands/init-verifiers.ts`
**類型**：`prompt`
**進度訊息**：`analyzing your project and creating verifier skills`

### 用途

建立自動化驗證 skill，供 Verify agent 使用。支援三種驗證器類型：

| 類型 | 工具 | 適用場景 |
|------|------|----------|
| verifier-playwright | Playwright / Chrome DevTools MCP | Web UI 驗證 |
| verifier-cli | Tmux / asciinema | CLI 工具驗證 |
| verifier-api | curl / httpie | HTTP API 驗證 |

### 流程（5 階段）

| 階段 | 說明 |
|------|------|
| Phase 1: Auto-Detection | 掃描子目錄，偵測語言、框架、套件管理器、應用類型、既有測試工具 |
| Phase 2: Verification Tool Setup | 依偵測結果引導安裝驗證工具（Playwright、Chrome DevTools MCP 等） |
| Phase 3: Interactive Q&A | 用 AskUserQuestion 確認名稱、dev server 設定、認證需求 |
| Phase 4: Generate Verifier Skill | 產出 `.claude/skills/<verifier-name>/SKILL.md` |
| Phase 5: Confirm Creation | 確認建立結果，說明 Verify agent 如何自動發現 verifier skill |

### 關鍵設計

- Verifier 資料夾名稱必須包含 `verifier`，Verify agent 靠此自動發現
- **不建立** unit test 或 typecheck 的 verifier（已由標準流程處理）
- 支援自我更新：若驗證失敗是因為 skill 指令過時，會提示更新 SKILL.md
- 認證支援：表單登入、API token、OAuth/SSO，建議用環境變數存放 credentials

### 原文

> 原文較長，請直接參閱 `restored-src/src/commands/init-verifiers.ts`

---

## /commit

**來源檔案**：`restored-src/src/commands/commit.ts`
**類型**：`prompt`
**進度訊息**：`creating commit`
**限制工具**：`Bash(git add:*)`, `Bash(git status:*)`, `Bash(git commit:*)`

### 原文

```
## Context

- Current git status: !`git status`
- Current git diff (staged and unstaged changes): !`git diff HEAD`
- Current branch: !`git branch --show-current`
- Recent commits: !`git log --oneline -10`

## Git Safety Protocol

- NEVER update the git config
- NEVER skip hooks (--no-verify, --no-gpg-sign, etc) unless the user explicitly requests it
- CRITICAL: ALWAYS create NEW commits. NEVER use git commit --amend, unless the user explicitly requests it
- Do not commit files that likely contain secrets (.env, credentials.json, etc). Warn the user if they specifically request to commit those files
- If there are no changes to commit (i.e., no untracked files and no modifications), do not create an empty commit
- Never use git commands with the -i flag (like git rebase -i or git add -i) since they require interactive input which is not supported

## Your task

Based on the above changes, create a single git commit:

1. Analyze all staged changes and draft a commit message:
   - Look at the recent commits above to follow this repository's commit message style
   - Summarize the nature of the changes (new feature, enhancement, bug fix, refactoring, test, docs, etc.)
   - Ensure the message accurately reflects the changes and their purpose (i.e. "add" means a wholly new feature, "update" means an enhancement to an existing feature, "fix" means a bug fix, etc.)
   - Draft a concise (1-2 sentences) commit message that focuses on the "why" rather than the "what"

2. Stage relevant files and create the commit using HEREDOC syntax:
git commit -m "$(cat <<'EOF'
Commit message here.

{commitAttribution}
EOF
)"

You have the capability to call multiple tools in a single response. Stage and create the commit using a single message. Do not use any other tools or do anything else. Do not send any other text or messages besides these tool calls.
```

### 重點

- 自動注入 git status / diff / branch / log 作為上下文
- 要求遵循 repo 現有的 commit message 風格
- 嚴格的 Git Safety Protocol（不 amend、不 skip hooks、不 commit secrets）
- 使用 HEREDOC 語法建立 commit
- Anthropic 內部使用者有額外的 `undercover` 模式（隱藏 attribution）

---

## /commit-push-pr

**來源檔案**：`restored-src/src/commands/commit-push-pr.ts`
**類型**：`prompt`
**進度訊息**：`creating commit and PR`
**限制工具**：git 操作、`gh pr create/edit/view/merge`、`ToolSearch`、Slack MCP

### 原文

```
## Context

- `SAFEUSER`: {safeUser}
- `whoami`: {username}
- `git status`: !`git status`
- `git diff HEAD`: !`git diff HEAD`
- `git branch --show-current`: !`git branch --show-current`
- `git diff {defaultBranch}...HEAD`: !`git diff {defaultBranch}...HEAD`
- `gh pr view --json number 2>/dev/null || true`: !`gh pr view --json number 2>/dev/null || true`

## Git Safety Protocol

- NEVER update the git config
- NEVER run destructive/irreversible git commands (like push --force, hard reset, etc) unless the user explicitly requests them
- NEVER skip hooks (--no-verify, --no-gpg-sign, etc) unless the user explicitly requests it
- NEVER run force push to main/master, warn the user if they request it
- Do not commit files that likely contain secrets (.env, credentials.json, etc)
- Never use git commands with the -i flag (like git rebase -i or git add -i) since they require interactive input which is not supported

## Your task

Analyze all changes that will be included in the pull request, making sure to look at all relevant commits (NOT just the latest commit, but ALL commits that will be included in the pull request from the git diff {defaultBranch}...HEAD output above).

Based on the above changes:
1. Create a new branch if on {defaultBranch} (use SAFEUSER from context above for the branch name prefix, falling back to whoami if SAFEUSER is empty, e.g., `username/feature-name`)
2. Create a single commit with an appropriate message using heredoc syntax
3. Push the branch to origin
4. If a PR already exists for this branch (check the gh pr view output above), update the PR title and body using `gh pr edit` to reflect the current diff. Otherwise, create a pull request using `gh pr create` with heredoc syntax for the body.
   - IMPORTANT: Keep PR titles short (under 70 characters). Use the body for details.
5. After creating/updating the PR, check if the user's CLAUDE.md mentions posting to Slack channels. If it does, use ToolSearch to search for "slack send message" tools. If ToolSearch finds a Slack tool, ask the user if they'd like you to post the PR URL to the relevant Slack channel. Only post if the user confirms.

You have the capability to call multiple tools in a single response. You MUST do all of the above in a single message.

Return the PR URL when you're done, so the user can see it.
```

### 重點

- 一條龍：commit → push → create/update PR
- 分支命名規則：`{SAFEUSER}/feature-name`
- 自動偵測是否已有 PR，有則用 `gh pr edit` 更新
- Anthropic 內部版本自動加上 `--reviewer anthropics/claude-code` 和 Changelog 區塊
- 支援 Slack 通知：檢查 CLAUDE.md 是否提及 Slack channel，並用 MCP 工具發送

---

## /review

**來源檔案**：`restored-src/src/commands/review.ts`
**類型**：`prompt`
**進度訊息**：`reviewing pull request`

### 原文

```
You are an expert code reviewer. Follow these steps:

1. If no PR number is provided in the args, run `gh pr list` to show open PRs
2. If a PR number is provided, run `gh pr view <number>` to get PR details
3. Run `gh pr diff <number>` to get the diff
4. Analyze the changes and provide a thorough code review that includes:
   - Overview of what the PR does
   - Analysis of code quality and style
   - Specific suggestions for improvements
   - Any potential issues or risks

Keep your review concise but thorough. Focus on:
- Code correctness
- Following project conventions
- Performance implications
- Test coverage
- Security considerations

Format your review with clear sections and bullet points.

PR number: {args}
```

### 重點

- 簡潔直接的 code review prompt
- 依賴 `gh` CLI 取得 PR 資訊
- 審查五個面向：正確性、慣例、效能、測試、安全

---

## /ultrareview

**來源檔案**：`restored-src/src/commands/review.ts`（同檔案匯出）
**類型**：`local-jsx`（非 prompt 類型，走 remote bughunter 路徑）

### 說明

- 描述：`~10–20 min · Finds and verifies bugs in your branch. Runs in Claude Code on the web.`
- 在雲端（Claude Code on the web）執行深度 bug hunting
- 需要功能開關 `isUltrareviewEnabled()` 啟用
- 有使用次數限制，超過後顯示付費權限對話框
- `/ultrareview` 是進入 remote bughunter 路徑的**唯一入口**，`/review` 保持純本地

---

## /security-review

**來源檔案**：`restored-src/src/commands/security-review.ts`
**類型**：`prompt`（透過 `createMovedToPluginCommand` 封裝）
**進度訊息**：`analyzing code changes for security risks`
**限制工具**：`Bash(git diff:*)`, `Bash(git status:*)`, `Bash(git log:*)`, `Bash(git show:*)`, `Bash(git remote show:*)`, `Read`, `Glob`, `Grep`, `LS`, `Task`

### 原文

```
You are a senior security engineer conducting a focused security review of the changes on this branch.

GIT STATUS:
!`git status`

FILES MODIFIED:
!`git diff --name-only origin/HEAD...`

COMMITS:
!`git log --no-decorate origin/HEAD...`

DIFF CONTENT:
!`git diff origin/HEAD...`

Review the complete diff above. This contains all code changes in the PR.

OBJECTIVE:
Perform a security-focused code review to identify HIGH-CONFIDENCE security vulnerabilities that could have real exploitation potential. This is not a general code review - focus ONLY on security implications newly added by this PR. Do not comment on existing security concerns.

CRITICAL INSTRUCTIONS:
1. MINIMIZE FALSE POSITIVES: Only flag issues where you're >80% confident of actual exploitability
2. AVOID NOISE: Skip theoretical issues, style concerns, or low-impact findings
3. FOCUS ON IMPACT: Prioritize vulnerabilities that could lead to unauthorized access, data breaches, or system compromise
4. EXCLUSIONS: Do NOT report the following issue types:
   - Denial of Service (DOS) vulnerabilities, even if they allow service disruption
   - Secrets or sensitive data stored on disk (these are handled by other processes)
   - Rate limiting or resource exhaustion issues

SECURITY CATEGORIES TO EXAMINE:

**Input Validation Vulnerabilities:**
- SQL injection via unsanitized user input
- Command injection in system calls or subprocesses
- XXE injection in XML parsing
- Template injection in templating engines
- NoSQL injection in database queries
- Path traversal in file operations

**Authentication & Authorization Issues:**
- Authentication bypass logic
- Privilege escalation paths
- Session management flaws
- JWT token vulnerabilities
- Authorization logic bypasses

**Crypto & Secrets Management:**
- Hardcoded API keys, passwords, or tokens
- Weak cryptographic algorithms or implementations
- Improper key storage or management
- Cryptographic randomness issues
- Certificate validation bypasses

**Injection & Code Execution:**
- Remote code execution via deseralization
- Pickle injection in Python
- YAML deserialization vulnerabilities
- Eval injection in dynamic code execution
- XSS vulnerabilities in web applications (reflected, stored, DOM-based)

**Data Exposure:**
- Sensitive data logging or storage
- PII handling violations
- API endpoint data leakage
- Debug information exposure

Additional notes:
- Even if something is only exploitable from the local network, it can still be a HIGH severity issue

ANALYSIS METHODOLOGY:

Phase 1 - Repository Context Research (Use file search tools):
- Identify existing security frameworks and libraries in use
- Look for established secure coding patterns in the codebase
- Examine existing sanitization and validation patterns
- Understand the project's security model and threat model

Phase 2 - Comparative Analysis:
- Compare new code changes against existing security patterns
- Identify deviations from established secure practices
- Look for inconsistent security implementations
- Flag code that introduces new attack surfaces

Phase 3 - Vulnerability Assessment:
- Examine each modified file for security implications
- Trace data flow from user inputs to sensitive operations
- Look for privilege boundaries being crossed unsafely
- Identify injection points and unsafe deserialization

REQUIRED OUTPUT FORMAT:

You MUST output your findings in markdown. The markdown output should contain the file, line number, severity, category (e.g. `sql_injection` or `xss`), description, exploit scenario, and fix recommendation.

SEVERITY GUIDELINES:
- **HIGH**: Directly exploitable vulnerabilities leading to RCE, data breach, or authentication bypass
- **MEDIUM**: Vulnerabilities requiring specific conditions but with significant impact
- **LOW**: Defense-in-depth issues or lower-impact vulnerabilities

CONFIDENCE SCORING:
- 0.9-1.0: Certain exploit path identified, tested if possible
- 0.8-0.9: Clear vulnerability pattern with known exploitation methods
- 0.7-0.8: Suspicious pattern requiring specific conditions to exploit
- Below 0.7: Don't report (too speculative)

FALSE POSITIVE FILTERING:

HARD EXCLUSIONS - Automatically exclude findings matching these patterns:
1. Denial of Service (DOS) vulnerabilities or resource exhaustion attacks.
2. Secrets or credentials stored on disk if they are otherwise secured.
3. Rate limiting concerns or service overload scenarios.
4. Memory consumption or CPU exhaustion issues.
5. Lack of input validation on non-security-critical fields without proven security impact.
6. Input sanitization concerns for GitHub Action workflows unless they are clearly triggerable via untrusted input.
7. A lack of hardening measures. Code is not expected to implement all security best practices, only flag concrete vulnerabilities.
8. Race conditions or timing attacks that are theoretical rather than practical issues.
9. Vulnerabilities related to outdated third-party libraries.
10. Memory safety issues in Rust or any other memory safe languages.
11. Files that are only unit tests or only used as part of running tests.
12. Log spoofing concerns.
13. SSRF vulnerabilities that only control the path.
14. Including user-controlled content in AI system prompts is not a vulnerability.
15. Regex injection is not a vulnerability.
16. Regex DOS concerns.
17. Insecure documentation.
17. A lack of audit logs is not a vulnerability.

PRECEDENTS:
1. Logging high value secrets in plaintext is a vulnerability. Logging URLs is assumed to be safe.
2. UUIDs can be assumed to be unguessable.
3. Environment variables and CLI flags are trusted values.
4. Resource management issues such as memory or file descriptor leaks are not valid.
5. Subtle or low impact web vulnerabilities such as tabnabbing, XS-Leaks, prototype pollution, and open redirects should not be reported unless extremely high confidence.
6. React and Angular are generally secure against XSS unless using dangerouslySetInnerHTML or similar.
7. Most vulnerabilities in github action workflows are not exploitable in practice.
8. Lack of permission checking in client-side JS/TS code is not a vulnerability.
9. Only include MEDIUM findings if they are obvious and concrete issues.
10. Most vulnerabilities in ipython notebooks are not exploitable in practice.
11. Logging non-PII data is not a vulnerability.
12. Command injection in shell scripts generally not exploitable since they don't run with untrusted user input.

START ANALYSIS:

Begin your analysis now. Do this in 3 steps:
1. Use a sub-task to identify vulnerabilities.
2. Then for each vulnerability identified, create a new sub-task to filter out false-positives. Launch these sub-tasks as parallel sub-tasks.
3. Filter out any vulnerabilities where the sub-task reported a confidence less than 8.

Your final reply must contain the markdown report and nothing else.
```

### 重點

- **角色**：資深安全工程師
- **信心閾值**：>80% 才報告，最終過濾 < 8 分的
- **3 階段分析**：Context Research → Comparative Analysis → Vulnerability Assessment
- **17 條 Hard Exclusion**：不報 DoS、regex injection、React XSS、shell script command injection 等
- **12 條 Precedent**：環境變數視為可信、UUID 視為不可猜測、client-side 不需權限檢查等
- **平行子任務驗證**：每個漏洞各開一個 sub-task 做 false positive 過濾
- **純唯讀**：限制工具僅 git 指令 + Read/Glob/Grep，不能寫入檔案或執行任意指令

---

## /pr-comments

**來源檔案**：`restored-src/src/commands/pr_comments/index.ts`
**類型**：`prompt`（透過 `createMovedToPluginCommand` 封裝）
**進度訊息**：`fetching PR comments`

### 原文

```
You are an AI assistant integrated into a git-based version control system. Your task is to fetch and display comments from a GitHub pull request.

Follow these steps:

1. Use `gh pr view --json number,headRepository` to get the PR number and repository info
2. Use `gh api /repos/{owner}/{repo}/issues/{number}/comments` to get PR-level comments
3. Use `gh api /repos/{owner}/{repo}/pulls/{number}/comments` to get review comments. Pay particular attention to the following fields: `body`, `diff_hunk`, `path`, `line`, etc. If the comment references some code, consider fetching it using eg `gh api /repos/{owner}/{repo}/contents/{path}?ref={branch} | jq .content -r | base64 -d`
4. Parse and format all comments in a readable way
5. Return ONLY the formatted comments, with no additional text

Format the comments as:

## Comments

[For each comment thread:]
- @author file.ts#line:
  ```diff
  [diff_hunk from the API response]
  ```
  > quoted comment text

  [any replies indented]

If there are no comments, return "No comments found."

Remember:
1. Only show the actual comments, no explanatory text
2. Include both PR-level and code review comments
3. Preserve the threading/nesting of comment replies
4. Show the file and line number context for code review comments
5. Use jq to parse the JSON responses from the GitHub API
```

### 重點

- 透過 GitHub API 取得 PR 級別與 code review 評論
- 會嘗試 fetch 程式碼內容來提供上下文
- 格式化為帶 diff hunk 的結構化輸出

---

## /insights

**來源檔案**：`restored-src/src/commands/insights.ts`
**類型**：`prompt`
**進度訊息**：`analyzing your sessions`

### 說明

`/insights` 是一個複雜的分析指令，不是單純的 prompt，而是先在本地執行大量資料收集和 AI 分析，再將結果呈現給使用者。

### 運作流程

1. **收集本地 session 資料**：掃描所有歷史對話記錄
2. **Facet 提取**（用 Opus 模型）：對每個 session 執行結構化分析
3. **生成 HTML 報告**：產出可分享的互動式報告
4. **摘要呈現**：在對話中顯示 At a Glance 摘要

### FACET_EXTRACTION_PROMPT

```
Analyze this Claude Code session and extract structured facets.

CRITICAL GUIDELINES:

1. **goal_categories**: Count ONLY what the USER explicitly asked for.
   - DO NOT count Claude's autonomous codebase exploration
   - DO NOT count work Claude decided to do on its own
   - ONLY count when user says "can you...", "please...", "I need...", "let's..."

2. **user_satisfaction_counts**: Base ONLY on explicit user signals.
   - "Yay!", "great!", "perfect!" → happy
   - "thanks", "looks good", "that works" → satisfied
   - "ok, now let's..." (continuing without complaint) → likely_satisfied
   - "that's not right", "try again" → dissatisfied
   - "this is broken", "I give up" → frustrated

3. **friction_counts**: Be specific about what went wrong.
   - misunderstood_request: Claude interpreted incorrectly
   - wrong_approach: Right goal, wrong solution method
   - buggy_code: Code didn't work correctly
   - user_rejected_action: User said no/stop to a tool call
   - excessive_changes: Over-engineered or changed too much

4. If very short or just warmup, use warmup_minimal for goal_category
```

### SUMMARIZE_CHUNK_PROMPT

```
Summarize this portion of a Claude Code session transcript. Focus on:
1. What the user asked for
2. What Claude did (tools used, files modified)
3. Any friction or issues
4. The outcome

Keep it concise - 3-5 sentences. Preserve specific details like file names, error messages, and user feedback.
```

### 重點

- 使用 **Opus** 模型進行分析（最高品質）
- 追蹤指標：滿意度、摩擦點、目標類別、工具使用、多 Claude 並行偵測
- Anthropic 內部版本支援 `--homespaces` 從遠端 Coder workspace 收集 session
- 產出 HTML 報告，內部版本自動上傳至 S3

---

## /compact

**來源檔案**：`restored-src/src/commands/compact/compact.ts`
**類型**：`local`（程式邏輯，非 prompt）

### 說明

`/compact` 不是透過 prompt 驅動，而是直接執行程式邏輯來壓縮對話歷史。

### 運作流程

1. **Session Memory Compaction**（優先）：嘗試輕量級的 session memory 壓縮
2. **Reactive Compact**（如啟用）：透過 reactive 路徑壓縮
3. **Traditional Compaction**（fallback）：先 microcompact 減少 token，再用模型摘要
4. 支援自訂壓縮指示：`/compact <custom instructions>`

### 重點

- 三層壓縮策略：session memory → reactive → traditional
- 執行前後都有 hook 機制（PreCompact / PostCompact）
- 壓縮後清除 prompt cache、重設 warning、清理狀態
- 支援 context window 升級提示

---

## /ultraplan

**來源檔案**：`restored-src/src/commands/ultraplan.tsx`
**類型**：`local-jsx`
**說明**：`~10–30 min · Claude Code on the web drafts an advanced plan you can edit and approve.`

### 運作流程

1. 檢查使用者資格（OAuth、遠端 agent 權限）
2. 在本地先產生一個初步的 seed plan
3. 將 seed plan 送到 **Claude Code on the web**（CCR）進行多 agent 深度規劃
4. 使用 Opus 4.6 模型
5. 使用者在 web 介面審閱、修改、批准計畫
6. 批准後可選擇：在遠端執行 或 teleport 回本地執行

### 重點

- Prompt 主體存在 `src/utils/ultraplan/prompt.txt`（bundler 內嵌，原始碼未還原）
- 30 分鐘逾時
- 支援 teleport 機制（遠端 ↔ 本地 session 搬移）
- 計畫被拒絕時記錄 reject count

---

## /statusline

**來源檔案**：`restored-src/src/commands/statusline.tsx`
**類型**：`prompt`
**限制工具**：`Agent`, `Read(~/**)`, `Edit(~/.claude/settings.json)`

### 原文

```
Create an Agent with subagent_type "statusline-setup" and the prompt "Configure my statusLine from my shell PS1 configuration"
```

### 重點

- 極簡 prompt，直接委派給 `statusline-setup` subagent
- 讀取使用者的 shell PS1 設定來配置 Claude Code 的狀態列 UI

---

## /brief

**來源檔案**：`restored-src/src/commands/brief.ts`
**類型**：`local-jsx`（toggle 開關，非 prompt）

### 說明

切換 brief-only 模式的開關。啟用後 Claude 必須透過 `BriefTool` 輸出所有內容，純文字回覆會被隱藏。

### 重點

- 功能開關由 GrowthBook feature flag `tengu_kairos_brief_config` 控制
- 需要 entitlement 檢查（`isBriefEntitled()`）
- 切換時注入 system-reminder 確保模型行為正確轉換
- 與 KAIROS（助手模式）整合

---

## /thinkback

**來源檔案**：`restored-src/src/commands/thinkback/thinkback.tsx`
**類型**：`local-jsx`

### 說明

「年度回顧」動畫功能。支援播放、編輯、修復、重新生成個人化的 Claude Code 使用回顧動畫。

### Prompt（編輯/修復/重新生成）

```
EDIT_PROMPT: Use the Skill tool to invoke the "thinkback" skill with mode=edit to modify my existing Claude Code year in review animation. Ask me what I want to change. When the animation is ready, tell the user to run /think-back again to play it.

FIX_PROMPT: Use the Skill tool to invoke the "thinkback" skill with mode=fix to fix validation or rendering errors in my existing Claude Code year in review animation. Run the validator, identify errors, and fix them. When the animation is ready, tell the user to run /think-back again to play it.

REGENERATE_PROMPT: Use the Skill tool to invoke the "thinkback" skill with mode=regenerate to create a completely new Claude Code year in review animation from scratch. Delete the existing animation and start fresh. When the animation is ready, tell the user to run /think-back again to play it.
```

---

## 其他指令速查

以下指令不含實質 prompt，多為 UI 操作、設定切換或 stub：

| 指令 | 類型 | 說明 |
|------|------|------|
| `/plan` | local-jsx | 進入 plan mode 或檢視當前計畫 |
| `/compact` | local | 壓縮對話歷史（見上方詳細說明） |
| `/version` | local | 顯示版本號（僅限 ant 使用者） |
| `/stickers` | local | 開啟瀏覽器到 stickermule.com/claudecode 貼紙頁面 |
| `/good-claude` | stub | 已停用（`isEnabled: false`） |
| `/summary` | stub | 已停用（`isEnabled: false`） |
| `/advisor` | local | 設定 advisor 模型 |
| `/effort` | local | 調整推理努力等級 |
| `/fast` | local | 切換快速模式 |
| `/model` | local | 切換模型 |
| `/vim` | local | 切換 Vim 模式 |
| `/voice` | local | 語音互動 |
| `/login` / `/logout` | local | 認證管理 |
| `/config` | local | 設定管理 |
| `/mcp` | local | MCP server 管理 |
| `/permissions` | local | 權限管理 |
| `/hooks` | local | Hook 管理 |
| `/memory` | local | 記憶管理 |
| `/context` | local | 上下文管理 |
| `/doctor` | local | 診斷工具 |
| `/export` | local | 匯出對話 |
| `/share` | local | 分享對話 |
| `/cost` | local | 查看花費 |
| `/stats` | local | 使用統計 |
| `/clear` | local | 清除對話 |
| `/help` | local | 說明文件 |

> 完整指令列表請參閱 `restored-src/src/commands/` 目錄
