# ARIS External Integrations

## Integration Philosophy

Every integration in ARIS follows the same principle: **graceful degradation**.

If an integration is not configured, the skill **silently skips** it and continues with whatever is available. You never get a crash or an error because Zotero isn't set up. The system just works with less.

This is intentional. It means you can start with the simplest setup (just Claude Code + Codex) and progressively add integrations without breaking anything.

---

## 1. Codex CLI / GPT-5.4 (Required for Review Skills)

### What It Does
Provides the adversarial reviewer. Without it, review skills (`auto-review-loop`, `research-review`, `novelty-check`, etc.) can't function.

### Setup
```bash
npm install -g @openai/codex
codex setup                           # sets model to gpt-5.4 interactively
claude mcp add codex -s user -- codex mcp-server
```

### What Gets Configured
- `~/.codex/config.toml`: `model = "gpt-5.4"`
- Claude Code's MCP config: adds `codex` as an available tool

### Alternatives
See the "Alternative Model Combinations" section in `docs/LLM_API_MIX_MATCH_GUIDE.md` for swapping to other models.

---

## 2. Zotero (Optional — Literature Management)

### What It Does
Lets `/research-lit` search your personal paper library, read your annotations, and export BibTeX — before searching the internet.

### Why It's Valuable
Zotero annotations capture what you personally highlighted as important in papers you've read. This is more valuable than a fresh web search because it reflects your domain knowledge and prior reading.

### Recommended: zotero-mcp (1.8k⭐)
```bash
# Option A: Local API (requires Zotero desktop running)
uv tool install zotero-mcp-server
claude mcp add zotero -s user -- zotero-mcp -e ZOTERO_LOCAL=true

# Option B: Web API (works without Zotero running)
claude mcp add zotero -s user -- zotero-mcp \
  -e ZOTERO_API_KEY=your_key -e ZOTERO_USER_ID=your_id
```

### What Skills Use It
- `research-lit` (Step 0a): Searches collections, reads annotations, exports BibTeX
- Detected at runtime: tries `mcp__zotero__*` tool, skips if unavailable

### Data Flow
```
research-lit invokes:
    mcp__zotero__search(query="diffusion models")
        → list of papers with metadata
    mcp__zotero__get_item(key="ABCD1234")
        → paper with annotations/highlights
    mcp__zotero__get_bibtex(key="ABCD1234")
        → BibTeX entry for later use in paper-write
```

---

## 3. Obsidian (Optional — Research Notes)

### What It Does
Lets `/research-lit` search your vault for paper summaries, tagged references, and your own insights — treating your personal notes as a literature source.

### Why It's Valuable
Obsidian notes often contain your **processed understanding** of papers (what you concluded, how it relates to your work), which is more useful than re-reading raw papers.

### Recommended: mcpvault (760⭐)
```bash
claude mcp add obsidian-vault -s user -- npx @bitbonsai/mcpvault@latest /path/to/your/vault
```

### What Skills Use It
- `research-lit` (Step 0b): Searches vault, reads tagged notes, follows wikilinks
- Detected at runtime: tries `mcp__obsidian-vault__*` tool, skips if unavailable

### Tags Convention (Recommended)
```
#paper-review     — papers you've read and summarized
#diffusion-models — topic tag for quick search
#todo             — papers flagged for deeper reading
```

### Zotero + Obsidian Together
Many researchers use Zotero for paper storage and Obsidian for notes. ARIS supports both simultaneously:
1. Zotero → raw papers + your PDF annotations
2. Obsidian → your processed summaries and insights
3. Local PDFs → any papers not in Zotero
4. Web search → gaps

---

## 4. arXiv (Built-in — No Setup)

### What It Does
Searches arXiv for structured paper metadata. Used by `research-lit` and the standalone `arxiv` skill.

### How It Works
Uses `tools/arxiv_fetch.py` which calls the public arXiv Atom API:

```
http://export.arxiv.org/api/query?search_query=QUERY&max_results=10&sortBy=relevance
```

Returns structured JSON:
```json
{
  "id": "2301.07041",
  "title": "...",
  "authors": ["Smith, John", "Doe, Jane"],
  "abstract": "...",
  "published": "2023-01-17",
  "categories": ["cs.LG", "cs.AI"],
  "pdf_url": "https://arxiv.org/pdf/2301.07041.pdf",
  "abs_url": "https://arxiv.org/abs/2301.07041"
}
```

