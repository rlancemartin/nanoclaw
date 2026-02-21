# NanoClaw

A personal Claude assistant accessible via WhatsApp. Lightweight, secure alternative to OpenClaw.

OpenClaw became a monstrosity — 4-5 processes, endless config files, no real isolation, impossible to understand. NanoClaw gives you the core functionality without that mess.

---

## Philosophy

**Small enough to understand.** One Node.js process. A handful of source files. No microservices, no message queues, no abstraction layers.

**Security through true isolation.** Agents run in Linux containers. Isolation is at the OS level. Agents can only see what's explicitly mounted. Bash is safe because it runs inside the container, not on your host.

**Built for one user.** This isn't a framework. It's working software for specific needs. It supports what you use; add integrations you actually want.

**Customization = code changes.** No configuration sprawl. The codebase is small enough that modifying code is safe and practical.

**AI-native development.** No installation wizard — Claude Code guides setup. No monitoring dashboard — ask Claude Code. No debugging tools — describe the problem, Claude fixes it. The codebase assumes you have an AI collaborator.

**Skills over features.** Contributors don't add "Telegram support alongside WhatsApp." They contribute a skill like `/add-telegram` that transforms the codebase. Users fork, run skills, and end up with clean code — not a bloated system.

---

## Architecture

### Three Actors

- **User** — wants things done. Steers the agent via WhatsApp. Adds capabilities via Claude Code.
- **Agent** — does things. Receives context and tools, performs actions, returns results. Runs in a container.
- **Claude Code** — the user's development tool. Modifies the orchestration layer: adding MCP tools, changing behavior, debugging. The user never edits the orchestrator directly.

### The Stack

1. **Communication plane** — carries user intent to the agent and delivers responses back. NanoClaw polls WhatsApp for inbound messages and routes outbound replies. The agent doesn't know or care about the transport.

2. **Orchestration layer (NanoClaw)** — everything that happens *around* the agent:
   - Defines the MCP action space
   - Initializes the agent with a system prompt
   - Manages memory (per-group CLAUDE.md files)
   - Runs scheduled tasks
   - Spawns containers

3. **Agent (Claude Agent SDK in container)** — receives context and tools, acts through MCP. Containerization is what makes it safe to give the agent real capabilities — bash, web, APIs — all sandboxed.

New capabilities: user tells Claude Code → Claude Code wires up an MCP tool → next agent run sees it.

### Two MCP Planes

All agent capabilities are MCP tools, but they serve two distinct roles:

**Communication plane** — how the user steers the agent. WhatsApp, Telegram, email. The channel itself is an MCP tool (`send_message`) the agent calls to reply.

**Action plane** — what the agent does in the world. Twitter, browser, shell, filesystem. Capabilities the agent uses to fulfill requests.

```
User ──[channel]──> Router ──> Agent Container
                                    │
                                    ├── MCP: send_message (reply)
                                    ├── MCP: schedule_task (reminders)
                                    ├── MCP: X (tweets)
                                    ├── MCP: Browser (pages)
                                    └── MCP: Filesystem (memory)
```

A channel like WhatsApp appears on both sides — it routes inbound messages *and* provides the MCP tool for replies — but these are separate concerns: input routing (Node.js) vs. output action (MCP in container).

### Always-On Requirement

