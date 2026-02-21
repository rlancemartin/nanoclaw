# NanoClaw Architecture — Layer on Top of the Agent Loop

The Claude Agent SDK gives you an agent that takes a prompt and returns a result. NanoClaw is the ~500 lines that turn that into a service: ears (messaging), a heartbeat (polling), hands (IPC tools), a clock (scheduler), an editable toolset, and memory.

This doc explains each piece, why the SDK doesn't handle it, and how NanoClaw does.

---

## 1. Interaction Mode

**What the SDK gives you:** `query(prompt="...", options={...})` — one prompt in, one result out.

**What's missing:** Where does the prompt come from? Who decides when to wake the agent?

**Philosophy:** The unit of interaction is a *group* — a WhatsApp group chat, a DM, a Telegram channel. Each group is independently registered with its own identity, trigger, and isolated filesystem. The system doesn't try to be a platform; it's a single-user tool that happens to participate in multiple conversations.

**How it works:**

Groups are registered in SQLite with a JID (chat ID), a folder name, and a trigger pattern:

```python
registered_groups = {
    "120363...@g.us": Group(name="Family", folder="family", trigger="@Andy"),
    "1234...@s.whatsapp.net": Group(name="Main", folder="main", trigger=None),
}
```

Two tiers:
- **Main** = your private channel (self-chat). Always listening, no trigger needed. Gets the entire project mounted read-write. Can manage other groups, schedule tasks for them, edit global memory.
- **Every other group** = isolated. Only wakes when someone says `@Andy`. Gets only its own folder + read-only global memory. Can't see or affect other groups.

Messages between triggers accumulate silently in the DB. When a trigger arrives, *all* pending messages are pulled as context — the agent sees the full conversation it missed, not just the trigger message.

```python
# Trigger arrives → pull everything since last agent response
all_pending = db.get_messages_since(group_jid, last_agent_timestamp)
prompt = format_as_xml(all_pending)
```

---

## 2. Heartbeat

**What the SDK gives you:** Nothing between queries. The SDK runs one prompt and returns.

**What's missing:** Something has to keep running, check for new messages, manage container lifecycles, fire scheduled tasks.

**Philosophy:** Polling, not events. Every subsystem is a `while True` loop with a sleep interval. This is deliberate — polling is stateless, restartable, debuggable, and has no race conditions. You can kill the process and restart it, and it picks up exactly where it left off because all state is in SQLite.

**How it works:**

Four independent poll loops in one process:

| Loop | Interval | What it checks |
|------|----------|---------------|
| Message loop | 2s | SQLite for new messages from registered groups |
| IPC watcher | 1s | Filesystem for JSON files written by containers |
| Scheduler | 60s | SQLite for tasks where `next_run <= now` |
| In-container input | 500ms | `/workspace/ipc/input/` for piped follow-up messages |

The WhatsApp client (Baileys library) is event-driven internally, but its only job is dumping messages into SQLite via a callback:

```python
on_message = lambda jid, msg: db.store_message(msg)
```

Everything downstream is poll-based. The message loop reads the DB, not the WebSocket.

**Container lifecycle:** When a container is spawned for a group, it stays alive. Follow-up messages are piped into the running container via IPC files (no new container spawn). After 30 minutes of no output, the host writes a `_close` sentinel file and the container exits gracefully. Next message spawns a fresh container.

---

## 3. Actions — IPC via Filesystem

**What the SDK gives you:** Tools for the agent to use (Bash, file I/O, web search). But these are all *local to the container*.

**What's missing:** The agent needs to affect the outside world — send messages back to WhatsApp, schedule recurring jobs, register new groups. The container is sandboxed; it can't call WhatsApp directly.

**Philosophy:** Filesystem as message bus. The agent writes JSON files to a shared directory. The host polls, reads, acts, deletes. No HTTP, no sockets, no RPC. Atomic writes (write to `.tmp`, then rename) prevent partial reads. Failed files move to `errors/` for debugging.

**How it works:**

