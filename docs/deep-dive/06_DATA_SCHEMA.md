# ARIS Data Schema

## Configuration Files

### `CLAUDE.md` (Project-level configuration)
Written by the user in their research project directory. Tells Claude Code about the environment.

```markdown
## Remote Server

- SSH: `ssh my-gpu-server`         # SSH alias (key-based auth required)
- GPU: 4x A100 80GB
- Conda env: `research`
- Activate: `eval "$(/opt/conda/bin/conda shell.bash hook)" && conda activate research`
- Code directory: `/home/user/experiments/`
- code_sync: rsync                 # or: git
- wandb: false                     # or: true to auto-add W&B logging
- wandb_project: my-project        # W&B project name
- wandb_entity: my-team            # W&B team (optional)

## Paper Library

- papers/                          # local PDF collection path
```

**Fields:**
| Field | Type | Default | Description |
|-------|------|---------|-------------|
| SSH | string | required | SSH alias for remote server |
| GPU | string | info only | GPU description |
| Conda env | string | required for remote | Conda environment name |
| Activate | string | required for remote | Full activation command |
| Code directory | string | required for remote | Path to code on server |
| code_sync | `rsync` \| `git` | `rsync` | Code sync method |
| wandb | boolean | `false` | Enable W&B auto-logging |
| wandb_project | string | required if wandb=true | W&B project name |
| wandb_entity | string | optional | W&B team/user |

---

### `~/.claude/feishu.json` (Feishu notification config)
Optional. Controls mobile notifications. Absence = silent mode.

**Push only mode:**
```json
{
  "mode": "push",
  "webhook_url": "https://open.feishu.cn/open-apis/bot/v2/hook/YOUR_WEBHOOK_ID"
}
```

**Interactive mode:**
```json
{
  "mode": "interactive",
  "webhook_url": "https://open.feishu.cn/open-apis/bot/v2/hook/YOUR_WEBHOOK_ID",
  "interactive": {
    "bridge_url": "http://localhost:5000",
    "timeout_seconds": 300
  }
}
```

**Off mode (explicit):**
```json
{
  "mode": "off"
}
```

**Fields:**
| Field | Type | Values | Description |
|-------|------|--------|-------------|
| mode | string | `off`, `push`, `interactive` | Notification mode |
| webhook_url | string | Feishu webhook URL | Required for push/interactive |
| interactive.bridge_url | string | HTTP URL | feishu-bridge server URL |
| interactive.timeout_seconds | integer | default 300 | How long to wait for user reply |

---

### `~/.codex/config.toml` (Codex CLI configuration)
Controls which OpenAI model GPT-5.4 the reviewer uses.

```toml
model = "gpt-5.4"          # Recommended for best review quality
# model = "gpt-5.3-codex"  # Alternative
# model = "gpt-5.2-codex"  # Alternative
# model = "o3"             # Alternative
```

---

### `~/.claude/settings.json` (Claude Code global settings)
Used for alternative model configurations (e.g., GLM as executor).

```json
{
  "env": {
    "ANTHROPIC_AUTH_TOKEN": "your_api_key",
    "ANTHROPIC_BASE_URL": "https://api.z.ai/api/anthropic",
    "ANTHROPIC_DEFAULT_OPUS_MODEL": "glm-5",
    "ANTHROPIC_DEFAULT_SONNET_MODEL": "glm-4.7",
    "ANTHROPIC_DEFAULT_HAIKU_MODEL": "glm-4.5-air",
    "API_TIMEOUT_MS": "3000000"
  },
  "mcpServers": {
    "codex": {
      "command": "/opt/homebrew/bin/codex",
      "args": ["mcp-server"]
    }
  }
}
```

**For overnight auto-permission (optional):**
```json
{
  "permissions": {
    "allow": [
      "mcp__codex__codex",
      "mcp__codex__codex-reply",
      "Write",
      "Edit",
      "Skill(auto-review-loop)"
    ]
  }
}
```

---

## State Files (Written by ARIS)

### `REVIEW_STATE.json`
Machine-readable state for crash recovery in `auto-review-loop`.

```json
{
  "round": 2,
  "threadId": "019cd392-7a8b-4c2d-9e1f-abc123def456",
  "status": "in_progress",
  "last_score": 5.0,
  "last_verdict": "not ready",
  "pending_experiments": ["exp_ablation_1", "exp_baseline_2"],
  "timestamp": "2026-03-13T21:00:00"
}
```

