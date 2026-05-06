# MCP System — `tools/mcp_tool.py`

## What MCP Is Here

MCP (Model Context Protocol) is a protocol for connecting an AI agent to external tool servers. Each MCP server is an independent process or HTTP endpoint that exposes its own set of tools. The agent calls those tools the same way it calls built-in tools — it doesn't know the difference.

`tools/mcp_tool.py` is the MCP client layer. It owns:
- The background asyncio event loop that all MCP I/O runs on
- The `MCPServerTask` class — one instance per server, one asyncio Task per instance
- Tool registration into the global registry
- Circuit breaker, keepalive, auth recovery, dynamic tool refresh
- Shutdown and orphan process cleanup

---

## Config and Init

Servers are declared in `~/.hermes/config.yaml` under `mcp_servers`:

```yaml
mcp_servers:
  my-server:
    command: uvx my-mcp-server   # stdio transport
    args: ["--flag"]
    env:
      API_KEY: "${MY_API_KEY}"   # env var interpolation at load time

  remote-server:
    url: https://api.example.com/mcp   # HTTP/StreamableHTTP transport
    auth: oauth
    timeout: 30
    tools:
      include: ["tool_a", "tool_b"]    # whitelist
```

`_load_mcp_server_configs()` reads this block and expands `$VAR` references from the process environment. The config is passed to `register_mcp_servers()` which is the public entry point.

### Background Event Loop

All MCP I/O runs on a dedicated background asyncio event loop — not the gateway's main event loop. The loop is module-global:

```python
_mcp_loop: asyncio.AbstractEventLoop | None = None
_mcp_thread: threading.Thread | None = None
```

`_get_mcp_loop()` lazily creates it: `asyncio.new_event_loop()` + daemon thread running `loop.run_forever()`. Callers on the main/gateway thread submit work via `asyncio.run_coroutine_threadsafe(coro, _mcp_loop)`.

One shared loop means one connection pool and one cleanup path at process exit. See `asyncio.md` for why per-instance loops leak resources.

At any point the loop contains N permanent Tasks (one per configured server) plus transient coroutines for in-flight tool calls:

```
_mcp_loop
├── Task: MCPServerTask("github")    ← parked in _wait_for_lifecycle_event()
├── Task: MCPServerTask("postgres")  ← parked in _wait_for_lifecycle_event()
├── Task: MCPServerTask("search")    ← parked in _wait_for_lifecycle_event()
│
└── (transient) call_tool("github", "search_repos", {...})   ← added for this tool call, gone when done
```

The server Tasks are permanent for the process lifetime — they keep `server.session` alive. Tool call coroutines are ephemeral: submitted by the agent worker thread, run to completion, return the result, removed.

---

## MCPServerTask — One Per Server

```python
class MCPServerTask:
    __slots__ = (
        "name", "session", "tool_timeout",
        "_task", "_ready", "_shutdown_event", "_reconnect_event",
        "_tools", "_error", "_config",
        "_sampling", "_registered_tool_names", "_auth_type", "_refresh_lock",
        "_rpc_lock", "_pending_refresh_tasks",
    )
```

Each server lives in **one long-lived asyncio Task**. This is not optional — anyio cancel-scopes created by the transport client must be entered and exited in the same Task. The entire connection lifecycle (connect → discover → serve → disconnect) stays inside `run()`.

### Key slots

| Slot | Purpose |
|------|---------|
| `session` | Live `ClientSession` from the MCP SDK — None when disconnected |
| `_ready` | `asyncio.Event` set when initial tool discovery completes (or permanently fails) |
| `_shutdown_event` | Set by `shutdown()` to cleanly exit the run loop |
| `_reconnect_event` | Set on auth failure — signals the transport to tear down and rebuild with fresh credentials |
| `_rpc_lock` | Serializes ALL client-initiated RPCs per server — prevents wedged stdio JSON-RPC streams when notifications arrive mid-call |
| `_refresh_lock` | Prevents overlapping dynamic tool refreshes |
| `_pending_refresh_tasks` | Strong references to background refresh tasks (prevents GC before they complete) |

---

## Connection Lifecycle

`register_mcp_servers()` submits `_discover_all()` to the MCP loop via `run_coroutine_threadsafe`, which connects all servers in parallel via `asyncio.gather`. Per server:

```
_discover_and_register_server(name, config)      ← coroutine on MCP loop
        ↓
_connect_server(name, config)
    → server = MCPServerTask(name)                ← instance created here
    → _servers[name] = server                     ← stored in module-level dict after successful start
        ↓
await server.start(config)                        ← Task 1: start() on MCP loop
    → self._task = ensure_future(server.run(config))  ← Task 2: run() scheduled as sibling
    → await self._ready.wait()                    ← start() suspends, loop is free to run run()
```

With `start()` suspended, the loop advances `run()`:

```
server.run(config):
    while True:
        _run_stdio(config)  OR  _run_http(config)
            → spawn subprocess / open HTTP client
            → async with ClientSession(...) as session:
                → await session.initialize()
                → self.session = session
                → await self._discover_tools()        ← list_tools() via _rpc_lock
                → self._ready.set()                   ← unblocks start()
                → await self._wait_for_lifecycle_event()
                        ↓
                    returns "shutdown" or "reconnect"
        ↓
        if shutdown: break
        if reconnect: continue   ← no retry counter increment, clean credential rebuild
        except Exception:
            exponential backoff, retry up to _MAX_RECONNECT_RETRIES
```

`self._ready.set()` is the handshake — it wakes `start()` which checks `self._error` and returns to the caller. After that, `run()` parks in `_wait_for_lifecycle_event()` for the process lifetime, keeping `self.session` alive. `start()` is done; `run()` is the permanent Task.

### Transports

**Stdio** (`command` key in config):
- `_build_safe_env()` strips the environment down to a safe allowlist — the subprocess does not inherit parent credentials
- OSV malware check on the command before spawning
- stderr redirected to `~/.hermes/logs/mcp-stderr.log` (not TTY)
- Child PIDs tracked in `_stdio_pids` for orphan cleanup

**HTTP/StreamableHTTP** (`url` key in config):
- `mcp >= 1.24.0`: caller-owned `httpx.AsyncClient` — explicit lifecycle, proper cleanup
- `mcp < 1.24.0`: deprecated path, SDK manages the client internally
- OAuth 2.1 PKCE via `MCPOAuthManager.get_or_build_provider()` when `auth: oauth`
- Authorization headers stripped on cross-origin redirects

---

## Tool Call Dispatch

When the LLM calls an MCP tool, `_invoke_tool()` routes it to `registry.dispatch()` which calls the sync handler closure registered at startup. Everything from the agent worker thread down is synchronous — the MCP loop is only entered via `run_coroutine_threadsafe`:

```
LLM tool call → _invoke_tool() → registry.dispatch() → _handler(args)   ← agent worker thread (sync)

_handler:
1. circuit breaker check
       → if open and cooldown not elapsed: return error JSON, don't call server
       → if cooldown elapsed: fall through as half-open probe

2. fetch server.session under _lock
       → if not connected: bump breaker, return error JSON

3. define _call() coroutine:
       async with server._rpc_lock:           ← serializes against other RPCs on this server
           result = await session.call_tool(tool_name, arguments=args)
       → parse result.content blocks → JSON string
       → {"result": "..."} or {"result": ..., "structuredContent": {...}} or {"error": "..."}

4. _run_on_mcp_loop(_call(), timeout=tool_timeout)
       → run_coroutine_threadsafe(_call(), _mcp_loop).result()
       → agent worker thread BLOCKS here until MCP server responds

5. parse returned JSON:
       error key present → _bump_server_error()
       success           → _reset_server_error()

6. exceptions:
       auth error         → _handle_auth_error_and_retry()    ← reconnect + retry once
       session expiry     → _handle_session_expired_and_retry()
       other              → _bump_server_error(), return error JSON
```

`_call()` is a fresh coroutine defined per invocation — a new coroutine object capturing the current `args`. The handler closure itself captures `server_name` and `tool_name` at registration time and is reused across all calls.

---

## Tool Registration

`_register_server_tools(name, server, config)` fires after `_discover_tools()` and on every dynamic refresh.

### Filtering

```yaml
tools:
  include: ["tool_a"]   # whitelist — takes precedence
  exclude: ["tool_b"]   # blacklist — used only if include absent
```

Tools absent from `include`, or present in `exclude`, are silently skipped.

### Naming

