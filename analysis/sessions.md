# Session Architecture — `gateway/session.py`

## Core Data Model

Three layered dataclasses, each with a distinct responsibility:

```
SessionSource       → WHERE the message came from
SessionContext      → WHAT goes into the system prompt
SessionEntry        → STATE persisted across turns and restarts
```

### SessionSource
The input layer. Holds all origin metadata: `platform`, `chat_id`, `user_id`, `thread_id`, `chat_type`, `guild_id`, etc. This is the raw identity of an incoming message before any processing.

### SessionContext
Built from `SessionSource` + `GatewayConfig`. Injected into the agent's system prompt as a `## Current Session Context` block. Contains:
- Platform source + user identity (PII-redacted on WhatsApp/Signal/Telegram)
- Platform-specific behavioral notes (Slack, Discord, iMessage, Yuanbao)
- Connected platforms list
- Home channels — named delivery targets for cron/scheduled tasks
- Delivery options: `"origin"`, `"local"`, `"<platform>"`

### SessionEntry
The persistent state record. Holds:
- `session_key` → `session_id` pointer
- Token counts (input, output, cache read/write, total, estimated cost)
- State flags: `suspended`, `resume_pending`, `expiry_finalized`, `is_fresh_reset`, `was_auto_reset`
- Reset metadata: `auto_reset_reason`, `reset_had_activity`
- Timestamps: `created_at`, `updated_at`

---

## Session Identity

`build_session_key(source)` produces a deterministic string like:

```
agent:main:telegram:dm:<chat_id>
agent:main:discord:group:<chat_id>:<user_id>
agent:main:slack:channel:<chat_id>:<thread_id>
```

Rules:
- **DMs** → isolated per `chat_id`
- **Groups** → isolated per `user_id` by default (`group_sessions_per_user=True`)
- **Threads** → shared across all participants by default (`thread_sessions_per_user=False`)

`session_key` is **permanent** — it never changes for a given chat/user.
`session_id` is **mutable** — it changes on reset, `/new`, or idle expiry.

---

## Storage — Two-Layer Dual-Write

```
SessionStore
├── sessions.json              ← session_key → session_id pointer + lightweight metadata
└── SessionDB (SQLite)         ← full message transcripts (role, content, tool calls, reasoning)
    └── <session_id>.jsonl     ← parallel JSONL write (live backup, legacy compat)
```

### sessions.json
- In-memory dict (`self._entries`) loaded once on first access, written back on every mutation
- Atomic write: `tempfile.mkstemp` + `atomic_replace` — crash-safe
- Stores: `session_key`, `session_id`, token counts, state flags, timestamps
- Purpose: fast lookup of "which transcript does this chat point to right now"

### SQLite (SessionDB)
- Stores the actual message-by-message transcript
- `session_id` is the foreign key tying rows to a session
- Written via `append_to_transcript()` on every turn
- Rewritten atomically via `rewrite_transcript()` for `/retry`, `/undo`, `/compress`

### JSONL (`<session_id>.jsonl`)
- Written in parallel with every SQLite write — live backup
- On `load_transcript()`, whichever source (SQLite vs JSONL) has **more messages** wins
- Handles sessions that predate the SQLite layer gracefully

---

## Session Lifecycle

```
get_or_create_session(source)
├── build_session_key(source)       → stable lookup key
├── Check suspended?                → force new session (from /stop or stuck-loop escalation)
├── Check resume_pending?           → return same session_id (restart recovery, transcript intact)
├── Check _should_reset()?          → idle timeout or daily reset
│       └── if reset: end old session in DB, generate new session_id
└── Return existing or new SessionEntry
```

Reset policies (per platform + session type):
- `none` — never reset
- `idle` — reset after N idle minutes
- `daily` — reset at a fixed hour each day
- `both` — idle + daily

### Session ID rotation on context compression

Context compression is a third path that rotates `session_id` outside of the normal reset flow. When the agent's context exceeds the compression threshold, `_compress_context` closes the current SQLite session with reason `"compression"`, generates a new `session_id`, and creates a fresh SQLite row with `parent_session_id` pointing to the old one — preserving lineage.

The agent owns this rotation entirely. The gateway's `sessions.json` pointer is updated lazily — the gateway reads `agent.session_id` after each `run_conversation` call and syncs if it changed. This means there is a window where `sessions.json` maps the session key to the pre-compression session ID. If the gateway reads the stale pointer before syncing (e.g. on a concurrent request), it loads the pre-compression transcript from SQLite and passes the full uncompressed history back to the agent — exactly what compression was trying to shrink. At worst this triggers one redundant compression pass; no data is lost since the old SQLite row is preserved intact.

---

## State Flags on SessionEntry

| Flag | Set by | Meaning |
|------|--------|---------|
| `suspended` | `/stop` or stuck-loop counter | Hard wipe — force new session on next access |
| `resume_pending` | Gateway restart/drain timeout | Soft recovery — preserve transcript, auto-resume |
| `expiry_finalized` | Background expiry watcher | Memory flush hooks already ran — don't re-run |
| `is_fresh_reset` | `reset_session()` | First turn of a manual `/new` or `/reset` |
| `was_auto_reset` | `get_or_create_session()` | Session expired and was silently reset |

