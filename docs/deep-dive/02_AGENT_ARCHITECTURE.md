# ARIS Agent Architecture — The Two-Agent System

## The Core Idea: Adversarial Collaboration

ARIS is built around one insight: **a single AI model reviewing its own work is useless**. It's like asking someone to proofread their own essay right after writing it — they'll read what they meant to write, not what they actually wrote.

Instead, ARIS uses two fundamentally different models:

```
┌─────────────────────────────────────────────────────────┐
│                  ARIS Agent Pair                         │
│                                                         │
│   ┌───────────────────┐       ┌─────────────────────┐  │
│   │   EXECUTOR        │       │   REVIEWER          │  │
│   │  Claude Code      │◀─────▶│  GPT-5.4 xhigh      │  │
│   │                   │       │                     │  │
│   │  • Fast execution │       │  • Deep reasoning   │  │
│   │  • File system    │       │  • Critical eye     │  │
│   │  • Shell access   │       │  • No blind spots   │  │
│   │  • Code writing   │       │    shared with      │  │
│   │  • Multi-step     │       │    executor         │  │
│   │    planning       │       │                     │  │
│   └───────────────────┘       └─────────────────────┘  │
│            │                           │                │
│          writes                      scores             │
│            │                           │                │
│            ▼                           ▼                │
│   ┌──────────────────────────────────────────────────┐  │
│   │            Research Artifacts                    │  │
│   │  code, paper, experiments, IDEA_REPORT.md, etc.  │  │
│   └──────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────┘
```

---

## Why Two Models?

### The Self-Play Problem

When a model reviews its own work:
- It reinforces its own patterns (can't see what it's missing)
- It can't critique its own assumptions
- It tends to validate instead of challenge
- Score inflation is systematic — hard to get honest low scores

