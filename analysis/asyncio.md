# asyncio — Event Loops and Coroutines

## The Problem Python Async Solves

Normal blocking code: call a function, wait for it, get the result. If that function does network I/O, your thread sits idle waiting for bytes. One thread = one request at a time.

Async flips this: instead of blocking, a function *suspends* at the I/O boundary, hands control back, and resumes when the data arrives. One thread can juggle hundreds of suspended-and-waiting operations simultaneously.

---

## The Event Loop

The event loop is the scheduler. It's a `while True` loop that:

```
while True:
    1. drain _ready queue        ← run all callbacks already queued
    2. compute timeout           ← how long until next scheduled timer fires?
    3. epoll_wait(timeout)       ← sleep until I/O ready OR timeout expires
    4. process I/O events        ← mark I/O callbacks as ready
    5. fire elapsed timers       ← mark timer callbacks as ready
    goto 1
```

It only enters `epoll_wait()` after `_ready` is **empty**. If there's work queued it never sleeps — it runs it first.

**Steps 4 and 5 don't execute callbacks — they only move them into `_ready`.**

Step 4: `epoll_wait` returns a list of `(fd, event)` pairs. For each ready fd the loop looks up the registered callback in `_selector._map` and appends it to `_ready`. Examples: TCP socket with data (HTTP response), subprocess stdout (MCP reply), self-pipe read end (`call_soon_threadsafe` doorbell).

Step 5: Any `TimerHandle` in `_scheduled` with `when <= loop.time()` is moved to `_ready`. Execution only happens back at step 1 — by then the loop makes no distinction between I/O and timer callbacks.

**Internal storage** — a suspended coroutine lives in exactly one of three places:

```
loop._ready              # deque — callbacks ready to run now
loop._scheduled          # heap  — TimerHandles sorted by deadline
loop._selector._map      # dict  — fd → callback, watched by epoll
```

**Timeout calculation** — computed by peeking at the soonest `_scheduled` entry: `timeout = max(0, soonest.when - loop.time())`. Edge cases:
- `_ready` non-empty after draining → `epoll_wait(0)`, don't sleep
- No timers, no pending work → large timeout, sleep until I/O arrives
- epoll can wake early — any I/O event (including self-pipe) interrupts the timeout

Coroutines carry no wait time themselves. The timeout only reflects explicit timer primitives. `await asyncio.sleep(5.0)` calls `loop.call_later(5.0, callback)` internally, inserting a `TimerHandle` into `_scheduled`. `await socket.read()` registers an fd with epoll — no timer created at all.

```python
loop = asyncio.new_event_loop()
loop.run_forever()   # starts the scheduler, never returns
```

---

## Coroutines

A coroutine is a function that can suspend mid-execution:

```python
async def fetch():
    result = await http.get(url)   # ← suspends here, hands control back to loop
    return result.text
```

`await` is the suspension point. The coroutine doesn't block — it yields control and the loop can run other coroutines while this one waits.

A coroutine object is **inert until scheduled**. Calling `fetch()` returns a coroutine object, nothing runs:

```python
coro = fetch()                      # nothing happens
loop.run_until_complete(coro)       # schedule + block calling thread until done
asyncio.ensure_future(coro)         # schedule, return Task immediately (fire and forget)
```

`ensure_future` is the inside-the-loop way to add a coroutine — must be called from async code already on the loop thread. Internally:

```python
task = Task(coro, loop=loop)   # allocates Task, sets up callbacks
loop.call_soon(task.__step)    # appends first step to _ready (no lock needed — on loop thread)
return task
```

`task.__step` calls `coro.send(None)` to advance the coroutine to its first `await`. The coroutine doesn't run until `_ready` is drained on the next iteration.

`Task` is a subclass of `asyncio.Future`, not `Handle`. The loop never sees Tasks directly — `_ready` only ever holds `Handle` objects. A Task schedules itself by appending `Handle(task.__step)` to `_ready` each time it's ready to advance. After each `await` suspends the coroutine, a new `Handle(task.__step)` is queued for the next wakeup. `TimerHandle` (used in `_scheduled`) is a subclass of `Handle` with an added `when` field.

---

## The Worker Thread Pattern

`run_forever()` doesn't return — it blocks the calling thread permanently. For code that mixes sync and async (e.g. a sync MemoryManager calling async HTTP), the standard pattern is:

