# ARIS: Deep Insight Into Prompting, Context Architecture, and Agent Rationality

> This document is a technical deep-dive for contributors, power users, and AI researchers who want to understand *why* each agent in ARIS receives the context it does, *how* the prompt templates are engineered, and *what reasoning* underlies every design decision at the prompting layer. It is meant to be read alongside `02_AGENT_ARCHITECTURE.md` and `04_SKILLS_REFERENCE.md`.

---

## Part 1 — The Fundamental Prompting Paradigm

### 1.1 Skills as Prompt Programs

The single most important insight for understanding ARIS prompting is this:

**Every SKILL.md file is a prompt program.** It is not configuration, not Python code, not a workflow DAG — it is a natural-language procedure that an LLM reads and executes at runtime. The "agent" is not a Python process wrapping an LLM; the LLM *is* the agent, and the SKILL.md is its full system prompt for that task.

This has deep implications for how prompts are structured:

- Instructions must be unambiguous enough that any sufficiently capable LLM will interpret them the same way
- State must be externalized to files (the LLM has no persistent memory)
- Branching logic must be expressed as natural-language conditionals, not code
- Tool calls are described by name, and the executor framework (Claude Code) resolves them

The frontmatter of each skill is the most compact part of the prompt, but arguably the most important:

```yaml
---
name: auto-review-loop
description: Autonomous multi-round research review loop...
allowed-tools: Bash(*), Read, Grep, Glob, Write, Edit, Agent, Skill,
               mcp__codex__codex, mcp__codex__codex-reply
---
```

The `allowed-tools` line is a **security and capability contract**: it tells Claude Code exactly which tools this skill may invoke. This matters because Claude Code uses it to grant or deny tool permissions at execution time. Listing `mcp__codex__codex` here is what makes cross-model calls possible — without it, the skill cannot reach GPT-5.4 regardless of what the body says.

The `description` line doubles as a routing hint. When Claude Code is deciding which skill to invoke in response to natural-language user input, it uses this description for semantic matching. This is why descriptions are verbose and include multi-lingual phrasings ("找idea", "brainstorm ideas", "generate research ideas") — the routing layer benefits from diverse trigger phrases.

### 1.2 The `$ARGUMENTS` Injection Point

Every skill that takes user input references `$ARGUMENTS` — the string passed when invoking the skill. This appears at least twice in each skill:

1. In the header line: `Generate publishable research ideas for: $ARGUMENTS`
2. In downstream prompt templates that embed the user's direction into reviewer calls

This design enforces a clean contract: the skill's instructions are fixed, and user intent flows through a single, well-defined injection point. There is no prompt injection surface beyond `$ARGUMENTS`, and its placement is explicit.

A key subtlety: `$ARGUMENTS` is passed unprocessed to the reviewer LLM's prompt. This means the user's wording of their research direction directly shapes what GPT-5.4 brainstorms. The skills are tuned to encourage concise, specific direction strings ("factorized gap in discrete diffusion LMs") because vague directions produce vague brainstorming. The `idea-creator` skill even has a hard rule: *"If the user's direction is too broad, STOP and ask them to narrow it."*

### 1.3 Constants as Configurable Defaults

Every skill defines a `## Constants` section with settings like `MAX_ROUNDS = 4` or `PILOT_MAX_HOURS = 2`. These are not environment variables or config files — they are part of the prompt text itself.

The implication: changing a constant means editing the SKILL.md and the LLM will read the new value on its next invocation. There is no compilation, no restart, no deployment. This "soft configuration" pattern keeps the system truly zero-dependency.

The override syntax — `> 💡 Override: /auto-review-loop "topic" — human checkpoint: true` — works because Claude Code parses the `$ARGUMENTS` string and the skill instructions tell the executor to look for these override directives. This is natural-language configuration parsing, not traditional CLI flag parsing.

---

## Part 2 — Context Architecture per Workflow

### 2.1 Workflow 1: Idea Discovery — Context Accumulation Chain

Workflow 1 is architecturally a **context accumulation pipeline**: each phase adds information to a shared working context that flows into the next phase.

#### Phase 1: Literature Survey (`/research-lit`)

**Context at invocation time:**
- `$ARGUMENTS`: the user's research direction string
- File system: `papers/`, `literature/` local PDFs (if present)
- Optional: Zotero library (via MCP), Obsidian vault (via MCP)

**What is read:**
The skill reads sources in a strict priority order — Zotero first (most curated), Obsidian second (most processed), local PDFs third, web last. This priority is rational: Zotero contains papers the user has *already* found relevant and annotated, making them higher-signal than a fresh web search. Obsidian notes represent the user's *processed understanding*, not raw paper content. The web is only a fallback for gaps.

The `arxiv_fetch.py` tool is called directly for structured metadata (JSON with full author lists, categories, dates) rather than scraping web search snippets. This produces cleaner data for the landscape map.