MCP tools are prefixed: `mcp_{server_name}_{tool_name}` with name components sanitized (non-alphanumeric → underscores). Example: server `my-server`, tool `do_thing` → `mcp_my_server_do_thing`.

### Collision guard

If the prefixed name already exists in the registry **in a non-MCP toolset**, the MCP tool is skipped with a warning. Built-in tools win; MCP tools of the same name do not shadow them.

### Utility tools

Beyond the server's declared tools, four utility tools are conditionally registered per server:
- `mcp_{server}_list_resources` / `mcp_{server}_read_resource`
- `mcp_{server}_list_prompts` / `mcp_{server}_get_prompt`

These are only registered if the server's session object actually has the corresponding capability method. Controllable via `tools.resources: false` / `tools.prompts: false` in config.

### Toolset alias

All tools from one server share a toolset named `mcp-{server_name}`. A bare `server_name` alias is also registered so the agent can request `toolset: my-server` instead of `toolset: mcp-my-server`.

### Prompt injection scan

`_scan_mcp_description()` runs on each tool's description before registration, flagging patterns that look like jailbreak attempts embedded in server-supplied tool descriptions.

---

## Keepalive

`_wait_for_lifecycle_event()` is not a simple `await event.wait()`. It loops with a 180-second timeout. On timeout, it sends `session.list_tools()` (30s timeout) as a keepalive to detect stale TCP connections. If the keepalive fails, `_reconnect_event` is set — the server rebuilds the session rather than silently degrading.

---

## Dynamic Tool Discovery

When a server sends `notifications/tools/list_changed`, the message handler on `ClientSession` receives a `ToolListChangedNotification`. It cannot refresh inline — it shares the same stdio JSON-RPC stream as tool calls, and calling `list_tools()` there would wedge the stream if a tool call was already in flight (notably `mongodb-mcp-server` emits this during startup).

Instead it calls `_schedule_tools_refresh()`:

```python
task = asyncio.create_task(_refresh_tools_task())   # new one-shot Task on MCP loop
self._pending_refresh_tasks.add(task)               # strong reference — prevents GC mid-execution
task.add_done_callback(self._pending_refresh_tasks.discard)  # self-removes on completion
```

`_refresh_tools()` is a one-shot coroutine — it runs once, exits, and is discarded. A fresh Task is created per notification per server. There is no shared or infinite refresh coroutine. One per server rather than shared because each refresh locks `_rpc_lock` and `_refresh_lock` scoped to its own server — sharing would add cross-server contention for something rare and inherently per-server.

