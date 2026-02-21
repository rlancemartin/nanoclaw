# NanoClaw Architecture — The 6 Core Ideas

A layer on top of the Claude Code agent loop. NanoClaw doesn't build an agent — it wraps one, giving it ears (messaging), a heartbeat (polling), hands (IPC actions), a schedule (cron), an editable toolset (MCP + CLAUDE.md), and a memory (filesystem + sessions).

---

## 1. Interaction Mode — "What App Group"

The unit of interaction is a **group** (WhatsApp group, DM, Telegram chat). Each group is registered with a JID (chat ID), a folder name, and a trigger word.

```
registeredGroups = {
  "120363...@g.us": { name: "Family", folder: "family", trigger: "@Andy" },
  "1234...@s.whatsapp.net": { name: "Main", folder: "main", trigger: null }
}
```

**Main group** = admin channel. Always listening (no trigger needed). Gets full project access.

**Other groups** = isolated. Only wake on `@Andy`. Get only their own folder + read-only global memory.

The trigger check is simple:
```python
if not is_main and not any(TRIGGER_PATTERN.match(m.content) for m in messages):
    skip()  # accumulate messages silently until triggered
```

Messages between triggers are *not lost* — they accumulate in the DB and all get pulled as context when a trigger finally arrives.

---

## 2. Heartbeat — Polling for Messages

There is no webhook, no event-driven architecture. It's a `while True` loop polling every 2 seconds:

```python
while True:
    messages = db.get_new_messages(registered_jids, last_timestamp)
    for group_jid, group_messages in group_by(messages, 'chat_jid'):
        if needs_trigger(group) and no_trigger(group_messages):
            continue
        queue.enqueue(group_jid)  # or pipe to existing container
    sleep(2)
```

The WhatsApp client (Baileys) runs separately and dumps every incoming message into SQLite via a callback. The main loop just reads from the DB.

Same for IPC (1s poll), scheduler (60s poll), and input to the running agent (500ms poll). Everything is poll-based. Simple, debuggable, no race conditions.

---

## 3. Actions — IPC via Filesystem

The agent runs inside a container. It can't call the host directly. Instead, it writes JSON files to a shared directory, and the host polls for them.

**Agent → Host** (the agent has an MCP server with these tools):

| Tool | What it does |
|------|-------------|
| `send_message` | Write `{type: "message", chatJid, text}` → host sends it |
| `schedule_task` | Write `{type: "schedule_task", prompt, cron, ...}` → host creates cron job |
| `list_tasks` | Read `current_tasks.json` (written by host before container starts) |
| `pause_task` / `resume_task` / `cancel_task` | Write IPC file → host updates DB |
| `register_group` | Write IPC file → host registers new group (main only) |

The mechanism is dead simple:
```python
def write_ipc(directory, data):
    path = f"{directory}/{timestamp}-{random}.json"
    write(f"{path}.tmp", json.dumps(data))
    rename(f"{path}.tmp", path)  # atomic
```

Host polls, reads, deletes. If it fails, file goes to `errors/`.

**"Check Twitter daily"** = the agent calls `schedule_task` with:
```json
{
  "prompt": "Check Twitter for mentions and summarize. Use send_message to report.",
  "schedule_type": "cron",
  "schedule_value": "0 9 * * *",
  "context_mode": "isolated"
}
```

The host's scheduler loop polls `getDueTasks()` every 60 seconds, and when it fires, spins up a fresh container with that prompt. The agent inside has all the same tools (Bash, WebSearch, send_message, etc.) so it can do whatever the prompt says.

---

## 4. Claude Code as Handler

The agent IS Claude Code. Not a custom agent framework. The container runs:

```python
from claude_agent_sdk import query

for message in query(
    prompt=user_messages,           # from stdin/IPC
    options={
        resume=session_id,          # resume previous conversation
        systemPrompt=global_prompt, # from CLAUDE.md
        allowedTools=[...],         # Bash, Read, Write, WebSearch, mcp__nanoclaw__*
        permissionMode='bypassPermissions',
        mcpServers={'nanoclaw': ...},  # the IPC MCP server above
        hooks={
            'PreCompact': [archive_hook],     # save transcript before compaction
            'PreToolUse': [sanitize_bash],    # strip secrets from Bash commands
        }
    }
):
    if message.type == 'result':
        emit(message.result)  # host captures, sends to WhatsApp
```

