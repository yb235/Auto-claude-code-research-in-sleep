# ARIS Skills Reference

## What Is a Skill?

A "skill" is a plain Markdown file (`SKILL.md`) that tells Claude Code how to perform a specific research task. It's the core primitive of ARIS.

Every skill has:
- **Frontmatter** (YAML): metadata, tool permissions
- **Constants**: configurable defaults with `— override: value` syntax
- **Workflow**: numbered steps the AI follows like a recipe

---

## Skill Frontmatter Anatomy

```yaml
---
name: skill-name                    # Invoked as /skill-name in Claude Code
description: "What it does..."      # Used for auto-detection and routing
argument-hint: [optional-arg]       # Hint shown to the user
allowed-tools: Bash(*), Read, Write, Edit, Grep, Glob, Agent, Skill,
               mcp__codex__codex, mcp__codex__codex-reply
---
```

**`allowed-tools` values explained:**
| Tool | What It Grants |
|------|---------------|
| `Bash(*)` | Run any shell command |
| `Read` | Read any file |
| `Write` | Create new files |
| `Edit` | Modify existing files |
| `Grep` | Search file contents |
| `Glob` | List files by pattern |
| `WebSearch` | Search the internet |
| `WebFetch` | Fetch a URL |
| `Agent` | Spawn sub-agents (parallelism) |
| `Skill` | Invoke another skill |
| `mcp__codex__codex` | Call GPT-5.4 (new thread) |
| `mcp__codex__codex-reply` | Continue GPT-5.4 thread |
| `mcp__zotero__*` | Access Zotero library |
| `mcp__obsidian-vault__*` | Access Obsidian vault |

---

## Complete Skills Catalog

### 🚀 Full Pipeline

#### `research-pipeline`
**Purpose:** One command from idea to submission-ready paper  
**Invocation:** `/research-pipeline "your research direction"`  
**Chains:** `idea-discovery` → implement → `run-experiment` → `auto-review-loop`  
**Key constants:** `AUTO_PROCEED=true`, `HUMAN_CHECKPOINT=false`, `ARXIV_DOWNLOAD=false`  
**Reviewer model:** GPT-5.4 xhigh  
**Inputs:** Research direction string  
**Outputs:** `IDEA_REPORT.md`, `AUTO_REVIEW.md`, REVIEW_STATE.json, experiment results

---

### 🔍 Workflow 1: Idea Discovery

#### `idea-discovery`
**Purpose:** Pipeline orchestrator for idea discovery  
**Invocation:** `/idea-discovery "your research direction"`  
**Chains:** `research-lit` → `idea-creator` → `novelty-check` → `research-review` → `research-refine-pipeline`  
**Key constants:** `PILOT_MAX_HOURS=2`, `MAX_PILOT_IDEAS=3`, `MAX_TOTAL_GPU_HOURS=8`, `AUTO_PROCEED=true`  
**Inputs:** Research direction string  
**Outputs:** `IDEA_REPORT.md`, `refine-logs/FINAL_PROPOSAL.md`, `refine-logs/EXPERIMENT_PLAN.md`  
**Codex MCP:** Yes

---

#### `research-lit`
**Purpose:** Multi-source literature survey  
**Invocation:** `/research-lit "topic"` or with `— sources: zotero, web`  
**Key constants:** `PAPER_LIBRARY=papers/,literature/`, `MAX_LOCAL_PAPERS=20`, `ARXIV_DOWNLOAD=false`  
**Data sources (in priority order):**
1. Zotero (via `mcp__zotero__*`) — if configured
2. Obsidian vault (via `mcp__obsidian-vault__*`) — if configured
3. Local PDFs in `papers/` or `literature/`
4. Web: arXiv API (`tools/arxiv_fetch.py`), Semantic Scholar, Google Scholar
**Inputs:** Topic string  
**Outputs:** Literature summary in working notes, optional `papers/*.pdf` downloads  
**Codex MCP:** No (pure executor)

---

