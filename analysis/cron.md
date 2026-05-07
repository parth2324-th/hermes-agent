# Cron — `cron/jobs.py` + `cron/scheduler.py`

## What It Is

A production-grade job scheduler built into the gateway. Cron jobs are agent runs triggered on a schedule — the agent gets a prompt, executes with a restricted toolset, and the result is delivered to a platform channel. Jobs are defined by operators or created by the agent itself via the `cronjob` tool.

---

## Data Model

Jobs live in `~/.hermes/cron/jobs.json`. Key fields:

| Field | Purpose |
|---|---|
| `id` | 12-char hex UUID |
| `prompt` | The instruction sent to the agent |
| `skills` | Ordered list of skill names prepended to the prompt |
| `schedule` | `{kind: "once|interval|cron", run_at/minutes/expr}` |
| `next_run_at` / `last_run_at` | ISO timestamps driving the tick |
| `repeat` | `{times: null\|int, completed: int}` — null = forever |
| `deliver` | Delivery target (see Delivery section) |
| `origin` | `{platform, chat_id, thread_id}` — where job was created |
| `enabled_toolsets` | Per-job toolset restriction |
| `workdir` | Absolute project directory — loads AGENTS.md/CLAUDE.md |
| `context_from` | Job IDs whose last output is injected as context |
| `script` | Path relative to `~/.hermes/scripts/` — pre-run script |
| `no_agent` | Skip LLM entirely, run script and deliver stdout |
| `model` / `provider` / `base_url` | Per-job overrides |
| `state` | `scheduled\|paused\|completed\|error` |

A threading lock `_jobs_file_lock` (`cron/jobs.py:43`) serialises all load→modify→save cycles across concurrent job threads.

---

## Scheduling

### The ticker

At gateway startup, `_start_cron_ticker()` (`gateway/run.py:14869`) spawns a background daemon thread that calls `cron.scheduler.tick()` every 60 seconds (configurable).

`tick()` (`cron/scheduler.py:1424`) acquires a file-level lock (`~/.hermes/cron/.tick.lock`, `fcntl` on Unix) before doing anything — this prevents overlap between the gateway ticker, a standalone daemon process, and manual `hermes cron tick` invocations.

### Due detection and missed-fire handling

`get_due_jobs()` returns jobs where `next_run_at <= now`. For jobs that were missed (process was down), a grace window decides whether to fire or fast-forward:

- **One-shot jobs** — 120s grace. Beyond that, skipped.
- **Recurring jobs** — grace = half the period, clamped to [120s, 2h]. A daily job has a 2-hour catchup window; a 10-minute job has 5 minutes.

Beyond the grace window the job is fast-forwarded to the next scheduled occurrence — no burst-firing on restart.

### At-most-once semantics

For recurring jobs, `advance_next_run()` preemptively advances `next_run_at` **before** execution. If the process crashes mid-run the job won't re-fire on restart. One-shot jobs skip this — they're allowed to retry.

### Parallel execution

Due jobs run concurrently via `ThreadPoolExecutor` (capped by `HERMES_CRON_MAX_PARALLEL`). Exception: jobs with `workdir` set run sequentially because they mutate `os.environ["TERMINAL_CWD"]`, which is process-global.

---

## Execution — `run_job()`

Entry point: `cron/scheduler.py:846`.

### LLM agent path (default)

1. **Script pre-run** (optional) — if `script` is set, runs before building the prompt. If the script outputs `{"wakeAgent": false}`, the agent is skipped entirely (silent conditional run). Script stdout is injected into the prompt as context.

2. **Context injection** — if `context_from` is set, the most recent output file from each referenced job ID is read and prepended (truncated to 8000 chars).

3. **Skill loading** — each skill name in `skills` is resolved via `skill_view()` and prepended to the prompt. Missing skills log a warning but don't abort.

