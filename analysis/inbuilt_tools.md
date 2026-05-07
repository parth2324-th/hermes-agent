# Built-in Tool Implementations

---

## File Tools — `tools/file_tools.py` + `tools/file_operations.py`

### Architecture

`file_tools.py` owns the tool handlers and all guard logic. `file_operations.py` owns the actual I/O via `ShellFileOperations` — all file operations are expressed as shell commands routed through the active terminal backend (local, docker, singularity, ssh, modal, daytona). This means file tools work identically across all execution environments.

### `read_file`

Guard chain runs before any I/O, in order:

1. **Device path block** — rejects `/dev/zero`, `/dev/stdin`, `/dev/urandom`, `/proc/self/fd/0-2`, etc. Pure path check, no I/O. Without this, reading these paths hangs the process (infinite output or blocking on input).
2. **Binary extension block** — rejects known binary extensions. No I/O.
3. **Hermes internal path block** (`agent.file_safety.get_read_block_error`) — blocks catalog and hub metadata files to prevent prompt injection via file content.
4. **Dedup check** — keyed on `(resolved_path, offset, limit)`. If the file's mtime matches the cached value from the last read, returns a lightweight stub (`"File unchanged since last read..."`) instead of re-sending content. After 2 stub hits for the same key, escalates to a hard `BLOCKED:` error to break infinite read loops. Saves context tokens.
5. **Actual read** via `ShellFileOperations.read_file()`.
6. **Char count guard** — default 100K chars (configurable via `file_read_max_chars` in config.yaml). Rejects oversized reads and tells the model to use `offset`/`limit`. Checked on formatted content (with line-number prefixes) since that's what actually enters context.

Path resolution: relative paths resolved against the task's live terminal cwd (`_get_live_tracking_cwd(task_id)`) → `TERMINAL_CWD` env var → `os.getcwd()`.

### `write_file`

Two staleness checks before writing:
- **Cross-agent**: `file_state.check_stale(task_id, resolved_path)` — did another task write this file since this task last read it? Names the sibling task in the warning.
- **Per-task**: `_check_file_staleness(path, task_id)` — same but scoped to this task's own read timestamps.

Write serialized via `file_state.lock_path(resolved_path)` — concurrent subagents cannot interleave on the same file. Different paths remain fully parallel.

After successful write:
- `file_state.note_write(task_id, resolved_path)` — updates the cross-agent registry
- `_update_read_timestamp(path, task_id)` — resets per-task staleness so consecutive writes by the same task don't false-alarm

Sensitive path guard blocks writes to `/etc/`, `/boot/`, `/usr/lib/systemd/`, `/private/etc/`, `/private/var/`, and exact paths like `/var/run/docker.sock`.

Guard against writing internal read_file status text as file content — prevents the agent from accidentally writing a dedup stub message into a file.

### `patch`

Two modes:
- `replace` — exact string find-and-replace (`old_string` → `new_string`, optional `replace_all`)
- `patch` (V4A format) — structured diff format supporting multi-file patches

For multi-file V4A patches, paths are **sorted and locked in sorted order** via `contextlib.ExitStack` — prevents deadlock when two concurrent callers patch overlapping file sets (both acquire locks in the same order).

Same staleness checks as `write_file` per path.

### `search_files`

Implemented via `ShellFileOperations.search()` — expressed as shell commands (ripgrep or grep depending on backend). Supports content search + file glob filtering.

---

## Skills Tools — `tools/skills_tool.py` + `tools/skill_manager_tool.py`

### What Skills Are

Skills are procedural memory — SKILL.md files with YAML frontmatter (`name`, `description`, `version`, `platforms`, `prerequisites`) and markdown body. They live in `~/.hermes/skills/` (seeded from the bundled `skills/` directory at install). Plugins can provide skills too via the plugin manager registry.

### Progressive Disclosure Architecture

Two tiers:
- **Tier 1** (`skills_list`) — name + description + category only. Never loads full content. Token-efficient index.
- **Tier 2/3** (`skill_view`) — full SKILL.md body + optional linked files (references, templates). Loaded on demand.

This mirrors how skills appear in the system prompt: the index (all names + descriptions) is always present; full content is only loaded when the agent calls `skill_view`.

### `skills_list`

Scans `~/.hermes/skills/` recursively for `SKILL.md` files, reads only YAML frontmatter, returns `{name, description, category}` per skill. Filters by platform (macOS/linux/windows) — skills with a `platforms` field that doesn't match the current OS are excluded. Optional category filter. Prompt injection scanning runs on each skill's description before inclusion.

### `skill_view`

Two dispatch paths:
- **Qualified names** (`"plugin:skill"`) → plugin manager registry via `get_plugin_manager().find_plugin_skill(name)`
- **Bare names** → flat tree scan of all skills directories