NanoClaw is literally "put Claude Code behind a message queue." The agent gets all standard tools (file I/O, web search, Bash) plus the MCP tools for messaging and scheduling.

---

## 5. Editable Action Space

The agent's capabilities come from three layers, all editable without code changes:

**Layer 1: CLAUDE.md files** (natural language instructions)
```
groups/global/CLAUDE.md   → read by all groups (mounted read-only)
groups/{name}/CLAUDE.md   → per-group personality and rules
```

The agent reads these as part of its system prompt. Change what the agent *knows* and *how it behaves* by editing text files.

**Layer 2: MCP server** (structured tools)
```
container/agent-runner/src/ipc-mcp-stdio.ts → send_message, schedule_task, etc.
```

Add a new tool here and the agent can call it. The host-side handler goes in `src/ipc.ts`.

**Layer 3: Skills** (`.claude/skills/`)
```
container/skills/agent-browser/  → browser automation via Bash
```

Synced to each group's `.claude/skills/` directory before container start. The agent discovers them automatically.

**Layer 4: Additional mounts** (per-group)
```json
{ "containerConfig": { "additionalMounts": ["/path/to/code:/workspace/extra/code"] } }
```

Mount external directories into the container. The agent can read/write them and any CLAUDE.md files in them get loaded automatically.

To add a new capability: edit a CLAUDE.md, add an MCP tool, or mount a directory. No framework code to learn.

---

## 6. Memory

No custom memory system. Three mechanisms, all native:

**A. Session resumption** — Claude Code's built-in session persistence.
```python
sessions = {"family": "session-abc123", "main": "session-def456"}

# On each run:
output = query(prompt=msg, options={resume: sessions[group]})
sessions[group] = output.new_session_id  # save to SQLite
```

One session ID per group, stored in SQLite. The agent picks up where it left off — full conversation history, tool results, everything.

**B. Filesystem** — The agent's working directory persists between runs.
```
groups/{name}/
  CLAUDE.md           → editable personality/instructions
  conversations/      → archived transcripts (pre-compaction hook)
  *.md, *.txt, etc.   → anything the agent writes
```

The conversations/ archive is created by a `PreCompact` hook: before Claude Code compresses old context, the hook dumps the full transcript to a markdown file the agent can search later.

**C. Auto-memory** — Claude Code's built-in preference learning (enabled via `CLAUDE_CODE_DISABLE_AUTO_MEMORY=0`).

That's it. The agent remembers by resuming its session, by reading files it previously wrote, and by Claude Code's native memory. No vector DB, no RAG, no embeddings.

---

## The Layer

```
┌──────────────────────────────────────────────────┐
│                  WhatsApp / Telegram              │
└──────────────────┬───────────────────────────────┘
                   │ messages
┌──────────────────▼───────────────────────────────┐
│              NanoClaw Host Process                │
│                                                  │
│  poll loop ──► trigger check ──► queue ──► spawn │
│  scheduler ──► cron check ──────────────► spawn  │
│  IPC watcher ◄── filesystem polls ◄── containers │
│                                                  │
│  SQLite: messages, sessions, tasks, groups        │
└──────────────────┬───────────────────────────────┘
                   │ docker run -i
┌──────────────────▼───────────────────────────────┐
│              Container (per group)                │
│                                                  │
│  Claude Code SDK ◄── session resume              │
│    ├── Bash, Read, Write, WebSearch, ...         │
│    ├── MCP: send_message, schedule_task, ...     │
│    └── CLAUDE.md (system prompt)                 │
│                                                  │
│  /workspace/group/   (persistent, read-write)    │
│  /workspace/global/  (shared, read-only)         │
│  /workspace/ipc/     (host communication)        │
└──────────────────────────────────────────────────┘
```

The host is a router and scheduler. The container is the agent. Everything between them is JSON files on disk.