4. **Agent construction** — a fresh `AIAgent` per job run. Key isolation:
   - `skip_context_files=True` unless `workdir` is set (workdir loads AGENTS.md/CLAUDE.md from the project)
   - `disabled_toolsets=["cronjob", "messaging", "clarify"]` — prevents recursive scheduling, message spam, and clarification loops
   - `HERMES_CRON_SESSION=1` set in env — signals approval.py to apply `cron_mode` rules
   - ContextVars carry delivery targets into the agent thread

5. **Inactivity timeout** — default 600s (`HERMES_CRON_TIMEOUT`). Main thread polls agent activity every 5s; on timeout calls `agent.interrupt()`.

6. **Output** — saved to `~/.hermes/cron/output/{job_id}/{timestamp}.md`.

### No-agent script path (`no_agent=True`)

Script runs directly. Stdout is the response. Empty stdout = silent (no delivery). Non-zero exit = deliver error alert.

---

## Delivery

The `deliver` field controls where output goes. Resolution order in `cron/scheduler.py:173`:

| Value | Behaviour |
|---|---|
| `"origin"` | Deliver to the chat where the job was created |
| `"local"` | Save to output file only, no platform delivery |
| `"platform"` | Use that platform's home channel |
| `"platform:chat_id[:thread_id]"` | Explicit target |
| comma-separated | Multi-delivery to all listed targets |

If `deliver="origin"` but origin is unavailable (job created via CLI), falls back to the first platform with a configured home channel.

Home channels per platform are set via env vars (`TELEGRAM_HOME_CHANNEL`, `DISCORD_HOME_CHANNEL`, etc.) or the `/sethome` slash command. See `gateway.md § Home channel`.

Delivery prefers a live gateway adapter (E2EE support for Matrix). Falls back to standalone HTTP if the adapter is unavailable. `MEDIA:` tags in the response are extracted and sent as native file attachments. A job metadata header wraps the response (toggleable via `cron.wrap_response` in config.yaml).

---

## Job Creation

Three paths:

1. **CLI** — `hermes cron create` (`hermes_cli/cron.py`)
2. **Slash command / agent tool** — `cronjob()` in `tools/cronjob_tools.py:257`. The agent calls this to schedule its own recurring tasks.
3. **Direct API** — `cron.jobs.create_job()`

On creation (`cron/jobs.py:422`):
- Schedule is parsed via `parse_schedule()` — accepts `"30m"`, `"every 2h"`, cron expressions, ISO timestamps
- Prompt injection scanning blocks exfiltration patterns and invisible unicode
- Script paths validated for containment under `~/.hermes/scripts/` — no absolute paths
- `context_from` job IDs validated to exist
- `next_run_at` pre-computed
- `deliver` auto-set to `"origin"` if origin available, else `"local"`
- One-shot jobs auto-set `repeat.times=1`

---

## Safety

- **Approval mode** (`approval.py:730`) — `approvals.cron_mode` config: `"deny"` (default, blocks dangerous commands in unattended runs) or `"approve"` (allow all). The `HERMES_CRON_SESSION=1` env var is how approval.py detects it's in a cron context.
- **Disabled toolsets** — `cronjob`, `messaging`, `clarify` always stripped from cron agents regardless of config.
- **Prompt scanning** — injection and exfiltration patterns blocked at creation time.
- **No retries** — failed jobs record `last_status="error"` and wait for the next scheduled time. Manual re-trigger via `hermes cron run <job_id>`.

---

## `cron/jobs.py` — Key Sections

| Section | Lines | Purpose |
|---|---|---|
| Config + locks | 32–96 | Paths, `_jobs_file_lock`, secure dir setup |
| Schedule parsing | 99–335 | `parse_schedule()`, `compute_next_run()`, grace logic, timezone handling |
| Job CRUD | 337–641 | create, get, list, update, pause/resume, trigger, remove |
| `advance_next_run` | 776–802 | Pre-execution next-run advancement (at-most-once) |
| `mark_job_run` | 703–773 | Post-execution state update, repeat counter, next-run recompute |
| Output saving | 907–932 | Markdown output to disk with secure permissions |
| Skill ref rewriting | 939–1050 | Curator integration — rewrites job skill lists when skills consolidate |
