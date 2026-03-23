# ARIS MCP Servers

## What Is MCP?

MCP (Model Context Protocol) is Anthropic's standard for extending AI agents with tools. In ARIS, MCP servers bridge Claude Code to external services (other LLMs, notification systems).

Every MCP server communicates via **JSON-RPC over stdio** (or HTTP in some cases). Claude Code discovers tools via the `tools/list` method and calls them via `tools/call`.

---

## Overview of ARIS MCP Servers

| Server | Directory | Protocol | Purpose |
|--------|-----------|----------|---------|
| Codex CLI | (external, from OpenAI) | stdio JSON-RPC | GPT-5.4 reviewer (primary) |
| `llm-chat` | `mcp-servers/llm-chat/` | stdio JSON-RPC | Any OpenAI-compatible API |
| `feishu-bridge` | `mcp-servers/feishu-bridge/` | HTTP REST | Feishu/Lark notifications |
| `claude-review` | `mcp-servers/claude-review/` | stdio JSON-RPC | Claude as reviewer |
| `minimax-chat` | `mcp-servers/minimax-chat/` | stdio JSON-RPC | MiniMax API specifically |

---

## 1. Codex CLI (Primary Reviewer)

### What It Is
The Codex CLI (`@openai/codex`) from OpenAI, run in MCP server mode. This is the default reviewer and the only one that supports `codex-reply` for multi-turn conversations.

### Setup
```bash
npm install -g @openai/codex
codex setup    # set model to gpt-5.4
claude mcp add codex -s user -- codex mcp-server
```

### Model Configuration
```toml
# ~/.codex/config.toml
model = "gpt-5.4"
# model = "gpt-5.3-codex"
# model = "o3"
```

### Tools Exposed to Claude Code

#### `mcp__codex__codex`
Start a new conversation thread with GPT-5.4.

```
Input:
  prompt: string         — The full review request
  config: object         — {"model_reasoning_effort": "xhigh"}

Output:
  text response          — Full reviewer response
  threadId               — Saved for codex-reply in subsequent rounds
```

#### `mcp__codex__codex-reply`
Continue an existing conversation thread (used for rounds 2-4 of auto-review-loop).

```
Input:
  threadId: string       — From the initial codex call
  prompt: string         — The update since last round
  config: object         — {"model_reasoning_effort": "xhigh"}

Output:
  text response          — GPT-5.4's follow-up assessment
```

### Why `xhigh` Reasoning?
`model_reasoning_effort: xhigh` activates GPT-5.4's extended thinking mode. It's slower (30-120 seconds per call) but significantly more thorough in identifying weaknesses. For research review, thoroughness > speed.

---

## 2. `llm-chat` — Generic OpenAI-Compatible Bridge

### What It Is
A small Python server that bridges Claude Code to any LLM with an OpenAI-compatible API. This makes ARIS model-agnostic for the reviewer role.

**Location:** `mcp-servers/llm-chat/server.py`

### Why It Exists
Codex CLI only works with OpenAI models. But many researchers have access to DeepSeek, Kimi, MiniMax, GLM, or other models via OpenAI-compatible APIs. `llm-chat` bridges this gap with a single Python file and no dependencies besides `httpx`.

### Architecture
```
Claude Code
    │
    │  mcp__llm-chat__chat(prompt="Review this...", system="Act as a senior reviewer")
    │
    ▼
llm-chat/server.py (stdio JSON-RPC server)
    │
    │  POST /v1/chat/completions
    │  Authorization: Bearer {LLM_API_KEY}
    │
    ▼
Any OpenAI-compatible API (DeepSeek, Kimi, GLM, MiniMax, etc.)
    │
    ▼
Response text
    │
    ▼
Claude Code receives the reviewer's response
```

### Configuration (Environment Variables)
```bash
LLM_API_KEY=your_api_key
LLM_BASE_URL=https://api.deepseek.com/v1     # or any compatible endpoint
LLM_MODEL=deepseek-r1                         # or any model name
LLM_SERVER_NAME=llm-chat                      # affects MCP tool name
```