This is analogous to stochastic bandits vs adversarial bandits in reinforcement learning:
- **Self-review** = stochastic bandit (the same model's "noise" each time — predictable)
- **Cross-model review** = adversarial bandit (the reviewer actively probes weaknesses the executor didn't anticipate)

Adversarial bandits are fundamentally harder to game, which is exactly what you want in peer review.

### Why Two Models, Not Three or Four?

The biggest gain is going from 1→2 agents. Two-player games converge to Nash equilibrium efficiently. Adding more reviewers:
- Increases API cost
- Creates coordination overhead
- Produces diminishing returns after the second model

The sweet spot is 2: minimum needed to break self-play, maximum efficiency.

### Why Claude + GPT-5.4 Specifically?

Their **complementary failure modes** make them ideal partners:
- **Claude Code**: Fast, fluid, great at multi-step planning and execution. Weaker at adversarial critique.
- **GPT-5.4 xhigh**: Slower, more deliberate, much better at finding logical gaps and weaknesses. Uses extended reasoning (`model_reasoning_effort: xhigh`).

Speed × Rigor = better outcomes than either alone.

---

## How the Two Agents Communicate

### The MCP Bridge

MCP (Model Context Protocol) is the communication protocol. It works like a function call:

```
Claude Code (executor)
    │
    │  calls: mcp__codex__codex(prompt="Review this paper...")
    │
    ▼
Codex CLI running in --mcp-server mode
    │
    │  forwards request to OpenAI API
    │  with model=gpt-5.4, reasoning_effort=xhigh
    │
    ▼
GPT-5.4 processes the request (may take 30-120 seconds)
    │
    │  returns structured response (score, weaknesses, fixes)
    │
    ▼
Claude Code receives the response
    │
    │  parses the score, extracts action items
    │  implements the suggested fixes
```

### Thread Continuity

For multi-round reviews, ARIS maintains conversation continuity:

**Round 1:** `mcp__codex__codex(prompt="...full context...")` → gets `threadId`

**Round 2+:** `mcp__codex__codex-reply(threadId="019cd392-...", prompt="...updates since last review...")` → GPT-5.4 has full context of the previous conversation

This is important because GPT-5.4 can reference its previous critiques when assessing whether fixes were actually applied correctly.

---

## Agent Roles in Detail

### The Executor (Claude Code)

**What it knows:**
- Your entire file system
- How to run shell commands (SSH, rsync, screen, conda, etc.)
- How to write code in any language
- Multi-step planning across many files

**What it does:**
- Reads SKILL.md files and follows them
- Writes experiment scripts
- SSHes to GPU servers
- Monitors running experiments
- Rewrites paper sections
- Fetches BibTeX from DBLP/CrossRef
- Compiles LaTeX

**What it doesn't do:**
- Review its own work critically
- Score itself
- Decide what experiments are most valuable (that's the reviewer's job)

### The Reviewer (GPT-5.4 xhigh)

**What it knows:**
- The research content you share with it (no file system access)
- Deep academic knowledge for scoring and critique
- What makes a paper pass at NeurIPS/ICML/ICLR

**What it does:**
- Scores research 1-10 on conference standards
- Lists critical weaknesses (ranked by severity)
- Specifies minimum viable fixes for each weakness
- Gives a clear verdict: ready/almost/not ready
- Reviews code for logic bugs before GPU deployment
- Brainstorms 8-12 research ideas in Workflow 1

**What it doesn't do:**
- Access files directly
- Write code
- Run experiments

**Key configuration:**
```
config: {"model_reasoning_effort": "xhigh"}
```
This activates extended thinking mode in GPT-5.4 — much more thorough but slower.

---

## The Review Protocol

Every review request follows a structured format:

```
[Round N/MAX_ROUNDS of autonomous review loop]

[Full research context: claims, methods, results, known weaknesses]
[Changes since last round, if any]

Please act as a senior ML reviewer (NeurIPS/ICML level).

1. Score this work 1-10 for a top venue
2. List remaining critical weaknesses (ranked by severity)
3. For each weakness, specify the MINIMUM fix (experiment, analysis, or reframing)
4. State clearly: is this READY for submission? Yes/No/Almost

Be brutally honest. If the work is ready, say so clearly.
```

The reviewer always responds with:
- **Numeric score** (1-10)
- **Verdict** (ready / almost / not ready)
- **Action items** (ranked, with severity: CRITICAL / MAJOR / MINOR)

### Stop Condition

The loop stops when:
- Score ≥ 6 AND verdict contains "ready" or "almost"
- OR max rounds (4) reached

Why 6/10? That's the NeurIPS/ICML "weak accept" threshold — papers above this line are typically publishable.

---

## Alternative Reviewer Models

You can swap GPT-5.4 for other models:

| Reviewer | How | Guide |
|----------|-----|-------|
| GPT-5.4 xhigh (default) | Codex MCP server | Built-in |
| o3, gpt-5.3-codex | Change `model` in `~/.codex/config.toml` | `codex setup` |
| MiniMax-M2.7 | `mcp-servers/minimax-chat/` | `docs/MINIMAX_MCP_GUIDE.md` |
| DeepSeek, Kimi, GLM | `mcp-servers/llm-chat/` | `docs/LLM_API_MIX_MATCH_GUIDE.md` |
| Claude Code CLI | `mcp-servers/claude-review/` | `docs/CODEX_CLAUDE_REVIEW_GUIDE.md` |
| Any OpenAI-compatible | `mcp-servers/llm-chat/` | `docs/LLM_API_MIX_MATCH_GUIDE.md` |

**Important:** When using `llm-chat` instead of Codex, you replace `mcp__codex__codex` with `mcp__llm-chat__chat` in all skill calls. The skill behavior is identical.

---

## Idea Generation: The Reviewer as Brainstormer

In Workflow 1 (idea discovery), the roles are slightly different:

- **Claude Code** does the literature survey (reads papers, searches arXiv)
- **GPT-5.4** brainstorms 8-12 research ideas based on the literature landscape
- **Claude Code** filters ideas by feasibility and compute cost
- **GPT-5.4** then runs a deep novelty check on top ideas
- **Claude Code** runs pilot experiments on GPU
- **GPT-5.4** acts as a devil's advocate critic on the piloted ideas

Here GPT-5.4 is in "generator" mode first, then "critic" mode. The same model, two different personae.

---

## The Human in the Loop

ARIS has configurable human override points:

```
AUTO_PROCEED = true   ← Skip and auto-select (overnight mode)
AUTO_PROCEED = false  ← Wait for your approval before moving to next stage

HUMAN_CHECKPOINT = false  ← Review loop runs fully autonomous
HUMAN_CHECKPOINT = true   ← Pause after each review round, show score, wait for input
```

At each checkpoint, you can:
- Say "go" → proceed with suggested fixes
- Give custom instructions → executor uses your guidance instead
- Say "skip 2" → skip action item #2
- Say "stop" → terminate and document state

This design lets ARIS work as either **full autopilot** (set it and sleep) or **guided assistant** (you approve each step).
