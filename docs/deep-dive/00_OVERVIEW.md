# ARIS — The Complete Beginner's Guide

> **ARIS** = **A**uto-claude-code **R**esearch **I**n **S**leep  
> Also written as: ⚔️🌙 (sword + moon = fight battles while you sleep)

---

## What Is ARIS? (ELI5)

Imagine you're a researcher. You have an idea for a machine learning paper, but:
- Finding related work takes days
- Running experiments takes nights
- Writing and polishing a paper takes weeks
- Getting feedback from experts takes months

**ARIS automates all of that.** You type one command before bed, and wake up with a reviewed, scored, partially written research paper — with real experiments run on your GPU server.

It's not magic. It's a very well-designed assembly line of AI tools working together.

---

## The "Sleeping Researcher" Mental Model

Think of it like a factory that runs overnight:

```
You (8pm)     ──▶  You give ARIS a research direction
                         │
                         ▼
                   [Factory runs all night]
                         │
                   ┌─────┴──────────────────────────┐
                   │  Worker 1 (Claude Code):         │
                   │  Reads papers, writes code,      │
                   │  manages files, runs commands     │
                   │                                  │
                   │  Worker 2 (GPT-5.4):             │
                   │  Acts as a tough professor —     │
                   │  reviews everything, scores it,  │
                   │  points out flaws                │
                   └────────────────────────────────-─┘
                         │
You (8am)     ◀──  "Here's your scored paper with
                    20 GPU experiments done."
```

The key insight: **Two different AI models reviewing each other's work**, instead of one model talking to itself. This avoids the "yes-man" problem where an AI only validates its own ideas.

---

## Why Does This Exist?

Traditional research automation fails because:
1. **Self-review is blind** — if GPT writes a paper and GPT reviews it, it finds no problems (it literally cannot see its own blind spots)
2. **Single model = single perspective** — different models have different strengths and knowledge bases
3. **Most tools are too complex** — databases, frameworks, Docker containers... ARIS is just Markdown files

ARIS solves this with a radical simplicity principle: **the entire system is plain Markdown files**. No database, no framework, no daemon. Just text files that AI models read and act on.

---

## The Four Research Workflows

ARIS organizes research into four stages you can run independently or chain together:

| Workflow | What It Does | When to Use It |
|----------|-------------|----------------|
| **Workflow 1** — Idea Discovery | Survey literature → brainstorm ideas → verify novelty → pilot on GPU → rank | "I have a vague direction, help me find something concrete" |
| **Workflow 1.5** — Experiment Bridge | Parse your experiment plan → write code → review code → deploy to GPU → collect results | "I have a plan, implement and run it" |
| **Workflow 2** — Auto Review Loop | Review paper → fix weaknesses → run experiments → re-review (4 rounds max) | "I have results, iterate until submission-ready" |
| **Workflow 3** — Paper Writing | Narrative → outline → figures → LaTeX → PDF → polish | "Ready to write the paper" |

**Full Pipeline** (`/research-pipeline`): chains all four end-to-end.

---

## What Makes ARIS Unique

### 1. Zero Dependencies (Truly)
No pip install, no Docker, no database. Copy the `skills/` folder to `~/.claude/skills/` and you're done.

### 2. Cross-Model Adversarial Review
Claude Code (fast, fluid executor) × GPT-5.4 xhigh (slow, rigorous critic). They don't agree with each other — that's the point.

### 3. "Skills" Are Just Text
Every skill is a single `SKILL.md` file that any LLM can read. Want to use Cursor instead of Claude Code? Read the file. Want to use Gemini? Read the file. The instructions are model-agnostic.

### 4. Real Experiments, Not Fake Ones
When the reviewer says "run an ablation study," Claude Code actually SSHes into your GPU server, writes the script, launches it in a `screen` session, and monitors it.

### 5. Anti-Hallucination Citations
Instead of letting the AI make up BibTeX entries (a common disaster), ARIS fetches real citations from DBLP and CrossRef APIs.

---

## Quick Orientation: Folder Structure

```
Auto-claude-code-research-in-sleep/
│
├── skills/                 ← The heart of ARIS. 31 "skill" folders.
│   ├── idea-discovery/     ← Each folder has a SKILL.md
│   ├── auto-review-loop/
│   ├── paper-write/
│   └── ... (28 more)
│
├── mcp-servers/            ← 4 tiny Python servers for external communication
│   ├── llm-chat/           ← Bridge to any OpenAI-compatible API
│   ├── feishu-bridge/      ← Mobile notifications via Feishu/Lark
│   ├── claude-review/      ← Claude as reviewer (for Codex-executor setups)
│   └── minimax-chat/       ← Bridge to MiniMax API specifically
│
├── tools/                  ← Python utilities
│   ├── arxiv_fetch.py      ← arXiv API search/download tool
│   └── generate_codex_claude_review_overrides.py  ← Codex skill generator
│
├── templates/              ← Input templates for each workflow
│   ├── RESEARCH_BRIEF_TEMPLATE.md    ← For Workflow 1
│   ├── EXPERIMENT_PLAN_TEMPLATE.md   ← For Workflow 1.5
│   ├── NARRATIVE_REPORT_TEMPLATE.md  ← For Workflow 3
│   └── PAPER_PLAN_TEMPLATE.md        ← For Workflow 3
│
└── docs/                   ← Documentation (you are here)
    └── deep-dive/          ← This folder: deep technical docs
```

---

## How to Read These Docs

This `deep-dive/` folder contains 9 documents that progressively go deeper:

| File | What's Inside | Who Should Read It |
|------|-------------|-------------------|
| `00_OVERVIEW.md` | This file — the big picture | Everyone |
| `01_ARCHITECTURE.md` | System components and how they connect | Curious minds |
| `02_AGENT_ARCHITECTURE.md` | The two-agent design and why it works | ML researchers |
| `03_WORKFLOWS.md` | Each workflow in detail | Users building pipelines |
| `04_SKILLS_REFERENCE.md` | Every skill, its inputs/outputs/purpose | Skill authors |
| `05_DATA_FLOW.md` | How data moves through the system | Debuggers |
| `06_DATA_SCHEMA.md` | All file formats and schemas | Integrators |
| `07_MCP_SERVERS.md` | The Python servers and their APIs | Developers |
| `08_INTEGRATIONS.md` | Feishu, Zotero, W&B, arXiv, etc. | Power users |
| `09_DESIGN_RATIONALE.md` | Why things are designed this way | Everyone curious |

---

## The "Surprising" Things About ARIS

Things that will raise your eyebrows when you first see them:

1. **The entire "AI agent" is a Markdown file** — there's no Python agent framework. Claude Code just reads a `.md` file and follows its instructions like a recipe.

2. **State is stored in plain JSON** — `REVIEW_STATE.json` is how the system survives crashes and resumes overnight loops.

3. **The reviewer model runs inside a CLI tool** — Codex CLI (OpenAI's terminal tool) acts as an MCP server, meaning Claude Code can call GPT-5.4 like a function.

4. **You can swap every component** — Claude → GLM, GPT-5.4 → MiniMax, Codex MCP → llm-chat MCP. The whole thing is model-agnostic.

5. **Papers accepted at AAAI 2026 were written with this** — This is not a toy. Real papers (including an AAAI 2026 accepted paper) came out of this pipeline.
