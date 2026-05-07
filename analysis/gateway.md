# Gateway Architecture — `gateway/run.py`

## What the Gateway Is

The gateway is the layer between platform adapters (Telegram, Discord, Slack, iMessage, etc.) and `AIAgent`. It is a long-lived process — one instance serves all sessions across all platforms simultaneously. `AIAgent` has no concept of platforms, sessions, or delivery; the gateway handles all of that.

`run_sync()` is the per-turn sync closure that does all gateway-level setup and then calls `agent.run_conversation()` as its final step. The agent handles the LLM tool loop; the gateway handles everything around it.

### The gateway event loop

The gateway runs on a single asyncio event loop created by `asyncio.run(start_gateway(...))` at process entry (`gateway/run.py:14938`). `start_gateway` is a coroutine that never returns — it starts all platform adapters as async tasks on this loop and runs indefinitely. All platform I/O (Telegram polling, Discord websocket, Slack RTM, HTTP webhook handlers) lives as async tasks on this same loop.

This is the loop that `asyncio.get_running_loop()` returns inside any async context during gateway operation. It is why `model_tools._run_async()` cannot call `run_until_complete()` when invoked from tool handlers during a gateway turn — the loop is already running. Instead it spins a fresh thread with its own loop for the duration of the async tool call.

---

## What the Gateway Adds

### Platform bridging

The gateway holds `self.adapters` — one adapter per connected platform. It receives a raw inbound event (Telegram message, Discord mention, Slack slash command), normalizes it into a `SessionSource`, runs the turn, and delivers the response back via `adapter.send()`. `AIAgent` receives a plain string and returns a plain string — it never touches a platform API.

### Session lifecycle

The gateway calls `get_or_create_session(source)` to resolve the `session_key`, load or create a `SessionEntry`, and determine the active `session_id`. It then loads the full message transcript from SQLite as `agent_history` and passes it as `conversation_history` into `agent.run_conversation()`. The agent sees a flat list of messages — it has no persistence layer of its own.

### Agent cache

The gateway owns the LRU `_agent_cache` (an `OrderedDict` keyed on `session_key + config_signature`). The agent is stateless between turns in the sense that the gateway decides whether to reuse a cached instance or create a fresh one. Per-message state (callbacks, reasoning config, stream handlers) is set on the agent after cache hit — these change every turn and must not be baked into the cached constructor.

### Config resolution per turn

Before each `run_conversation` call the gateway:
- Re-reads `.env` and `config.yaml` (the process is long-lived; credentials may rotate without restart)
- Resolves model and provider for this session (`_resolve_session_agent_runtime`)
- Builds reasoning config, service tier, and toolset lists
- Assembles the ephemeral system prompt by combining: platform context + per-channel context + global ephemeral prompt from config

### Streaming and progress delivery

The gateway creates a `GatewayStreamConsumer` and wires `stream_delta_callback` into the agent. As the LLM streams tokens the consumer edits a live message bubble on the platform side. A separate `progress_queue` accumulates tool-progress events ("⚙️ bash: ...") and a background coroutine edits a second bubble with dedup and throttling. Neither the streaming consumer nor the progress queue exist inside `AIAgent`.

See **Streaming Pipeline** below for the full breakdown.

### Recovery injection

The `resume_pending` / fresh-tool-tail check happens here, before the `agent.run_conversation()` call. When either condition is detected, the gateway prepends a system note to the user message (e.g. "Your previous turn was interrupted by a gateway restart..."). See `sessions.md` for the full lifecycle.

### Slash command routing

`/stop`, `/new`, `/steer`, `/model`, `/compress`, `/reload-skills`, etc. are intercepted at the gateway level before the message reaches the agent. One-shot notes (e.g. from `/reload-skills`) are queued and prepended to the next user message.

### Multimodal attachment

If image paths were buffered for the current session, the gateway wraps the user message as an OpenAI-style multimodal content list (`build_native_content_parts`) before passing it to the agent. The agent itself handles multimodal content but does not know where the images came from.

---

## Turn Flow (gateway layer only)

```
inbound platform event
        ↓
normalize → SessionSource
        ↓
get_or_create_session()   → session_key, session_id, SessionEntry
        ↓
load transcript           → agent_history (from SQLite)
        ↓
resolve model/runtime     → re-read .env + config, per-session overrides
        ↓
build ephemeral prompt    → platform context + channel context + global ephemeral
        ↓
check resume_pending /    → prepend system recovery note to message if needed
      tool-tail
        ↓
wire callbacks            → stream_delta_cb, progress_callback, status_callback,
                            step_callback (plugin hooks), interim_assistant_cb
        ↓
cache lookup              → reuse AIAgent (config_sig match) or create fresh
        ↓
agent.run_conversation()  ← gateway hands off here
        ↓
deliver response          → adapter.send() back to originating platform
        ↓
persist session           → token counts, timestamps to sessions.json
```