### PDF Download (Optional)
```bash
# In research-lit:
/research-lit "topic" — arxiv download: true     # download top 5
/research-lit "topic" — arxiv download: true, max download: 10

# Standalone:
/arxiv "2301.07041" — download
```

Rate limiting: 1-second delay between downloads. Retry on 429. Verifies file > 10KB (error pages are smaller).

### Why a Custom Script Instead of the arxiv Python Package?
Zero dependencies. `tools/arxiv_fetch.py` uses only Python stdlib (`urllib`, `xml.etree`, `json`). No pip install needed.

---

## 5. DBLP / CrossRef (Built-in — Anti-Hallucination Citations)

### What It Does
Fetches real, verified BibTeX entries for citations in papers, preventing LLM-hallucinated citations.

### Why It Matters
LLM-generated BibTeX frequently:
- Invents venue names ("International Workshop on...")
- Gets page numbers wrong
- Adds non-existent co-authors
- Makes up DOIs

DBLP and CrossRef return publisher-verified metadata.

### The Three-Step Chain

```bash
# Step A: DBLP (best quality)
curl -s "https://dblp.org/search/publ/api?q=TITLE+AUTHOR&format=json&h=3"
# extract DBLP key (e.g., conf/nips/VaswaniSPUJGKP17)
curl -s "https://dblp.org/rec/conf/nips/VaswaniSPUJGKP17.bib"

# Step B: CrossRef (fallback — works for arXiv preprints)
# If paper has a DOI (arXiv DOI = 10.48550/arXiv.{id})
curl -sLH "Accept: application/x-bibtex" "https://doi.org/{doi}"

# Step C: Mark [VERIFY] (last resort)
# If both fail, add % [VERIFY] comment — NEVER fabricate
```

### Usage
Controlled by `DBLP_BIBTEX=true` (default) in `paper-write`. Set to `false` to disable.

Triggered by: `/paper-write`, `/auto-review-loop` (when adding citations during fixes)

### Bib Cleaning
After building the bibliography, a Python script filters `references.bib` to only include entries that are actually `\cite`d in the paper:
```
Before: 948 entries (full research group bib)
After: 215 entries (only cited)
```

---

## 6. GPU Servers (Required for Auto-Experiments)

### What It Does
Enables Claude Code to SSH to your GPU server, sync code, launch experiments in screen sessions, and monitor results.

### Setup
Add to your project's `CLAUDE.md`:
```markdown
## Remote Server
- SSH: `ssh my-gpu-server`   # key-based auth (no password prompt)
- GPU: 4x A100 (80GB each)
- Conda: `eval "$(/opt/conda/bin/conda shell.bash hook)" && conda activate research`
- Code dir: `/home/user/experiments/`
```

Or if you're already on the GPU server:
```markdown
## GPU Environment
- This machine has direct GPU access (no SSH needed)
- GPU: 4x A100 80GB
- Activate: `conda activate research`
- Code directory: `/home/user/experiments/`
```

### What Happens
```bash
# Check GPU availability
nvidia-smi --query-gpu=index,memory.used,memory.total --format=csv,noheader

# Sync code
rsync -avz --include='*.py' --exclude='*' local/ server:/remote/

# Launch experiment
screen -dmS exp_name bash -c 'conda activate research && CUDA_VISIBLE_DEVICES=0 python train.py ...'

# Monitor
screen -ls
tail -50 exp_name.log
```

### Without GPU Access
Review and rewriting skills still work. Only experiment-related fixes are skipped and flagged for manual follow-up.

---

## 7. Weights & Biases / W&B (Optional — Experiment Tracking)

### What It Does
Auto-adds W&B logging to experiment scripts and lets `/monitor-experiment` pull training curves.

### Setup
1. `wandb: true` in your `CLAUDE.md`
2. `wandb_project: my-project` in `CLAUDE.md`
3. `wandb login` on the GPU server (or set `WANDB_API_KEY` env var)

### What Gets Added to Scripts
```python
import wandb
wandb.init(project=WANDB_PROJECT, name=EXP_NAME, config={...hyperparams...})

# Inside training loop:
wandb.log({"train/loss": loss, "train/lr": lr, "step": step})
wandb.log({"eval/loss": eval_loss, "eval/ppl": ppl, "eval/accuracy": acc})

wandb.finish()
```