**What is produced:**
A literature summary stored in working notes (e.g., `diffusion_lit_notes.md`). This is *not* yet passed to the reviewer LLM — it is read by the executor (Claude Code) and assembled into a structured landscape map.

**Rationale for executor-only phase:**
Literature survey is a data-gathering and synthesis task, not a judgment task. Claude Code is good at this: reading files, fetching URLs, de-duplicating results, and building structured summaries. Calling GPT-5.4 for literature search would be expensive and slower without improving quality, since the reviewer model has no file system access and cannot search arXiv directly.

#### Phase 2: Idea Generation (`/idea-creator`)

**Context passed to GPT-5.4:**

This is the first time the external reviewer receives a prompt. The exact template is:

```
You are a senior ML researcher brainstorming research ideas.

Research direction: [user's direction]

Here is the current landscape:
[paste landscape map from Phase 1]

Key gaps identified:
[paste gaps from Phase 1]

Generate 8-12 concrete research ideas. For each idea:
1. One-sentence summary
2. Core hypothesis (what you expect to find and why)
3. Minimum viable experiment (what's the cheapest way to test this?)
4. Expected contribution type: empirical finding / new method / theoretical result / diagnostic
5. Risk level: LOW / MEDIUM / HIGH
6. Estimated effort: days / weeks / months

Prioritize ideas that are:
- Testable with moderate compute (8x RTX 3090 or less)
- Likely to produce a clear positive OR negative result (both are publishable)
- Not "apply X to Y" unless the application reveals genuinely surprising insights
- Differentiated from the 10-15 papers above

Be creative but grounded. A great idea is one where the answer matters regardless of which way it goes.
```

**Why this prompt structure:**

- **"Senior ML researcher"** persona: establishes domain authority. GPT-5.4 with `xhigh` reasoning produces measurably better domain-specific output when given an expert role than when prompted neutrally.
- **Landscape map first, then gaps**: the ideas generated will be more targeted if the model first has a full picture of what exists, then explicitly has the gaps pointed out. Ordering matters for context-window attention.
- **Forced 6-field structure per idea**: unstructured brainstorming produces ideas that are hard to compare. The structured output (hypothesis, MVE, contribution type, risk, effort) allows the executor to filter by fields programmatically — specifically, to filter by `effort` (exclude "months") and `risk` before running pilots.
- **"Both directions publishable"** constraint: this is a subtle but important research heuristic. Many researchers only pursue ideas they think will succeed. By explicitly asking for ideas where the answer matters regardless of direction, the prompt pushes toward stronger experimental designs.
- **Explicit differentiation from the 10-15 papers**: prevents the reviewer from suggesting ideas that are variations of the papers in the landscape map.
- **`config: {"model_reasoning_effort": "xhigh"}`**: activates extended chain-of-thought reasoning in GPT-5.4. For brainstorming, this means the model can spend more inference time on exploring the idea space before committing to a list. The latency cost (30-120 seconds) is acceptable for an overnight run.

**The threadId continuity design:**

After Phase 2 generates ideas, the skill saves the `threadId` from the first `mcp__codex__codex` call. All subsequent calls in the same session use `mcp__codex__codex-reply` with this ID. This gives GPT-5.4 full memory of its own brainstormed ideas when later asked to do the devil's advocate critique:

```
Here are our top ideas after filtering:
[paste surviving ideas with novelty check results]

For each, play devil's advocate:
- What's the strongest objection a reviewer would raise?
- What's the most likely failure mode?
- How would you rank these for a top venue submission?
- Which 2-3 would you actually work on?
```

The `codex-reply` continuation means GPT-5.4 has its own brainstormed ideas in context when critiquing them. This is important: a model critiquing ideas it just generated will give more targeted and specific objections than a model critiquing ideas presented cold.

#### Phase 3: Novelty Check (`/novelty-check`)

**Context passed to GPT-5.4:**

```
[Proposed method description]
[Papers found via multi-source search]

Is this method novel? What is the closest prior work? What is the delta?
```

**Key design decision — why search before querying GPT-5.4:**

The executor runs a multi-source web search (arXiv, Scholar, Semantic Scholar) *before* calling the reviewer. This is essential: GPT-5.4's training data has a knowledge cutoff, and the most dangerous overlap is always in the last 6 months before submission. By searching first and including found papers in the prompt, the skill grounds the novelty check in current literature rather than relying on the model's possibly-stale knowledge.

The `config: {"model_reasoning_effort": "xhigh"}` is used here too. Novelty assessment requires careful reasoning about whether a proposed method's claims genuinely differentiate it from each cited paper — this benefits from extended reasoning.

#### Phase 4: Critical Review (`/research-review`)

**Context passed to GPT-5.4:**

