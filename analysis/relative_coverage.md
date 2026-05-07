# Relative Coverage

## Context — system prompt assembly
- **Covered** in `agents.md`
- All 7 layers: identity, project-rules, user-profile, procedural (skills index + skill_view), episodic (memory), runtime, tool-schema
- Ephemeral system prompt and prefill messages
- Memory store vs memory manager system prompt contributions
- Progressive skill disclosure (index in prompt, full body on-demand via skill_view)

## Control — the kernel loop
- **Covered** in `agents.md`
- `run_conversation()` structure and tool loop
- `_invoke_tool()` dispatch table and priority order
- `IterationBudget` (counts LLM calls, not tokens)
- `_drop_thinking_only_and_merge_users` (two-pass cleanup before API call)
- Reasoning/thinking field lifecycle (`_copy_reasoning_content_for_api`, cross-provider guard)
- `_spawn_background_review` (forked agent post-loop)
- `_persist_session` (SQLite flush per turn)

## Capability — tools + MCP + guardrails
- **Partially covered**
- Registry dispatch and tool handler architecture covered in `agents.md` and `mcp.md`
- MCP outbounds fully covered in `mcp.md` (init, lifecycle, circuit breaker, auth recovery, dynamic refresh, sampling)
- Delegation fully covered in `delegation.md` — toolset, schema, child construction, toolset resolution, credential pool, TLS approval callback, heartbeat, timeout, file-state reminder, result assembly, cost rollup
- Guardrails (path restrictions, output filtering, approval gates) **not covered**

## Model — LLM call
- **Partially covered** in `agents.md`
- `_call_llm()`, prompt caching, streaming pipeline covered
- Reasoning/thinking wire format (`_copy_reasoning_content_for_api`) covered
- Provider selection and model resolution internals **not covered**
- `providers.md` covers memory providers, not LLM providers

## LifeCycle — persistence of self
- **Covered** in `sessions.md` and `agents.md`
- SQLite transcript storage, sessions.json pointer map, token counts
- Session flags (`suspended`, `resume_pending`, `expiry_finalized`, etc.)
- Session expiry watcher and memory handoff chain
- Context compression and session ID rotation
- `_persist_session` dedup and flush logic

## World — external I/O
- **Partially covered**
- `delegate_task` fully covered in `delegation.md` — toolset, schema, top-level flow, `_build_child_agent`, `_run_single_child`, heartbeat, timeout, result assembly
- `model_tools.py` covered in `delegation.md` and `agents.md` — `get_tool_definitions`, `handle_function_call`, `_run_async`, `_last_resolved_tool_names`
- Cron jobs fully covered in `cron.md` — data model, ticker, missed-fire grace logic, at-most-once semantics, agent construction, delivery resolution, safety
- Individual tool implementations (bash, web_search, code_execution, etc.) not explored

## Proactive Read arrow — memory prefetch
- **Covered** in `providers.md`
- `prefetch_all` (blocking, turn start) vs `queue_prefetch_all` (non-blocking, turn end)
- One-turn topic lag design flaw and tradeoff discussed

## Reactive READ/WRITE arrow — tool execution
- **Partially covered**
- `_invoke_tool()` dispatch path covered in `agents.md`
- MCP tool call path fully covered in `mcp.md`
- File tools (`read_file`, `write_file`, `patch`, `search_files`) covered in `inbuilt_tools.md`
- Skills tools (`skills_list`, `skill_view`, `skill_manage`) covered in `inbuilt_tools.md`
- Memory tool (`add`, `replace`, `remove`, snapshot vs live state, injection guards) covered in `inbuilt_tools.md`
- Web tools (`web_search`, `web_extract`, backend dispatch, LLM post-processing, SSRF guards) covered in `inbuilt_tools.md`
- Terminal tool (backend routing, approval gate, CWD tracking, output limits, background processes) covered in `inbuilt_tools.md`
- Code execution tool (UDS vs file-based RPC, sandbox tool allowlist, env scrubbing, execution modes) covered in `inbuilt_tools.md`
- Clarify tool (callback dispatch, two modes, subagent guard) covered in `inbuilt_tools.md`
- `messaging` tool **not covered**