#### `idea-creator`
**Purpose:** Brainstorm research ideas, run GPU pilots, rank by signal  
**Invocation:** `/idea-creator "topic"`  
**What it does:**
- Asks GPT-5.4 to brainstorm 8-12 concrete ideas
- Filters by feasibility and compute cost
- Runs parallel GPU pilots on top 2-3 ideas
- Deep novelty + devil's advocate check on survivors
- Ranks ideas by empirical signal
**Inputs:** Topic string + literature survey context  
**Outputs:** `IDEA_REPORT.md`  
**Codex MCP:** Yes

---

#### `novelty-check`
**Purpose:** Deep multi-source novelty verification  
**Invocation:** `/novelty-check "idea description"`  
**What it does:**
- Multi-source search (arXiv, Scholar, Semantic Scholar)
- Cross-verifies with GPT-5.4
- Checks for concurrent work (last 3-6 months)
- Identifies closest existing work + differentiation points
**Inputs:** Idea description  
**Outputs:** Novelty verdict added to working notes  
**Codex MCP:** Yes

---

#### `research-review`
**Purpose:** Single-round deep review by external LLM  
**Invocation:** `/research-review "your paper/idea"`  
**What it does:** GPT-5.4 acts as a senior reviewer (NeurIPS/ICML level), scores 1-10, identifies weaknesses, suggests minimum viable improvements  
**Inputs:** Research content string  
**Outputs:** Review report with score, weaknesses, action items  
**Codex MCP:** Yes

---

#### `research-refine-pipeline`
**Purpose:** Refine method + plan experiments in one chain  
**Invocation:** `/research-refine-pipeline "idea + results + reviewer feedback"`  
**Chains:** `research-refine` → `experiment-plan`  
**Outputs:** `refine-logs/FINAL_PROPOSAL.md`, `refine-logs/EXPERIMENT_PLAN.md`  
**Codex MCP:** Yes

---

#### `research-refine`
**Purpose:** Iterative method refinement (up to 5 rounds, target score ≥ 9)  
**What it does:**
- Freezes a "Problem Anchor" to prevent scope drift
- Each round: executor proposes improvements, reviewer scores
- Stops when score ≥ 9 or 5 rounds elapsed
**Inputs:** Idea description + pilot results + reviewer feedback  
**Outputs:** `refine-logs/FINAL_PROPOSAL.md`  
**Codex MCP:** Yes

---

#### `experiment-plan`
**Purpose:** Claim-driven experiment roadmap  
**What it does:** Turns a refined proposal into an ordered list of experiments with ablations, compute budgets, and priority ranking  
**Inputs:** `refine-logs/FINAL_PROPOSAL.md`  
**Outputs:** `refine-logs/EXPERIMENT_PLAN.md`  
**Codex MCP:** No

---

### 🔗 Workflow 1.5: Experiment Bridge

#### `experiment-bridge`
**Purpose:** Read plan → write code → review code → deploy → collect  
**Invocation:** `/experiment-bridge` or `/experiment-bridge "my_plan.md"`  
**Key constants:** `CODE_REVIEW=true`, `AUTO_DEPLOY=true`, `SANITY_FIRST=true`, `MAX_PARALLEL_RUNS=4`  
**Chains:** (internal) → `run-experiment` → `monitor-experiment`  
**Inputs:** `refine-logs/EXPERIMENT_PLAN.md`  
**Outputs:** Experiment scripts, updated `refine-logs/EXPERIMENT_TRACKER.md`, initial results  
**Codex MCP:** No (but invokes it for code review step)

---

#### `run-experiment`
**Purpose:** Deploy experiments to local (MPS/CUDA) or remote GPU servers  
**Invocation:** `/run-experiment train.py --lr 1e-4`  
**What it reads:** `CLAUDE.md` for server configuration  
**Code sync options:**
- `rsync` (default): sync only `.py` files
- `git` (optional): `git push` + `ssh "git pull"`
**Deployment:** `screen -dmS exp_name bash -c '...'` on remote  
**W&B integration:** Auto-adds `wandb.init()` + `wandb.log()` when `wandb=true`  
**Inputs:** Experiment command + server config in `CLAUDE.md`  
**Outputs:** Running screen sessions on GPU server, Feishu notification if configured  
**Codex MCP:** No