---

## `resume_pending` — Gateway Restart Recovery

### Why the Gateway Shuts Down

- **Intentional:** `systemd` restart for a new deployment, operator `hermes restart`, graceful SIGTERM on server reboot or container stop
- **Unintentional:** uncaught exception in non-turn code, OOM kill (SIGKILL, no drain runs), `TimeoutStopSec` expiry from systemd

### The Drain

When the gateway receives SIGTERM it stops accepting new messages and waits for all currently-running turns to finish naturally — this is the drain:

```
session A: LLM streaming a response
session B: mid tool call (running bash)
session C: waiting on HTTP
        ↓
_drain_active_agents(timeout=restart_drain_timeout)
    → watches _running_agents shrink as turns complete
    → if all finish in time → clean shutdown
    → if timeout expires → force-interrupt remaining agents
```

`_running_agents` is the live set of sessions mid-turn. Drain polls it until empty or timeout.

### What Happens on Drain Timeout

Some turns can't finish fast — slow LLM provider, long-running bash tool, hung network call. After `restart_drain_timeout` seconds the gateway stops waiting. **Before** interrupting:

```python
for session_key, agent in _running_agents.items():
    session_store.mark_resume_pending(session_key, reason="restart_timeout")
```

Then agents are force-interrupted (async tasks cancelled), the process exits cleanly under its own control rather than waiting for systemd SIGKILL.

### The Flag Lifecycle

```
drain timeout
        ↓
mark_resume_pending(session_key)
    → entry.resume_pending = True
    → entry.resume_reason = "restart_timeout"
    → persisted to sessions.json  (survives the restart)
        ↓
gateway restarts, new message arrives on same session_key
        ↓
get_or_create_session()
    → sees resume_pending=True
    → returns existing session_id unchanged  (bypasses _should_reset())
    → transcript intact
        ↓
gateway prepends system note to user message:
    "[System note: Your previous turn was interrupted by a gateway restart.
     The conversation history is intact. If it contains unfinished tool
     result(s), process them first, then address the user's new message.]"
        ↓
run_conversation() completes successfully
        ↓
clear_resume_pending(session_key)
    → resume_pending = False
    → next turn is normal
```

`suspended=True` always overrides — a hard-stopped session is never marked resumable.

A parallel case — `_has_fresh_tool_tail` — fires when the transcript ends with `role: tool` (mid-turn kill, no `resume_pending` flag set). Same system note, same recovery path. Both are gated on a freshness window — if the last transcript timestamp is too old, the note is suppressed (a confusing interruption notice hours later is worse than silence).

### Without `resume_pending`

`get_or_create_session()` would fall through to the normal `_should_reset()` check. The interrupted session looks **idle** from the store's perspective — `updated_at` stopped the moment the turn was killed. Two bad outcomes depending on reset policy:

- **`idle` policy:** if enough time passed during the restart, the session crosses the idle threshold → new `session_id`, full context wipe. User's next message starts a blank session with no explanation.
- **`none` policy:** session survives but transcript ends mid-turn with an unfinished tool result or truncated assistant message. Agent picks up broken state with no recovery instruction — confused or incorrect response, no indication of what happened.

`resume_pending` solves both: locks the `session_id` to bypass the idle check, and delivers recovery instructions so the agent handles the broken transcript correctly.

---

## Session Expiry & Memory Handoff

### The Expiry Watcher (`gateway/run.py`)

A background daemon (`_session_expiry_watcher`) runs every 300s with a 60s initial delay. On each tick it iterates all `SessionEntry` records and checks `_is_session_expired()`. Entries with `expiry_finalized=True` are skipped.

### Full Shutdown Chain

```
_session_expiry_watcher() detects expired session
        ↓
invoke_hook("on_session_finalize")      ← plugin system notified first
        ↓
_cleanup_agent_resources(agent)
  ├── agent.shutdown_memory_provider(session_messages)
  │       ↓
  │   memory_manager.on_session_end(messages)
  │       ↓
  │   provider.on_session_end(messages)  ← each backend flushes/summarizes
  │       ↓
  │   memory_manager.shutdown_all()      ← releases SDK clients, reverse order
  │
  ├── agent.close()                      ← tears down MCP servers, tool subprocesses, HTTP clients
  └── stale async client cleanup
        ↓
entry.expiry_finalized = True            ← persisted to sessions.json (crash-safe)
        ↓
_evict_cached_agent(key)                 ← removed from _agent_cache, GC eligible
```

**Ordering is intentional:**
- Plugin hook fires before agent teardown — external plugins notified while agent is still alive
- `expiry_finalized` set only after clean shutdown — if cleanup crashes, flag stays `False` and watcher retries (up to 3 attempts, then force-sets)
- Eviction last — agent reference stays valid throughout entire cleanup chain

