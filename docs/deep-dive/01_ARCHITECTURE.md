# ARIS System Architecture

## The 30,000-Foot View

ARIS is built on one radical assumption: **AI agents don't need a framework; they just need clear written instructions.** Every component is either a Markdown file (instructions for an AI) or a tiny Python server (bridge to an external service).

```
┌─────────────────────────────────────────────────────────────────────┐
│                          USER MACHINE                               │
│                                                                     │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │                    Claude Code CLI                            │  │
│  │  (Reads SKILL.md files → executes their instructions)        │  │
│  │                                                              │  │
│  │   Tools available to Claude:                                 │  │
│  │   • Bash(*) — run any shell command                          │  │
│  │   • Read/Write/Edit/Glob/Grep — file system                  │  │
│  │   • WebSearch/WebFetch — internet                            │  │
│  │   • Agent — spawn sub-agents                                 │  │
│  │   • Skill — invoke other SKILL.md files                      │  │
│  │   • mcp__codex__* — call GPT-5.4 via Codex MCP server        │  │
│  └──────────────────────────────────────────────────────────────┘  │
│         │                          │                               │
│         ▼                          ▼                               │
│  ┌─────────────┐         ┌─────────────────────┐                  │
│  │  skills/    │         │   mcp-servers/       │                  │
│  │  (Markdown  │         │   (Python servers)   │                  │
│  │  recipes)   │         │   llm-chat           │                  │
│  └─────────────┘         │   feishu-bridge      │                  │
│                          │   claude-review      │                  │
│                          │   minimax-chat       │                  │
│                          └─────────────────────┘                  │
│                                   │                               │
└───────────────────────────────────┼───────────────────────────────┘
                                    │
            ┌───────────────────────┼───────────────────┐
            ▼                       ▼                   ▼
    ┌──────────────┐      ┌──────────────────┐  ┌─────────────┐
    │  OpenAI API  │      │  Feishu/Lark API  │  │  GPU Server │
    │  (GPT-5.4)   │      │  (notifications) │  │  (via SSH)  │
    └──────────────┘      └──────────────────┘  └─────────────┘
            ▲                                          ▲
            │                                          │
    ┌───────────────┐                        ┌─────────────────┐
    │ Codex CLI     │                        │  screen sessions│
    │ (mcp-server   │                        │  (experiments)  │
    │  mode)        │                        └─────────────────┘
    └───────────────┘
```

---

## Five Architectural Layers

### Layer 1: Instructions Layer (skills/)
Plain Markdown files that tell Claude Code exactly what to do, step by step.

**Structure of every skill:**
```
skills/
└── skill-name/
    ├── SKILL.md          ← The "recipe" file
    └── templates/        ← (optional) LaTeX/output templates
```

**SKILL.md format:**
```yaml
---
name: skill-name
description: "What this skill does and when to invoke it"
argument-hint: [optional-arg]
allowed-tools: Bash(*), Read, Write, Edit, Grep, Glob, mcp__codex__codex
---

# Skill Title

## Constants
- SOME_SETTING = default_value

## Workflow
### Step 1: Do this
...
### Step 2: Then this
...
```

The `allowed-tools` frontmatter is a security boundary — it declares exactly which capabilities this skill is allowed to use.

### Layer 2: Execution Layer (Claude Code)
Claude Code is the execution backbone. It:
- Reads SKILL.md files when you type `/skill-name`
- Has direct access to your file system, shell, and internet
- Can spawn sub-agents (for parallelism)
- Can call MCP servers (for cross-model communication)

Think of Claude Code as an extremely capable intern who reads a procedure document and executes it faithfully.

### Layer 3: Review Layer (Codex MCP / external reviewer)
The second AI model acts as a critic. It is accessed via the Model Context Protocol (MCP):

```
Claude Code  ─── mcp__codex__codex(prompt) ───▶  Codex CLI ──▶  GPT-5.4
                                                      │
                ◀── reviewer response ────────────────┘
```

The Codex CLI is run in `--mcp-server` mode, which means it becomes a tool that Claude Code can call like a function.

### Layer 4: Bridge Layer (mcp-servers/)
Small Python HTTP servers that bridge Claude Code to external services:

| Server | Protocol | Purpose |
|--------|----------|---------|
| `llm-chat` | JSON-RPC (stdio) | Generic OpenAI-compatible API bridge |
| `feishu-bridge` | HTTP REST | Send/receive messages via Feishu |
| `claude-review` | JSON-RPC (stdio) | Claude Code as reviewer (for Codex-executor setups) |
| `minimax-chat` | JSON-RPC (stdio) | MiniMax API (non-OpenAI format) |

### Layer 5: State Layer (project files)
ARIS uses the file system as its database. Every workflow writes and reads from specific files:

| File | Purpose |
|------|---------|
| `CLAUDE.md` | Project configuration (server info, env settings) |
| `IDEA_REPORT.md` | Output of Workflow 1 |
| `AUTO_REVIEW.md` | Cumulative review log for Workflow 2 |
| `REVIEW_STATE.json` | Machine-readable state for crash recovery |
| `NARRATIVE_REPORT.md` | Human-written research narrative (Workflow 3 input) |
| `PAPER_PLAN.md` | Auto-generated paper outline |
| `paper/` | Final LaTeX paper directory |
| `refine-logs/` | Method refinement outputs |

---

## Component Interactions

### Synchronous Call Pattern (most common)
```
User: /skill-name "argument"
    ↓
Claude Code reads ~/.claude/skills/skill-name/SKILL.md
    ↓
Claude Code follows the instructions in the skill
    ↓ (when skill needs GPT-5.4 review)
Claude Code calls mcp__codex__codex(prompt="...")
    ↓
Codex CLI receives the call, forwards to OpenAI API
    ↓
GPT-5.4 returns a response
    ↓
Claude Code receives response, continues skill execution
```

### Asynchronous Experiment Pattern
```
Claude Code writes experiment script
    ↓
Claude Code SSHes to GPU server
    ↓
Launches: screen -dmS exp_name bash -c 'python train.py ...'
    ↓ (non-blocking — returns immediately)
Claude Code continues doing other work
    ↓ (periodically)
Claude Code polls: ssh server 'screen -ls' and checks log files
    ↓
When done, collects results and continues
```

### State Recovery Pattern
```
REVIEW_STATE.json exists AND status="in_progress" AND timestamp < 24h ago?
    YES → Read state, restore round/threadId/scores, resume from next round
    NO  → Fresh start
```

---

## Design Principles (Architectural)

### Principle 1: Stateless Instructions, Stateful Files
SKILL.md files are stateless instructions. All state lives in files (REVIEW_STATE.json, AUTO_REVIEW.md, etc.). This means: crash the agent, restart it, and it picks up where it left off.

### Principle 2: Fail-Open Integrations
Every optional integration (Zotero, Obsidian, Feishu, W&B) is checked at runtime and silently skipped if not configured. The core workflow always works, extras are purely additive.

### Principle 3: Human-Override at Every Gate
Every pipeline has configurable human checkpoints (`AUTO_PROCEED`, `HUMAN_CHECKPOINT`). You can run fully autonomous or pause at any step.

### Principle 4: Adversarial, Not Self-Play
The executor (Claude) and reviewer (GPT-5.4) are intentionally different models. They have different training data, different biases, different blind spots. This makes the review adversarial, not confirmatory.

### Principle 5: Model-Agnostic
Skills reference generic tool names (`mcp__codex__codex`, `mcp__llm-chat__chat`). Swapping the model means only changing the MCP server configuration, not rewriting any skills.

---

## Supported Execution Environments

ARIS skills can be executed by:

| Executor | How Skills Are Loaded | Notes |
|----------|-----------------------|-------|
| Claude Code (default) | `/skill-name` slash command | Native support |
| Codex CLI | `spawn_agent` tool | See `skills/skills-codex/` |
| Cursor | `@skill-name` reference | See `docs/CURSOR_ADAPTATION.md` |
| Trae (ByteDance) | Native SKILL.md support | See `docs/TRAE_ARIS_RUNBOOK_EN.md` |
| Antigravity (Google) | Native SKILL.md support | See `docs/ANTIGRAVITY_ADAPTATION.md` |
| OpenClaw | Manual stage mapping | See `docs/OPENCLAW_ADAPTATION.md` |

This portability is possible because **SKILL.md files are just text** — any LLM can read and follow them.