The research-review skill assembles a comprehensive package:
- Top idea description
- Pilot experiment results (including **negative results and failed pilots**)
- Novelty check verdict and closest prior work

The instructions explicitly mandate including negative results: *"Be honest — include negative results and failed experiments."* This is rationalized as follows: if the reviewer only sees the positive results, it will give an inflated score. When the reviewer score is used to decide whether to commit GPU resources to full experiments, an inflated score leads to wasted compute. Honest input produces actionable output.

The review prompt follows a structured format:
```
Please act as a senior ML reviewer (NeurIPS/ICML level).

1. Score this work 1-10 for a top venue
2. List remaining critical weaknesses (ranked by severity)
3. For each weakness, specify the MINIMUM fix (experiment, analysis, or reframing)
4. State clearly: is this READY for submission? Yes/No/Almost
```

**Why "minimum fix" framing:**

Asking for "minimum viable fixes" instead of "how to make this perfect" is deliberate. A reviewer asked to make a paper perfect will suggest extensive experiments. A reviewer asked for the minimum to address each weakness will suggest the smallest intervention — which is what an autonomous executor can actually implement in a finite overnight run.

#### Phase 4.5: Method Refinement (`/research-refine`)

This is the most sophisticated prompting phase in Workflow 1. The executor-reviewer loop runs up to 5 rounds with a hard stop at score ≥ 9.

**The Problem Anchor mechanism:**

Before any prompting begins, the executor extracts an immutable "Problem Anchor" from the user's input:

```markdown
## Problem Anchor
- Bottom-line problem: [extracted]
- Must-solve bottleneck: [specific failure in current methods]
- Non-goals: [explicit exclusions]
- Constraints: [compute, data, time, venue]
- Success condition: [what would confirm the method works]
```

This anchor is then **copied verbatim** into every subsequent proposal and every reviewer prompt. The rationale: without an anchor, iterative refinement through a reviewer loop consistently drifts — each round's suggested improvements subtly shift the problem being solved. After 5 rounds, the proposal may be optimized for a different problem than the original. The anchor prevents this by making drift explicit and detectable.

**The reviewer prompt for refinement rounds:**

```
Review this research proposal for fidelity, specificity, contribution quality, and frontier leverage.

## Problem Anchor (fixed — do not suggest changing this):
[verbatim Problem Anchor]

## Current Proposal:
[full proposal text]

Score this proposal on:
1. Problem-anchor fidelity (1-10): Does it solve the stated problem?
2. Method specificity (1-10): Is it concrete enough to implement?
3. Contribution quality (1-10): Is the novelty claim defensible?
4. Frontier leverage (1-10): Does it use modern techniques appropriately?
5. OVERALL SCORE (1-10): would this pass top-venue review?

For each dimension, specify:
- What is missing or weak?
- The minimum fix that would address it?
```

The 4-dimensional scoring is a prompting technique to force the reviewer to reason about specific aspects before producing an overall score. Without this decomposition, reviewers tend to produce holistic impressions that are less actionable.

The threshold `SCORE_THRESHOLD = 9` (vs the `6` used in the auto-review-loop) reflects a deliberate difference in goals: method refinement should aim for a near-perfect proposal *before* experiments are run (when changes are cheap), while the review loop accepts "good enough for submission" (≥6) because experiments are already run and expensive to repeat.

---

### 2.2 Workflow 1.5: Experiment Bridge — Cross-Model Code Review

The experiment bridge introduces a specialized prompting pattern: **code review by the reviewer model**.

**Context passed to GPT-5.4 for code review:**

```
Review the following experiment implementation for correctness.

## Experiment Plan:
[key sections from EXPERIMENT_PLAN.md]

## Method Description:
[from FINAL_PROPOSAL.md]

## Implementation:
[experiment scripts]

Check for:
1. Does the code correctly implement the method in the proposal?
2. Are hyperparameters from the plan reflected in the code?
3. Are there logic bugs (wrong loss function, incorrect split, missing eval)?
4. Is the evaluation metric computed correctly?
5. **CRITICAL: Does evaluation use the dataset's actual ground truth labels —
   NOT another model's output as ground truth?**
6. OOM risk, numerical instability, missing seeds?

For each issue: CRITICAL / MAJOR / MINOR and the exact fix.
```

**Why item 5 is in bold as CRITICAL:**

This addresses a specific and common failure mode in ML research: using a base model's outputs as "ground truth" for an evaluation metric, when the ground truth should come from the dataset's actual labels. This produces circular evaluation (the new model is measured relative to the old model, not against the truth), which inflates results and would be caught by a reviewer. The explicit call-out in the prompt reflects experience from real research automation where this bug appeared and was caught by the code review step — catching it before GPU deployment saved significant compute.

**The rationale for cross-model code review:**

