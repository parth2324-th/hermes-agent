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
- Individual tool implementations (bash, filesystem, web_search, etc.) not explored
- Cron jobs and scheduled tasks not covered

## Proactive Read arrow — memory prefetch
- **Covered** in `providers.md`
- `prefetch_all` (blocking, turn start) vs `queue_prefetch_all` (non-blocking, turn end)
- One-turn topic lag design flaw and tradeoff discussed

## Reactive READ/WRITE arrow — tool execution
- **Partially covered**
- `_invoke_tool()` dispatch path covered in `agents.md`
- MCP tool call path fully covered in `mcp.md`
- Individual built-in tool implementations (bash, edit, web) **not covered**
