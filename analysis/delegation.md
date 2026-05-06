# Delegation — `tools/delegate_tool.py`

## What Delegation Is Here

Delegation is the mechanism by which an agent spawns child agents to handle subtasks in isolated contexts. The parent never sees the child's intermediate tool calls — only the final summary is returned. This keeps the parent's context window clean and enables parallel workstreams.

`tools/delegate_tool.py` owns:
- The `delegate_task` tool schema and handler
- Child `AIAgent` construction and lifecycle
- Parallel execution via `ThreadPoolExecutor`
- Toolset restriction and role/depth enforcement
- Cost rollup, file-state side-effect tracking, hook invocation

---

## Toolset Registration

Defined in `toolsets.py:201`:

```python
"delegation": {
    "description": "Spawn subagents with isolated context for complex subtasks",
    "tools": ["delegate_task"],
    "includes": []
}
```

Single tool, no includes. The toolset name `"delegation"` is what gets stripped from child agents — that is the recursion guard. A leaf child never gets this toolset, so it cannot call `delegate_task`.

---

## Tool Schema — `delegate_task`

Defined at `tools/delegate_tool.py:2382`. The tool has two modes selected by which top-level param is provided:

| Mode | Required param | Behaviour |
|------|---------------|-----------|
| Single | `goal` | 1 subagent, runs on the calling thread (no thread pool) |
| Batch | `tasks` | N subagents, `ThreadPoolExecutor`, parallel |

### Parameters

**`goal`** `string` — What the subagent should accomplish. Must be self-contained; the subagent has no conversation history.

**`context`** `string` — Background the subagent needs: file paths, error messages, constraints.

**`toolsets`** `string[]` — Toolsets to enable. Defaults to inheriting parent's enabled toolsets.

**`tasks`** `object[]` — Batch mode. Each item: `{goal, context?, toolsets?, role?, acp_command?, acp_args?}`. `required: ["goal"]`. No `maxItems` in schema — the cap (`max_concurrent_children`, default 3) is enforced at runtime with a clear error.

**`role`** `enum["leaf", "orchestrator"]` — Controls whether the child can further delegate. Default `"leaf"`. Can be set per-task inside `tasks[]` to override.

**`acp_command`** `string` — Spawn a different ACP-capable subprocess as the child (e.g. `claude`) instead of a native `AIAgent`. Enables cross-agent delegation from any parent platform.

**`acp_args`** `string[]` — Args for `acp_command`. Default `["--acp", "--stdio"]`.

`required: []` at schema level — the runtime enforces "either `goal` or `tasks`".

---

## `delegate_task` Function — Top-Level Flow

Entry point at `tools/delegate_tool.py:1836`. Execution order:

### 1. Guards
- No `parent_agent` → error (tool requires agent context injected by the registry)
- `is_spawn_paused()` → error (operator kill switch, clearable from TUI `/agents` overlay)
- `depth >= max_spawn_depth` → error (depth read from `parent_agent._delegate_depth`, default 0)

### 2. Config + credentials
- `_load_config()` reads `delegation.*` block from `config.yaml`
- `effective_max_iter` set from config; model-supplied `max_iterations` is silently ignored (config is authoritative so users get predictable budgets)
- `_resolve_delegation_credentials()` resolves provider/model/api_key/base_url for children — None values mean "inherit from parent"

### 3. Normalize to task list
- `tasks` array → use directly (error if `len > max_concurrent_children`)
- `goal` string → wrap into `[{goal, context, toolsets, role}]`
- Neither → error

### 4. Build children (main thread)
All `AIAgent` instances are constructed on the main thread before any worker thread starts. Wrapped in `try/finally` to restore `model_tools._last_resolved_tool_names` even if a build raises — `AIAgent.__init__` calls `get_tool_definitions()` which, as a side effect, overwrites this global with the names of the child's narrower toolset. The parent needs the global to stay pointing at its own tool list so `execute_code` generates correct sandbox imports for the parent's session.