The agent has an MCP server (`ipc-mcp-stdio.ts`) that exposes tools. Each tool writes a JSON file:

```python
# Agent calls send_message("Hello!")
# MCP tool writes:
write_ipc("/workspace/ipc/messages/", {
    "type": "message",
    "chatJid": "120363...@g.us",
    "text": "Hello!"
})

# Host-side (ipc.ts) polls, reads the file, calls:
channel.send_message(chat_jid, text)
# Then deletes the file.
```

Available tools:

| Tool | IPC type | Host handler |
|------|----------|-------------|
| `send_message` | `message` | Route text to the right channel (WhatsApp, Telegram, etc.) |
| `schedule_task` | `schedule_task` | Insert into SQLite tasks table, compute `next_run` |
| `list_tasks` | *(reads file)* | Host writes `current_tasks.json` before container starts |
| `pause_task` | `pause_task` | Update task status in DB |
| `resume_task` | `resume_task` | Update task status in DB |
| `cancel_task` | `cancel_task` | Delete task from DB |
| `register_group` | `register_group` | Add group to SQLite, create folder (main only) |

Authorization is enforced by directory path — each group writes to its own `data/ipc/{folder}/` namespace. The host knows which group wrote the file by which directory it's in. Non-main groups can only `send_message` to their own JID and can only manage their own tasks.

**Adding a new action** = add a tool to the MCP server + add a case to the host's `processTaskIpc` switch statement. Two files, two changes.

---

## 4. Claude Code as Handler

**What the SDK gives you:** The full Claude Code agent — file I/O, web search, Bash, subagents, memory, hooks.

**Why use it instead of building a custom agent?** Because the harness matters more than people think. Claude Code handles: tool orchestration, error recovery, context management, permission boundaries, session persistence, auto-compaction, subagent teams, and a battle-tested system prompt. Reimplementing any of these is months of work. Reimplementing all of them well is what the Claude Code team does full-time.

**Philosophy:** Don't build an agent. Wrap the best one that exists. NanoClaw's only job is getting prompts *to* Claude Code and results *from* Claude Code. Everything between the `query()` call and the result is Claude Code's problem.

**How it works:**

The container runs one file (`agent-runner/src/index.ts`). Its job:

1. Read input from stdin (prompt, session ID, group context)
2. Call `query()` with the right options
3. Emit results back to stdout (wrapped in sentinel markers for parsing)
4. Poll for follow-up messages via IPC
5. Loop until `_close` sentinel

```python
for message in sdk.query(
    prompt=user_messages,
    options={
        resume=session_id,              # pick up previous conversation
        cwd="/workspace/group",         # agent works in group's folder
        system_prompt=global_claude_md, # from groups/global/CLAUDE.md
        allowed_tools=[
            "Bash", "Read", "Write", "Edit",
            "WebSearch", "WebFetch",
            "Task",          # subagents
            "mcp__nanoclaw__*"  # IPC tools
        ],
        permission_mode="bypass",  # safe because container is sandboxed
        mcp_servers={"nanoclaw": ipc_mcp_server},
        hooks={
            "PreCompact": archive_transcript,  # save before context compression
            "PreToolUse": sanitize_bash,        # strip API keys from Bash commands
        }
    }
):
    if message.type == "result":
        emit(message.result)  # host captures via stdout, sends to WhatsApp
```

The `permissionMode: "bypassPermissions"` flag looks scary but is safe here because the agent runs inside a container. It can only see mounted directories. Bash commands run inside the sandbox. The container IS the permission boundary.

---

## 5. Editable Action Space

**What the SDK gives you:** A fixed set of tools (Bash, Read, Write, etc.) plus MCP server support.

**What's missing:** A way for the *user* to change what the agent can do without writing code. And a way for *contributors* to add capabilities without bloating the codebase.

**Philosophy:** Three layers of customization, from easiest to most powerful. Most changes are Layer 1 (edit a text file). Some need Layer 2 (add an MCP tool). Almost none need Layer 3 (fork + skill).