---

---

## Streaming Pipeline — `gateway/stream_consumer.py`

### Roles

- **LLM provider** — produces tokens.
- **Agent worker thread** — receives tokens from the provider, fires `stream_delta_callback(token)` synchronously for each one. This is the token producer from the gateway's perspective.
- **`GatewayStreamConsumer.run()`** — an async task on the gateway's event loop. Drains the queue and calls `adapter.edit_message()` toward the platform. This is the platform delivery loop.

```
LLM provider → agent worker thread → queue.put(token) → GatewayStreamConsumer.run() → adapter.edit_message() → Telegram/etc.
  (produces)      (fires callback)   (no backpressure)     (drains + batches)               (slow path)
```

`stream_delta_callback` is the handoff point: the agent calls it synchronously and returns immediately. It does nothing but `queue.put(text)`. The agent never blocks on platform delivery.

### The queue is unbounded

`queue.Queue()` with no `maxsize`. There is no backpressure. Tokens accumulate in the queue regardless of how slow the platform is.

### The live-edit pattern

The delivery loop sends **one platform message** and then edits it in place as tokens arrive:

1. First drain that exceeds `buffer_threshold` (40 chars) or `edit_interval` (1s) → `adapter.send()` → platform message created, `_message_id` captured.
2. Each subsequent tick: drain all queued tokens into `_accumulated`, then `adapter.edit_message(message_id, accumulated + "▉")`. User sees the message growing.
3. On `_DONE` sentinel → final edit without cursor.

The delivery loop fires `await adapter.edit_message(...)` and suspends. During that await the agent thread keeps pushing tokens into the queue. When the edit returns, the loop drains everything that accumulated — a larger batch. **Slow platforms auto-batch**: a platform that takes 2s per edit gets fewer, larger edits rather than many small ones. The agent is never stalled.

### `None` delta = tool boundary

When the LLM stops generating to call a tool, the agent fires `stream_delta_callback(None)`. This lands as a `_NEW_SEGMENT` sentinel in the queue:

- Delivery loop finalizes the current platform message (strips the cursor `▉`).
- Resets `_message_id = None`.
- Next text after the tool creates a **fresh message below** the tool-progress bubble.

Without this, text before and after a tool call would edit the same message, making it appear above the tool-progress bubbles — visually out of order.

### Flood-control fallback

Platforms rate-limit edits (Telegram flood control). The delivery loop tracks consecutive edit failures with `_flood_strikes`. Behavior:

- Strikes 1–2: double `_current_edit_interval` (adaptive backoff, up to 10s), keep trying.
- Strike 3: enter **fallback mode**.
  - Record `_fallback_prefix` = what the user has already seen.
  - Stop all edits for the rest of the stream.
  - Strip the stuck `▉` from the last visible message (best-effort final edit).
  - At stream end: send only the **tail** (accumulated text minus what was already shown) as a new message.

This means under sustained flood control the user sees partial text in one message, then the remainder as a follow-up — rather than a frozen cursor and lost content.

### Additional handling

| Concern | Mechanism |
|---|---|
| Think-block filtering (`<think>...</think>`) | State machine in `_filter_and_accumulate()` strips inline reasoning tags before any edit reaches the platform. The agent's final-response stripping happens too late for intermediate edits. |
| `buffer_only` mode (Matrix) | Matrix clients render the cursor as a visible tofu artifact. Consumer buffers internally but skips intermediate edits; only the final accumulated text is sent. |
| `fresh_final_after_seconds` (Telegram only) | For slow reasoning-model responses the first-token timestamp becomes stale. When the stream ends and the preview has been visible longer than the threshold, the consumer sends a **fresh new message** (deletes the old one) so the platform timestamp reflects completion time. |
| Overflow splitting | If `_accumulated` exceeds `MAX_MESSAGE_LENGTH` (e.g. 4096 for Telegram), `adapter.truncate_message()` splits into chunks with `(1/2)` indicators. |
| No-edit platforms (iMessage) | `SUPPORTS_MESSAGE_EDITING = False` detected at setup time. Streaming disabled entirely; gateway falls back to single final send. |
| Tool-progress bubble linearization | When a fresh content bubble is created, `_notify_new_message()` fires. The gateway uses this to close the current tool-progress bubble so the next tool opens a new one **below** the content, preserving chronological order. |

---

## Channels