**Fields:**
| Field | Type | Description |
|-------|------|-------------|
| round | integer | Last completed round number |
| threadId | string | Codex conversation thread ID for `codex-reply` |
| status | `in_progress` \| `completed` | Whether the loop is running |
| last_score | float | Most recent reviewer score (1.0-10.0) |
| last_verdict | string | `"ready"`, `"almost"`, or `"not ready"` |
| pending_experiments | array[string] | Screen session names still running |
| timestamp | ISO 8601 string | When state was last written |

**Recovery logic:**
- `status=in_progress` AND `timestamp < 24h` → resume
- `status=completed` OR `timestamp > 24h` → fresh start

---

### `AUTO_REVIEW.md`
Cumulative review log. Appended every round. Never overwritten (only appended).

```markdown
# Auto Review Log

**Direction**: [research topic]
**Started**: 2026-03-13T19:00:00

---

## Round 1 (2026-03-13T19:30:00)

### Assessment (Summary)
- Score: 5.0/10
- Verdict: not ready
- Key criticisms:
  - Missing standard baselines (CRITICAL)
  - No ablation study (MAJOR)
  - Claims not supported by evidence (MAJOR)

### Reviewer Raw Response

<details>
<summary>Click to expand full reviewer response</summary>

[Full verbatim GPT-5.4 response — never truncated]

</details>

### Actions Taken
- Added 3 baseline comparisons
- Ran ablation: seed study (screen session: exp_seed_study)

### Results
- Baseline comparisons: +2.3% over best baseline
- Seed study: variance within acceptable bounds

### Status
- Continuing to Round 2

---

## Round 2 (2026-03-13T23:15:00)

...

---

## Final Summary (2026-03-14T08:00:00)
- Rounds completed: 4/4
- Score progression: 5.0 → 6.5 → 6.8 → 7.5
- Final verdict: READY for submission

## Method Description

[1-2 paragraph concise description of the final method, its architecture, and data flow.
Used as input to /paper-illustration for diagram generation.]
```

---

### `IDEA_REPORT.md`
Output of Workflow 1. Comprehensive ranked idea list.

```markdown
# Idea Discovery Report

**Direction**: [research direction]
**Date**: 2026-03-12
**Pipeline**: research-lit → idea-creator → novelty-check → research-review → research-refine-pipeline

## Executive Summary
[2-3 sentences: best idea, key evidence, recommended next step]

## Literature Landscape
[landscape summary from /research-lit]

## Ranked Ideas

### 🏆 Idea 1: [Title] — RECOMMENDED
- Pilot: POSITIVE (+8.3% over baseline)
- Novelty: CONFIRMED (closest: [Smith 2024], differentiation: [key difference])
- Reviewer score: 7.5/10
- Reviewer feedback: [summary]
- Next step: implement full experiment → /auto-review-loop

### Idea 2: [Title] — BACKUP
...

## Eliminated Ideas

| Idea | Killed at Phase | Reason |
|------|----------------|--------|
| [idea] | Phase 3 (Novelty) | Already published as [paper] |
| [idea] | Phase 2 (Pilot) | Negative pilot signal (-3%) |

## Refined Proposal
- Proposal: `refine-logs/FINAL_PROPOSAL.md`
- Experiment plan: `refine-logs/EXPERIMENT_PLAN.md`
- Tracker: `refine-logs/EXPERIMENT_TRACKER.md`

## Next Steps
- [ ] /run-experiment to deploy experiments from the plan
- [ ] /auto-review-loop to iterate until submission-ready
```

---

### `refine-logs/FINAL_PROPOSAL.md`
Output of `/research-refine`. The refined method proposal.

```markdown
# Final Method Proposal

**Problem Anchor**: [one sentence — the specific problem this addresses]
**Method Thesis**: [one sentence — what the method does]
**Dominant Contribution**: [what's novel]
**Reviewer Score**: 9.2/10 (5 refinement rounds)

## Problem Statement
[anchored problem definition]

## Method
[detailed method description]

## Key Claims
1. [falsifiable claim 1]
2. [falsifiable claim 2]

## Differentiation from Prior Work
[what's different from closest existing work]
```

---

### `refine-logs/EXPERIMENT_PLAN.md`
Output of `/experiment-plan`. Ordered list of experiments to run.

