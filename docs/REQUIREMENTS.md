# NanoClaw Requirements

Original requirements and design decisions from the project creator.

---

## Why This Exists

This is a lightweight, secure alternative to OpenClaw (formerly ClawBot). That project became a monstrosity - 4-5 different processes running different gateways, endless configuration files, endless integrations. It's a security nightmare where agents don't run in isolated processes; there's all kinds of leaky workarounds trying to prevent them from accessing parts of the system they shouldn't. It's impossible for anyone to realistically understand the whole codebase. When you run it you're kind of just yoloing it.

NanoClaw gives you the core functionality without that mess.

---

## Philosophy

### Small Enough to Understand

The entire codebase should be something you can read and understand. One Node.js process. A handful of source files. No microservices, no message queues, no abstraction layers.

### Security Through True Isolation

Instead of application-level permission systems trying to prevent agents from accessing things, agents run in actual Linux containers. The isolation is at the OS level. Agents can only see what's explicitly mounted. Bash access is safe because commands run inside the container, not on your Mac.

### Built for One User

This isn't a framework or a platform. It's working software for my specific needs. I use WhatsApp and Email, so it supports WhatsApp and Email. I don't use Telegram, so it doesn't support Telegram. I add the integrations I actually want, not every possible integration.

### Customization = Code Changes

No configuration sprawl. If you want different behavior, modify the code. The codebase is small enough that this is safe and practical. Very minimal things like the trigger word are in config. Everything else - just change the code to do what you want.

### AI-Native Development

I don't need an installation wizard - Claude Code guides the setup. I don't need a monitoring dashboard - I ask Claude Code what's happening. I don't need elaborate logging UIs - I ask Claude to read the logs. I don't need debugging tools - I describe the problem and Claude fixes it.

The codebase assumes you have an AI collaborator. It doesn't need to be excessively self-documenting or self-debugging because Claude is always there.

### Skills Over Features

When people contribute, they shouldn't add "Telegram support alongside WhatsApp." They should contribute a skill like `/add-telegram` that transforms the codebase. Users fork the repo, run skills to customize, and end up with clean code that does exactly what they need - not a bloated system trying to support everyone's use case simultaneously.

---

## RFS (Request for Skills)

Skills we'd love contributors to build:

### Communication Channels
Skills to add or switch to different messaging platforms:
- `/add-telegram` - Add Telegram as an input channel
- `/add-slack` - Add Slack as an input channel
- `/add-discord` - Add Discord as an input channel
- `/add-sms` - Add SMS via Twilio or similar
- `/convert-to-telegram` - Replace WhatsApp with Telegram entirely

### Container Runtime
The project uses Docker by default (cross-platform). For macOS users who prefer Apple Container:
- `/convert-to-apple-container` - Switch from Docker to Apple Container (macOS-only)

### Platform Support
- `/setup-linux` - Make the full setup work on Linux (depends on Docker conversion)
- `/setup-windows` - Windows support via WSL2 + Docker

---

## Vision

A personal Claude assistant accessible via WhatsApp, with minimal custom code.

**Core components:**
- **Claude Agent SDK** as the core agent
- **Containers** for isolated agent execution (Linux VMs)
- **WhatsApp** as the primary I/O channel
- **Persistent memory** per conversation and globally
- **Scheduled tasks** that run Claude and can message back
- **Web access** for search and browsing
- **Browser automation** via agent-browser

**Implementation approach:**
- Use existing tools (WhatsApp connector, Claude Agent SDK, MCP servers)
- Minimal glue code
- File-based systems where possible (CLAUDE.md for memory, folders for groups)

---

## Architecture Decisions

### Two MCP Planes

The agent's capabilities are all MCP tools, but they serve two distinct roles:

**Communication plane** — How the user steers the agent. WhatsApp, Telegram, email — these are channels that carry user intent to the agent and deliver responses back. The channel itself is an MCP tool (`send_message`) that the agent calls to reply.

**Action plane** — What the agent does in the world. Read a Twitter timeline, post a tweet, browse a URL, run a shell command. These are the capabilities the agent uses to fulfill requests.

Both planes are MCP. The difference is purpose: communication tools move messages between the user and agent, action tools interact with external systems on the user's behalf. Both are customizable by forking the repo and modifying or adding MCP servers.