```
worker thread:   loop.run_forever()   ← owns the loop, runs forever, does nothing else
main thread:     run_coroutine_threadsafe(coro, loop).result()
other threads:   run_coroutine_threadsafe(coro, loop).result()
```

The worker thread is the loop's dedicated executor. All application logic stays synchronous; async I/O is offloaded to the loop thread.

**Why not `asyncio.run()` per call?**

```python
asyncio.run(coro)   # creates a fresh loop, runs coro, closes loop
```

Simple but creates and destroys a loop on every call. aiohttp's `ClientSession` gets orphaned each time because its cleanup coroutines need to run on the same loop that created them — which is already closed. Produces "Unclosed client session" warnings and leaks file descriptors at scale. Fine for scripts, bad for long-running services.

---

## The Cross-Thread Problem — Self-Pipe and run_coroutine_threadsafe

The event loop is **not thread-safe** — calling `loop.run_until_complete()` from another thread raises `RuntimeError: This event loop is already running`. A loop can only be driven by one caller at a time.

The solution is `run_coroutine_threadsafe`, which bridges into the loop via the **self-pipe**:

Every asyncio event loop creates an internal self-pipe at startup — a connected socket pair:

```
write_end ──→ [ pipe ] ──→ read_end
```

The `read_end` is registered with epoll. The loop always watches it.

`run_coroutine_threadsafe` submits work by wrapping `ensure_future` in a closure that captures `coro` and a `concurrent.futures.Future`, then calling `call_soon_threadsafe(callback)`:

```python
def run_coroutine_threadsafe(coro, loop):
    future = concurrent.futures.Future()

    def callback():
        task = ensure_future(coro, loop=loop)
        asyncio.futures._chain_future(task, future)
        # when task completes → future.set_result(task.result())
        # when task raises    → future.set_exception(task.exception())

    loop.call_soon_threadsafe(callback)
    return future
```

`_chain_future` adds a done callback on the `asyncio.Task` that calls `set_result` or `set_exception` on the `concurrent.futures.Future` — waking the calling thread's `.result()`. Without this chain the Task completes but nobody notifies the `concurrent.futures.Future`, and the calling thread blocks forever. `future.set_result()` can't be called inline right after `ensure_future` because the Task hasn't run yet — only the done callback fires with the actual result.

```
calling thread:
    acquire lock → append(closure callback) → release lock → write pipe byte
                   ↑ lock held only for deque append — microseconds

loop thread wakes from epoll, drains _ready:
    popleft() → release lock → call closure()
                                    → ensure_future(coro)    ← Task created here, outside lock
                                        → Task(coro)
                                        → call_soon(task.__step)
```

The byte written to the pipe is a doorbell — meaningless payload. The real work is the closure in `_ready`. The pipe just wakes `epoll_wait` if the loop is sleeping.

`call_soon_threadsafe` vs `call_soon`:
- `call_soon_threadsafe` — lock + append + pipe write. Safe from any thread. Accepts any callable — not just coroutines. Returns immediately after the pipe write.
- `call_soon` — append only, no lock, no pipe. Only safe from the loop thread (`ensure_future` uses this).
- `call_later(delay, callback)` — creates `TimerHandle(when=loop.time() + delay)` and inserts into `_scheduled`. Fired when the timeout elapses and the loop moves it to `_ready`.

Both `call_soon_threadsafe` and `run_coroutine_threadsafe` return immediately after the pipe write — the difference is whether you can get a result back:
- `call_soon_threadsafe(fn)` — fire and forget only, no result mechanism. For plain callables (e.g. `event.set()`).
- `run_coroutine_threadsafe(coro)` — returns a `concurrent.futures.Future` linked to the Task via `_chain_future`. Blocking on `.result()` is optional — skip it for fire and forget, call it to wait for the result.

`asyncio.to_thread(fn)` is the inverse — submits a blocking sync callable to the default `ThreadPoolExecutor` and returns an awaitable. The loop stays unblocked while the thread runs:

```python
result = await asyncio.to_thread(blocking_fn)
# blocking_fn runs on thread pool thread
# loop is free to handle other coroutines during the call
# resumes with result when thread finishes
```

Internally it's `loop.run_in_executor(None, fn)` — when the thread completes, `future.set_result()` is called via `call_soon_threadsafe` back to the loop, which resumes the awaiting coroutine.

