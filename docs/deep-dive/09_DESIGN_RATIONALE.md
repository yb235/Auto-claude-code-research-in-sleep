# ARIS Design Rationale

## Why Things Are Designed The Way They Are

This document explains the "why" behind every significant design decision in ARIS — the ones that might make you raise an eyebrow and ask "but why not just...?"

---

## 1. "Why is everything just a Markdown file?"

**The decision:** Skills are plain `.md` files, not Python scripts, JSON configs, or a framework.

**The rationale:**

### a) Any LLM Can Read Markdown
A Python script is specific to a Python executor. A YAML config requires a specific parser. A Markdown file is universal — any LLM, any CLI tool, any human can read and follow it. This is what makes ARIS portable: the same `SKILL.md` works in Claude Code, Codex CLI, Cursor, Trae, Antigravity, and OpenClaw.

### b) Skills Are Self-Documenting
You can read a SKILL.md and immediately understand what it does. No source code to read, no documentation to find. The instructions *are* the documentation.

### c) Fork-Friendly
You can modify a skill by editing a text file. No build step, no compilation, no tests to run. Change a constant, save, done. This lowers the barrier for contributions enormously (12 community skills added in 2 weeks).

### d) Zero Lock-In
If Claude Code disappears tomorrow, your skills still work with any LLM that can read Markdown. You don't lose your research workflows.

**The cost:** Markdown instructions are less precise than code. An LLM might interpret the same instructions slightly differently. But for research workflows, flexibility often outweighs precision.

---

## 2. "Why two models instead of one?"

**The decision:** Use GPT-5.4 as the reviewer instead of having Claude review its own work.

**The rationale:**

