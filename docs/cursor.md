# Cursor Agent Integration

This document explains how cc-connect integrates with [Cursor](https://cursor.com) — the architecture, communication protocol, and setup.

## Overview

cc-connect controls Cursor through the **Cursor Agent CLI** (`agent` binary), **not** the Cursor IDE directly. The `agent` CLI is a headless, terminal-based tool that provides the same AI coding capabilities as the Cursor editor but runs as a subprocess controlled by cc-connect.

```
User (Phone/Chat App)
     │
     ▼
Platform (Feishu / Telegram / Slack / ...)
     │
     ▼
cc-connect Engine (routes messages)
     │
     ▼
agent/cursor/ (adapter layer)
     │  Spawns subprocess per turn:
     │  agent --print --output-format stream-json --trust ...
     ▼
Cursor Agent CLI (`agent` binary)
     │
     │  stdout: newline-delimited JSON events
     │
     ▼
cc-connect Engine (parses events → sends to platform)
     │
     ▼
User sees response in chat
```

## How It Works

### 1. Subprocess Communication

Each time a user sends a message, cc-connect launches the `agent` CLI as a **subprocess** with flags that enable programmatic communication:

```bash
agent \
  --print \                          # Enable non-interactive output
  --output-format stream-json \      # Output as newline-delimited JSON
  --trust \                          # Trust the workspace
  [--force] \                        # Auto-approve tools (mode=force)
  [--mode plan|ask] \                # Plan or Ask mode
  [--resume <session_id>] \          # Resume previous conversation
  [--model <model_id>] \             # Specify AI model
  --workspace /path/to/project \     # Project directory
  -- "<user prompt>"                 # The user's message
```

### 2. JSON Event Stream

The `agent` CLI writes structured JSON events to stdout, one per line. cc-connect reads these events in real-time and translates them into chat messages:

| Event Type | Description | Example |
|------------|-------------|---------|
| `system` | Session initialization | `{"type":"system","session_id":"abc123","model":"claude-sonnet-4"}` |
| `thinking` | Extended reasoning | `{"type":"thinking","subtype":"delta","text":"Let me analyze..."}` |
| `assistant` | Response text | `{"type":"assistant","message":{"content":[{"type":"text","text":"Here is..."}]}}` |
| `tool_call` | Tool invocation | `{"type":"tool_call","subtype":"started","tool_call":{"shellToolCall":{...}}}` |
| `interaction_query` | Permission request | `{"type":"interaction_query","subtype":"request","query_type":"shellRequestQuery",...}` |
| `result` | Final result | `{"type":"result","result":"Done.","session_id":"abc123"}` |

### 3. Session Continuity

Multi-turn conversations are handled via session resumption:

1. First message → `agent --print --output-format stream-json --trust -- "user prompt"`
2. Agent responds with a `system` event containing `session_id`
3. cc-connect stores this `session_id`
4. Next message → `agent --print --output-format stream-json --trust --resume <session_id> -- "follow-up prompt"`

The agent CLI maintains conversation history internally, so each resumed session has full context.

### 4. Tool Call Handling

When the agent invokes tools (file edits, shell commands, web fetches), cc-connect extracts tool information and displays it in the chat:

- **Bash** — shell commands (`shellToolCall`)
- **Read** — file reads (`readToolCall`)
- **Edit** — file edits (`editToolCall`)
- **Write** — file writes (`writeToolCall`)
- **Grep** — pattern search (`grepToolCall`)
- **Glob** — file matching (`globToolCall`)
- **WebFetch** — web requests (`webFetchToolCall`)

### 5. Permission Modes

The Cursor Agent CLI supports different trust levels, controlled by CLI flags:

| Mode | CLI Flags | Behavior |
|------|-----------|----------|
| `default` | `--trust` | Trust workspace, ask before tool use |
| `force` | `--trust --force` | Auto-approve all tool calls |
| `plan` | `--trust --mode plan` | Read-only analysis, no edits |
| `ask` | `--trust --mode ask` | Q&A style, read-only |

Switch modes at runtime via `/mode` in chat.

### 6. Session Storage

Cursor stores chat sessions locally at:

```
~/.cursor/chats/<MD5_HASH_OF_WORKSPACE_PATH>/
```

Each session directory contains a `store.db` SQLite database with conversation metadata and message blobs. cc-connect reads this database to list and manage sessions via `/list`, `/switch`, and other commands.

---

## Prerequisites

1. **Cursor Agent CLI** (`agent`) must be installed and available in your `$PATH`.

   Install from: https://docs.cursor.com/agent

2. Verify installation:

   ```bash
   agent --version
   agent models      # List available models
   ```

---

## Configuration

### Basic Setup

```toml
[[projects]]
name = "my-project"

[projects.agent]
type = "cursor"

[projects.agent.options]
work_dir = "/path/to/project"
```

### Full Configuration

```toml
[[projects]]
name = "my-project"

[projects.agent]
type = "cursor"

[projects.agent.options]
work_dir = "/path/to/project"
mode = "force"                       # "default" | "force" | "plan" | "ask"
cmd = "agent"                        # CLI binary name (default: "agent")
model = "claude-sonnet-4-20250514"   # Optional model override

# Optional: API provider management
[[projects.agent.providers]]
name = "anthropic"
api_key = "sk-ant-xxx"

[[projects.platforms]]
type = "feishu"   # or telegram, slack, discord, etc.

[projects.platforms.options]
# ... platform-specific config
```

### Configuration Options

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `work_dir` | string | `"."` | Project working directory |
| `mode` | string | `"default"` | Permission mode |
| `cmd` | string | `"agent"` | CLI binary name |
| `model` | string | *(auto)* | AI model to use |

---

## Chat Commands

Once connected, use these commands in your chat app:

| Command | Description |
|---------|-------------|
| `/mode` | Show available permission modes |
| `/mode force` | Switch to force mode (auto-approve all) |
| `/model` | List available models |
| `/model <name>` | Switch to a specific model |
| `/list` | List all Cursor Agent sessions |
| `/switch <id>` | Switch to a different session |
| `/new [name]` | Start a new session |
| `/current` | Show current session info |
| `/stop` | Stop current execution |

---

## Architecture Details

### Code Structure

```
agent/cursor/
├── cursor.go              # Agent interface: modes, models, providers, session listing
└── session.go             # Session: subprocess management, JSON event parsing
```

### Interfaces Implemented

The Cursor agent implements these core interfaces:

| Interface | Purpose |
|-----------|---------|
| `Agent` | Basic agent contract (Name, StartSession, ListSessions, Stop) |
| `WorkDirSwitcher` | Runtime work directory switching |
| `ModelSwitcher` | Runtime model selection |
| `ModeSwitcher` | Permission mode switching |
| `ProviderSwitcher` | API provider management |
| `SessionDeleter` | Session removal |
| `SessionEnvInjector` | Per-session environment variables |
| `SkillProvider` | Custom skill discovery |
| `AgentDoctorInfo` | CLI binary metadata for diagnostics |

### Why CLI, Not IDE?

cc-connect integrates with the **Cursor Agent CLI** rather than the Cursor IDE because:

1. **Headless operation** — The CLI runs without a GUI, ideal for server or daemon deployment
2. **Programmatic control** — JSON stream output enables precise event parsing
3. **Session management** — `--resume` enables stateful multi-turn conversations
4. **Process isolation** — Each conversation turn runs in its own process
5. **Cross-platform** — Works on any machine with Node.js, no display server needed

---

## Troubleshooting

### "agent" CLI not found

```
cursor: "agent" CLI not found in PATH
```

Install the Cursor Agent CLI from https://docs.cursor.com/agent and ensure the `agent` binary is in your `$PATH`.

### No models returned

If `/model` shows only fallback models, verify the CLI works standalone:

```bash
agent models
```

### Session listing is empty

Sessions are stored per-workspace. Ensure `work_dir` in your config matches the directory you used when chatting with the agent. Sessions are stored at `~/.cursor/chats/<md5_of_work_dir>/`.
