# ARIS Data Flow

## Overview

ARIS is fundamentally file-based. Every piece of state, every intermediate result, every configuration lives in a file. This section traces exactly how data moves through each workflow.

---

## Workflow 1: Idea Discovery — Data Flow

```
User Input: "/idea-discovery 'discrete diffusion models'"
     │
     ▼
┌─────────────────────────────────────────────────────────────┐
│ Phase 1: Literature Survey (/research-lit)                   │
│                                                             │
│ Sources checked in order:                                   │
│   [Zotero] ──────────────┐                                 │
│   [Obsidian vault] ───────┤                                 │
│   [Local PDFs] ───────────┤──▶ Literature summary          │
│   [arXiv API] ────────────┤    (working notes)             │
│   [Web search] ───────────┘                                 │
│                                                             │
│   tools/arxiv_fetch.py ──▶ Structured metadata (JSON)       │
│   DBLP API ──────────────▶ (optional citation export)       │
└─────────────────────────────────────────────────────────────┘
     │
     ▼ (literature summary)
┌─────────────────────────────────────────────────────────────┐
│ Phase 2: Idea Generation (/idea-creator)                     │
│                                                             │
│   Literature summary ──▶ mcp__codex__codex (GPT-5.4 brainstorm) │
│                              │                              │
│                              ▼                              │
│                       8-12 idea candidates                   │
│                              │                              │
│                       Filter by feasibility                  │
│                              │                              │
│              ┌───────────────┼───────────────┐             │
│              ▼               ▼               ▼             │
│          Idea 1           Idea 2          Idea 3           │
│           GPU             GPU             GPU              │
│           pilot           pilot           pilot            │
│          (screen)        (screen)        (screen)          │
│              │               │               │             │
│              └───────────────┼───────────────┘             │
│                              ▼                              │
│                       Ranked by signal                       │
│                              ▼                              │
│                       IDEA_REPORT.md (draft)                 │
└─────────────────────────────────────────────────────────────┘
     │
     ▼
┌─────────────────────────────────────────────────────────────┐
│ Phase 3: Novelty Check (/novelty-check)                      │
│                                                             │
│   Top ideas ──▶ arXiv/Scholar/Semantic Scholar search       │
│              ──▶ mcp__codex__codex (cross-verify)           │
│                              │                              │
│                       Novelty verdicts                       │
│                              │                              │
│                   IDEA_REPORT.md (updated)                   │
└─────────────────────────────────────────────────────────────┘
     │
     ▼
┌─────────────────────────────────────────────────────────────┐
│ Phase 4: Critical Review (/research-review)                  │
│                                                             │
│   Top idea + pilots + novelty ──▶ mcp__codex__codex         │
│                                        │                    │
│                                 Score + weaknesses          │
│                                        │                    │
│                              IDEA_REPORT.md (final)          │
└─────────────────────────────────────────────────────────────┘
     │
     ▼
┌─────────────────────────────────────────────────────────────┐
│ Phase 4.5: Method Refinement (/research-refine-pipeline)     │
│                                                             │
│   Top idea + feedback ──▶ /research-refine (up to 5 rounds) │
│                                │                            │
│                                ▼                            │
│              refine-logs/FINAL_PROPOSAL.md                   │
│                                │                            │
│                                ▼ (fed to experiment-plan)   │
│              refine-logs/EXPERIMENT_PLAN.md                  │
│              refine-logs/EXPERIMENT_TRACKER.md               │
└─────────────────────────────────────────────────────────────┘
     │
     ▼
Final Output: IDEA_REPORT.md + refine-logs/
```

---

## Workflow 1.5: Experiment Bridge — Data Flow

```
refine-logs/EXPERIMENT_PLAN.md
     │
     ▼
┌─────────────────────────────────────────────────────────────┐
│ /experiment-bridge                                           │
│                                                             │
│ 1. Parse EXPERIMENT_PLAN.md ──▶ List of experiments         │
│                                                             │
│ 2. Implement experiment scripts                             │
│    (reads CLAUDE.md for conventions)                        │
│    Outputs: *.py scripts                                    │
│                                                             │
│ 3. Code review                                              │
│    *.py scripts ──▶ mcp__codex__codex (GPT-5.4 code review) │
│                         │                                   │
│                   CRITICAL/MAJOR issues? ──▶ fix ──▶ retry  │
│                   PASS? ──▶ continue                        │
│                                                             │
│ 4. Sanity check                                             │
│    Smallest experiment ──▶ local run ──▶ verify no errors   │
│                                                             │
│ 5. Deploy via /run-experiment                               │
│    CLAUDE.md (server config) ──▶ SSH + rsync/git ──▶ screen │
│                                                             │
│ 6. Collect initial results                                  │
│    Screen logs ──▶ JSON/CSV ──▶ EXPERIMENT_TRACKER.md        │
└─────────────────────────────────────────────────────────────┘
     │
     ▼
Output: Running experiments on GPU + initial results
```