### Claude Code Config (settings.json)
```json
{
  "mcpServers": {
    "llm-chat": {
      "command": "python3",
      "args": ["/path/to/mcp-servers/llm-chat/server.py"],
      "env": {
        "LLM_API_KEY": "your-key",
        "LLM_BASE_URL": "https://api.deepseek.com/v1",
        "LLM_MODEL": "deepseek-r1"
      }
    }
  }
}
```

### API

#### Tool: `chat`
```
Input:
  prompt: string         — The message to send
  model: string          — Optional model override
  system: string         — Optional system prompt

Output:
  text response          — LLM response content
```

### Protocol Details
The server implements JSON-RPC over stdio in two transport modes:
- **Content-Length framing** (default): `Content-Length: N\r\n\r\n{json}`
- **NDJSON** (fallback): `{json}\n` — auto-detected from first byte

```python
def read_message():
    line = sys.stdin.readline()
    if line.lower().startswith("content-length:"):
        # Read Content-Length framed message
        ...
    elif line.startswith("{"):
        # NDJSON mode
        _use_ndjson = True
        ...
```

**Why two modes?** Different MCP clients send messages differently. The server auto-detects and sticks with the detected format.

### Supported Providers (Tested)
| Provider | Base URL | Model Example |
|----------|----------|--------------|
| OpenAI | `https://api.openai.com/v1` | `gpt-4o` |
| DeepSeek | `https://api.deepseek.com/v1` | `deepseek-chat` |
| Kimi (Moonshot) | `https://api.moonshot.cn/v1` | `moonshot-v1-32k` |
| MiniMax | `https://api.minimax.chat/v1` | `MiniMax-M2.5` |
| GLM (Z.ai) | `https://api.z.ai/api/anthropic` | `glm-5` |
| LongCat (Meituan) | (custom) | (custom) |
| ModelScope | `https://dashscope.aliyuncs.com/compatible-mode/v1` | various |

---

## 3. `feishu-bridge` — Feishu Notification Server

### What It Is
A Python HTTP server that bridges ARIS skills to Feishu (Lark). It provides two-way communication:
- **Send** rich card notifications to Feishu users
- **Poll** for user replies (used for interactive mode)

**Location:** `mcp-servers/feishu-bridge/server.py`

### Architecture

```
Skill (feishu-notify) ──▶ HTTP POST /send ──▶ feishu-bridge ──▶ Feishu API
                                                    │
                       HTTP GET /poll ◀─────────────┘
                       (long-polling)
                           │
                      User replies in ──▶ POST /reply ──▶ threading.Event.set()
                      Feishu app          (webhook from     (wake up /poll)
                                          Feishu server)
```

### API Endpoints

#### `POST /send`
Send a notification to a Feishu user.

```json
Request:
{
  "user_id": "ou_...",          // Feishu open_id (optional if FEISHU_USER_ID set)
  "type": "card",               // "card" or "text"
  "title": "ARIS Notification", // Card header
  "body": "Round 2: 6.5/10...", // Card body (markdown)
  "color": "green"              // Header color: blue/green/orange/red/purple/yellow
}

Response:
{
  "ok": true,
  "message_id": "om_..."        // Used for polling
}
```

#### `GET /poll?message_id=om_...&timeout=300`
Long-poll for user reply to a specific message.

```json
Response (when user replies):
{
  "reply": "go"                  // User's text reply
}

Response (on timeout):
{
  "timeout": true
}
```

#### `POST /reply`
Called by Feishu's webhook when user sends a message. Wakes up any waiting `/poll`.

```json
Request:
{
  "message_id": "om_...",
  "text": "go"
}
```

#### `GET /health`
```json
{"status": "ok", "port": 5000}
```

### Threading Model
```python
reply_store = {}      # message_id → reply text
reply_events = {}     # message_id → threading.Event
reply_lock = threading.Lock()

# When /send is called:
reply_events[msg_id] = threading.Event()
reply_store[msg_id] = None

# When /poll is called:
event.wait(timeout=300)  # blocks until reply or timeout

# When /reply is called (webhook):
reply_store[msg_id] = text
reply_events[msg_id].set()  # wakes up the waiting /poll
```