### What `/monitor-experiment` Does
```python
api = wandb.Api()
runs = api.runs(f"{entity}/{project}")
# Pull training curves, compute stats, report results
```

---

## 8. Feishu / Lark (Optional — Mobile Notifications)

### What It Does
Sends mobile notifications when experiments finish, papers get reviewed, or checkpoints need your input — without sitting at your terminal.

### Three Modes

| Mode | What Happens | Setup Time |
|------|-------------|-----------|
| **Off** (default) | Nothing | None |
| **Push** | Rich cards in a group chat | 5 minutes |
| **Interactive** | Cards in group + private chat (bidirectional) | 15 minutes |

### Push Mode Setup
```bash
# 1. Create a Feishu group bot (webhook)
# 2. Create config
cat > ~/.claude/feishu.json << 'EOF'
{
  "mode": "push",
  "webhook_url": "https://open.feishu.cn/open-apis/bot/v2/hook/YOUR_ID"
}
EOF
```

### Interactive Mode
Requires the `mcp-servers/feishu-bridge/` server running and a Feishu app with these permissions:
- `im:message` — send/receive messages
- `im:message:send_as_bot` — send as bot
- `im:message.p2p_msg:readonly` — ⚠️ Critical: receive private messages
- `im:message.group_at_msg:readonly` — group @mentions
- `im:resource` — attachments

### Notification Events

| Event | Color | Content |
|-------|-------|---------|
| Review scored ≥ 6 | 🟢 green | Score, verdict, top weaknesses |
| Review scored < 6 | 🟠 orange | Score, verdict, action items |
| Experiment deployed | 🔵 blue | GPU assignments, ETAs |
| Experiment complete | 🟢 green | Results table |
| Checkpoint waiting | 🟡 yellow | Question + options |
| Error | 🔴 red | Error message, suggested fix |
| Pipeline complete | 🟣 purple | Score progression, deliverables |

### Detection Pattern
Every skill that sends notifications does:
```python
# Check if feishu.json exists and mode != "off"
if config_exists and mode != "off":
    send_notification(...)
else:
    pass  # silent no-op
```

### Alternative Notification Platforms
The push-only webhook pattern works with any service accepting incoming webhooks:
- Slack: change webhook URL + card format
- Discord: change webhook URL + embed format
- DingTalk (钉钉): compatible webhook format
- WeChat Work: compatible webhook format

---

## 9. Gemini API (Optional — AI Figure Generation)

### What It Does
Generates publication-quality architecture diagrams and method figures for papers. Used by the `paper-illustration` skill.

### Setup
```bash
export GEMINI_API_KEY="your-key"
# Get key at: https://aistudio.google.com/apikey
```

### How It Works
Claude plans the diagram → Gemini renders it → iterative refinement until score ≥ 9

Built on [PaperBanana](https://github.com/dwzhu-pku/PaperBanana).

### Free Alternative
Use `illustration: mermaid` instead — generates Mermaid diagrams, no API key needed. Less polished but free and instant.

---

## Integration Priority Map

```
For /research-lit:
  1. Zotero (your annotated collection)
  2. Obsidian (your processed notes)
  3. Local PDFs (papers/ or literature/)
  4. arXiv API (structured metadata, free)
  5. Web search (Google Scholar, Semantic Scholar)

For paper citations:
  1. DBLP (best quality, peer-reviewed)
  2. CrossRef (fallback, arXiv preprints)
  3. [VERIFY] marker (manual check required)

For reviewer LLM:
  1. Codex CLI → GPT-5.4 xhigh (default)
  2. llm-chat → DeepSeek/Kimi/GLM/MiniMax (alternative)
  3. claude-review → Claude Code CLI (alternative)

For notifications:
  1. ~/.claude/feishu.json mode=interactive → bidirectional
  2. ~/.claude/feishu.json mode=push → one-way
  3. No config → silent (always works)

For experiments:
  1. Remote GPU server (SSH + screen)
  2. Local GPU (CUDA)
  3. Local MPS (Apple Silicon)
  4. No GPU → skip experiments, flag for manual
```