Claude Code (the executor) wrote the code. If Claude Code also reviewed it, it would make the same mistakes it made when writing. The cross-model review catches systematic issues in how Claude Code tends to implement evaluation — different models have different blind spots, so GPT-5.4 can see what Claude Code cannot.

The dual "max 2 rounds" rule on code review (fix CRITICAL issues, re-review once more) avoids infinite loops while still guaranteeing that critical issues are fixed before any GPU time is spent.

---

### 2.3 Workflow 2: Auto Review Loop — Multi-Round Context Threading

The auto-review-loop is the most technically interesting prompting pattern in ARIS, primarily because it manages multi-round state across a conversation thread with GPT-5.4.

#### Round 1 Prompt Structure

The full context assembled for the first review round includes:

1. **Round indicator**: `[Round 1/4 of autonomous review loop]` — gives the reviewer anchor for how many rounds remain
2. **Research context**: claims, methods, experimental results (read from project files at runtime)
3. **Known weaknesses**: extracted from prior reviews if resuming, or identified by the executor if fresh
4. **Changes since last round** (rounds 2+): explicit list of what was implemented
5. **Reviewer instruction block**: the structured 4-question format

The reviewer instruction block is worth examining in full:

```
Please act as a senior ML reviewer (NeurIPS/ICML level).

1. Score this work 1-10 for a top venue
2. List remaining critical weaknesses (ranked by severity)
3. For each weakness, specify the MINIMUM fix (experiment, analysis, or reframing)
4. State clearly: is this READY for submission? Yes/No/Almost

Be brutally honest. If the work is ready, say so clearly.
```

Each line is deliberate:

- **"Senior ML reviewer (NeurIPS/ICML level)"**: higher-specificity persona produces higher-quality critique than "be a reviewer." Naming specific venues tells the model what standard to apply.
- **"Score this work 1-10 for a top venue"**: the numeric score enables the stop condition to be evaluated programmatically. Asking for a letter grade or qualitative assessment would require parsing; asking for 1-10 produces a directly comparable number.
- **"Ranked by severity"**: ensures that the executor works on the most important fixes first. Without ranking, the executor might implement minor stylistic fixes before addressing fundamental methodological weaknesses.
- **"Minimum fix"**: as discussed above, constraints the scope of suggested experiments to what's achievable in one iteration.
- **"Yes/No/Almost"**: the three-valued verdict maps onto the stop condition: `score >= 6 AND verdict contains "ready" or "almost"`. A score of 6 with "not ready" verdict means the reviewer has concerns that the number alone doesn't capture.
- **"Be brutally honest"**: explicit instruction to override any tendency toward politeness. LLMs can be sycophantic; this instruction counteracts it. The follow-up "If the work is ready, say so clearly" is equally important — it prevents the model from giving unnecessarily negative feedback on genuinely good work.

#### Thread Continuity Design

Round 2+ uses `mcp__codex__codex-reply` with the saved `threadId`:

```
mcp__codex__codex-reply:
  threadId: [saved from round 1]
  config: {"model_reasoning_effort": "xhigh"}
  prompt: |
    [Round N update]

    Since your last review, we have:
    1. [Action 1]: [result]
    2. [Action 2]: [result]
    3. [Action 3]: [result]

    Updated results table:
    [paste metrics]

    Please re-score and re-assess. Are the remaining concerns addressed?
    Same format: Score, Verdict, Remaining Weaknesses, Minimum Fixes.
```

**Why thread continuity matters:**

Without thread continuity, each review round is a new conversation. The reviewer cannot tell the difference between:
- "We addressed weakness X by running experiment Y (result: Z)"
- "We claim to have addressed weakness X but haven't shown the result"

With thread continuity, GPT-5.4 has the full history of its own previous critiques in context. When it gave CRITICAL weakness X in round 1 and the round 2 prompt shows the experiment that addresses X with specific results, the reviewer can verify that the fix actually addresses the stated concern — not just that an experiment was run. This produces more targeted re-assessments than cold-start reviews.

The "same format" instruction at the end of the round 2+ prompt is important: it ensures the response remains parseable by the executor even after several conversational exchanges. Without it, the model might naturally produce a more conversational response that's harder to parse for score and verdict.

#### The Stop Condition Rationale

```
STOP CONDITION: If score >= 6 AND verdict contains "ready" or "almost"
```

The 6/10 threshold corresponds to "weak accept" at NeurIPS/ICML. Papers above this line are typically publishable; papers below are typically rejected. This is a calibrated threshold, not an arbitrary one.

The dual condition (score AND verdict) addresses a specific failure mode: a reviewer that gives 6/10 but says "not ready — needs one more ablation study" should not trigger early termination. The conjunction prevents the stop condition from triggering on a score that happens to be ≥6 but doesn't reflect genuine submission readiness.