---

#### `monitor-experiment`
**Purpose:** Check experiment progress and collect results  
**Invocation:** `/monitor-experiment server5`  
**What it does:**
- SSH to server, check screen sessions
- Read log files for training metrics
- If W&B configured, pull training curves
- Collect results to JSON/CSV
**Inputs:** Server name  
**Outputs:** Updated experiment results, Feishu notification if configured  
**Codex MCP:** No

---

### 🔁 Workflow 2: Auto Research Loop

#### `auto-review-loop`
**Purpose:** Autonomous multi-round review → fix → re-review cycle  
**Invocation:** `/auto-review-loop "topic"` or with `— human checkpoint: true`  
**Key constants:** `MAX_ROUNDS=4`, `POSITIVE_THRESHOLD=6/10`, `HUMAN_CHECKPOINT=false`  
**Review format:** `mcp__codex__codex` (round 1) then `mcp__codex__codex-reply` (rounds 2-4)  
**State persistence:** `REVIEW_STATE.json` (crash recovery)  
**Inputs:** Research context (reads project files)  
**Outputs:** `AUTO_REVIEW.md` (cumulative), `REVIEW_STATE.json`  
**Codex MCP:** Yes

---

#### `auto-review-loop-llm`
**Purpose:** Same as `auto-review-loop` but uses `mcp__llm-chat__chat` instead of Codex  
**Use case:** When using DeepSeek, MiniMax, GLM, or other OpenAI-compatible APIs as reviewer  
**Codex MCP:** No (uses `llm-chat` MCP server)

---

#### `analyze-results`
**Purpose:** Analyze experiment results, compute statistics, generate insights  
**Invocation:** `/analyze-results figures/*.json`  
**Inputs:** JSON/CSV result files  
**Outputs:** Statistical analysis, insights added to working notes  
**Codex MCP:** No

---

### 📝 Workflow 3: Paper Writing

#### `paper-writing`
**Purpose:** Full paper writing pipeline orchestrator  
**Invocation:** `/paper-writing "NARRATIVE_REPORT.md"`  
**Chains:** `paper-plan` → `paper-figure` → `paper-illustration` (optional) → `paper-write` → `paper-compile` → `auto-paper-improvement-loop`  
**Codex MCP:** Yes

---

#### `paper-plan`
**Purpose:** Claims-Evidence Matrix + section structure + figure plan  
**Inputs:** `NARRATIVE_REPORT.md`  
**Outputs:** `PAPER_PLAN.md`  
**What it verifies:** Every claim has evidence, every experiment supports a claim  
**Codex MCP:** Yes

---

#### `paper-figure`
**Purpose:** Auto-generate publication-quality figures from data  
**What it generates:**
- Training curves (matplotlib/seaborn)
- Bar charts, heatmaps
- LaTeX comparison tables
- `figures/latex_includes.tex` (ready-to-paste snippets)
**Inputs:** JSON/CSV data files + `PAPER_PLAN.md`  
**Outputs:** PDF/PNG files in `figures/`  
**Codex MCP:** Optional

---

#### `paper-illustration`
**Purpose:** AI-generated architecture diagrams and method figures  
**How it works:** Claude plans the diagram → Gemini renders it → iterative refinement until score ≥ 9  
**Requirements:** `GEMINI_API_KEY` environment variable  
**Inputs:** Method description  
**Outputs:** SVG/PNG architecture figures  
**Codex MCP:** No (uses Gemini API directly)

---