Returns full SKILL.md content. `preprocess=True` (default) applies configured template rendering and inline shell expansion. Prompt injection scanning runs on content before returning. Stale registry entries (file deleted out of band) are cleaned up on access.

### `skill_manage`

Agent-writable procedural memory. All writes go to `~/.hermes/skills/`. Actions:

| Action | What it does |
|--------|-------------|
| `create` | Creates new skill directory + SKILL.md |
| `edit` | Full rewrite of SKILL.md |
| `patch` | Targeted find-and-replace within SKILL.md or any supporting file |
| `delete` | `shutil.rmtree` on the skill directory |
| `write_file` | Add/overwrite a supporting file (must be under `references/`, `templates/`, `scripts/`, or `assets/`) |
| `remove_file` | Remove a supporting file |

**No hardcoded bundled vs user distinction** — `_find_skill` searches all skill directories and `delete` operates on whatever it finds. The only protection is the **pin system**: skills pinned via `hermes curator pin <name>` cannot be deleted (`_pinned_guard` check). Unpinning requires `hermes curator unpin <name>` from the CLI.

`write_file` enforces path traversal prevention (`has_traversal_component`) and restricts files to `ALLOWED_SUBDIRS` only — no escaping the skill directory.

Optional security scan (`skills.guard_agent_created`) — off by default since the agent can already execute arbitrary code via terminal, so the scan adds friction without meaningful security gain.

### Plugin Skills

Plugins can register skills via `plugin_context.register_skill(name, path)` during their `setup()` hook. These are stored in `PluginManager._plugin_skills` as `{"plugin_name:skill_name": {"path": Path, ...}}` and resolved via `skill_view("plugin:skill")`.

Plugin skills differ from regular skills in three ways:
- They do **not** live in `~/.hermes/skills/` — they live wherever the plugin's files are
- They are **not** listed in `skills_list` or the system prompt's `<available_skills>` index — invisible by default
- They are **read-only** — `skill_manage` cannot edit or delete them

The LLM discovers plugin skills through the **tool schema** — a plugin ships a tool alongside its skill, and the tool's `"description"` field (which is sent to the LLM on every API call as part of the `tools` array) can tell the agent to call `skill_view('plugin:skill-name')` to load instructions. The skill provides the detailed how-to; the tool provides the capability. Skills are loaded on demand rather than baked into the system prompt permanently.

---

## Memory Tool — `tools/memory_tool.py`

The agent-facing tool for reading and writing persistent memory mid-conversation. Backed by `BuiltinMemoryProvider` (see `providers.md`).

### Actions

Three actions, no explicit read — the current state is returned in every response so the agent always sees the result of its write:

| Action | Required params | What it does |
|---|---|---|
| `add` | `target`, `content` | Appends a new entry |
| `replace` | `target`, `content`, `old_text` | Finds `old_text` (substring match) and replaces with `content` |
| `remove` | `target`, `old_text` | Deletes the entry matching `old_text` |