```
run_coroutine_threadsafe  — sync thread → async loop   (submit coro, block for result)
asyncio.to_thread         — async loop  → sync thread   (submit blocking fn, await result)
```

Task allocation and init happen outside the lock — no caller blocks waiting for Task construction. The calling thread never touches the Task; it just blocks on `.result()` of the returned `concurrent.futures.Future`:

```python
future = asyncio.run_coroutine_threadsafe(coro, loop)  # returns immediately after pipe write
result = future.result(timeout=30)                      # this blocks
```

`call_soon_threadsafe` also accepts `*args` stored in a `Handle`:

```python
loop.call_soon_threadsafe(fn, arg1, arg2)
# stores Handle(fn, (arg1, arg2)) in _ready
# when drained: fn(arg1, arg2)
```

`run_coroutine_threadsafe` uses a closure instead of args since `coro` and `future` are already in scope.

---

## concurrent.futures.Future vs asyncio.Future

`asyncio.Future` is not thread-safe — designed to be awaited from within the loop. `concurrent.futures.Future` uses `threading.Condition` (lock + wait/notify — blocks thread until notified, no spinning) internally, so `.result()` can block a non-loop thread safely and `.set_result()` from the loop thread wakes it up correctly.

The self-pipe is the bridge **into** the loop. The `concurrent.futures.Future` is the bridge **back out** — when the Task completes it calls `future.set_result(value)` which wakes the blocked `.result()` on the calling thread.

`asyncio.Future` is the async-side "I'll have a value later" primitive — awaited from within the loop:

```python
async def something():
    fut = loop.create_future()
    # hand fut to something that will call fut.set_result() later
    result = await fut   # suspends this coroutine, loop keeps running
```

One coroutine creates it, another resolves it. `asyncio.Task` is built on top of it — when a Task hits `await`, it's internally waiting on a Future. In practice you rarely create one manually; `ensure_future`, `gather`, `wait` handle it.

---

## One Shared Loop Per Process

When multiple objects need async I/O (e.g. one `HindsightMemoryProvider` per chat session), creating a loop per instance is wrong:

```
session_1 provider → loop_1 → ClientSession_1 → TCPConnector_1
session_2 provider → loop_2 → ClientSession_2 → TCPConnector_2
```

Closing loop_1 on shutdown strands aiohttp's cleanup coroutines — they can't run on a dead loop. At scale this leaks file descriptors.

One shared loop means one connection pool, reused across all instances, properly closed at process exit:

```python
# module-global
_loop: asyncio.AbstractEventLoop | None = None
_loop_thread: threading.Thread | None = None

def _get_loop():
    global _loop, _loop_thread
    if _loop and _loop.is_running():
        return _loop
    _loop = asyncio.new_event_loop()
    _loop_thread = threading.Thread(target=_loop.run_forever, daemon=True)
    _loop_thread.start()
    return _loop
```

The loop thread is intentionally NOT stopped on per-instance shutdown — other instances still need it. It runs as a daemon thread and is reclaimed on process exit.

---

## `_run_async` — Tool Handler Sync→Async Bridge

`model_tools._run_async(coro)` is the single source of truth for running async tool handlers from sync code. Three paths depending on the calling context:

```
# Common paths — tool handlers run on sync threads, no running loop
#
# CLI: main thread calls tool directly
main CLI thread  →  asyncio.get_running_loop() raises  →  _get_tool_loop()  →  tool_loop.run_until_complete(coro)
                                                            (process-global persistent loop, reused across all calls)

# Gateway: agent worker thread calls tool (gateway loop is on a separate thread, invisible here)
agent worker thread  →  asyncio.get_running_loop() raises  →  _get_worker_loop()  →  worker_loop.run_until_complete(coro)
                                                               (per-thread persistent loop in threading.local)

# Rare: tool handler called from within a coroutine already on the gateway loop
coroutine on gateway loop  →  asyncio.get_running_loop() succeeds  →  spin fresh thread with own loop
                               (can't run_until_complete — loop already running; can't block — would deadlock)
                               →  future.result(timeout=300), cancel via call_soon_threadsafe on timeout
```

Persistent loops (not `asyncio.run()`) in the CLI and worker paths because cached `httpx`/`AsyncOpenAI` clients stay bound to the loop that created them. `asyncio.run()` closes the loop after each call — those clients raise `RuntimeError: Event loop is closed` on GC cleanup.