The four-round maximum reflects a cost-benefit calculation: most of the improvement in research quality comes from the first 2-3 rounds of review. After 4 rounds, a paper that hasn't reached 6/10 typically has fundamental issues (invalid core claim, insufficient differentiation from prior work) that cannot be fixed by additional experiments — they require the researcher's judgment on whether to pivot or publish at a lower-tier venue.

#### State Persistence as Fault Tolerance

The `REVIEW_STATE.json` file is a prompting-level design decision:

```json
{
  "round": 2,
  "threadId": "019cd392-...",
  "status": "in_progress",
  "last_score": 5.0,
  "pending_experiments": ["exp_ablation_1"],
  "timestamp": "2026-03-13T21:00:00"
}
```

The file is written at the end of every Phase E. On restart, the skill reads this file and the executor can reconstruct exactly where it was without needing to remember anything.

The `threadId` in the state file is the key insight: even if Claude Code's session crashes and restarts, it can reconnect to the *same GPT-5.4 conversation thread* and resume the multi-round review dialogue. The reviewer retains full context of all prior rounds even though the executor has lost its local state.

The 24-hour staleness check (`timestamp < 24h ago`) is a pragmatic heuristic: if the state is more than 24 hours old, it's likely the user abandoned the run and started a new research session. Resuming a stale loop would be confusing.

---

### 2.4 Workflow 3: Paper Writing — Grounded Generation Anti-Hallucination

The paper writing workflow introduces the most concentrated prompting for quality control: section-by-section LaTeX generation with anti-hallucination constraints at multiple levels.

#### The Narrative Report as a Ground Truth Document

The most important prompting design decision in Workflow 3 is that paper writing starts from a *human-written* `NARRATIVE_REPORT.md`, not directly from experiment results.

This is an anti-hallucination measure. When an LLM is asked to write a paper from raw experimental numbers, it must invent the narrative (problem framing, motivation, related work context, conclusion). This narrative invention is where hallucinations occur — the model fills gaps with plausible-sounding but potentially false claims.

When the input is a human-written narrative that includes the actual story the researcher wants to tell, the LLM's task shifts from *invention* to *translation* (from narrative prose to academic LaTeX). This is much safer: the claims come from the human, the AI provides the prose structure and citation infrastructure.

#### The `paper-plan` Skill: Claims-Evidence Matrix

Before writing, the paper-plan skill creates a structured map:

```markdown
## Claims-Evidence Matrix
| Claim | Evidence | Section | Figure/Table |
|-------|----------|---------|-------------|
| Claim 1 | Experiment 1 result | §4.2 | Table 1 |
| Claim 2 | Ablation | §4.3 | Figure 2 |
```

The rationale: a paper that maps all claims to evidence and all evidence to claims is structurally coherent. The paper-plan review step passes this matrix to GPT-5.4 with the explicit instruction: *"Does every claim have supporting evidence? Does every piece of evidence support a stated claim? Are there orphaned claims or unused experiments?"*

This is a prompt engineering pattern for structural consistency checking — it is much harder to catch structural gaps in running prose than in a matrix. The matrix format forces the issue.

#### The 8-Step Paper-Write Workflow

The paper-write skill follows a precise 8-step sequence. Each step's prompting design is intentional:

**Step 3 (Write each section):** The executor writes section by section in a fixed order. This order matters: Abstract and Introduction are written *after* the outline but *before* the method and experiments sections. This is counter-intuitive for humans (we usually write intro last) but correct for LLM generation: the intro needs to be consistent with the paper plan, and writing it first while the plan is in context produces a cleaner first draft than retrofitting it to match a completed paper.

**Step 5 (Build bibliography via DBLP → CrossRef → `[VERIFY]`):**

The anti-hallucination citation chain is a three-fallback system:

```bash
# Step A: DBLP (best quality — publisher-verified metadata)
curl "https://dblp.org/search/publ/api?q=TITLE+AUTHOR&format=json"
→ get key → curl "https://dblp.org/rec/{key}.bib"

# Step B: CrossRef (covers journals, arXiv via DOI)
curl -H "Accept: application/x-bibtex" "https://doi.org/{doi}"

# Step C: [VERIFY] marker — never fabricate
```

The prompt to the LLM never says "generate a BibTeX entry." It says "search DBLP for this paper and fetch the BibTeX." The BibTeX is never generated by the model; it is fetched from canonical databases. The model's only role is to identify the correct search query and paper key. This eliminates the most common category of LLM error in academic writing: hallucinated citations with wrong venue names, wrong authors, or non-existent papers.

The `[VERIFY]` marker for cases where both APIs fail is better than hallucination: it flags the entry for human review rather than submitting a fabricated citation.

**Step 6 (De-AI polish pass):**

After drafting, a dedicated polish step removes LLM-characteristic language. The prompt lists prohibited words:

> Remove: *delve, pivotal, groundbreaking, revolutionize, paradigm shift, crucial, intricately, noteworthy, commendable, synergistic*

These words appear disproportionately in LLM-generated text and are immediately recognizable to experienced reviewers. Their removal is a prompting-layer quality control step — better to explicitly list and remove them than to hope the model doesn't use them.

**Step 7 (Cross-review by GPT-5.4):**

The paper draft is sent to the reviewer for content review. The prompt structure:

```
Review this [venue] paper as a senior area chair.

## Paper content:
[full LaTeX source]

1. Does the abstract match the actual contributions?
2. Are all claims in the introduction supported by experiments?
3. Is the related work section honest about the relation to prior work?
4. Is the method description clear enough to reproduce?
5. Are the experimental results presented fairly?

Score: 1-10. List CRITICAL/MAJOR/MINOR issues.
```

The "area chair" persona (vs. reviewer) prompts a higher-level assessment of paper coherence, not just technical correctness. An area chair evaluates whether the paper's story is coherent and its claims are consistent — exactly the level of review needed at this stage.

**Step 8 (Reverse outline test):**

The final step is a structural consistency check: extract the topic sentence from each paragraph and read them in sequence. If the topic sentences form a coherent narrative, the paper has good paragraph structure. If they don't connect, sections need reorganization.

This is a prompting trick borrowed from academic writing pedagogy: the reverse outline surfaces logical flow issues that are invisible when reading full paragraphs.

#### The `auto-paper-improvement-loop` Round Structure

The paper improvement loop runs 3 rounds with different objectives:

- **Round 1**: Content review — CRITICAL/MAJOR issues (substance)
- **Round 2**: Re-review after fixes — remaining substantive issues
  
- **Round 3**: Format compliance — page count, overfull boxes, required sections

The separation of content and format review is deliberate. Mixing them in a single pass produces confusion: the model may fix formatting issues while missing substantive problems, or may suggest moving sections to save space when the content itself is the problem. Separating rounds ensures each type of issue gets full attention.

The reported score progression (4/10 → 8.5/10 across 3 rounds on a real paper) validates the design: a 4.5-point improvement in 3 rounds of automated review demonstrates that the structured prompting is producing actionable critique.

---

## Part 3 — Cross-Cutting Prompting Patterns

### 3.1 The "Minimum Fix" Constraint

Across every review context — idea generation, method refinement, code review, paper review — the prompts request the *minimum* fix for each weakness, never the *ideal* fix.

This is the most important single prompting pattern in ARIS.

**Why it works:** When you ask an LLM reviewer for the ideal improvement, you get suggestions calibrated to perfection ("run 50 seeds, test on 5 datasets, derive theoretical convergence guarantees"). When you ask for the minimum fix, you get suggestions calibrated to *sufficiency* ("run 3 seeds on the primary dataset to verify the effect is not noise-driven"). The minimum fix is implementable overnight; the ideal fix is not.

An autonomous executor that must implement reviewer suggestions needs *actionable* suggestions. The "minimum fix" framing systematically produces actionable output by constraining the response space.

### 3.2 Severity Tiers (CRITICAL / MAJOR / MINOR)

Reviews consistently request severity-tagged action items. The executor's fix prioritization logic uses these tags directly:

```
For each action item (highest priority first):
1. CRITICAL → must fix before continuing
2. MAJOR → fix in this round
3. MINOR → fix if time permits
```

This is prompt-level priority encoding: by asking the reviewer to tag severity, the executor can implement a simple priority queue without additional reasoning. The alternative — asking the executor to judge which issues are most important — introduces another reasoning step that could go wrong.

### 3.3 Verbatim Response Preservation

Every skill that calls the reviewer includes this instruction:

> **CRITICAL: Save the FULL raw response from the external reviewer verbatim.** Do NOT discard or summarize — the raw text is the primary record.

The AUTO_REVIEW.md format wraps this in a `<details>` block:

```markdown
### Reviewer Raw Response
<details>
<summary>Click to expand full reviewer response</summary>
[Complete verbatim GPT-5.4 response]
</details>
```

**The rationale:** When reviewing research outcomes months later, summaries are lossy. The reviewer might have said something precise about a specific equation or experimental setup that the executor's summary missed. The verbatim record is the authoritative source of what the reviewer actually said. This is also an accountability mechanism: the full review is preserved even if some critiques were ultimately ignored.

### 3.4 Graceful Degradation as a Prompting Principle

Every optional integration in ARIS is written into the skill instructions as a conditional that silently degrades:

```
Check if ~/.claude/feishu.json exists and mode is not "off":
- Send notification: ...
- If config absent or mode off: skip entirely (no-op)
```

```
Try calling a Zotero MCP tool. If it succeeds: [use Zotero]
If unavailable: skip silently
```