**`target`** is `"memory"` (agent's personal notes, `~/.hermes/memories/MEMORY.md`) or `"user"` (user profile, `~/.hermes/memories/USER.md`). Character limits: 2200 for memory, 1375 for user profile — configurable via `memory.memory_char_limit` / `memory.user_char_limit`.

### Dispatch

The `MemoryStore` instance is constructed in `run_agent.py` and injected into the tool handler via a lambda closure. If `store=None` (memory disabled in config or not available in this environment), the tool returns an error immediately without dispatching.

### Guards

Injection scanning runs on every `content` write before touching disk (`memory_tool.py:67`). Blocked patterns:

- Prompt injection phrases ("ignore previous instructions", "system prompt override")
- Role hijack ("you are now...")
- Deception hiding ("do not tell the user")
- Exfiltration (curl/wget with `${TOKEN}`, reading `.env`, `.ssh`)
- Invisible unicode (zero-width joiners, RTL marks)

### Snapshot vs live state

Writes are durable (atomic `os.replace()` + `fcntl` file lock) but not reflected in the current session's system prompt. The `_system_prompt_snapshot` is frozen at session start and never mutated — this keeps the prefix cache stable for the entire session. The model sees its own writes only after re-initialization. See `providers.md § BuiltinMemoryProvider` for the full frozen snapshot design.

Entries are delimited by `\n§\n` in the markdown files. Duplicates are dropped on load (first occurrence kept).

---

## Web Tools — `tools/web_tools.py`

Two registered tools: `web_search` and `web_extract`. A third (`web_crawl`) exists in the file but is not registered as an agent tool.

### Backends

Dispatch is via `_get_backend()` (`web_tools.py:121`), which reads `web.backend` from config. Fallback priority when not explicitly set: **firecrawl → parallel → tavily → exa**. Each backend requires its own API key env var (`FIRECRAWL_API_KEY`, `PARALLEL_API_KEY`, `TAVILY_API_KEY`, `EXA_API_KEY`). Firecrawl also supports self-hosted instances via `FIRECRAWL_API_URL` and a managed gateway path for Nous subscribers.

### `web_search`

Schema: `query` (string, required), `limit` (int, default 5, max 100). Returns up to `limit` results as `{title, url, description, position}`.

Each backend has its own search implementation: `parallel.beta.search()` for Parallel, Exa SDK with highlights, HTTP POST to `api.tavily.com/search` for Tavily, `firecrawl_client.search()` for Firecrawl.

### `web_extract`

Schema: `urls` (array, max 5 items). Fetches and returns each URL as markdown.

Safety guards run before any fetch:
- **Secret blocking** — rejects URLs containing embedded API keys/tokens
- **SSRF protection** (`is_safe_url()`) — blocks private/internal IP ranges
- **Website policy** (`check_website_access()`) — checks operator blocklist; re-checked after redirects

Content processing pipeline per URL:
1. Backend fetches raw content (markdown + html via Firecrawl, `exa.get_contents()`, Tavily extract endpoint, or Parallel extract)
2. `clean_base64_images()` strips embedded base64 images to reduce token count
3. **Optional LLM post-processing** — if `use_llm_processing=True` and an auxiliary model is available, content is summarised by a secondary LLM (default: Gemini Flash via OpenRouter)

LLM post-processing thresholds:
- Trigger: content > 5000 chars
- Chunked mode: content > 500KB (split into 100KB chunks, parallel summarisation, then synthesis pass)
- Hard refusal: content > 2MB
- Output cap: 5000 chars

Falls back to truncated raw content (first 5000 chars) if LLM processing fails — never returns an error in place of content.

### Debug mode

`WEB_TOOLS_DEBUG=true` writes per-session JSON logs to `logs/web_tools_debug_{uuid}.json` with raw API response sizes, LLM compression ratios, and processing stages applied.

---

## Terminal Tool — `tools/terminal_tool.py`

Single tool: `terminal_tool(command, background, timeout, workdir, pty, ...)`. Executes shell commands and returns `{output, exit_code, error, status}`.

### Backends

Routed via `TERMINAL_ENV` env var (default: `"local"`). Seven backends:

| Backend | Mechanism |
|---|---|
| `local` | `subprocess.Popen` on host, `os.setsid()` for process-group isolation |
| `docker` | Named containers, configurable image, optional volume mounts |
| `singularity` | Apptainer/Singularity `.sif` images or `docker://` URIs |
| `ssh` | Remote shell via SSH, persistent session |
| `modal` | Cloud sandbox via Modal SDK (direct or managed via Nous gateway) |
| `daytona` | Cloud sandbox via Daytona SDK |
| `vercel` | Cloud sandbox via Vercel API, fixed CWD `/vercel/sandbox` |

All backends share a **session snapshot** pattern: login shell environment (exports, functions, aliases) is captured once at init and re-sourced before every command, giving the illusion of a persistent shell across individual subprocess calls.

### CWD tracking

The working directory is tracked across calls:
- **Local**: written to a temp file after each command, read back by the parent
- **Remote**: backends emit `__HERMES_CWD_{session_id}__<path>__` stdout markers which are parsed and stripped from the output

The resolved CWD is exposed via `TERMINAL_CWD` and `_get_live_tracking_cwd(task_id)` — file tools use this to resolve relative paths.

### Approval gate

Two-tier dangerous command checking before execution:

1. **Hardline blocklist** — unconditional blocks (`rm /`, `mkfs`, `dd` to block device, fork bomb, `kill -1`, `reboot/shutdown`). Cannot be bypassed by `force=True` or any config.
2. **Dangerous patterns** — 47 patterns (`rm -r`, `chmod 777`, `git reset --hard`, `curl | sh`, SQL `DROP/DELETE`, etc.). Can be approved per-session, permanently via `command_allowlist` in config, or by the approval callback.

`force=True` on the tool call skips tier-2 checks (used after the user confirms via the UI). Approval callbacks are stored in TLS (`_callback_tls.approval`) — subagents get `_subagent_auto_deny` or `_subagent_auto_approve` depending on config (see `delegation.md`).

### Output and timeouts

- Output capped at **50KB**, kept as 40% head + 60% tail to preserve both early errors and final output
- ANSI escape sequences stripped before returning to the model
- Secrets redacted via `redact_sensitive_text()`
- Default timeout: **180s** (`TERMINAL_TIMEOUT`), hard cap **600s** (`TERMINAL_MAX_FOREGROUND_TIMEOUT`)
- Commands exceeding the foreground cap are rejected with a suggestion to use `background=True`
- Interrupts are flag-based via `tools/interrupt.py` — a process-level `set[int]` of interrupted thread IDs. `agent.interrupt()` calls `set_interrupt(True, tid)` for the execution thread and every active tool worker thread. The terminal backend's output-reading loop checks `is_interrupted()` on each iteration (sleeps ~0.2s between reads); when the flag is set it kills the process group (SIGTERM → 5s → SIGKILL)

### Background processes

`background=True` spawns via the process registry and returns immediately with a `session_id`. Results polled later with `process(action="poll"|"wait")`. Supports `notify_on_complete` and `watch_patterns` (regex match on output, rate-limited to 1 notification per 15s).

---

## Code Execution Tool — `tools/code_execution_tool.py`

Single tool: `execute_code(code)`. Runs a Python script in an isolated child process and returns `{status, output, tool_calls_made, duration_seconds}`. The script has access to a `hermes_tools` module exposing 7 tools — this is the key difference from `terminal_tool`, which runs single commands. `execute_code` is for multi-step logic with conditionals, loops, and tool calls woven together.

### Two transports

**Local (Unix Domain Socket):**
1. Parent generates a `hermes_tools.py` stub with UDS transport and starts a UDS server thread
2. Child process spawned with the script; child imports `hermes_tools` and calls tool functions which send JSON-RPC over the socket
3. Parent's RPC thread dispatches via `model_tools.handle_function_call()` and returns results

**Remote (Docker/SSH/Modal/etc., file-based RPC):**
1. Parent ships `hermes_tools.py` (file transport variant) + script to the sandbox via base64
2. Parent starts a polling thread that watches for `req_<N>` files written by the child
3. Child writes request files, parent dispatches and writes `res_<N>` response files (atomic `.tmp` + rename)
4. Adaptive polling: 50ms → 250ms backoff

### Available tools in sandbox

```python
SANDBOX_ALLOWED_TOOLS = {"web_search", "web_extract", "read_file", 
                          "write_file", "search_files", "patch", "terminal"}
```

`terminal` calls inside the sandbox are restricted to foreground-only — `background`, `pty`, `notify_on_complete`, and `watch_patterns` are blocked.

### Execution modes

| Mode | CWD | Python interpreter |
|---|---|---|
| `strict` | Isolated `/tmp` staging dir | `sys.executable` (hermes interpreter) |
| `project` (default) | Session's `TERMINAL_CWD` | Venv/Conda python if available, else `sys.executable` |

Project mode resolves project dependencies and relative paths. Strict mode is reproducible but can't see project packages.

### Environment scrubbing (local backend)

Child process gets a clean env — not inherited from parent:
- **Allowed prefixes**: `PATH`, `HOME`, `LANG`, `LC_`, `TERM`, `TMPDIR`, `VIRTUAL_ENV`, `PYTHONPATH`, `HERMES_*`
- **Blocked substrings**: `KEY`, `TOKEN`, `SECRET`, `PASSWORD`, `CREDENTIAL`, `AUTH`
- Passthrough vars from the registry (skills or user config) are re-added explicitly
- Socket path injected as `HERMES_RPC_SOCKET`

Output is still run through `redact_sensitive_text()` as a backstop in case secrets leak via disk.

### Resource limits

| Limit | Default | Config |
|---|---|---|
| Timeout | 300s | `code_execution.timeout` |
| Max tool calls | 50 | `code_execution.max_tool_calls` |
| Stdout cap | 50KB (40/60 head/tail) | `tool_output.max_bytes` |
| Stderr cap | 10KB | hardcoded |
| Min Python | 3.8 | hardcoded |

On timeout: SIGTERM → 5s → SIGKILL. Exit code 124 = timeout, 130 = interrupted.

---

## Clarify Tool — `tools/clarify_tool.py`

Single tool: `clarify(question, choices?)`. Pauses the agent to ask the user a question and returns their response as JSON `{question, choices_offered, user_response}`.

Two modes:
- **Multiple choice** — up to 4 predefined options. The platform UI always appends a 5th "Other (type your answer)" option automatically.
- **Open-ended** — `choices` omitted entirely, user types a free-form response.

### Dispatch

The tool itself contains no UI logic. The actual interaction is handled by a platform-provided `callback(question, choices) -> str` injected at registration time — CLI renders arrow-key navigation, gateway renders a numbered list. The tool validates input, calls the callback, and wraps the result in JSON.

If `callback=None` (subagents, cron, any context without a user present) the tool returns an error immediately without blocking. This is why `"clarify"` is in `_strip_blocked_tools()` — a subagent that can't reach a user should never be able to call it.

### Scope

Explicitly not for dangerous command confirmation — the schema description says so directly. That path goes through the terminal tool's approval gate. Clarify is for task ambiguity, trade-off decisions, post-task feedback, and offering to save skills or update memory.