```python
_parent_tool_names = list(_model_tools._last_resolved_tool_names)
try:
    for i, t in enumerate(task_list):
        child = _build_child_agent(...)
        child._delegate_saved_tool_names = _parent_tool_names
        children.append((i, t, child))
finally:
    _model_tools._last_resolved_tool_names = _parent_tool_names
```

### 5. Run
- **n=1**: `_run_single_child()` called directly on the calling thread — no `ThreadPoolExecutor`, the parent blocks here until the child finishes
- **n>1**: `ThreadPoolExecutor(max_workers=max_concurrent_children)`, one `submit` per child

Parent blocks in a `wait(..., timeout=0.5, return_when=FIRST_COMPLETED)` polling loop rather than `as_completed()` so it can check `parent_agent._interrupt_requested` on each iteration and bail mid-batch without blocking forever on a stuck child.

### 6. Post-run
- `parent_agent._memory_manager.on_delegation(task, result, child_session_id)` — memory provider notified of each outcome
- `subagent_stop` hooks fired once per child, serialized on the parent thread (not worker threads)
- Child costs folded into `parent_agent.session_estimated_cost_usd` (additive, rolls up naturally through orchestrator→worker trees)
- Returns `{"results": [...], "total_duration_seconds": float}`

---

## `_build_child_agent` — Child Construction

Called on the main thread for every task before any worker starts (`tools/delegate_tool.py:834`). Returns a fully configured `AIAgent` without running it.

### Role resolution

```
child_depth = parent._delegate_depth + 1
orchestrator_ok = orchestrator_enabled AND child_depth < max_spawn_depth
effective_role = role if (role == "orchestrator" AND orchestrator_ok) else "leaf"
```

Single gate — role degrades to `"leaf"` here and nowhere else. Callers always receive the normalised string.

### Toolset resolution

Parent toolsets are derived from `parent_agent.enabled_toolsets`. If that is `None` (all tools enabled), toolsets are reverse-mapped from `parent_agent.valid_tool_names` via `model_tools.get_toolset_for_tool()`.

Resolution order:
1. If `toolsets` param given: intersect with parent toolsets (child cannot gain tools parent lacks), then apply MCP inheritance, then `_strip_blocked_tools()`
2. Else: `_strip_blocked_tools(parent_enabled)`

`_strip_blocked_tools()` removes: `"delegation"`, `"clarify"`, `"memory"`, `"code_execution"` — these are the four toolsets containing blocked tools.

Orchestrators get `"delegation"` re-added unconditionally after the strip, regardless of parent toolset membership — orchestrator capability is granted by role, not inherited.

### `AIAgent` construction

Key isolation params passed to `AIAgent.__init__`:

| Param | Value |
|-------|-------|
| `ephemeral_system_prompt` | Built by `_build_child_system_prompt(goal, context, workspace, role)` |
| `skip_memory` | `True` — no parent memory.md access |
| `skip_context_files` | `True` — no parent context imports |
| `enabled_toolsets` | Restricted child toolsets (from above) |
| `quiet_mode` | `True` — no terminal output |
| `max_iterations` | From `delegation.max_iterations` config (default 50) |
| `session_db` | Inherited from parent — child transcript written to same DB |
| `parent_session_id` | Parent's `session_id` — links child row in DB |
| `iteration_budget` | `None` — fresh budget per subagent |
| `clarify_callback` | `None` — subagents cannot ask user questions |

Credential resolution: `delegation.provider/model/api_key/base_url` override > parent inherit. If `override_provider` is set but `override_acp_command` is not, `acp_command` is cleared — otherwise the child's ACP transport would bypass the credential override.

### Post-construction wiring

