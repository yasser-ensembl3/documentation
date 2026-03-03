# Kai — Claude Code Configuration — Process Documentation

## Overview

Complete backup of Kai's Claude Code (Anthropic CLI) configuration. Kai is Yasser's personal AI assistant built on top of Claude Code's extensibility system — skills, hooks, tools, MCP servers, and settings. The architecture follows a **Personal AI Infrastructure (PAI)** pattern where all configuration lives in `~/.claude/` on macOS, with Bun as the runtime for all TypeScript hooks and tools.

**Repo**: [yasser-ensembl3/KAI_Ensembl3](https://github.com/yasser-ensembl3/KAI_Ensembl3)

---

## Tech Stack

| Component | Technology |
|-----------|------------|
| Runtime | Claude Code (Anthropic CLI) |
| Hook/Tool runtime | Bun |
| Language | TypeScript |
| MCP SDK | @modelcontextprotocol/sdk |
| Notion SDK | @notionhq/client |
| Shopify API | REST Admin API 2024-10 |
| Browser automation | Playwright (via Browser skill) |
| Template engine | Handlebars (via Prompting skill) |

---

## Architecture

```
Kai-Claude-Code-Config/
├── skills/                          # 12 skill modules
│   ├── CORE/                        # Identity, preferences, orchestrator
│   │   ├── SKILL.md                 # Frontmatter + body
│   │   ├── SkillSystem.md           # Mandatory skill structure spec
│   │   └── Workflows/
│   │       └── UpdateDocumentation.md
│   ├── Agents/                      # Dynamic agent composition
│   │   ├── SKILL.md
│   │   ├── AgentPersonalities.md    # Named agent definitions
│   │   ├── Data/Traits.yaml         # 28 composable traits
│   │   ├── Templates/DynamicAgent.hbs
│   │   ├── Tools/
│   │   │   ├── AgentFactory.ts      # Composition engine
│   │   │   └── BarRaiser.ts         # Quality gate
│   │   └── Workflows/
│   │       ├── CreateCustomAgent.md
│   │       ├── ListTraits.md
│   │       └── SpawnParallelAgents.md
│   ├── Browser/                     # Code-first browser automation
│   │   ├── SKILL.md
│   │   ├── index.ts                 # Playwright wrapper API
│   │   ├── Tools/Browse.ts          # CLI: screenshot/verify/open
│   │   ├── Examples/                # 3 usage examples
│   │   ├── Workflows/              # 4 workflows
│   │   └── package.json             # Playwright dependency
│   ├── Coaching/                    # Personal growth
│   │   ├── SKILL.md
│   │   └── Tools/Coach.ts
│   ├── Coding/                      # Technical development
│   │   ├── SKILL.md
│   │   └── Tools/Coder.ts
│   ├── CreateSkill/                 # Skill creation helper
│   │   └── SKILL.md
│   ├── NotionManager/               # Notion MCP management
│   │   ├── SKILL.md
│   │   ├── install.sh
│   │   ├── mcp-server/
│   │   │   ├── src/index.ts
│   │   │   └── export-page.ts
│   │   └── scripts/notion-to-md.ts
│   ├── Organizer/                   # Multi-destination file organization
│   │   ├── SKILL.md
│   │   ├── Tools/
│   │   │   ├── FileSort.ts
│   │   │   ├── Operator.ts          # Business/pragmatic agent
│   │   │   ├── ObsidianCapture.ts
│   │   │   ├── NotionSync.ts
│   │   │   ├── GitHubIssue.ts
│   │   │   ├── drive_para_organizer.py
│   │   │   └── shared_files_organizer.py
│   │   └── Workflows/              # 5 workflows
│   ├── Prompting/                   # Meta-prompting system
│   │   ├── SKILL.md
│   │   ├── Standards.md
│   │   ├── Templates/
│   │   │   ├── README.md
│   │   │   └── Primitives/         # 5 Handlebars primitives
│   │   │       ├── Briefing.hbs
│   │   │       ├── Gate.hbs
│   │   │       ├── Roster.hbs
│   │   │       ├── Structure.hbs
│   │   │       └── Voice.hbs
│   │   └── Tools/
│   │       ├── RenderTemplate.ts
│   │       └── ValidateTemplate.ts
│   ├── Research/                    # Knowledge synthesis
│   │   ├── SKILL.md
│   │   └── Tools/Researcher.ts
│   ├── Strategy/                    # Planning & goals
│   │   ├── SKILL.md
│   │   └── Tools/Planner.ts
│   └── Writing/                     # Content creation
│       ├── SKILL.md
│       └── Tools/Writer.ts
├── hooks/                           # 8 lifecycle hooks
│   ├── initialize-session.ts        # SessionStart: environment setup
│   ├── load-core-context.ts         # SessionStart: inject CORE context
│   ├── capture-all-events.ts        # ALL events: JSONL logging
│   ├── update-tab-titles.ts         # UserPromptSubmit: terminal tab
│   ├── security-validator.ts        # PreToolUse: command validation
│   ├── stop-hook.ts                 # Stop: capture work summaries
│   ├── subagent-stop-hook.ts        # SubagentStop: route agent outputs
│   ├── capture-session-summary.ts   # SessionEnd: session summary
│   └── lib/
│       ├── observability.ts         # Event dispatch to dashboard
│       └── metadata-extraction.ts   # Agent instance metadata
├── tools/                           # 3 global CLI tools
│   ├── GenerateSkillIndex.ts        # Build searchable skill index
│   ├── PaiArchitecture.ts           # Generate Architecture.md
│   └── SkillSearch.ts               # Query skill index
├── mcp-servers/
│   ├── minivault/                   # Notion MCP server (27 tools)
│   │   ├── src/index.ts
│   │   ├── package.json
│   │   └── tsconfig.json
│   └── shopify/                     # Shopify MCP server (9 tools)
│       ├── src/index.ts
│       ├── package.json
│       └── tsconfig.json
├── project-memory/                  # Per-project persistent memory
│   ├── Quarterly-results/MEMORY.md
│   └── Template-book-manager/
│       ├── MEMORY.md
│       └── architecture.md
├── settings.json                    # Global hooks + flags
├── settings.local.json              # Permission allowlist
└── env-config                       # Environment variables
```

---

## Skill System

All 12 skills follow a mandatory structure defined in `CORE/SkillSystem.md`.

### Skill Structure Convention

Every skill requires:

1. **SKILL.md** with YAML frontmatter containing `name` (TitleCase) and `description` (single-line with mandatory `USE WHEN` clause)
2. **Markdown body** with `## Workflow Routing` table and `## Examples` section
3. **Tools/ directory** containing TypeScript CLI tools executed via Bun
4. **Workflows/ directory** (optional) with step-by-step execution guides

```yaml
---
name: SkillName
description: [What it does]. USE WHEN [intent triggers using OR]. [Additional capabilities].
---
```

### Skill Inventory

| # | Skill | Traits | Purpose | Key Tools |
|---|-------|--------|---------|-----------|
| 1 | **CORE** | — | Identity (Kai), personality calibration, orchestrator routing, learner system | — |
| 2 | **Agents** | — | Dynamic agent composition from 28 traits, BarRaiser quality gate | AgentFactory.ts, BarRaiser.ts |
| 3 | **Browser** | — | Code-first Playwright automation (99% token savings vs MCP) | Browse.ts (screenshot/verify/open) |
| 4 | **Coaching** | empathetic, consultative, synthesizing | Personal growth, breakthroughs, getting unstuck | Coach.ts |
| 5 | **Coding** | technical, meticulous, systematic | Software development, debugging, prototyping | Coder.ts |
| 6 | **CreateSkill** | — | Skill creation and validation helper | — |
| 7 | **NotionManager** | — | Notion database management via MCP (27 tools) | MCP server |
| 8 | **Organizer** | business, pragmatic, systematic | Multi-destination file organization (Drive, Obsidian, Notion, GitHub) | FileSort.ts, Operator.ts, ObsidianCapture.ts, NotionSync.ts, GitHubIssue.ts, 2 Python scripts |
| 9 | **Prompting** | — | Meta-prompting with 5 Handlebars primitives | RenderTemplate.ts, ValidateTemplate.ts |
| 10 | **Research** | research, analytical, thorough | Knowledge synthesis, information gathering | Researcher.ts |
| 11 | **Strategy** | business, analytical, systematic | Goals, planning, reviews, strategic direction | Planner.ts |
| 12 | **Writing** | creative, enthusiastic, synthesizing | Content creation, editing, publishing | Writer.ts |

---

## CORE Skill (Detail)

The CORE skill auto-loads at every session start via the `load-core-context.ts` hook.

### Identity

- **Assistant name**: Kai
- **User**: Yasser
- **Voice rule**: First-person only ("my system", "I can") — never third-person

### Personality Calibration

| Trait | Value | Scale |
|-------|-------|-------|
| Humor | 70/100 | 0=serious, 100=witty |
| Curiosity | 65/100 | 0=focused, 100=exploratory |
| Precision | 60/100 | 0=approximate, 100=exact |
| Formality | 30/100 | 0=casual, 100=professional |
| Directness | 55/100 | 0=diplomatic, 100=blunt |

### Orchestrator Routing

Routes incoming tasks to the appropriate skill based on domain keywords:

| Domain | Skill | Trigger Keywords |
|--------|-------|------------------|
| Writing/Content | Writing/ | write, draft, content, article, blog |
| Technical/Code | Coding/ | code, debug, implement, build, fix bug |
| Research/Learning | Research/ | research, find, investigate, learn, summarize |
| Strategy/Goals | Strategy/ | plan, goals, strategy, review, prioritize |
| Personal Growth | Coaching/ | stuck, blocked, help me, motivation, growth |
| Admin/Logistics | Organizer/ | organize, manage, schedule, admin |
| Custom Agents | Agents/ | custom agents, spin up, specialized |
| Quality Review | Agents/BarRaiser | review quality, final check |

### Response Format Template

```
📋 SUMMARY: [One sentence]
🔍 ANALYSIS: [Key findings]
⚡ ACTIONS: [Steps taken]
✅ RESULTS: [Outcomes]
➡️ NEXT: [Recommended next steps]
🎯 COMPLETED: [12 words max]
```

---

## Agents Skill (Detail)

### Dynamic Agent Composition

Two agent types coexist:

| Type | Definition | Best For |
|------|------------|----------|
| **Named Agents** | Persistent identities with backstories | Recurring work, voice output |
| **Dynamic Agents** | Task-specific specialists from traits | One-off tasks, parallel work |

### Trait System

28 composable traits in `Data/Traits.yaml`, organized in 3 categories:

| Category | Traits |
|----------|--------|
| **Expertise** | security, legal, finance, medical, technical, research, creative, business, data, communications |
| **Personality** | skeptical, enthusiastic, cautious, bold, analytical, creative, empathetic, contrarian, pragmatic, meticulous |
| **Approach** | thorough, rapid, systematic, exploratory, comparative, synthesizing, adversarial, consultative |

### AgentFactory.ts

Mandatory execution before spawning any custom agent:

```bash
bun run $PAI_DIR/skills/Agents/Tools/AgentFactory.ts \
  --traits "<expertise>,<personality>,<approach>" \
  --task "<task description>" \
  --output json
```

Returns JSON with `prompt` (verbatim system prompt) and `voice_id` (mapped to traits).

### BarRaiser.ts

Quality gate inspired by Amazon's Bar Raiser program. Traits: `contrarian, skeptical, adversarial`.

Quality levels: `draft` | `review` | `ship` | `archive`

### Model Selection

| Task Type | Model | Rationale |
|-----------|-------|-----------|
| Grunt work | haiku | 10-20x faster |
| Standard analysis | sonnet | Balanced |
| Deep reasoning | opus | Maximum intelligence |

---

## Browser Skill (Detail)

Code-first Playwright automation that replaces traditional browser MCP servers with 99% token savings.

### CLI Commands

```bash
bun run $PAI_DIR/skills/Browser/Tools/Browse.ts screenshot <url>   # Capture page
bun run $PAI_DIR/skills/Browser/Tools/Browse.ts verify <url>       # Verify page state
bun run $PAI_DIR/skills/Browser/Tools/Browse.ts open <url>         # Open and interact
```

### Decision Tree

1. Simple page check → `screenshot` command
2. Verify page state → `verify` command
3. Fill forms, click buttons → `open` command with instructions
4. Complex multi-step automation → TypeScript API (`index.ts` Playwright wrapper)

### Playwright Wrapper API

`index.ts` exposes: navigation, capture (screenshot/PDF), interaction (click/type/select), waiting, JavaScript execution, viewport control.

---

## Hooks System

All 8 hooks run via Bun (`~/.bun/bin/bun run`). Configured in `settings.json` across 7 lifecycle events.

### Event Flow

```
SessionStart → initialize-session → load-core-context → capture-all-events
     ↓
UserPromptSubmit → update-tab-titles → capture-all-events
     ↓
PreToolUse → security-validator → capture-all-events
     ↓
PostToolUse → capture-all-events
     ↓
Stop → stop-hook → capture-all-events
SubagentStop → subagent-stop-hook → capture-all-events
     ↓
SessionEnd → capture-session-summary → capture-all-events
```

### Hook Details

#### initialize-session.ts (SessionStart)

Initializes session environment:
1. Sets terminal tab title with project name via OSC escape sequences
2. Creates required directories: `history/sessions`, `history/learnings`, `history/research`
3. Writes `.current-session` marker file with session ID, timestamp, cwd, project name
4. Sends event to observability dashboard (POST to `PAI_OBSERVABILITY_URL` or `localhost:4000`)
5. Background Claude Code version check (non-blocking)

#### load-core-context.ts (SessionStart)

Injects CORE skill context into Claude's conversation:
1. Skips for subagent sessions (`CLAUDE_CODE_AGENT` env or `SUBAGENT=true`)
2. Reads `skills/CORE/SKILL.md`
3. Wraps content in `<system-reminder>` tags with current date/time
4. Outputs to stdout so Claude Code processes it as context

#### capture-all-events.ts (ALL events)

Universal event logger:
1. Receives `--event-type` argument identifying the lifecycle event
2. Parses stdin payload (JSON from Claude Code)
3. Tracks agent type from Task tool calls, SubagentStop, and environment
4. Enriches events with agent metadata if spawning subagents
5. Appends to daily JSONL file at `history/raw-outputs/YYYY-MM/YYYY-MM-DD_all-events.jsonl`

#### update-tab-titles.ts (UserPromptSubmit)

Updates terminal tab title with task keywords:
1. Extracts prompt from payload
2. Removes stop words (30+ English filler words)
3. Takes first 4 significant words, capitalizes first
4. Sets tab title via OSC escape sequences

#### security-validator.ts (PreToolUse)

10-tier security validation for Bash commands:

| Tier | Category | Action | Examples |
|------|----------|--------|----------|
| 1 | Catastrophic | Block | `rm -rf /`, `mkfs`, `dd if=...of=/dev` |
| 2 | Reverse shells | Block | `bash -i >& /dev/tcp`, `nc -e /bin/sh` |
| 3 | Credential theft | Block | `curl ... | bash`, `base64 -d | sh` |
| 4 | Prompt injection | Block | `ignore previous instructions`, `[INST]`, `<\|im_start\|>` |
| 5 | Env manipulation | Warn | `export ANTHROPIC_`, `echo $OPENAI_` |
| 6 | Git dangerous | Confirm | `git push --force`, `git reset --hard` |
| 7 | System modification | Log | `chmod 777`, `sudo`, `systemctl stop` |
| 8 | Network operations | Log | `ssh`, `scp`, `curl -X POST` |
| 9 | Data exfiltration | Block | `curl --upload-file`, `tar | curl` |
| 10 | PAI protection | Block | `rm .claude/`, `git push PAI public` |

Exit code 2 signals block to Claude Code. Only validates `Bash` tool calls.

#### stop-hook.ts (Stop)

Captures main agent work summaries:
1. Reads response from payload or extracts last assistant message from transcript JSONL
2. Detects "learning indicators" (problem, solved, discovered, fixed, etc. — needs 2+ matches)
3. Routes to `history/sessions/` or `history/learnings/` based on detection
4. Writes markdown file with YAML frontmatter (capture_type, timestamp, session_id, executor)
5. Extracts summary from `🎯 COMPLETED` or `📋 SUMMARY` sections (first 100 chars)

#### subagent-stop-hook.ts (SubagentStop)

Routes subagent outputs to categorized history directories:
1. Reads transcript path from stdin
2. Finds Task tool results in transcript JSONL (searches backwards)
3. Extracts completion message from `🎯 COMPLETED` patterns
4. Routes by agent type:
   - `researcher`/`intern` → `history/research/`
   - `architect` → `history/decisions/`
   - `engineer`/`designer` → `history/execution/features/`
5. Writes markdown with agent metadata

#### capture-session-summary.ts (SessionEnd)

Creates end-of-session summary:
1. Analyzes raw event JSONL files for the current month
2. Extracts files changed (Edit/Write tool calls), commands executed (Bash), tools used
3. Determines session focus from file patterns (blog-work, hook-development, skill-updates, etc.)
4. Writes summary markdown to `history/sessions/YYYY-MM/`

### Hook Libraries

#### lib/observability.ts

Sends events to an observability dashboard via HTTP POST:
- Endpoint: `PAI_OBSERVABILITY_URL` env var or `http://localhost:4000/events`
- Fails silently if dashboard is offline
- Helpers: `getCurrentTimestamp()`, `getSourceApp()` (reads `PAI_SOURCE_APP` or `DA` env)

#### lib/metadata-extraction.ts

Extracts agent instance metadata from Task tool calls:
- Strategy 1: Parse `[agent-type-N]` from description
- Strategy 2: Parse `[AGENT_INSTANCE: ...]` from prompt
- Strategy 3: Fallback to `subagent_type`
- `isAgentSpawningCall()`: checks if tool call is `Task` with `subagent_type`

---

## Global Tools

### GenerateSkillIndex.ts

Parses all `SKILL.md` files and builds a searchable JSON index:

```bash
bun run $PAI_DIR/tools/GenerateSkillIndex.ts
```

- Scans `skills/` directory for directories with `SKILL.md`
- Extracts YAML frontmatter (name, description)
- Parses `USE WHEN` triggers from description
- Classifies skills as `always` (CORE, Development, Research) or `deferred`
- Writes `skills/skill-index.json`

### PaiArchitecture.ts

Scans the PAI installation and generates `Architecture.md`:

```bash
bun $PAI_DIR/tools/PaiArchitecture.ts generate     # Generate/refresh
bun $PAI_DIR/tools/PaiArchitecture.ts status        # Show current state
bun $PAI_DIR/tools/PaiArchitecture.ts check         # Verify health
bun $PAI_DIR/tools/PaiArchitecture.ts log-upgrade "description"
```

- Detects installed packs (skills with SKILL.md)
- Loads bundle info from `.installed-bundles.json`
- Reads upgrade history from `history/Upgrades.jsonl`
- Checks system health (hooks, history, skills directories)

### SkillSearch.ts

Searches the skill index for capabilities:

```bash
bun run $PAI_DIR/tools/SkillSearch.ts <query>   # Search by keyword
bun run $PAI_DIR/tools/SkillSearch.ts --list     # List all skills
```

- Scores results by: exact name match (10), trigger match (5), description match (2)
- Returns top 5 results sorted by score

---

## MCP Servers

### MiniVault (Notion)

TypeScript MCP server providing 27 tools for Notion database management.

**Tech**: `@modelcontextprotocol/sdk` + `@notionhq/client`
**Transport**: Stdio

**Databases supported** (11 total, configured via env vars):

| Database | Env Variable |
|----------|-------------|
| Tasks | `NOTION_DB_TASKS` |
| Recurring Tasks | `NOTION_DB_RECURRING_TASKS` |
| Orders | `NOTION_DB_ORDERS` |
| Essentials | `NOTION_DB_ESSENTIALS` |
| Metrics | `NOTION_DB_METRICS` |
| Goals | `NOTION_DB_GOALS` |
| Documents | `NOTION_DB_DOCUMENTS` |
| Feedback | `NOTION_DB_FEEDBACK` |
| Sales | `NOTION_DB_SALES` |
| Web Analytics | `NOTION_DB_WEB_ANALYTICS` |
| Assumptions | `NOTION_DB_ASSUMPTIONS` |

**Tool categories**:

| Category | Tools |
|----------|-------|
| Tasks | list_tasks, create_task, update_task, delete_task |
| Recurring Tasks | list_recurring_tasks, create_recurring_task, update_recurring_task |
| Orders | list_orders, update_order |
| Essentials | list_essentials, create_essential, update_essential, delete_essential |
| Documents | list_documents, create_document, delete_document |
| Metrics/Goals | list_metrics, create_metric, list_goals, create_goal |
| Feedback | list_feedback, create_feedback |
| Overview | get_project_status |
| Raw Access | notion_query, notion_update_page, notion_create_page |

**Property extraction**: `getTextFromProperty()` handles all Notion property types: title, rich_text, select, multi_select, number, date, checkbox, url, email, phone_number, formula, status.

### Shopify

TypeScript MCP server providing 9 tools for Shopify store management.

**Tech**: `@modelcontextprotocol/sdk` + Shopify REST Admin API v2024-10
**Transport**: Stdio
**Auth**: `SHOPIFY_ACCESS_TOKEN` + `SHOPIFY_SHOP_DOMAIN` env vars

**Tools**:

| Tool | Purpose |
|------|---------|
| get_shop_info | Store details (name, domain, plan, currency) |
| list_orders | Filter by status, fulfillment, financial status, date range |
| get_order | Single order details with line items |
| list_products | Filter by status (active/archived/draft) |
| get_product | Single product with variants and pricing |
| list_customers | Customer list with order count and total spent |
| get_inventory | Inventory levels by location |
| list_locations | All store locations with addresses |
| get_analytics_summary | Revenue, order count, fulfillment stats for N days |

---

## Settings

### settings.json — Global Configuration

Hooks configuration for all 7 lifecycle events, each running via `~/.bun/bin/bun run`:

| Event | Hooks |
|-------|-------|
| SessionStart | initialize-session, load-core-context, capture-all-events |
| PreToolUse | capture-all-events |
| PostToolUse | capture-all-events |
| UserPromptSubmit | update-tab-titles, capture-all-events |
| Stop | stop-hook, capture-all-events |
| SubagentStop | subagent-stop-hook, capture-all-events |
| SessionEnd | capture-session-summary, capture-all-events |

Global flags: `alwaysThinkingEnabled: true`

### settings.local.json — Permissions

Allowlisted commands include: `git`, `ls`, `cat`, `find`, `bun`, `npm`, `python3`, `gh` (GitHub CLI), `curl`, `open`, `tree`, `chmod`, `brew`, `wc`, plus project-specific commands. Skills `CORE` and `Organizer` are always allowed.

### env-config — Environment

```
PAI_DIR="$HOME/.claude"
DA="Kai"
TIME_ZONE="Europe/Paris"
```

---

## History System

The hooks collectively maintain a structured history:

```
history/
├── raw-outputs/
│   └── YYYY-MM/
│       └── YYYY-MM-DD_all-events.jsonl    # Every hook event
├── sessions/
│   └── YYYY-MM/
│       └── YYYYMMDDTHHMMSS_SESSION_*.md   # Session summaries
├── learnings/
│   └── YYYY-MM/
│       └── YYYYMMDDTHHMMSS_LEARNING_*.md  # Problem/solution captures
├── research/
│   └── YYYY-MM/
│       └── YYYYMMDDTHHMMSS_AGENT-*_RESEARCH_*.md
├── decisions/
│   └── YYYY-MM/
│       └── YYYYMMDDTHHMMSS_AGENT-architect_DECISION_*.md
└── execution/
    └── features/
        └── YYYY-MM/
            └── YYYYMMDDTHHMMSS_AGENT-engineer_FEATURE_*.md
```

All markdown files include YAML frontmatter with `capture_type`, `timestamp`, `session_id`, and `executor`.

---

## Domain-Specific Skills

### Coaching

**Agent traits**: empathetic, consultative, synthesizing

Principles: Ask don't tell, trust the process, challenge with care, values as compass, no judgment.

Workflows: Session (full coaching), Unstuck (quick unblock), Reflect (guided reflection).

Handoff triggers: Strategy/ (needs planning), Coding/ (ready to execute), Research/ (needs facts).

### Coding

**Agent traits**: technical, meticulous, systematic

Principles: Ship fast, minimal viable, no over-engineering, document decisions.

5-level complexity assessment: Trivial → Simple → Moderate → Complex → Unknown.

Workflows: Build (implement), Debug (with cautious trait), Review (with comparative trait).

### Research

**Agent traits**: research, analytical, thorough

Principles: Synthesize don't collect, source everything, flag uncertainty, update don't duplicate.

Output formats: Quick Answer (answer + sources + confidence) and Deep Dive (summary + findings + implications + open questions).

Workflows: Quick (rapid research), Deep (thorough analysis), Compare (comparative research).

### Strategy

**Agent traits**: business, analytical, systematic

Principles: Specific > vague, aligned with values, project vs area, progress visible.

Workflows: GoalSet, WeeklyReview, QuarterlyReview, Prioritize.

### Writing

**Agent traits**: creative, enthusiastic, synthesizing

Principles: Direct not academic, story-driven, no filler, ship > perfect.

Workflows: Draft (create content), Edit (refine text), Review (quality check via BarRaiser).

### Organizer

**Operator agent traits**: business, pragmatic, systematic

5 destination workflows:
1. **FileSort** — Sort files by type/date/project
2. **DriveSync** — Sync to Google Drive (PARA method)
3. **ObsidianCapture** — Add to Obsidian vault
4. **NotionSync** — Create/update Notion pages
5. **GitHubIssue** — Create GitHub issues from files

Python tools: `drive_para_organizer.py` (Google Drive PARA), `shared_files_organizer.py`.

### Prompting

5 Handlebars primitives for meta-prompting:

| Primitive | Purpose |
|-----------|---------|
| **Roster** | Define team composition |
| **Voice** | Set communication style |
| **Structure** | Define output format |
| **Briefing** | Provide context and constraints |
| **Gate** | Quality checkpoints |

Tools: `RenderTemplate.ts` (compile templates), `ValidateTemplate.ts` (check structure).

---

## Restoration

```bash
# 1. Install Claude Code
npm install -g @anthropic-ai/claude-code

# 2. Install Bun (required for hooks)
curl -fsSL https://bun.sh/install | bash

# 3. Copy configuration
cp -r skills/ ~/.claude/skills/
cp -r hooks/ ~/.claude/hooks/
cp -r tools/ ~/.claude/tools/
cp settings.json ~/.claude/settings.json
cp settings.local.json ~/.claude/settings.local.json
cp env-config ~/.claude/.env

# 4. Install MCP servers
cp -r mcp-servers/ ~/.claude/mcp-servers/
cd ~/.claude/mcp-servers/minivault && npm install && npm run build
cd ~/.claude/mcp-servers/shopify && npm install && npm run build

# 5. Configure MCP servers
claude mcp add minivault --scope user \
  --env NOTION_TOKEN=your_token \
  -- node ~/.claude/mcp-servers/minivault/dist/index.js

# 6. Install Browser skill dependencies
cd ~/.claude/skills/Browser && bun install

# 7. Install Organizer Python dependencies
cd ~/.claude/skills/Organizer/Tools && pip install -r requirements.txt

# 8. Restart Claude Code
```

---

## Credentials

| Service | Type | Used By |
|---------|------|---------|
| Notion | Internal integration token | MiniVault MCP server |
| Shopify | Access token + shop domain | Shopify MCP server |
| Google | OAuth token + credentials | Organizer skill (Drive) |

No credentials are included in the backup — they must be configured separately.

---

## Excluded from Backup

| Directory | Reason |
|-----------|--------|
| `~/.claude/debug/` | Debug logs (300+ session dirs) |
| `~/.claude/history/` | Command history and session archives |
| `~/.claude/session-env/` | Session environment snapshots |
| `~/.claude/todos/` | Session-specific todo lists |
| `~/.claude/plans/` | Session-specific plans |
| `~/.claude/cache/` | Cache files |
| `~/.claude/telemetry/` | Telemetry data |
| `skills/Browser/node_modules/` | Node dependencies |
| `skills/Organizer/Tools/venv/` | Python virtual environment |
| `skills/Organizer/Tools/token.json` | OAuth token (sensitive) |
| `skills/Organizer/Tools/credentials.json` | Google credentials (sensitive) |

---

## Design Decisions

1. **Bun as hook runtime** — All hooks and tools use Bun for fast TypeScript execution without compilation step. Hooks must never crash (fail silently) to avoid blocking Claude Code.
2. **File-based MCP pattern** — Browser skill uses pre-written code execution instead of runtime code generation, achieving 99% token savings compared to traditional MCP browser tools.
3. **Trait-based agent composition** — 28 composable traits across 3 categories allow dynamic agent creation with unique voices, enforced through mandatory AgentFactory.ts execution.
4. **10-tier security validation** — Tiered approach from catastrophic blocking to logging, covering reverse shells, credential theft, prompt injection, and PAI infrastructure protection.
5. **Structured history system** — Automatic capture of sessions, learnings, research, and decisions into monthly directories with YAML frontmatter for searchability.
6. **Context injection via hooks** — CORE skill auto-loads at session start through `<system-reminder>` tags, establishing identity and routing without manual activation.
7. **Dual MCP servers** — Separate servers for Notion (project management) and Shopify (e-commerce), both using stdio transport with the MCP SDK.
8. **Observability-first** — Every hook event is captured to daily JSONL files and optionally forwarded to an HTTP dashboard endpoint.