**Layer 1: CLAUDE.md files** — change behavior with natural language.

```
groups/global/CLAUDE.md   → read by all groups (read-only mount)
groups/{name}/CLAUDE.md   → per-group personality, rules, context
```

This is the most common edit. "Be more concise." "Always respond in Spanish." "You have access to my Obsidian vault at /workspace/extra/vault." The agent reads these as system prompt. No restart needed — next container picks up the changes.

**Layer 2: MCP tools** — give the agent new structured actions.

The MCP server (`ipc-mcp-stdio.ts`) is ~280 lines. Each tool is ~20 lines: a schema, a handler that writes a JSON file. The host-side handler (`ipc.ts`) is a switch statement.

To add "post to Slack":

```python
# In MCP server:
@tool("post_to_slack")
def post_to_slack(channel: str, text: str):
    write_ipc(TASKS_DIR, {"type": "slack_post", "channel": channel, "text": text})

# In host ipc.ts:
case "slack_post":
    await slack.postMessage(data.channel, data.text)
```

**Layer 3: Skills** — contributor-friendly capability packages.

This is the fork model. Instead of adding Telegram support *to the codebase*, a contributor writes a skill file (`.claude/skills/add-telegram/SKILL.md`) that teaches Claude Code how to *transform your fork*. You run `/add-telegram`, Claude Code modifies your code, and you end up with clean code that does exactly what you need.

Skills can:
- Add new source files (e.g., `src/channels/telegram.ts`)
- Three-way merge changes into existing files (e.g., add Telegram to `index.ts`)
- Install dependencies
- Walk through interactive setup (API keys, configuration)

The result is NOT a plugin system. After the skill runs, it's just code in your repo. You own it, you can read it, you can modify it. No runtime plugin loading, no abstraction layers.

**Layer 4: Additional mounts** — expose external data to the agent.

```python
# In group registration:
Group(
    name="Work",
    folder="work",
    container_config={
        "additional_mounts": [
            {"host_path": "~/projects/myapp", "readonly": True}
        ]
    }
)
```

This mounts `~/projects/myapp` at `/workspace/extra/myapp` inside the container. If that directory has a `CLAUDE.md`, it's automatically loaded as additional context. Validated against an allowlist stored outside the project root (so containers can't tamper with it).

---

## 6. Memory

**What the SDK gives you:** Session persistence (`resume` parameter), auto-memory (learns preferences), context compaction (compresses old messages when context gets long).

**What's missing:** Nothing, really. The SDK handles memory. NanoClaw just needs to store session IDs and give the agent a persistent filesystem.

**Philosophy:** Don't build a memory system. The agent already has one — it's called the filesystem. Let it write notes, save files, organize however it wants. The only infrastructure NanoClaw adds is (a) persisting the session ID so the agent can resume, and (b) archiving transcripts before compaction so they're searchable.

**How it works:**

**A. Session resumption** — one session ID per group, stored in SQLite.

```python
# Before running agent:
session_id = db.get_session(group_folder)  # might be None for first run

# After agent finishes:
db.set_session(group_folder, output.new_session_id)
```

When the agent resumes, it has its full conversation history — every message, every tool call, every result. This is Claude Code's native session system.

**B. Persistent filesystem** — the group's folder survives container restarts.

```
groups/{name}/
  CLAUDE.md           → agent reads this (and can edit its own instructions)
  conversations/      → archived transcripts
  notes.md            → anything the agent decides to write
  data/               → whatever it needs
```

The agent can create files, read them back later, organize its own knowledge. No schema imposed.

**C. Transcript archival** — a `PreCompact` hook fires before Claude Code compresses context:

```python
def pre_compact_hook(transcript_path, session_id):
    messages = parse_jsonl(transcript_path)
    summary = get_session_summary(session_id)  # from sessions-index.json
    filename = f"{date}-{summary}.md"
    write(f"/workspace/group/conversations/{filename}", format_as_markdown(messages))
```

This means the agent can `grep` or `read` its own past conversations — even ones that have been compacted out of the active context window.

