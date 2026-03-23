# ARIS Workflows — In Depth

## The Full Pipeline at a Glance

```
/research-lit → /idea-creator → /novelty-check → /research-refine → /experiment-bridge → /auto-review-loop → /paper-plan → /paper-figure → /paper-write → submit
  (survey)       (brainstorm)    (verify novel)   (refine method)   (implement+deploy)  (review & fix)      (outline)      (plots)         (LaTeX+PDF)    (done!)
  ├────────────────── Workflow 1: Idea Discovery ─────────────────┤ ├── W1.5 ──┤ ├────── Workflow 2 ──────┤   ├──────────────── Workflow 3 ────────────────────┤
```

---

## Workflow 1: Idea Discovery & Method Refinement

> **Command:** `/idea-discovery "your research direction"`
> **Duration:** 1-4 hours  
> **Can run overnight:** Yes (if `AUTO_PROCEED=true`)

### What It Does

Turns a vague research direction into a validated, pilot-tested idea with an experiment plan ready to run.

### The 5 Phases

#### Phase 1: Literature Survey (`/research-lit`)

The executor sweeps multiple sources to build a "landscape map" of the field:

```
Priority order:
1. Zotero library (if configured) — your annotated paper collection
2. Obsidian vault (if configured) — your research notes
3. Local PDFs in papers/ or literature/ directories
4. Web: arXiv API, Semantic Scholar, Google Scholar
```

**What it produces:**
- Which sub-directions exist
- What approaches have been tried
- Recurring limitations in current work
- Structural gaps that could be addressed

**Key design:** Zotero and Obsidian are checked first because they contain papers you've already read and annotated — your personal knowledge is more targeted than fresh web search.

**arXiv integration:** The `tools/arxiv_fetch.py` script is called directly for structured metadata (title, abstract, full author list, categories) — richer than web scraping. PDF download is optional (`arxiv download: true`).

**Output:** Literature summary saved to working notes.

**Checkpoint:** ARIS shows you the landscape. You can redirect scope before continuing.

#### Phase 2: Idea Generation + Pilots (`/idea-creator`)

GPT-5.4 xhigh brainstorms 8-12 concrete research ideas based on the literature survey. Then:

1. **Filter by feasibility** — remove ideas that need unavailable resources or >2hr GPU pilot
2. **Quick novelty scan** — check if the idea already exists (fast, shallow check)
3. **Run parallel pilots** on top 2-3 ideas (up to MAX_PILOT_IDEAS=3)
4. **Rank by empirical signal** — a positive pilot outranks "sounds great"

**Key constants:**
- `PILOT_MAX_HOURS = 2` — skip any pilot estimated >2h per GPU
- `PILOT_TIMEOUT_HOURS = 3` — hard kill if a pilot runs too long
- `MAX_TOTAL_GPU_HOURS = 8` — budget across all pilots

**Output:** `IDEA_REPORT.md` with ranked ideas.

**Checkpoint:** You see the ranked ideas. You can pick, reject, or ask for regeneration.

#### Phase 3: Deep Novelty Check (`/novelty-check`)

For top ideas with positive pilot signal, a thorough novelty verification:
- Multi-source literature search (arXiv, Scholar, Semantic Scholar)
- Cross-verify with GPT-5.4
- Check for concurrent work (last 3-6 months)
- Identify closest existing work and differentiation points