```
User ──[channel]──> Router ──> Agent Container
                                    │
                                    ├── MCP: send_message (reply to user)
                                    ├── MCP: schedule_task (set reminders)
                                    ├── MCP: X (read/post tweets)
                                    ├── MCP: Browser (fetch pages)
                                    └── MCP: Filesystem (memory)
```

A channel like WhatsApp appears on both sides — it routes inbound messages to the agent *and* provides the MCP tool the agent calls to send replies — but these are separate concerns: input routing (Node.js process) vs. output action (MCP tool in container).

### The Actors

There are three actors:

- **User** — wants things done. Steers the agent via a communication channel (WhatsApp). Adds capabilities to the agent via Claude Code.
- **Agent** — does things. Receives context and tools, performs actions, returns results. Runs inside a container so it can act safely.
- **Claude Code** — the user's development tool. Modifies the orchestration layer — adding MCP tools, changing behavior, debugging. The user doesn't edit config files; they tell Claude Code what they want and it changes the code.

### The Stack

Three concerns sit between these actors:

1. **Communication plane (WhatsApp)** — carries user intent to the agent and delivers responses back. NanoClaw polls this for inbound messages and routes outbound replies through it. The agent doesn't know or care about the transport.

2. **Orchestration layer (NanoClaw)** — everything that happens *around* the agent. It:
   - Defines the MCP action space (what the agent can do)
   - Initializes the agent with a system prompt
   - Manages memory (per-group CLAUDE.md files)
   - Runs cron / scheduled tasks
   - Captures agent logs
   - Spawns the container the agent runs in

3. **Agent (Claude Agent SDK in container)** — receives context and a set of tools, performs actions, returns results. The SDK is the harness that runs the agent loop. The agent lives in an isolated container and acts through the MCP tools the orchestration layer provides. Containerization is what makes it safe to give the agent real capabilities — it can run bash, browse the web, call APIs, but only within its sandbox.

The user adds new capabilities by telling Claude Code to wire up a new MCP tool. Claude Code modifies the orchestration layer. Next time the agent runs, it sees the new tool. The user never touches the orchestration layer directly.

### Always-On Requirement