### The Self-Play Blind Spots Problem
When Claude Code reviews its own paper:
1. It tends to validate its own writing style
2. It can't detect its own knowledge gaps
3. It reinforces its own assumptions
4. Review scores inflate systematically (it's hard to criticize yourself honestly)

This is mathematically analogous to a stochastic bandit — the noise in self-review is predictable and gameable.

### Cross-Model Review Is Adversarial
Different models have different training data, different emphasis, different failure modes. What Claude Code finds convincing, GPT-5.4 may find poorly justified. Their disagreement is information.

This is like an adversarial bandit: the reviewer actively probes weaknesses the executor didn't anticipate.

### Why Two, Not Three?
Two-player games converge to Nash equilibrium. The biggest gain is 1→2. Going from 2→3 adds:
- ~50% more API cost
- Coordination overhead (resolving conflicting reviews)
- Diminishing signal (the third model mostly agrees with one of the first two)

The diminishing returns argument is strong enough that ARIS settled on exactly 2.

---

## 3. "Why `model_reasoning_effort: xhigh`?"

**The decision:** All reviewer calls use GPT-5.4 with `xhigh` reasoning effort.

**The rationale:**

For research review, the cost of missing a critical weakness is much higher than the cost of slower/more expensive reviews. If the reviewer misses an obvious flaw, you might submit a paper that gets rejected for something easily fixable.

`xhigh` activates extended thinking (o3-style chain-of-thought reasoning). This:
- Catches more subtle logical gaps
- Identifies assumption violations
- Provides more actionable critique
- Is worth the 30-120 second latency for overnight runs

You're asleep when the review happens. Speed doesn't matter.

---

## 4. "Why persist state in JSON files instead of a database?"

**The decision:** `REVIEW_STATE.json` stores loop state in a plain file.

**The rationale:**

### Zero Setup
A JSON file requires no database, no connection string, no migrations, no schema. It exists as long as the project directory exists.

### Human-Readable Debugging
When something goes wrong at 3am and you wake up to a stopped loop, you can `cat REVIEW_STATE.json` and immediately see what happened: which round failed, what the last score was, which experiments were running.

### Git-Friendly
State files can be tracked in git, giving you a history of review rounds. This is valuable for understanding the evolution of a project.

### Stateless Instructions, Stateful Files
This pattern (instructions that read/write state from files) is robust to crashes: the skill doesn't "remember" anything — it reads the state file on startup. There's no in-memory state to lose.

**The cost:** No ACID transactions. Two concurrent runs could corrupt the state file. But ARIS is designed for single-user research, not distributed teams, so this is acceptable.

---

## 5. "Why is `AUTO_REVIEW.md` append-only?"

**The decision:** The review log is never overwritten — only appended.

**The rationale:**

Research history matters. When you look at a paper 3 months later and wonder "why did we pivot away from the original hypothesis?", you need the full audit trail. Overwriting the log would lose this.

The `## Reviewer Raw Response` section stores the **verbatim, unedited** GPT-5.4 response. This is the authoritative record. Summaries can be wrong; the raw response cannot be wrong.

This design also prevents the executor from "cleaning up" inconvenient reviews. The reviewer's full criticism is preserved, even the parts that were ultimately ignored.

---

## 6. "Why is the DBLP/CrossRef chain the default?"

**The decision:** `DBLP_BIBTEX=true` is the default for paper writing.

**The rationale:**

LLM-hallucinated citations are a serious problem in academic writing:
- They cite papers that don't exist
- They get venue names wrong ("Workshop on..." instead of "Advances in...")
- They invent page numbers
- They swap co-authors

A paper submitted with hallucinated citations is immediately credibility-damaging. Reviewers check citations.

DBLP covers virtually all CS/ML conference papers with publisher-verified metadata. CrossRef covers journal papers and arXiv preprints via DOI. Together they handle ~95% of ML citations.

The `[VERIFY]` marker for the remaining 5% is better than fabrication — it tells the human author "I couldn't verify this, please check manually."

**The cost:** Requires internet access during paper writing. But `latexmk` already requires internet (for package downloads), so this is acceptable.

---

## 7. "Why use `screen` for experiment management instead of something modern?"

**The decision:** Remote experiments run in `screen` sessions rather than tmux, Docker, slurm, etc.

**The rationale:**

### Universal Availability
`screen` is pre-installed on virtually every Linux server. No setup, no permissions needed, no cluster configuration. An experiment deployed to any SSH-accessible machine just works.

### Inspectable State
`screen -ls` shows running experiments. `screen -r exp_name` lets you attach to a running experiment. `tail -50 exp_name.log` shows progress. All without any special tooling.

### Simple Kill
`screen -S exp_name -X quit` kills an experiment cleanly. No job IDs to track, no scheduler to talk to.

### Crash Recovery
If Claude Code disconnects mid-deployment, the `screen` sessions keep running. The next `/monitor-experiment` call finds them exactly where they left off.

**The cost:** No job scheduling, no resource management, no queue. For research projects with <10 concurrent experiments, this is fine. For large-scale sweeps, users should configure slurm/kubernetes in their CLAUDE.md and the skill will adapt.

---

## 8. "Why does Workflow 1 run pilot experiments before deep novelty checks?"

**The decision:** GPU pilots happen before deep novelty verification.

**The rationale:** This is a deliberate prioritization of empirical over theoretical.

Traditional research flow:
1. Brainstorm idea
2. Extensively verify novelty (search for weeks)
3. Implement and run experiments

ARIS flow:
1. Brainstorm 8-12 ideas
2. Quick (shallow) novelty filter — eliminates obvious duplicates
3. **Run 2-hour GPU pilots on top 3 ideas**
4. Deep novelty check — only for ideas with positive pilot signal

**Why?** A 2-hour pilot that shows negative results is much cheaper than:
- Days of deep novelty research on an idea that doesn't empirically work
- Fully implementing an idea before discovering it doesn't work

The "positive pilot signal" filter is the key gate. Ideas that don't show even weak positive signal in a pilot are deprioritized regardless of their theoretical novelty.

**The philosophical claim:** In ML research, empirical evidence should come before theoretical justification. "Does it work at all?" is more important than "Is it novel?" when deciding where to invest compute.

---

## 9. "Why does Workflow 3 start with a Narrative Report instead of directly from results?"

**The decision:** Paper writing starts from a human-written `NARRATIVE_REPORT.md`, not from experiment results directly.

**The rationale:**

### Humans Provide the Story
Experiment results are data. A paper needs a story: why this problem matters, what insight led to the solution, how the method works conceptually. These require human judgment about significance and framing.

### Separation of Concerns
- Human: "What is the story we're telling?"
- AI: "How do we tell it in perfect academic prose?"

This division is more reliable than asking the AI to invent the story from raw numbers.

### The Narrative Report as a Forcing Function
Writing the narrative forces you to think carefully about: What's the key claim? What evidence supports it? What's the take-home message? This thinking often improves the research, not just the paper.

### Anti-Hallucination
When the AI writes from a narrative that contains your actual results, it can't hallucinate results. The narrative is the ground truth.

---

## 10. "Why is `AUTO_PROCEED=true` the default?"

**The decision:** The full pipeline auto-selects the top idea without waiting for user confirmation.

**The rationale:** The name of the project is "Research in Sleep." The primary use case is overnight autonomous operation.

With `AUTO_PROCEED=false`, every phase transition waits for user input. This is appropriate during the day when you're actively working, but it defeats the purpose of overnight automation.

The design provides both modes:
- `AUTO_PROCEED=true` (default): full autopilot — suitable for overnight runs
- `AUTO_PROCEED=false`: step-by-step approval — suitable for careful research decision-making

The important safety valve is that Stage 1 (idea discovery) should be run separately before bed, so you can review the ideas before launching the expensive Stage 3-4 operations. The pipeline documents this as the "sweet spot" workflow.

---

## 11. "Why no framework like LangChain or AutoGPT?"

**The decision:** ARIS uses no agent framework.

**The rationale:**

### Frameworks Add Complexity
LangChain, AutoGPT, and similar frameworks have:
- Version dependencies to manage
- APIs that break between versions
- Abstractions that obscure what's happening
- Significant learning curves

### The Core Is Just Tool Calls
An ARIS "skill" is fundamentally: read files → generate content → call external APIs → write files. This maps directly to Claude Code's native tool capabilities. A framework would add a layer of abstraction without adding capability.

### Portability Requires Frameworklessness
A skill that depends on LangChain can only run in Python environments with LangChain installed. A Markdown file can run anywhere.

### Debugging Is Easier
When something goes wrong, you can read the SKILL.md and understand exactly what should have happened. With a framework, you have to trace through multiple layers of abstraction.

---

## 12. "What's curious about the `feishu-bridge` threading model?"

This one deserves special attention because it's elegantly simple.

**The problem:** You want Claude Code to "pause and wait for user input via Feishu." But HTTP is request/response — you can't block an HTTP request until a user replies (connection timeout).

**The solution:**
1. When Claude Code sends a message (`POST /send`), the server creates a `threading.Event` for that message ID
2. Claude Code then calls `GET /poll?message_id=...` which blocks (waits) for up to 300 seconds
3. When the user replies in Feishu, Feishu calls the server's `/reply` webhook with the message text
4. The server sets the `threading.Event`, which wakes up the blocked `/poll` call
5. Claude Code receives the user's reply and continues

This is essentially a **semaphore-based async pattern** implemented in synchronous HTTP servers using `threading.Event`. No async framework required.

The 300-second default is "5 minutes to read a review score and decide what to do" — reasonable for checkpoint decisions.