**Why this is separate from Phase 2's quick check:** Phase 2's novelty filter is fast and shallow (just enough to eliminate obvious duplicates). Phase 3 is deep and adversarial (designed to kill your idea if it's not novel enough).

#### Phase 4: Critical Review (`/research-review`)

GPT-5.4 acts as a senior NeurIPS/ICML reviewer on your top idea:
- Scores the idea's potential
- Lists fundamental weaknesses
- Suggests minimum viable improvements to the experimental design
- Provides concrete feedback on methodology

**Key behavior:** The executor includes negative results and failed pilots in the review prompt. Honesty here prevents wasted effort later.

#### Phase 4.5: Method Refinement + Experiment Planning (`/research-refine-pipeline`)

This chains two skills:

1. **`/research-refine`**: Iteratively refines the method via GPT-5.4 review (up to 5 rounds, target score ≥ 9)
   - Freezes a "Problem Anchor" to prevent scope drift
   - Each round: executor proposes improvements, reviewer scores and critiques
   - Output: `refine-logs/FINAL_PROPOSAL.md`

2. **`/experiment-plan`**: Turns the refined proposal into a claim-driven experiment roadmap
   - Maps claims to experiments
   - Prioritizes by budget and impact
   - Output: `refine-logs/EXPERIMENT_PLAN.md`

**Output files:**
- `refine-logs/FINAL_PROPOSAL.md` — the refined method
- `refine-logs/EXPERIMENT_PLAN.md` — ordered list of experiments to run
- `refine-logs/EXPERIMENT_TRACKER.md` — status tracker for each experiment

#### Phase 5: Final Report

`IDEA_REPORT.md` is finalized with all accumulated information: ranked ideas, novelty verdicts, reviewer scores, refined proposal reference, and next steps.

---

## Workflow 1.5: Experiment Bridge

> **Command:** `/experiment-bridge` (reads `refine-logs/EXPERIMENT_PLAN.md` automatically)
> **Duration:** 30-120 min  
> **Can run overnight:** Yes

### What It Does

Takes an experiment plan and turns it into running code on your GPU server.

### The 6 Steps

1. **Parse** the experiment plan (reads EXPERIMENT_PLAN.md)
2. **Implement** experiment scripts
   - Reuses existing code where possible
   - Adds proper argparse, logging, random seeds
   - Makes results machine-readable (JSON/CSV output)
3. **Code review** by GPT-5.4 xhigh (`CODE_REVIEW=true` by default)
   - Catches logic bugs before wasting GPU hours
   - CRITICAL/MAJOR issues must be fixed before deployment
4. **Sanity check** — runs the smallest experiment first to catch setup errors
5. **Deploy** full experiment suite via `/run-experiment`
6. **Collect** initial results and update the tracker

### Key Configuration

| Setting | Default | Effect |
|---------|---------|--------|
| `CODE_REVIEW` | true | GPT-5.4 reviews code before deployment |
| `AUTO_DEPLOY` | true | Auto-deploy after code review passes |
| `SANITY_FIRST` | true | Run 1 small experiment before full suite |
| `MAX_PARALLEL_RUNS` | 4 | Max experiments per deployment wave |
| `BASE_REPO` | false | Clone a GitHub repo as the base codebase |

### The Deployment Mechanism

For remote GPU servers:
```bash
# Creates a dedicated screen session per experiment
ssh <server> "screen -dmS exp_name bash -c '
  eval "$(/opt/conda/bin/conda shell.bash hook)" &&
  conda activate <env> &&
  CUDA_VISIBLE_DEVICES=<gpu_id> python train.py <args> 2>&1 | tee <log_file>
'"
```

Code sync options:
- **rsync** (default): `rsync -avz --include='*.py' --exclude='*' src/ server:dst/`
- **git** (optional): `git push && ssh server "git pull"` — preferred when multiple servers need the same code

---

## Workflow 2: Auto Research Loop

> **Command:** `/auto-review-loop "your paper topic"`
> **Duration:** 2-12 hours (depends on experiments needed)
> **Can run overnight:** Yes ✅ (the main "sleep" workflow)

### What It Does

Repeatedly reviews your work, implements fixes, and re-reviews until it reaches submission quality (score ≥ 6/10) or 4 rounds are completed.

### The Loop Structure

```
┌─────────────────────────────────────────────────────────┐
│               Auto Review Loop                           │
│                                                         │
│  ┌─────────────────────────────────────────────────┐   │
│  │ Phase A: Review                                 │   │
│  │ → Send full context to GPT-5.4 xhigh            │   │
│  │ → Get score (1-10), verdict, action items       │   │
│  └────────────────────┬────────────────────────────┘   │
│                       │                                 │
│  ┌────────────────────▼────────────────────────────┐   │
│  │ Phase B: Parse + Check Stop Condition           │   │
│  │ → score ≥ 6 AND "ready/almost"? → STOP          │   │
│  │ → Otherwise continue                            │   │
│  └────────────────────┬────────────────────────────┘   │
│                       │                                 │
│  ┌────────────────────▼────────────────────────────┐   │
│  │ Human Checkpoint (if HUMAN_CHECKPOINT=true)     │   │
│  │ → Show score + fixes → Wait for user input      │   │
│  └────────────────────┬────────────────────────────┘   │
│                       │                                 │
│  ┌────────────────────▼────────────────────────────┐   │
│  │ Phase C: Implement Fixes                        │   │
│  │ → Code changes, new experiments, reframing      │   │
│  │ → Deploy to GPU, monitor results                │   │
│  └────────────────────┬────────────────────────────┘   │
│                       │                                 │
│  ┌────────────────────▼────────────────────────────┐   │
│  │ Phase D: Wait for Experiments                   │   │
│  │ → Poll screen sessions / log files              │   │
│  └────────────────────┬────────────────────────────┘   │
│                       │                                 │
│  ┌────────────────────▼────────────────────────────┐   │
│  │ Phase E: Document Round                         │   │
│  │ → Append to AUTO_REVIEW.md (full verbatim resp) │   │
│  │ → Update REVIEW_STATE.json                      │   │
│  └────────────────────┬────────────────────────────┘   │
│                       │                                 │
│         (loop back to Phase A, up to 4 rounds)         │
└─────────────────────────────────────────────────────────┘
```

### State Persistence (Critical Feature)

Long runs hit context window limits and trigger "compaction" (the agent loses memory). ARIS survives this by writing machine-readable state before it can happen:

```json
{
  "round": 2,
  "threadId": "019cd392-...",
  "status": "in_progress",
  "last_score": 5.0,
  "last_verdict": "not ready",
  "pending_experiments": ["screen_name_1"],
  "timestamp": "2026-03-13T21:00:00"
}
```

On restart:
- If `status=in_progress` and `timestamp < 24h ago` → resume from round+1
- If `status=completed` or `timestamp > 24h ago` → fresh start

### Fix Prioritization

When implementing fixes from the reviewer:
1. Skip fixes requiring excessive compute (flag for manual follow-up)
2. Skip fixes requiring unavailable data/models
3. Prefer reframing/analysis over new experiments when both work
4. Always implement metric additions (cheap, high reviewer impact)

### Real Score Progression (From Production Run)

| Round | Score | What Happened |
|-------|-------|---------------|
| Initial | 5.0/10 | Borderline reject |
| Round 1 | 6.5/10 | Added standard metrics, discovered metric decoupling |
| Round 2 | 6.8/10 | Key claim failed to reproduce, pivoted narrative |
| Round 3 | 7.0/10 | Large seed study killed main improvement claim |
| Round 4 | **7.5/10** ✅ | Diagnostic evidence solidified, submission ready |

20+ GPU experiments, full narrative rewrite, invalid claims removed — all autonomous.

---

## Workflow 3: Paper Writing

> **Command:** `/paper-writing "NARRATIVE_REPORT.md"`
> **Duration:** 2-4 hours  
> **Can run overnight:** Yes

### What It Does

Takes a human-written narrative report and produces a submission-ready LaTeX paper.

### The 5+1 Steps

#### Step 1: Paper Plan (`/paper-plan`)
Builds a "Claims-Evidence Matrix" and section structure from your narrative:
- Every claim maps to specific evidence
- Every experiment supports a stated claim
- No orphaned evidence, no unsupported claims
- Figure plan: what to generate for each section
- Citation scaffolding

Output: `PAPER_PLAN.md`

#### Step 2: Figure Generation (`/paper-figure`)
Auto-generates data-driven plots from JSON/CSV:
- Training curves (matplotlib/seaborn)
- Bar charts, heatmaps, comparison tables
- LaTeX-ready files in `figures/`
- `figures/latex_includes.tex` with ready-to-paste snippets

**Note:** For architecture diagrams, use `illustration: gemini` (requires `GEMINI_API_KEY`) or `illustration: mermaid` (free).

#### Step 3: Paper Write (`/paper-write`)
Section-by-section LaTeX generation. The skill includes:
- Venue-specific templates (ICLR, NeurIPS, ICML, CVPR, ACL, AAAI, ACM)
- Anti-hallucination BibTeX: DBLP → CrossRef → `[VERIFY]` chain
- 8-step writing workflow with quality checks
- Cross-review by GPT-5.4 after drafting
- "Reverse outline test" (topic sentences should form coherent narrative)
- "De-AI polish" (removes: delve, pivotal, groundbreaking, etc.)

**Anti-hallucination citation chain:**
```bash
# Step A: DBLP (best quality)
curl "https://dblp.org/search/publ/api?q=TITLE+AUTHOR&format=json&h=3"
curl "https://dblp.org/rec/{key}.bib"

# Step B: CrossRef (fallback)
curl -H "Accept: application/x-bibtex" "https://doi.org/{doi}"

# Step C: Mark [VERIFY] — never fabricate
```

Output: `paper/` directory with `main.tex`, section files, `references.bib`, style files.

#### Step 4: Compile (`/paper-compile`)
Runs `latexmk` to build the PDF:
- Auto-fixes common LaTeX errors
- Checks page count with `pdftotext` (precise, not approximate)
- Verifies no undefined references or missing citations
- Submission readiness checklist

#### Step 5: Auto Paper Improvement Loop (`/auto-paper-improvement-loop`)
2 rounds of GPT-5.4 xhigh content review + 1 round of format compliance:

- **Round 1**: Content review → CRITICAL/MAJOR fixes
- **Round 2**: Re-review → remaining fixes
- **Round 3**: Format compliance (page limit, overfull boxes, required sections)

**Real result:** 4/10 → 8.5/10 across 3 rounds on a 9-page ICLR theory paper.

### Venue Support

| Venue | Style File | Page Limit | Citation Style |
|-------|------------|------------|---------------|
| ICLR | `iclr2026_conference.sty` | 9 pages | natbib |
| NeurIPS | `neurips_2025.sty` | 9 pages | natbib |
| ICML | `icml2025.sty` | 9 pages | natbib |
| CVPR | `cvpr2024.sty` | 8 pages | natbib |
| ACL | `acl_natbib.sty` | 8 pages | natbib |
| AAAI | `aaai24.sty` | 7 pages | natbib |
| ACM | `acmart.cls` | varies | natbib |

---

## Full Pipeline: `/research-pipeline`

Chains all workflows end-to-end:

```
Stage 1: /idea-discovery          (Workflow 1)
    ↓ Gate 1 (AUTO_PROCEED or user confirmation)
Stage 2: Implement experiments    (manual code within pipeline)
    ↓
Stage 3: /run-experiment          (Workflow 1.5)
    ↓
Stage 4: /auto-review-loop        (Workflow 2)
    ↓
Final: Report + ready for /paper-writing
```

**Timeline (typical overnight run):**

| Stage | Duration | Sleep mode? |
|-------|----------|-------------|
| Idea Discovery | 30-60 min | Yes (with AUTO_PROCEED=true) |
| Implementation | 15-60 min | Yes (autonomous after gate) |
| Deployment | 5 min + experiment time | Yes ✅ |
| Auto Review | 1-4 hours | Yes ✅ |

**The sweet spot:** Run Stage 1-2 in the evening → launch Stages 3-4 before bed → wake up to a reviewed paper.