**D. Auto-memory** — Claude Code's built-in preference learning, enabled by default. Learns things like "user prefers short responses" across sessions.

No vector DB. No RAG pipeline. No embeddings. The agent remembers by resuming its session, reading files it wrote, and Claude Code's native memory.

---

## Adding Output Channels

The host has a `Channel` interface:

```python
class Channel:
    name: str
    def connect() -> None
    def send_message(jid: str, text: str) -> None
    def is_connected() -> bool
    def owns_jid(jid: str) -> bool
    def disconnect() -> None
```

`findChannel(channels, jid)` picks the right channel based on which one owns that JID. The agent doesn't know or care which channel its messages go through — it calls `send_message` with a JID, the host routes it.

To add a channel:
1. Implement the `Channel` interface (see `src/channels/whatsapp.ts` as reference)
2. Push it to `channels[]` in `main()`
3. Done — message routing, IPC, scheduling all work automatically

Or run `/add-telegram` and Claude Code does this for you via the skills system.

The agent can also bypass the channel system entirely. It has Bash. It can `curl` an API, run a Python script, use any CLI tool inside the container. The browser skill works this way — it's a Node script the agent runs via Bash, no channel integration needed.

---

## The Full Picture

```
┌──────────────────────────────────────────────────────────┐
│                   Messaging Channels                      │
│              WhatsApp / Telegram / Email                   │
│              (each implements Channel)                     │
└──────────────────┬───────────────────────────────────────┘
                   │ on_message → db.store()
                   │
┌──────────────────▼───────────────────────────────────────┐
│                NanoClaw Host Process                      │
│                                                          │
│  ┌─────────────┐  ┌───────────┐  ┌───────────────────┐  │
│  │ Message Loop │  │ Scheduler │  │   IPC Watcher     │  │
│  │ (poll 2s)   │  │ (poll 60s)│  │   (poll 1s)       │  │
│  │             │  │           │  │                   │  │
│  │ check DB    │  │ check DB  │  │ read JSON files   │  │
│  │ for new msgs│  │ for due   │  │ from containers   │  │
│  │ check trigger│  │ tasks     │  │ → send messages   │  │
│  │ enqueue     │  │ enqueue   │  │ → create tasks    │  │
│  └──────┬──────┘  └─────┬─────┘  │ → register groups │  │
│         │               │        └───────────────────┘  │
│         └───────┬───────┘                                │
│                 ▼                                         │
│          ┌─────────────┐                                 │
│          │ Group Queue  │  max 5 concurrent containers    │
│          └──────┬──────┘                                 │
│                 │                                         │
│  SQLite: messages, sessions, tasks, groups, state         │
└─────────────────┼────────────────────────────────────────┘
                  │ docker run -i (stdin: prompt + secrets)
                  │
┌─────────────────▼────────────────────────────────────────┐
│                Container (per group)                      │
│                                                          │
│  agent-runner: read stdin → call SDK → emit results      │
│                                                          │
│  Claude Code SDK query()                                  │
│    ├── Built-in: Bash, Read, Write, WebSearch, ...       │
│    ├── MCP tools: send_message, schedule_task, ...       │
│    ├── Skills: agent-browser, custom skills              │
│    ├── CLAUDE.md: global + per-group instructions        │
│    └── Hooks: archive transcripts, sanitize Bash         │
│                                                          │
│  Mounts:                                                 │
│    /workspace/group/   (group folder, read-write)        │
│    /workspace/global/  (shared memory, read-only)        │
│    /workspace/extra/*  (additional dirs, per config)     │
│    /workspace/ipc/     (JSON file IPC with host)         │
│    /home/node/.claude/ (session data, skills)            │
│                                                          │
│  stdout → host parses results → routes to channel        │
└──────────────────────────────────────────────────────────┘
```

The host is a router, scheduler, and IPC relay. The container is the agent. Everything between them is a JSON file on disk or a row in SQLite.

The SDK is the brain. The layer is the body.