After `AIAgent()` returns:
- `child._delegate_depth = child_depth` — recursion guard for grandchildren
- `child._delegate_role = effective_role` — stashed for `subagent_stop` hook and cost rollup
- `child._subagent_id = "sa-{index}-{uuid8}"` — stable key shared across TUI events, `_active_subagents` registry, file_state
- `child._credential_pool = _resolve_child_credential_pool(...)` — shared with parent if same provider, else own pool
- `parent_agent._active_children.append(child)` — registered for interrupt propagation (parent's `interrupt()` will call `child.interrupt()`)
- `child_progress_cb("subagent.spawn_requested")` fired immediately — TUI gets a tree node before the child's thread starts

---

## `_run_single_child` — Child Execution

Called either directly on the calling thread (n=1) or from a worker thread in the `ThreadPoolExecutor` (n>1). Takes a pre-built child and runs it to completion (`tools/delegate_tool.py:1266`).

### Setup

1. **Credential lease**: acquires a slot from `child._credential_pool` if present, binds the credential to the child via `child._swap_credential()`
2. **Heartbeat thread**: daemon `threading.Thread` started immediately — fires every `_HEARTBEAT_INTERVAL` (30s), touches `parent_agent._touch_activity()` to prevent gateway inactivity timeout while the child works
3. **Stale detection**: heartbeat tracks `(child_iter, child_tool)` across cycles. If neither advances:
   - Idle child (no current tool): stale after `_HEARTBEAT_STALE_CYCLES_IDLE` (15 cycles = 450s)
   - In-tool child: stale after `_HEARTBEAT_STALE_CYCLES_IN_TOOL` (40 cycles = 1200s)
   - On stale: heartbeat stops touching parent, allowing the gateway timeout to fire naturally
4. **`_active_subagents` registration**: child registered by `subagent_id` with `{goal, model, depth, status, agent}` — lets the TUI target the child by ID for kill/pause

### Execution

Child runs inside a dedicated **one-worker `ThreadPoolExecutor`** regardless of single vs batch mode — this is the timeout mechanism, not parallelism:

```python
_timeout_executor = ThreadPoolExecutor(
    max_workers=1,
    initializer=_set_subagent_approval_cb,   # installs non-interactive approval cb
    initargs=(_get_subagent_approval_callback(),),
)
_child_future = _timeout_executor.submit(_run_with_thread_capture)
result = _child_future.result(timeout=child_timeout)   # blocks here
```

`_run_with_thread_capture` captures `threading.current_thread()` into `_worker_thread_holder` before calling `child.run_conversation(user_message=goal, task_id=child_task_id)`.

The `initializer` installs either `_subagent_auto_deny` or `_subagent_auto_approve` in the worker thread's `threading.local` — without this, a dangerous command prompt inside the child would call `input()` and deadlock the parent TUI.

### Timeout handling

On `FuturesTimeoutError`:
- `child.interrupt()` called to signal the child to stop cleanly
- If 0 API calls were made: `_dump_subagent_timeout_diagnostic()` writes `~/.hermes/logs/subagent-{sid}-{ts}.log` with child config, schema sizes, and worker thread stack — these hangs were previously black boxes
- Returns `{status: "timeout", ...}` — does not raise

`_timeout_executor.shutdown(wait=False)` — if the child thread is stuck on blocking I/O, `wait=True` would hang forever.

### Result assembly

On success, `child.run_conversation()` returns a dict with `final_response`, `completed`, `interrupted`, `api_calls`, `messages`.

**Tool trace** rebuilt from `messages`: walks assistant messages for `tool_calls`, walks tool messages for results, matches by `tool_call_id` to correctly pair parallel tool calls with their results.

**Status**:
- `"completed"` — `final_response` non-empty
- `"failed"` — ran without error but no summary
- `"interrupted"` — `interrupted=True` in result
- `"timeout"` / `"error"` — set in the exception branch above

**Exit reason**: `"completed"` | `"max_iterations"` | `"interrupted"`

**File-state reminder**: checks `file_state.writes_since()` for any writes by non-parent task IDs to files the parent had previously read — if found, appends a `[NOTE: subagent modified files ...]` reminder to `entry["summary"]` so the parent re-reads before editing.

**`_child_role` and `_child_cost_usd`** captured into the entry before `child.close()` — they are stripped before the dict is serialised back to the model but used by the post-run hook/cost-rollup loop in `delegate_task()`.