A **channel** is the platform concept for where a message arrives and where responses are delivered. Every inbound event is normalised by the platform adapter into a `SessionSource` — see `sessions.md` for the full `SessionSource` / `SessionContext` / `SessionEntry` data model and session key construction rules. This section covers the platform-specific mapping and channel-level config that live in the gateway.

### Platform → chat_type mapping

Each platform adapter extracts `chat_id` and `chat_type` from its native event and populates `SessionSource`. The gateway's session key logic then uses these fields — see `sessions.md § Session Identity`.

| Platform | Concept | chat_type | chat_id |
|---|---|---|---|
| Telegram | Private chat | `dm` | user_id |
| Telegram | Group/supergroup | `group` | group_id |
| Telegram | Forum topic | `forum` | group_id (thread_id = topic_id) |
| Discord | DM | `dm` | user_id |
| Discord | Channel | `group` | channel_id |
| Discord | Thread / Forum post | `thread` | thread_id (parent_chat_id = channel_id) |
| Slack | DM | `dm` | U-prefixed user ID |
| Slack | Channel | `group` | C-prefixed channel ID |
| Slack | Thread reply | `group` | channel_id (thread_id = message timestamp) |

Extraction points: `str(interaction.channel_id)` in Discord (`discord.py:3089`), `str(chat.id)` in Telegram (`telegram.py:3597`), `channel_id` from the Slack event payload (`slack.py:2112`).

### Channel prompt

An operator-configured per-channel ephemeral prompt override. Resolved by `resolve_channel_prompt(config.extra, channel_id, parent_id)` (`gateway/platforms/base.py:1121`) and stored on `MessageEvent.channel_prompt`. Discord threads fall back to the parent channel's prompt if no thread-specific one is set.

Configured in `config.yaml` under `channel_prompts: {"channel_id": "prompt text"}` inside the platform block.

### Ephemeral system prompt assembly

`combined_ephemeral` is built in the gateway each turn before the `AIAgent` is constructed or reused (`gateway/run.py:13160`):

```
combined_ephemeral = context_prompt                     # 1. SessionContext block — platform location, user, home channels
                   + "\n\n" + channel_prompt            # 2. Per-channel operator override (if set)
                   + "\n\n" + _ephemeral_system_prompt  # 3. Global ephemeral from config.yaml (if set)
```

This is passed as `ephemeral_system_prompt` to `AIAgent`. At API-call time the agent builds the effective system prompt (`run_agent.py:10309`):

```
effective_system = _cached_system_prompt + "\n\n" + ephemeral_system_prompt
```

`_cached_system_prompt` is frozen at agent construction (the cacheable prefix). `ephemeral_system_prompt` is appended fresh each call so it never breaks the provider's prefix cache. It is also excluded from trajectory persistence — subagent transcripts and session replays only store the base prompt.

### Channel-level config

| Setting | Platform | Effect |
|---|---|---|
| `channel_prompts` | All | Per-channel ephemeral system prompt |
| `channel_skill_bindings` | Discord, Slack | Auto-load skill(s) when the bot is in this channel |
| `allowed_channels` | Discord | Whitelist — bot only responds in listed channel IDs |
| `ignored_channels` | Discord | Blacklist — bot never responds here |
| `free_response_channels` | Discord, Slack | No @mention required to invoke the bot |
| `no_thread_channels` | Discord | Bot replies directly without spinning up a thread |

### Home channel

The default delivery target for a platform — used by cron jobs and platform-level notifications. Stored as `HomeChannel(platform, chat_id, name, thread_id?)`. Set via `/sethome` in any chat or via env vars (`TELEGRAM_HOME_CHANNEL`, `DISCORD_HOME_CHANNEL`, etc.). Surfaced in the `SessionContext` system prompt block so the agent knows where scheduled output will land.

---

## What Lives in the Gateway vs the Agent

| Concern | Gateway | AIAgent |
|---|---|---|
| Platform adapters (Telegram, Discord…) | ✓ | |
| Session persistence (SQLite, sessions.json) | ✓ | |
| Agent cache (LRU, config signature) | ✓ | |
| Credential refresh between turns | ✓ | |
| Streaming consumer (live-edit bubble) | ✓ | |
| Tool progress bubbles | ✓ | |
| Slash command routing | ✓ | |
| Resume/recovery injection | ✓ | |
| Multimodal attachment assembly | ✓ | |
| LLM tool loop | | ✓ |
| Prompt caching | | ✓ |
| Memory provider lifecycle | | ✓ |
| MCP tool dispatch | | ✓ |
| Subagent delegation | | ✓ |
| Context compression | | ✓ |