```markdown
# Experiment Plan

## Must-Run Experiments (Ordered by Priority)

### Block 1: Main Result
**Claim**: [claim this supports]
**Experiments**:
1. `python train.py --model ours --dataset X` — 2 GPU-hours
2. `python train.py --model baseline1 --dataset X` — 2 GPU-hours
**Budget**: 4 GPU-hours total

### Block 2: Ablation Study
...

## Optional Experiments (If Budget Allows)
...

## Total Budget Estimate
- Must-run: 12 GPU-hours
- Optional: 8 GPU-hours
```

---

### `refine-logs/EXPERIMENT_TRACKER.md`
Status tracking for each experiment.

```markdown
# Experiment Tracker

| Experiment | Status | GPU | Screen Session | Start Time | Result |
|------------|--------|-----|---------------|------------|--------|
| main_result_ours | DONE | GPU 0 | exp_main | 2026-03-13T20:00 | 84.3% |
| main_result_baseline | RUNNING | GPU 1 | exp_base | 2026-03-13T20:05 | - |
| ablation_no_aug | TODO | - | - | - | - |
```

---

## Output Files

### `PAPER_PLAN.md`
Output of `/paper-plan`. Claims-Evidence Matrix and paper structure.

```markdown
# Paper Plan

**One-Sentence Contribution**: [the key takeaway]

## Claims-Evidence Matrix

| Claim | Evidence | Section | Figure/Table |
|-------|----------|---------|-------------|
| [Claim 1] | [Experiment 1 result] | §4.2 | Table 1 |
| [Claim 2] | [Ablation] | §4.3 | Figure 2 |

## Section Structure

### §0 Abstract (150-250 words)
- What: [problem]
- Why hard: [challenge]
- How: [approach]
- Evidence: [key result]
- Strongest: [biggest number]

### §1 Introduction (1-1.5 pages)
...

## Figure Plan
- Figure 1: [description, data source]
- Table 1: [description, columns, data source]
```

---

### `paper/` directory
Output of `/paper-write` and `/paper-compile`.

```
paper/
├── main.tex                    # Master file (inputs sections)
├── iclr2026_conference.sty     # Venue style file
├── math_commands.tex           # Shared math macros
├── references.bib              # Bibliography (filtered — only cited entries)
├── sections/
│   ├── 0_abstract.tex
│   ├── 1_introduction.tex
│   ├── 2_related_work.tex
│   ├── 3_method.tex
│   ├── 4_experiments.tex
│   ├── 5_conclusion.tex
│   └── A_appendix.tex
├── figures/                    # Figures (symlink or copy)
└── main.pdf                    # Compiled output
```

---

## Input Templates

### `NARRATIVE_REPORT_TEMPLATE.md`
The main input for Workflow 3. User fills this in before running `/paper-writing`.

Key sections:
```markdown
# Research Narrative Report

## Problem and Motivation
[What problem you're solving and why it matters]

## Our Approach
[High-level description of the method]

## Experimental Setup
[Datasets, baselines, metrics, implementation details]

## Main Results
[Key numbers — e.g., "Our method achieves 84.3% on X, vs 81.2% for the best baseline"]

## Ablation Studies
[What each component contributes]

## Figures
[Description of each figure and what data it should show]
```

### `RESEARCH_BRIEF_TEMPLATE.md`
Input for Workflow 1 (more detailed than a single command).

### `EXPERIMENT_PLAN_TEMPLATE.md`
Input for Workflow 1.5 (when skipping Workflow 1).

### `PAPER_PLAN_TEMPLATE.md`
Alternative to auto-generated PAPER_PLAN.md (for manual outlines).

---

## Intermediate Data: Working Notes Convention

When Claude Code needs to save intermediate data (literature summaries, analysis notes), it creates files in the current directory following naming conventions:

| Convention | Example | Used by |
|------------|---------|---------|
| `TOPIC_notes.md` | `diffusion_lit_notes.md` | research-lit |
| `IDEA_REPORT.md` | fixed name | idea-creator |
| `AUTO_REVIEW.md` | fixed name | auto-review-loop |
| `REVIEW_STATE.json` | fixed name | auto-review-loop |
| `refine-logs/` | fixed directory | research-refine-pipeline |
| `paper/` | fixed directory | paper-write |
| `figures/` | fixed directory | paper-figure |
| `paper-backup-{timestamp}/` | timestamped | paper-write (safety backup) |