`_refresh_tools()` on the MCP loop:
- Acquires `_refresh_lock` — prevents overlapping refreshes from rapid-fire notifications
- `list_tools()` via `_rpc_lock` — serialized against other RPCs on this server's stream
- Diffs old vs new tool names
- Deregisters stale tools (removed from server's list) via `registry.deregister()`
- Re-registers with fresh list — in-place update for unchanged names avoids races with in-flight tool calls that already hold handler references
- Logs added/removed names as a warning — unexpected tool changes are user-visible

---

## Circuit Breaker

```
state: closed → open → half-open → closed
```

- **closed**: all calls go through, `_server_error_counts[name] < 3`
- **open**: after 3 consecutive failures, calls short-circuit immediately
  - Returns a JSON error telling the model the server is unreachable and to stop retrying
  - Prevents the 90-iteration burn loop from issue #10447
- **half-open**: after 60s cooldown, next call is a real probe
  - Probe succeeds → closed, error count reset
  - Probe fails → open again, cooldown re-armed

`_bump_server_error()` increments count and stamps `_server_breaker_opened_at` when threshold is crossed. `_reset_server_error()` clears both. A successful OAuth recovery also resets the breaker (otherwise a reconnect followed by a failing retry would leave the breaker pinned above threshold permanently).

---

## Auth Recovery

When a tool call raises an auth-related exception (`OAuthFlowError`, `OAuthTokenError`, `UnauthorizedError`, or `httpx.HTTPStatusError` with status 401):

```
tool handler raises auth exception
        ↓
_handle_auth_error_and_retry()
        ↓
MCPOAuthManager.handle_401(server_name)
    → checks if disk has fresh tokens / SDK can refresh in-place
        ↓
if recovered:
    # agent worker thread → MCP loop thread (cross-thread signal)
    loop.call_soon_threadsafe(server._reconnect_event.set)
    → MCPServerTask.run() exits _wait_for_lifecycle_event() with "reconnect"
    → _run_http/_run_stdio tears down current session cleanly
    → run() loops back, rebuilds MCP session with fresh credentials
    → self._ready.set()

    # agent worker thread busy-polls
    while time.monotonic() < deadline (15s):
        if server.session is not None and server._ready.is_set():
            break
        time.sleep(0.25)   ← polls every 250ms
        ↓
retry the tool call once
    → success: return result
    → failure: return needs_reauth JSON (tells model to stop)
```

Session expiry errors follow the same `_reconnect_event` path — detected by substring matching on the error message rather than exception type.

---

## Shutdown

`shutdown_mcp_servers()` — the process-level shutdown entry point:

```python
asyncio.gather(*(server.shutdown() for server in _servers.values()))
```

All servers shut down **in parallel** (15s total timeout). Per-server `MCPServerTask.shutdown()`:

1. Sets `_shutdown_event` and `_reconnect_event` (both, to ensure `_wait_for_lifecycle_event` unblocks)
2. `await asyncio.wait_for(self._task, timeout=10)`
3. If task times out: cancel it, swallow `CancelledError`
4. Cancel any pending refresh tasks
5. Deregister all registered tools from the global registry

After all tasks exit: `_stop_mcp_loop()` stops and closes the background asyncio loop, joins its thread.

For stdio servers: any subprocess PIDs that survived transport teardown are flagged as orphans in `_orphan_stdio_pids`. `_kill_orphaned_mcp_children()` then sends SIGTERM, waits 2s, escalates to SIGKILL for survivors.

---

## SamplingHandler

MCP servers can send `sampling/createMessage` requests back to the client — asking hermes's LLM to generate a response on their behalf. One `SamplingHandler` is created per server and passed to `ClientSession` as `sampling_callback`. The SDK calls it directly when the server sends a sampling request.

### Pathway

```
MCP server subprocess
    → sends sampling/createMessage over stdio/HTTP
        ↓
ClientSession (on MCP loop thread)
    → calls SamplingHandler.__call__(context, params)   ← async, runs on MCP loop
        ↓
1. rate limit check         ← sliding window, max_rpm (default 10/min)
2. model resolution         ← config override > server hint > default
3. allowed_models check     ← whitelist guard, returns ErrorData if rejected
4. _convert_messages()      ← MCP SamplingMessage format → OpenAI message format
                               handles text, image, tool_use, tool_result blocks
5. asyncio.to_thread(_sync_call)   ← offloads blocking call_llm() to thread pool
                                      MCP loop stays unblocked during LLM call
        ↓
call_llm() on thread pool thread
    → LLM response
        ↓
dispatch on finish_reason:
    "tool_calls" → _build_tool_use_result()   ← returns CreateMessageResultWithTools
                   tool loop counter checked against max_tool_rounds
    text         → _build_text_result()       ← returns CreateMessageResult
                   resets tool loop counter
        ↓
SDK receives CreateMessageResult / CreateMessageResultWithTools / ErrorData
    → returns to MCP server as sampling/createMessage response
```

`asyncio.to_thread()` is the inverse of `run_coroutine_threadsafe` — it takes a sync callable and runs it on a thread pool, returning an awaitable. `call_llm` is synchronous and blocking; running it directly on the MCP loop thread would freeze all other MCP coroutines for the duration of the LLM call.

Sampling is enabled by default when the MCP SDK supports it (`_MCP_SAMPLING_TYPES` import guard).

---

## What Lives Where

| Concern | Owner |
|---------|-------|
| Background event loop | `_mcp_loop` module global |
| Per-server connection lifecycle | `MCPServerTask.run()` — one asyncio Task |
| Tool registration + deregistration | `_register_server_tools()` / `registry.deregister()` |
| Circuit breaker state | `_server_error_counts`, `_server_breaker_opened_at` module globals |
| Auth recovery orchestration | `MCPOAuthManager` (separate module) |
| Tool call dispatch | `_make_tool_handler()` factory — sync wrapper, submits to MCP loop |
| Dynamic tool refresh | `_refresh_tools()` — separate background Task per notification |
| Orphan process cleanup | `_stdio_pids`, `_orphan_stdio_pids` + SIGTERM/SIGKILL sweep |