### Card Color Semantics
| Color | Use Case |
|-------|---------|
| 🟢 green | Review score ≥ 6 / experiment success |
| 🟠 orange | Review score < 6 |
| 🟡 yellow | Checkpoint waiting for user input |
| 🔴 red | Error occurred |
| 🟣 purple | Pipeline complete |
| 🔵 blue | General notification / experiment deployed |

### Configuration
```bash
export FEISHU_APP_ID=cli_your_app_id
export FEISHU_APP_SECRET=your_app_secret
export FEISHU_USER_ID=ou_your_open_id
export BRIDGE_PORT=5000
python3 mcp-servers/feishu-bridge/server.py
```

---

## 4. `claude-review` — Claude as Reviewer

### What It Is
A bridge that exposes Claude Code CLI as an MCP reviewer. Used when you want **Codex CLI as executor and Claude Code as reviewer** (the reverse of the default setup).

**Location:** `mcp-servers/claude-review/server.py`

### Why It Exists
Some users prefer Codex CLI for execution but want Claude's review capabilities. The `claude-review` server creates a Claude Code subprocess, sends it review prompts, and returns responses to Codex.

### Setup
```bash
# Generate Codex skill overrides for claude-review
python3 tools/generate_codex_claude_review_overrides.py
# Install overrides
cp -r skills/skills-codex-claude-review/* ~/.codex/skills/
```

Full guide: `docs/CODEX_CLAUDE_REVIEW_GUIDE.md`

### Communication Pattern
```
Codex CLI (executor)
    │
    │  mcp__claude-review__review(prompt="...")
    │
    ▼
claude-review server
    │
    │  spawn: claude --no-permissions "prompt"
    │  (or claude --resume with async polling)
    │
    ▼
Claude Code CLI (reviewer)
    │
    ▼
Response via stdout → captured → returned to Codex
```

---

## 5. `minimax-chat` — MiniMax API Bridge

### What It Is
Like `llm-chat` but specifically for MiniMax, which has a non-standard API format (not OpenAI-compatible).

**Location:** `mcp-servers/minimax-chat/server.py`

### Why Separate?
MiniMax's API uses different request/response formats than OpenAI. Rather than adding format detection to `llm-chat`, a dedicated server handles MiniMax's quirks.

Setup guide: `docs/MINIMAX_MCP_GUIDE.md`

---

## MCP Transport Protocol (Technical Details)

All ARIS MCP servers implement the **2024-11-05** MCP protocol version. The JSON-RPC message lifecycle:

### Initialization
```json
// Client → Server
{"jsonrpc": "2.0", "id": 1, "method": "initialize", "params": {...}}

// Server → Client
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": {
    "protocolVersion": "2024-11-05",
    "capabilities": {"tools": {}},
    "serverInfo": {"name": "llm-chat", "version": "2.0.0"}
  }
}
```

### Tool Discovery
```json
// Client → Server
{"jsonrpc": "2.0", "id": 2, "method": "tools/list"}

// Server → Client
{
  "jsonrpc": "2.0",
  "id": 2,
  "result": {
    "tools": [{
      "name": "chat",
      "description": "Send message to GPT-4o...",
      "inputSchema": {"type": "object", "properties": {...}, "required": ["prompt"]}
    }]
  }
}
```

### Tool Call
```json
// Client → Server
{
  "jsonrpc": "2.0",
  "id": 3,
  "method": "tools/call",
  "params": {
    "name": "chat",
    "arguments": {"prompt": "Review this paper...", "system": "Act as a senior reviewer"}
  }
}

// Server → Client
{
  "jsonrpc": "2.0",
  "id": 3,
  "result": {
    "content": [{"type": "text", "text": "Score: 7/10. Strengths: ..."}]
  }
}
```

### Notifications (No Response Expected)
```json
// Server → Client (no id, no response)
{"jsonrpc": "2.0", "method": "notifications/initialized"}
```