This is a prompting design pattern for robustness. An executor that fails on a missing optional dependency wastes time and may require human intervention at 3am. An executor that detects absence and silently continues produces the same core output regardless of environment.

The pattern works because the LLM executor is capable of conditional logic when the conditions are clearly stated in natural language. The instructions read like pseudocode: "if X then Y, else Z". The executor faithfully implements this conditional.

### 3.5 The `allowed-tools` Security Model

The frontmatter `allowed-tools` field creates a capability contract that Claude Code enforces. Consider what happens when it's absent or minimal:

- `research-lit` has no `mcp__codex__codex` in its allowed-tools — it cannot make cross-model calls even if the body instructions accidentally requested it
- `paper-compile` has only `Bash(*), Read, Write, Edit` — it cannot spawn sub-agents or call external models
- `auto-review-loop` explicitly lists both `mcp__codex__codex` and `mcp__codex__codex-reply` — both initial and continuation calls are authorized

This is least-privilege security applied to AI prompting: each skill gets exactly the tools it needs, and no more. The skill that does literature search does not need GPT-5.4 access. The skill that does multi-round review needs both initial and reply call tools.

The practical benefit: if a skill's body instructions contain a prompt injection (either from a malicious paper title in the literature or a corrupted file), the attacker can only invoke tools that are already allowed. A literature review skill cannot accidentally trigger a cross-model call that spends money or sends data externally, because `mcp__codex__codex` is not in its allowed-tools.

---

## Part 4 — Prompt Engineering Anti-Patterns That ARIS Avoids

Understanding the design rationale is clarified by examining what ARIS deliberately does not do.

### 4.1 Does NOT Use "Generate BibTeX from Memory"

The single most dangerous prompt in academic writing automation is: *"Generate a BibTeX entry for [paper title]."* LLMs hallucinate BibTeX with wrong venues, wrong years, wrong authors, and sometimes entirely non-existent papers. ARIS never makes this call — BibTeX is always fetched from DBLP or CrossRef, with the model only determining the search query.

### 4.2 Does NOT Ask for Self-Review

Nowhere in ARIS does Claude Code review its own writing, code, or analysis. The model that produced the content is never the model that reviews it. This isn't just a quality concern — it's a structural one: self-review systematically fails to catch errors because the same cognitive patterns that produced the error also fail to detect it.

### 4.3 Does NOT Use Open-Ended "Improve This" Prompts

Every review prompt asks for *specific, structured, minimum* improvements. "Improve this paper" produces vague feedback. "List remaining critical weaknesses ranked by severity, specify the minimum fix for each, score 1-10" produces actionable items. The structured output format is enforced in the prompt, not hoped for in the response.

### 4.4 Does NOT Include Positive Reinforcement

The review prompts explicitly say "Be brutally honest." There is no "first acknowledge what's good" instruction. Research review benefits from adversarial critique. Asking the reviewer to acknowledge strengths first activates politeness patterns that soften critique. The design prioritizes honest negative feedback over balanced feedback.

### 4.5 Does NOT Trust Implicit State

Every skill that depends on state from a previous skill reads the state *from files*, never assumes it's in the LLM's context. `auto-review-loop` reads `AUTO_REVIEW.md` on startup. `paper-write` reads `PAPER_PLAN.md`. `experiment-bridge` reads `EXPERIMENT_PLAN.md`. The prompt instructions explicitly say "Read X before proceeding" — they do not assume the prior skill's output is in context.

This is critical for overnight runs: context windows overflow, compaction happens, sessions restart. Files persist; LLM context does not.

---

## Part 5 — Prompt Quality Control Mechanisms

### 5.1 The Two-Layer Review System in Paper Writing

Paper writing uses two distinct review prompts targeting different levels:

1. **Section-level review** (during `paper-write`): GPT-5.4 reviews each section as it is written, using an "area chair" persona for holistic coherence checking
2. **Loop-level review** (in `auto-paper-improvement-loop`): GPT-5.4 reviews the complete paper, first for content, then for format compliance