#### `paper-write`
**Purpose:** Section-by-section LaTeX generation  
**Key constants:** `TARGET_VENUE=ICLR`, `DBLP_BIBTEX=true`, `ANONYMOUS=true`, `MAX_PAGES=9`  
**8-step workflow:**
1. Backup existing `paper/`
2. Initialize venue template
3. Generate `math_commands.tex`
4. Write each section (in order)
5. Build bibliography (DBLP → CrossRef → `[VERIFY]`)
6. De-AI polish pass
7. Cross-review with GPT-5.4
8. Reverse outline test
**Inputs:** `PAPER_PLAN.md`, `NARRATIVE_REPORT.md`, `figures/`  
**Outputs:** `paper/` directory with full LaTeX source  
**Codex MCP:** Yes

---

#### `paper-compile`
**Purpose:** Build PDF, auto-fix errors, submission readiness check  
**What it runs:** `latexmk -pdf -interaction=nonstopmode`  
**Checks:** Undefined references, missing citations, page count (via `pdftotext`), overfull boxes  
**Inputs:** `paper/` directory  
**Outputs:** `paper/main.pdf`  
**Codex MCP:** No

---

#### `auto-paper-improvement-loop`
**Purpose:** 2-round content review + format compliance check  
**Score progression (real test):** 4/10 → 8.5/10 in 3 rounds  
**Codex MCP:** Yes

---

### 🛠️ Standalone / Utility Skills

#### `arxiv`
**Purpose:** Search, download, and summarize arXiv papers  
**Invocation:** `/arxiv "topic"` or `/arxiv "2301.07041" — download`  
**Under the hood:** Calls `tools/arxiv_fetch.py`  
**Inputs:** Query string or arXiv ID  
**Outputs:** Structured metadata JSON, optional PDF download  
**Codex MCP:** No

---

#### `feishu-notify`
**Purpose:** Push notifications or interactive chat via Feishu/Lark  
**Used by:** All major skills (auto-detected, no-op if not configured)  
**Config:** `~/.claude/feishu.json`  
**Modes:** `off` (default), `push` (webhook), `interactive` (bidirectional)  
**Codex MCP:** No

---

#### `pixel-art`
**Purpose:** Generate pixel art SVG illustrations  
**Use cases:** READMEs, docs, slides  
**Codex MCP:** No

---

### 🎉 Community Skills (Not in Core Workflows)

| Skill | Purpose | Domain |
|-------|---------|--------|
| `research-refine` | Problem-anchored method refinement | General |
| `experiment-plan` | Claim-driven experiment roadmap | General |
| `grant-proposal` | Draft grant proposals (9 agencies) | General |
| `paper-poster` | Conference poster (tcbposter → A0/A1 PDF + PPTX) | General |
| `paper-slides` | Conference talk slides (beamer → PDF + PPTX + script) | General |
| `proof-writer` | Rigorous theorem proof drafting | ML Theory |
| `comm-lit-review` | Literature review for comms/wireless | Comms |
| `dse-loop` | Design space exploration (gem5, Yosys) | EDA/Architecture |
| `idea-discovery-robot` | Idea discovery adapted for robotics | Robotics |
| `mermaid-diagram` | Mermaid diagrams (free alternative to paper-illustration) | General |
| `formula-derivation` | Formula development and verification | Math |
| `paper-illustration` | AI-generated figures via Gemini | General |

---

## Skill Composition Patterns

Skills can be composed in three ways:

### 1. Pipeline Orchestration (using `Skill` tool)
```
/idea-discovery internally invokes:
  Skill(/research-lit, ...)
  Skill(/idea-creator, ...)
  Skill(/novelty-check, ...)
  Skill(/research-review, ...)
  Skill(/research-refine-pipeline, ...)
```

### 2. Parameter Pass-Through
Parameters flow down the call chain:
```
/research-pipeline "topic" — sources: zotero, arxiv download: true
    ↓ passes "sources" and "arxiv download" to
/idea-discovery
    ↓ passes "arxiv download" to
/research-lit
```

### 3. Manual Chaining
You can run skills manually in sequence:
```
/research-lit "discrete diffusion"
/idea-creator "DLLMs post training"
/novelty-check "top idea"
/research-refine "top idea + feedback"
/experiment-plan
/experiment-bridge
/auto-review-loop "topic"
/paper-writing "NARRATIVE_REPORT.md"
```
