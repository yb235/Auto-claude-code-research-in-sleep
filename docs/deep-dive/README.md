# ARIS Deep-Dive Documentation

Written for a "dummy" who wants to understand ARIS from the inside out — every design decision explained, every data format documented, every component demystified.

## Reading Order

| File | What's Inside | Read If... |
|------|-------------|-----------|
| [00_OVERVIEW.md](00_OVERVIEW.md) | What ARIS is, the mental model, surprising facts | Start here |
| [01_ARCHITECTURE.md](01_ARCHITECTURE.md) | System layers, components, how they connect | You want the full map |
| [02_AGENT_ARCHITECTURE.md](02_AGENT_ARCHITECTURE.md) | Two-agent design, adversarial collaboration, MCP bridge | You want to understand the AI architecture |
| [03_WORKFLOWS.md](03_WORKFLOWS.md) | Each workflow in deep detail, phase by phase | You're using a workflow |
| [04_SKILLS_REFERENCE.md](04_SKILLS_REFERENCE.md) | Every skill: purpose, inputs, outputs, constants | You're building or modifying a skill |
| [05_DATA_FLOW.md](05_DATA_FLOW.md) | How data moves through the system, step by step | You're debugging or tracing execution |
| [06_DATA_SCHEMA.md](06_DATA_SCHEMA.md) | All file formats, JSON schemas, config schemas | You're integrating or building tooling |
| [07_MCP_SERVERS.md](07_MCP_SERVERS.md) | Python servers, APIs, JSON-RPC protocol | You're adding a new model or bridge |
| [08_INTEGRATIONS.md](08_INTEGRATIONS.md) | Feishu, Zotero, arXiv, DBLP, GPU, W&B | You're configuring integrations |
| [09_DESIGN_RATIONALE.md](09_DESIGN_RATIONALE.md) | Why every major decision was made this way | You're curious or extending the system |

## Quick Facts

- **31 skills** — composable, model-agnostic Markdown files
- **4 MCP servers** — tiny Python bridges to external services
- **4 main workflows** — chain from idea discovery to submitted paper
- **Zero dependencies** — copy `skills/` to `~/.claude/skills/` and go
- **Any LLM** — swap Claude, GPT, GLM, MiniMax, DeepSeek freely
- **Real papers** — AAAI 2026 accepted paper built entirely with ARIS