### Code Sync Flow

**Option A (rsync):**
```
Local *.py files ──▶ rsync over SSH ──▶ Remote code directory
```

**Option B (git):**
```
Local repo ──▶ git add/commit/push ──▶ GitHub
                                           │
                                           ▼
                              SSH to server ──▶ git pull
```

---

## Workflow 2: Auto Review Loop — Data Flow

```
Research context (files in project dir)
     │
     ▼
┌─────────────────────────────────────────────────────────────┐
│ Round N of /auto-review-loop                                 │
│                                                             │
│ Phase A: Review                                             │
│   Research context + prior results                          │
│         │                                                   │
│         ▼                                                   │
│   mcp__codex__codex (Round 1) ─────────────────────────┐   │
│   mcp__codex__codex-reply (Rounds 2-4) ─────────────────┘  │
│         │                      │                            │
│         ▼                      ▼                            │
│   Response text           threadId (saved to               │
│   (verbatim)              REVIEW_STATE.json)                │
│                                                             │
│ Phase B: Parse                                              │
│   Response text ──▶ score (float), verdict, action items   │
│   score ≥ 6 AND "ready/almost"? ──▶ STOP                   │
│                                                             │
│ [Feishu notification if configured]                         │
│                                                             │
│ Phase C: Implement Fixes                                    │
│   Action items ──▶ code changes / new experiments           │
│   New experiments ──▶ /run-experiment ──▶ screen sessions   │
│                                                             │
│ Phase D: Wait for Results                                   │
│   Poll: ssh server 'screen -ls' + read log files            │
│   Results ──▶ JSON/CSV files                                │
│                                                             │
│ Phase E: Document                                           │
│   All of the above ──▶ AUTO_REVIEW.md (appended)            │
│   State ──▶ REVIEW_STATE.json (overwritten)                 │
│                                                             │
│ ──▶ Repeat up to MAX_ROUNDS                                 │
└─────────────────────────────────────────────────────────────┘
     │
     ▼
Final: AUTO_REVIEW.md with ## Method Description section
       (used as input to /paper-illustration in Workflow 3)
```

### REVIEW_STATE.json Flow

```
[Round 1 ends]
    │
    ▼
REVIEW_STATE.json = {round: 1, status: "in_progress", ...}
    │
[Potential crash / context compaction]
    │
[Restart]
    │
    ▼
Check REVIEW_STATE.json:
  - exists? YES
  - status == "in_progress"? YES
  - timestamp < 24h? YES
    ──▶ Resume from round 2 with saved threadId
  - timestamp > 24h?
    ──▶ Fresh start (stale state)
```

---

## Workflow 3: Paper Writing — Data Flow