The two layers catch different issues. Section-level review catches problems within a section. Loop-level review catches cross-section inconsistencies (abstract claims a result not shown in experiments, related work misrepresents a paper that's correctly cited in the method). Neither level alone is sufficient.

### 5.2 The "Method Description" Handoff for Diagrams

At the end of the auto-review-loop, the skill writes a specific section to AUTO_REVIEW.md:

```markdown
## Method Description
[1-2 paragraph concise description of the final method, its architecture, and data flow]
```

This section is written specifically to serve as input to `/paper-illustration`, which passes it to the Gemini API to generate architecture diagrams. This is a designed interface between two skills: the review loop produces a structured method description optimized for diagram generation, and the illustration skill reads exactly that section.

The iterative diagram refinement uses a dedicated scoring prompt:
```
Rate this diagram 1-10 for clarity and accuracy.
- Does it correctly represent the architecture?
- Is the data flow direction clear?
- Are the module names correct?
Suggest specific visual improvements.
```

The score threshold (≥9) for diagrams is higher than for paper content (≥6) because a bad diagram is more confusing than a weak section of text — readers process diagrams differently and a misleading diagram can undermine trust in the entire paper.

### 5.3 Feishu Interactive Loop Design

The `feishu-bridge` MCP server implements a semaphore-based blocking pattern for human-in-the-loop workflows. When `HUMAN_CHECKPOINT=true`, the prompt to the human (via Feishu) is structured:

```
📋 Round N/MAX_ROUNDS review complete.

Score: X/10 — [verdict]
Top weaknesses:
1. [weakness 1]
2. [weakness 2]
3. [weakness 3]

Suggested fixes:
1. [fix 1]
2. [fix 2]
3. [fix 3]

Options:
- Reply "go" or "continue" → implement all suggested fixes
- Reply with custom instructions → implement your modifications
- Reply "skip 2" → skip fix #2, implement the rest
- Reply "stop" → end the loop
```

The response parsing is deliberately flexible:
- Exact matches ("go", "continue") for the common case
- Free text for custom instructions (any other text)
- Skip syntax ("skip 1,3") for partial approval
- Stop ("stop", "enough", "done") for termination

This covers 99% of real use cases while keeping the instruction natural language. The human can send a Feishu message from their phone and the research pipeline responds correctly.

---

## Part 6 — Rationale Summary: The Ten Prompting Principles of ARIS

1. **Instructions over code**: Any capable LLM can follow well-written natural language. Markdown skills are model-agnostic, zero-dependency, and self-documenting.

2. **Structured output from reviewers**: Ask for specific fields (score, severity, minimum fix) rather than open-ended critique. Structured output enables programmatic parsing and prioritization.

3. **Cross-model review is adversarial, not confirmatory**: The reviewer model must be different from the executor. Different models have different training data, different biases, and different blind spots — their disagreement is information.

4. **State in files, not context**: LLM context windows are ephemeral. Files are persistent. Every piece of state that matters to the next step of a pipeline is written to a file before the current step ends.

5. **Thread continuity for multi-round reviews**: Using `codex-reply` with a saved `threadId` gives the reviewer memory of its own prior critiques. This enables genuine iterative improvement rather than disconnected re-reviews.

6. **Minimum fix framing**: Always ask for the smallest intervention that would address a weakness, not the ideal improvement. Minimum fixes are actionable; ideal improvements are not.

7. **Verbatim preservation**: Raw reviewer responses are preserved in full. Summaries are lossy; verbatim records are auditable.

8. **Graceful degradation**: Every optional integration is written as a conditional that silently skips if the integration is unavailable. The core pipeline works in any environment.

9. **Anti-hallucination at citation time**: BibTeX is fetched from canonical APIs (DBLP, CrossRef), never generated from model memory. Unresolvable citations are marked `[VERIFY]`, never fabricated.

10. **Least-privilege tool permissions**: Each skill's `allowed-tools` frontmatter grants only the capabilities the skill needs. This limits the blast radius of prompt injection or skill misuse.

---

## Appendix: Quick Reference — Context Passed to GPT-5.4 per Workflow Phase

| Phase | Skill | Context Passed to GPT-5.4 | Config |
|-------|-------|---------------------------|--------|
| Idea brainstorm | `idea-creator` | Research direction + literature landscape + identified gaps | xhigh |
| Devil's advocate | `idea-creator` | Previous brainstorm + filtered ideas + novelty results | xhigh (same thread) |
| Novelty check | `novelty-check` | Method description + papers found in multi-source search | xhigh |
| Critical review | `research-review` | Top idea + pilot results (incl. negatives) + novelty verdict | xhigh |
| Method refinement | `research-refine` | Problem Anchor + current proposal + literature scan | xhigh |
| Code review | `experiment-bridge` | Experiment plan + method description + implementation code | xhigh |
| Research review loop | `auto-review-loop` | Full research context + results + changes since last round | xhigh |
| Research review (reply) | `auto-review-loop` | Update summary + new results table | xhigh (same thread) |
| Paper plan review | `paper-plan` | Claims-Evidence Matrix + NARRATIVE_REPORT.md | xhigh |
| Section review | `paper-write` | Drafted section + paper plan + venue requirements | xhigh |
| Paper improvement | `auto-paper-improvement-loop` | Full paper LaTeX source | xhigh |

All calls use `config: {"model_reasoning_effort": "xhigh"}` to activate extended chain-of-thought reasoning, without exception. The latency cost (30-120 seconds per call) is acceptable because ARIS is designed for overnight unattended operation.