The orchestration layer must be persistent — it polls for messages, routes them, and spawns agents. This means dedicated hardware (Mac mini, home server). A laptop works but sleeps. This is an artifact of self-hosting, not a fundamental constraint. See [Addendum: Deployed Agent Model](#addendum-deployed-agent-model).

---

## How It Works

### Message Routing
- Router listens to WhatsApp, processes only registered groups
- Trigger: `@Andy` prefix (case insensitive), configurable via `ASSISTANT_NAME`
- Unregistered groups are ignored

### Memory
- **Per-group**: each group has a folder with its own `CLAUDE.md`
- **Global**: root `CLAUDE.md` readable by all, writable only from main (self-chat)
- Agent runs in the group's folder, inherits both CLAUDE.md files automatically

### Sessions
- Each group maintains a conversation session via Claude Agent SDK
- Sessions auto-compact when context grows long

### Container Isolation
- Every agent invocation spawns a container with mounted directories
- Agents can only see mounted paths
- Browser automation via agent-browser with Chromium in the container

### Scheduled Tasks
- Users ask Claude to schedule recurring or one-time tasks
- Tasks run as full agents in the creating group's context
- Can send messages to their group or complete silently
- Schedule types: cron, intervals, or one-time (ISO timestamp)
- Runs logged to SQLite with duration and result

### Groups
- Registered in SQLite via main channel or IPC
- Each gets a folder under `groups/`
- Can have additional directories mounted via `containerConfig`
- **Main channel** (self-chat) is admin: writes global memory, manages all tasks and groups

---

## Integration Points

| Integration | How |
|-------------|-----|
| **WhatsApp** | baileys library, messages in SQLite, QR auth |
| **Scheduler** | `nanoclaw` MCP server in container — `schedule_task`, `list_tasks`, `pause_task`, `resume_task`, `cancel_task`, `send_message` |
| **Web** | Built-in WebSearch and WebFetch (Claude Agent SDK) |
| **Browser** | agent-browser CLI + Chromium in container — snapshots, screenshots, PDFs, video, auth persistence |

---

## Setup & Customization

Clone the repo. Run Claude Code. It handles everything.

| Skill | Purpose |
|-------|---------|
| `/setup` | Install deps, authenticate WhatsApp, configure scheduler, start services |
| `/customize` | Add channels, integrations, change behavior |
| `/debug` | Troubleshoot containers, logs, auth |

Runs on local Mac via launchd. Single Node.js process.

---

## RFS (Request for Skills)

Skills we'd love contributors to build:

**Communication channels:**
`/add-telegram` · `/add-slack` · `/add-discord` · `/add-sms` · `/convert-to-telegram`

**Container runtime:**
`/convert-to-apple-container` — switch Docker to Apple Container (macOS-only)

**Platform support:**
`/setup-linux` · `/setup-windows` (WSL2 + Docker)

---

## Personal Configuration (Reference)

| Setting | Value |
|---------|-------|
| Trigger | `@Andy` (case insensitive) |
| Response prefix | `Andy:` |
| Persona | Default Claude |
| Main channel | Self-chat (WhatsApp) |

---

## Addendum: Deployed Agent Model

Today, NanoClaw is a persistent Node.js process that polls for messages, manages containers, and runs the agent — all on your hardware. This works, but requires always-on hardware.

If the agent SDK were a hosted service with built-in persistence and container isolation, the architecture shifts. The core design carries forward; what changes is where the agent runs.

### What Stays

1. **Forkable orchestrator** — your fork *is* your agent's configuration. No dashboards, no admin UIs — just code.
2. **Actions via MCP** — adding a capability means adding an MCP server. The orchestrator defines the action space.
3. **Claude Code as administrator** — "give my agent access to Twitter" → Claude Code wires up the MCP tool.
4. **Two planes** — communication tools carry intent, action tools do work. Both MCP.
5. **Memory writes back to orchestration** — CLAUDE.md files in the repo. Version-controlled, survives deployments.

### What Changes

- **Agent is deployed** — SDK provider handles persistence, execution, isolation. No Mac mini, no polling loop.
- **Orchestration becomes declarative** — define the action space, system prompt, memory, cron schedule. Deploy. The SDK handles the rest.
- **Communication plane moves to the SDK** — deployed SDK receives webhooks directly. Orchestrator just declares which channels to listen on.

### The Shift

NanoClaw goes from "always-on process that runs agents" to "declarative configuration that defines and deploys them." Same orchestrator, same MCP tools, same Claude Code workflow, same memory model — agent runs somewhere else.