The orchestration layer must be persistent — it polls for messages, routes them, and spawns agents. This is why it runs on dedicated hardware (Mac mini, home server, etc.). A laptop works but sleeps. This is an artifact of the agent runtime being self-hosted, not a fundamental requirement of the design. See [Addendum: Deployed Agent Model](#addendum-deployed-agent-model) for how this constraint goes away.

### Message Routing
- A router listens to WhatsApp and routes messages based on configuration
- Only messages from registered groups are processed
- Trigger: `@Andy` prefix (case insensitive), configurable via `ASSISTANT_NAME` env var
- Unregistered groups are ignored completely

### Memory System
- **Per-group memory**: Each group has a folder with its own `CLAUDE.md`
- **Global memory**: Root `CLAUDE.md` is read by all groups, but only writable from "main" (self-chat)
- **Files**: Groups can create/read files in their folder and reference them
- Agent runs in the group's folder, automatically inherits both CLAUDE.md files

### Session Management
- Each group maintains a conversation session (via Claude Agent SDK)
- Sessions auto-compact when context gets too long, preserving critical information

### Container Isolation
- All agents run inside containers (lightweight Linux VMs)
- Each agent invocation spawns a container with mounted directories
- Containers provide filesystem isolation - agents can only see mounted paths
- Bash access is safe because commands run inside the container, not on the host
- Browser automation via agent-browser with Chromium in the container

### Scheduled Tasks
- Users can ask Claude to schedule recurring or one-time tasks from any group
- Tasks run as full agents in the context of the group that created them
- Tasks have access to all tools including Bash (safe in container)
- Tasks can optionally send messages to their group via `send_message` tool, or complete silently
- Task runs are logged to the database with duration and result
- Schedule types: cron expressions, intervals (ms), or one-time (ISO timestamp)
- From main: can schedule tasks for any group, view/manage all tasks
- From other groups: can only manage that group's tasks

### Group Management
- New groups are added explicitly via the main channel
- Groups are registered in SQLite (via the main channel or IPC `register_group` command)
- Each group gets a dedicated folder under `groups/`
- Groups can have additional directories mounted via `containerConfig`

### Main Channel Privileges
- Main channel is the admin/control group (typically self-chat)
- Can write to global memory (`groups/CLAUDE.md`)
- Can schedule tasks for any group
- Can view and manage tasks from all groups
- Can configure additional directory mounts for any group

---

## Integration Points

### WhatsApp
- Using baileys library for WhatsApp Web connection
- Messages stored in SQLite, polled by router
- QR code authentication during setup

### Scheduler
- Built-in scheduler runs on the host, spawns containers for task execution
- Custom `nanoclaw` MCP server (inside container) provides scheduling tools
- Tools: `schedule_task`, `list_tasks`, `pause_task`, `resume_task`, `cancel_task`, `send_message`
- Tasks stored in SQLite with run history
- Scheduler loop checks for due tasks every minute
- Tasks execute Claude Agent SDK in containerized group context

### Web Access
- Built-in WebSearch and WebFetch tools
- Standard Claude Agent SDK capabilities

### Browser Automation
- agent-browser CLI with Chromium in container
- Snapshot-based interaction with element references (@e1, @e2, etc.)
- Screenshots, PDFs, video recording
- Authentication state persistence

---

## Setup & Customization

### Philosophy
- Minimal configuration files
- Setup and customization done via Claude Code
- Users clone the repo and run Claude Code to configure
- Each user gets a custom setup matching their exact needs

### Skills
- `/setup` - Install dependencies, authenticate WhatsApp, configure scheduler, start services
- `/customize` - General-purpose skill for adding capabilities (new channels like Telegram, new integrations, behavior changes)

### Deployment
- Runs on local Mac via launchd
- Single Node.js process handles everything

---

## Personal Configuration (Reference)

These are the creator's settings, stored here for reference:

- **Trigger**: `@Andy` (case insensitive)
- **Response prefix**: `Andy:`
- **Persona**: Default Claude (no custom personality)
- **Main channel**: Self-chat (messaging yourself in WhatsApp)

---

## Project Name

**NanoClaw** - A reference to Clawdbot (now OpenClaw).

---

## Addendum: Deployed Agent Model

Today, the orchestration layer and agent runtime are co-located on your hardware. NanoClaw is a persistent Node.js process that polls for messages, manages containers, and runs the agent. This works, but it means you need always-on hardware.

If the agent SDK were a hosted, deployed service — always-on by default, with persistence and container isolation built in — the architecture could shift. The core design principles carry forward; what changes is where the agent runs and what the orchestrator is responsible for.

### What Stays

1. **Forkable orchestrator** — the orchestration layer is a repo you fork and customize. Your fork *is* your agent's configuration. No dashboards, no admin UIs — just code.

2. **Actions via MCP** — the agent's capabilities are MCP tools. Adding a capability means adding an MCP server. The orchestrator defines the action space; the agent acts within it.

3. **Claude Code as administrator** — the user configures the orchestrator through Claude Code. "Give my agent access to Twitter" → Claude Code wires up the MCP tool. The user never edits the orchestrator directly.

4. **Communication and action as separate planes in the action space** — both are MCP, but they serve different purposes. Communication tools (WhatsApp, Telegram, email) carry user intent in and responses out. Action tools (Twitter, browser, filesystem) do work in the world. The agent sees both as tools; the orchestrator knows which is which.

5. **Memory writes back to orchestration** — learnings, preferences, and context from each session persist back to the orchestration layer (CLAUDE.md files in the repo). The agent's memory lives in the orchestrator, not in the SDK. This means memory survives across deployments and is version-controlled.

### What Changes

- **Agent is always-on and deployed** — the SDK provider handles persistence, execution, and container isolation. The orchestrator doesn't run the agent; it configures and deploys it. No Mac mini, no always-on process, no polling loop.

- **Orchestration becomes declarative** — instead of a Node.js process managing agent lifecycle, the orchestrator defines: here's the action space (MCP tools), here's the system prompt, here's the memory, here's the cron schedule. Deploy. The deployed SDK handles the rest.

- **Communication plane moves to the SDK** — instead of the orchestrator polling WhatsApp and feeding messages to the agent, the deployed SDK receives webhooks directly. The orchestrator just declares which channels to listen on.

### Summary

Same forkable orchestrator, same MCP action space, same Claude Code administration, same memory model — but the agent runs somewhere else. NanoClaw shifts from "always-on process that runs agents" to "declarative configuration that defines and deploys them."