```
NARRATIVE_REPORT.md (human-written)
     │
     ▼
┌─────────────────────────────────────────────────────────────┐
│ /paper-plan                                                  │
│                                                             │
│ NARRATIVE_REPORT.md ──▶ Claims-Evidence Matrix              │
│                     ──▶ Section structure                   │
│                     ──▶ Figure plan                         │
│                     ──▶ Citation scaffolding                │
│                              │                              │
│                       PAPER_PLAN.md                          │
└─────────────────────────────────────────────────────────────┘
     │
     ▼
┌─────────────────────────────────────────────────────────────┐
│ /paper-figure                                                │
│                                                             │
│ Experiment results (JSON/CSV) + PAPER_PLAN.md               │
│         │                                                   │
│         ▼                                                   │
│   matplotlib/seaborn scripts ──▶ figures/*.pdf/png          │
│   LaTeX comparison tables ──▶ figures/*.tex                 │
│   Combined ──▶ figures/latex_includes.tex                   │
└─────────────────────────────────────────────────────────────┘
     │
     ▼
┌─────────────────────────────────────────────────────────────┐
│ /paper-write                                                 │
│                                                             │
│ Inputs:                                                     │
│   PAPER_PLAN.md + NARRATIVE_REPORT.md + figures/            │
│         │                                                   │
│ For each section:                                           │
│   1. Read plan ──▶ Draft LaTeX                              │
│   2. Insert figures (from latex_includes.tex)               │
│   3. Add citations                                          │
│                                                             │
│ For each citation:                                          │
│   curl DBLP API ──▶ real BibTeX? ──▶ use it                 │
│   Not found ──▶ curl CrossRef ──▶ real BibTeX? ──▶ use it   │
│   Not found ──▶ mark [VERIFY]                               │
│                                                             │
│ After all sections:                                         │
│   De-AI polish (remove: delve, pivotal, groundbreaking...)  │
│   ──▶ mcp__codex__codex (GPT-5.4 full paper review)        │
│   Reverse outline test                                      │
│                                                             │
│ Outputs:                                                    │
│   paper/main.tex                                            │
│   paper/sections/0_abstract.tex ... N_conclusion.tex        │
│   paper/references.bib (only cited entries)                 │
│   paper/math_commands.tex                                   │
│   paper/{venue}.sty                                         │
└─────────────────────────────────────────────────────────────┘
     │
     ▼
┌─────────────────────────────────────────────────────────────┐
│ /paper-compile                                               │
│                                                             │
│ paper/ ──▶ latexmk -pdf ──▶ paper/main.pdf                  │
│                 │                                           │
│            Errors? ──▶ auto-fix ──▶ retry (up to 3x)        │
│                                                             │
│ Checks:                                                     │
│   pdftotext main.pdf | wc -l ──▶ page count                 │
│   grep undefined references, missing citations              │
└─────────────────────────────────────────────────────────────┘
     │
     ▼
┌─────────────────────────────────────────────────────────────┐
│ /auto-paper-improvement-loop                                 │
│                                                             │
│ Round 1: Content review ──▶ CRITICAL/MAJOR fixes ──▶ recompile │
│ Round 2: Re-review ──▶ fixes ──▶ recompile                  │
│ Round 3: Format compliance ──▶ page count, margins, etc.    │
│                                                             │
│ Final: paper/main.pdf (submission-ready)                    │
└─────────────────────────────────────────────────────────────┘
```

---

## External Service Data Flows

### arXiv API Flow
```
tools/arxiv_fetch.py search "query"
     │
     ▼
GET http://export.arxiv.org/api/query?search_query=...
     │
     ▼
Atom/XML response ──▶ parsed to JSON:
{id, title, authors, abstract, published, categories, pdf_url, abs_url}
     │
     ▼
Printed as JSON to stdout ──▶ Claude Code reads it
```

### DBLP/CrossRef Citation Flow
```
/paper-write encounters \citep{vaswani2017attention}
     │
     ▼
curl "https://dblp.org/search/publ/api?q=attention+vaswani&format=json&h=3"
     │
     ▼
Extract DBLP key (e.g., conf/nips/VaswaniSPUJGKP17)
     │
     ▼
curl "https://dblp.org/rec/conf/nips/VaswaniSPUJGKP17.bib"
     │
     ▼
Real BibTeX ──▶ references.bib
(Falls back to CrossRef, then [VERIFY] if not found)
```

### Feishu Notification Flow
```
Skill checks: ~/.claude/feishu.json exists AND mode != "off"?
     │
     YES ──▶ POST to webhook_url (push mode)
              or POST to feishu-bridge server:5000/send (interactive mode)
     │
     NO  ──▶ silent skip (no-op)
```

### GPU Experiment Flow
```
Claude Code ──▶ ssh server "screen -dmS exp_1 bash -c '...'"
     │
     ▼
screen session running on remote GPU (detached)
     │
[Periodically]
Claude Code ──▶ ssh server "screen -ls" ──▶ session still alive?
           ──▶ ssh server "tail -50 exp_1.log" ──▶ training metrics
           ──▶ ssh server "cat results/exp_1.json" ──▶ final results
```

---

## Data Lifecycle Summary

```
Input                    Processing              Output
─────────────────────────────────────────────────────────────
Research direction  ──▶  research-lit      ──▶  Literature notes
Literature notes    ──▶  idea-creator      ──▶  IDEA_REPORT.md
Top idea            ──▶  novelty-check     ──▶  Novelty verdict
Top idea + verdict  ──▶  research-review   ──▶  Score + action items
Idea + feedback     ──▶  research-refine   ──▶  FINAL_PROPOSAL.md
Proposal            ──▶  experiment-plan   ──▶  EXPERIMENT_PLAN.md
Plan                ──▶  experiment-bridge ──▶  Running experiments
Results             ──▶  auto-review-loop  ──▶  AUTO_REVIEW.md
Narrative           ──▶  paper-plan        ──▶  PAPER_PLAN.md
PAPER_PLAN.md       ──▶  paper-figure      ──▶  figures/
PAPER_PLAN.md       ──▶  paper-write       ──▶  paper/
paper/              ──▶  paper-compile     ──▶  paper/main.pdf
main.pdf            ──▶  improvement-loop  ──▶  submission-ready PDF
```
