# Provider Architecture

## Memory Manager — Unified Provider Interface

The agent never talks to a memory provider directly. `MemoryManager` is the single interface — it fans out to all registered providers, error-isolates each call (try/caught independently), and presents a unified surface to the agent.

### Provider Registration

- **`BuiltinMemoryProvider`** — always registered first, cannot be removed
- **External provider** — max one (honcho, mem0, holographic, supermemory, etc.), set via `memory.provider` in `config.yaml`. A second registration attempt is rejected with a warning.
- No configured provider → only builtin runs, no long-term memory extraction

### Per-Turn Lifecycle Hooks

The manager is not just a session-end concern — it hooks into every stage:

```
on_turn_start()        ← before each turn (token budget, model, platform passed in)
prefetch_all()         ← recalls relevant memories before model sees the message
queue_prefetch_all()   ← starts background recall for the next turn
sync_all()             ← after each completed turn, syncs to all providers
on_pre_compress()      ← providers can reshape messages before context compression
on_session_switch()    ← when /resume switches to a different session_id
on_delegation()        ← when a subagent completes
on_session_end()       ← session expiry/shutdown
```

`prefetch_all` is blocking — called at turn start, waits for each provider's recall and returns the merged result to inject into context. `queue_prefetch_all` is non-blocking — called at turn end, kicks off background recall so the result is pre-warmed by the time the next turn's `prefetch_all` fires.

This creates a one-turn lag: the pre-warmed result was recalled using the *previous* turn's query. If the user changes topic between turns the injected context is stale. There is no topic-shift detection or re-query with the new message — the model receives the stale block inside `<memory-context>` and must judge its relevance itself. Turn 1 always gets no pre-warmed context (no prior `queue_prefetch_all` has fired); providers that do synchronous recall (e.g. Holographic) still work correctly on turn 1 since `prefetch_all` calls them blocking with the real query.

`MemoryManager` is a pure push bus — it delivers lifecycle events to all providers unconditionally and each provider decides independently whether to act:

```
MemoryManager.sync_all()
    → Hindsight.sync_turn()     ← checks _turn_counter % N, maybe enqueues retain
    → Holographic.sync_turn()   ← pass (no-op, tools only)
    → ByteRover.sync_turn()     ← curates if len(user_content) > 10 chars
```

This extends to every hook: `prefetch()`, `on_session_end()`, `on_pre_compress()` all follow the same pattern. A provider can inspect message content, check counters, detect keywords, or apply any heuristic before deciding to act. The manager imposes no constraints on the decision logic.

### What Providers Contribute

Beyond lifecycle hooks, providers actively contribute to the agent's runtime:

- **`system_prompt_block()`** — each provider can inject recalled memories directly into the system prompt every turn
- **`get_tool_schemas()`** — providers can expose callable tools to the agent (e.g. `recall_memory`, `save_memory`)

### `shutdown_all()` Order

Runs in **reverse registration order** — last-registered provider shuts down first, ensuring clean teardown dependency order.

---

## BuiltinMemoryProvider — `tools/memory_tool.py`

Always-on, zero-config, no external deps. Backed by two flat markdown files:

```
~/.hermes/memories/MEMORY.md   ← agent's personal notes (env facts, conventions, lessons learned)
~/.hermes/memories/USER.md     ← user profile (preferences, habits, communication style)
```

Files are only created on the first `memory` tool call — empty until the agent writes something.

### Frozen Snapshot Pattern

The most important design decision in the builtin:

```
agent initializes → load_from_disk()
                        ↓
            _system_prompt_snapshot frozen ONCE
                        ↓
    injected into system prompt, never mutated mid-session
                        ↓ (prefix cache stays stable for entire session)

mid-session tool write → MEMORY.md on disk immediately (durable)
                       → NOT reflected in current session's system prompt
                       → visible on next agent initialization
```

Writes are durable but invisible to the current session. The model doesn't see its own mid-session writes until the agent is re-initialized.

### Cross-Session Leakage

Because writes go to disk immediately and the path is profile-scoped (not user/channel scoped):

```
Session A (turn 5) writes to MEMORY.md
                          ↓
Session B starts (different channel/user)
    → load_from_disk() reads Session A's mid-session write
    → injected into Session B's system prompt
```

Session B inherits Session A's learnings **before Session A ends** — no session boundary needed for write propagation. This works well for single-user CLI; in multi-user gateway it's a leakage vector. Per-user scoping is why external providers exist.

### Storage Mechanics

- **Entry delimiter:** `§` (section sign) — entries can be multiline
- **Char limits** (model-independent, not tokens): `MEMORY.md` 2,200 chars · `USER.md` 1,375 chars
- **Writes:** atomic `tempfile.mkstemp` + `atomic_replace` — crash-safe
- **File locking:** `fcntl` (Unix) / `msvcrt` (Windows) on a separate `.lock` file — concurrent gateway sessions safe
- **Before every mutation:** full file re-read under lock (`_reload_target`) to pick up concurrent writes. Files are ~3.5KB max so full re-read is effectively free.

### Tool Interface — LLM-Driven, No Automatic Trigger

Single `memory` tool, three actions:

```
add     → append new entry (rejects exact duplicates, enforces char limit)
replace → find by substring match, swap content
remove  → find by substring match, delete
```

**The agent decides when to call it** — no programmatic trigger, no session-end extraction pass. The tool schema instructs the model to call proactively on:
- User corrections or explicit "remember this"
- User preferences, habits, personal details
- Environment facts (OS, tools, project structure)
- Conventions, API quirks, workflow specifics

Explicitly **not** for: task progress, completed-work logs, temporary state, things easily re-discovered.

### At the Char Limit

Hard reject — no eviction, no truncation, no LRU:

```python
{
  "success": False,
  "error": "Memory at 2,200/2,200 chars. Replace or remove existing entries first.",
  "current_entries": [...],   # ← returned so model can decide what to evict
}
```

The eviction decision is delegated back to the model. It sees the full entry list and must consciously choose what to remove. No silent pruning.

### Security Scanning

Every `add`/`replace` runs `_scan_memory_content()` before accepting. Blocks:
- Prompt injection (`ignore previous instructions`, `you are now`, `system prompt override`, etc.)
- Exfiltration via curl/wget referencing secret env vars
- SSH backdoor patterns, reading `.env`/credentials files
- Invisible unicode characters (steganographic injection)

---

## Holographic Memory Provider — `plugins/memory/holographic/`

Fully local, zero external deps beyond optional numpy. Backed by SQLite. Implements `MemoryProvider` ABC alongside BuiltinMemoryProvider — both can run simultaneously.

### SQLite Schema

```
facts           → id, content, category, tags, trust_score, created_at, used_at, use_count
entities        → id, name, normalized_name
fact_entities   → fact_id, entity_id  (join table)
memory_banks    → category, vector BLOB, updated_at
facts_fts       → VIRTUAL TABLE (FTS5 over facts.content)
```

**facts**: the raw storage row. `content` is whatever string the LLM passed to `fact_store(action='add')`. `trust_score` starts at `default_trust` (0.5) and drifts via feedback.

**entities**: normalized names extracted from fact content. Same entity name always resolves to the same row — entity resolution deduplicates "TypeScript" and "typescript" to one entry.

**fact_entities**: the relationship table. "it's Thursday" → entity "Thursday"; "Parth uses TypeScript" → entities "Parth" + "TypeScript". Many-to-many.

**memory_banks**: one row per category. Stores a serialized 1024-dim float vector — the HRR superposition of all facts in that category. Rebuilt from scratch on every `add_fact()` call.

---

### FTS5 Virtual Table

`facts_fts` is SQLite's full-text search extension. It maintains an internal inverted index over `facts.content`.

Without it, `search("deploy")` would be a `LIKE '%deploy%'` scan — O(n) across every row. FTS5 is O(log n) with BM25 ranking (more specific matches rank higher).

SQLite triggers on `facts` keep it auto-synced — INSERT/UPDATE/DELETE on `facts` mirrors into `facts_fts` automatically. No manual sync needed. It stores no new data — just a different index structure over the same content.

---

### `add_fact()` Pipeline

```
LLM calls fact_store(action='add', content="...", category="user_pref")
        ↓
INSERT into facts (content, category, trust=default_trust)
        ↓
entity extraction  → regex/simple parse on content string
        ↓
entity resolution  → normalize name, lookup or INSERT into entities
        ↓
link               → INSERT into fact_entities (fact_id, entity_id)
        ↓
compute HRR vector → _hrr_vector(content)
        ↓
rebuild bank       → load all HRR vectors for category, bundle → store in memory_banks
```

Fact quality is entirely LLM-governed — no normalization layer exists between the tool call and storage. A vague fact ("user likes things") gets stored, vectorized, and retrieved as if well-formed. The tool schema description instructs the model to write specific, entity-rich facts, but this is prompt-governed, not structurally enforced.

---

### Hybrid Retrieval Pipeline (3 stages)

`search()` runs all three stages sequentially:

**Stage 1 — FTS5 candidates**
```sql
SELECT fact_id, rank FROM facts_fts WHERE facts_fts MATCH ? ORDER BY rank LIMIT N*3
```
BM25-ranked keyword matches. Overshoots by 3× to give the next stages room to rerank.

**Stage 2 — Jaccard rerank**
```
jaccard(query_tokens, fact_tokens) = |intersection| / |union|
```
Token-level overlap between the query and each candidate. Penalizes false positives from FTS5 (e.g. short common words matching irrelevant facts).

**Weakness:** short facts have smaller union sets, so they score disproportionately high. A 3-token fact matching 1 query token scores 1/3=0.33; a detailed 10-token fact matching the same token scores 1/10=0.10, even though the longer fact is likely more useful. Trust scoring and HRR similarity in stage 3 can override this — a short low-trust fact loses to a longer high-trust one in the final weighted blend.

**Stage 3 — Trust weighting + temporal decay**
```
final_score = fts_weight * fts_score
            + jaccard_weight * jaccard_score
            + hrr_weight * hrr_similarity
            + trust_bonus * trust_score
            - temporal_decay_penalty
```
`temporal_decay_half_life` is 0 by default (no decay). When set, recently-used facts are boosted and stale facts penalized.

---

### HRR Algebra

HRR (Holographic Reduced Representation) encodes structured relationships into fixed-size vectors while preserving algebraic compositionality. Three primitives:

#### Bind — circular convolution

Associates two concepts. Result has same dimensionality as inputs.

```
c = a ⊛ b
c[k] = Σ(j=0..N-1)  a[j] * b[(k-j) mod N]
```

The `mod N` makes it circular — shifts wrap around. For each of N output positions, N multiplications: **O(N²)** direct.

Quasi-invertible via conjugate unbind:
```
a ≈ bundle ⊛ b⁻¹  =  ifft( fft(bundle) * conj(fft(b)) )
```

Recovery is approximate — noise accumulates as more facts are bundled into the bank.

#### Bundle — superposition

Combines N vectors into one that "contains" all of them:
```
bank = normalize( fact_1 + fact_2 + fact_3 + ... )
```

The bank does not store facts separately — they are superimposed. Unbinding entity E from the bank produces a noisy approximation of all facts associated with E. Noise increases with N (more facts = more interference). Used as a filter for ranked similarity, not exact reconstruction.

#### Similarity — cosine

```
similarity(a, b) = dot(a, b) / (norm(a) * norm(b))
```

After unbinding produces a noisy result vector, cosine similarity against all candidate fact vectors ranks the closest matches above the noise floor.

#### Entity vectors are deterministic

Each entity name gets a fixed random vector seeded from its name hash. Same name → same vector, always, across sessions and additions. This is the property that makes algebraic queries stable — "Parth" always has the same vector regardless of when the fact was stored.

---

### Algebraic Query Types

**`probe(entity)`** — what do we know about this entity?
```
result = bank ⊛ E⁻¹
rank fact vectors by cosine(result, fact_vec)
```
Asks: which fact vectors, when bound with E, would reconstruct something close to the bank?

**`related(entity)`** — what entities are structurally connected?
```
result = bank ⊛ E⁻¹
rank entity vectors by cosine(result, entity_vec)   ← entity vectors, not fact vectors
```
Same unbind, different comparison set. High similarity to E_typescript means facts about the probed entity frequently co-occur with TypeScript.

**`reason(entities=[E1, E2])`** — AND semantics across entities
```
partial_1 = bank ⊛ E1⁻¹
partial_2 = bank ⊛ E2⁻¹
result    = elementwise_min(partial_1, partial_2)
rank fact vectors by cosine(result, fact_vec)
```
`min()` is fuzzy AND — only dimensions where both unbinds have signal survive. Facts mentioning only one entity are suppressed; facts mentioning both are amplified.

**`contradict()`** — conflicting claims
```
for each pair (f_i, f_j) sharing entities:
    score = entity_overlap(f_i, f_j) × content_divergence(f_i, f_j)
```
High entity overlap = same subject. High content divergence (low cosine between fact vectors) = different claims. Geometric — no language understanding required. Capped at 500 facts to avoid O(n²) blow-up.

---

### FFT-based Convolution

Direct circular convolution is O(N²). At N=1024: ~1M multiplications per bind call.

The convolution theorem states: convolution in time domain = elementwise multiplication in frequency domain.
```
fft(a ⊛ b) = fft(a) * fft(b)   ← elementwise complex multiply
```

Therefore:
```
a ⊛ b = ifft( fft(a) * fft(b) )
```

Three FFT passes + N multiplications = **O(N log N)**. At N=1024: ~30K operations vs ~1M. Result is mathematically identical to direct convolution — no approximation introduced here.

#### How FFT achieves O(N log N) — Cooley-Tukey

The naive DFT computes N frequency bins, each requiring N multiplications:
```
X[k] = Σ(n=0..N-1)  x[n] * e^(-2πi·k·n/N)
```

Cooley-Tukey splits the N-point DFT into two N/2-point DFTs (even and odd indexed elements):
```
X[k]       = E[k] + e^(-2πi·k/N) * O[k]
X[k + N/2] = E[k] - e^(-2πi·k/N) * O[k]
```

Each half-size DFT costs half the work. Recurse log₂(N) times — the N halves bottom out at 1-point DFTs (trivial). Total: N operations × log₂(N) levels = **O(N log N)**.

The exponential `e^(-2πi·k/N)` is a twiddle factor — a rotation in the complex plane. The `±` split is the butterfly operation, combining pairs of results from each recursion level.

#### Why conjugate for unbind

Bind in frequency domain: `C = fft(a) * fft(b)`

To recover `a`: `C / fft(b)` = `C * fft(b)⁻¹`

For complex `z = r·e^{iθ}`, the multiplicative inverse is `e^{-iθ}/r`. The conjugate `z* = r·e^{-iθ}` flips the phase but keeps magnitude. When HRR vectors are unit-norm (`r ≈ 1`), conjugate ≈ inverse:
```
ifft( fft(bundle) * conj(fft(b)) )  ≈  a
```

The approximation degrades as more facts bundle into the bank — each additional fact adds frequency-domain noise. Retrieval is therefore similarity-ranked, not exact: unbind recovers a noisy approximation, cosine finds the closest real fact vector.

#### Without numpy

HRR weight redistributed to FTS(0.6) + Jaccard(0.4). `probe()` and `reason()` fall back to SQL joins on `fact_entities`. Functionally correct, algebraically inert — no bank queries, no bind/unbind, no compositional reasoning.

---

### Trust Scoring

Asymmetric feedback:
```
helpful   → trust += 0.05   (small reward)
unhelpful → trust -= 0.10   (larger penalty)
```

Penalizes bad facts twice as fast as it rewards good ones. Trust floor is the `min_trust_threshold` config (default 0.3) — facts below this are filtered from all retrieval results. No eviction: low-trust facts stay in the DB, just invisible to queries above the threshold.

---

### `on_memory_write()` Hook

When the BuiltinMemoryProvider writes to MEMORY.md or USER.md, it fires `on_memory_write(action, target, content)`. Holographic mirrors the write:
```python
if action == "add" and content:
    category = "user_pref" if target == "user" else "general"
    store.add_fact(content)
```

Builtin memory writes automatically populate the structured fact store without any extra LLM tool calls.

---

### Auto-Extraction (disabled by default)

`on_session_end()` runs `_auto_extract_facts(messages)` if `auto_extract: true`. Regex patterns on `role == "user"` messages only:
```
I prefer/like/love/use/want/need ...
my favorite/preferred/default X is ...
we decided/agreed/chose to ...
the project uses/needs/requires ...
```

Stores the whole matching sentence (capped at 400 chars) as a fact. Naive — no semantic parsing. Disabled by default because false positive rate is high. Trust scoring mitigates this: auto-extracted facts that prove useless sink toward the `min_trust_threshold` floor via `fact_feedback`.

---

## Hindsight Memory Provider — `plugins/memory/hindsight/`

Long-term memory with knowledge graph, entity resolution, and multi-strategy retrieval. Three deployment modes: cloud API, local embedded daemon, local external instance.

### What Hindsight Actually Is

Two separable layers: the **client-side adapter** (`__init__.py`) and the **Hindsight service** (cloud API or local daemon). The intelligence — entity extraction, knowledge graph, vector embeddings, reranking — lives in the service. The adapter is purely orchestration.

### The Service Model

**Banks** are namespaced memory stores. A bank has a `bank_id` (the name), an optional `bank_mission` (framing for reflect reasoning), and a `bank_retain_mission` (steers what the LLM extracts during retain). All reads and writes are scoped to a bank.

**Bank ID templates** solve cross-user leakage that BuiltinMemoryProvider and Holographic can't address:
```
bank_id_template: "hermes-{platform}-{user}"
→ hermes-telegram-user_111
→ hermes-telegram-user_222   ← completely separate bank
```
Placeholders: `{profile}`, `{workspace}`, `{platform}`, `{user}`, `{session}`. Missing placeholders collapse cleanly (`hermes-{user}` with no user → `hermes`).

**Documents** are the unit of storage. Upserting the same `document_id` updates the document — it doesn't append rows. The adapter sends *all accumulated turns* on every retain call:

```python
content = "[" + ",".join(self._session_turns) + "]"
# turn 1: sends [turn_1]
# turn 2: sends [turn_1, turn_2]      ← same document_id
# turn 3: sends [turn_1, turn_2, turn_3]
```

The server always sees the full growing transcript, not deltas. Entity extraction benefits from full context.

**`retain_async=True`** (default): server acknowledges receipt immediately, processes entity extraction and indexing in the background. Content may not be immediately searchable. `False` blocks until fully indexed.

**Local embedded** runs PostgreSQL internally (not SQLite), plus local embedding models and a reranker. LLM needed only for extraction and reflect synthesis. Starts as a daemon, idles out after 5 minutes (`idle_timeout=300`). Logs: `~/.hermes/logs/hindsight-embed.log`.

### Recall vs Reflect

**`recall`** — retrieval: semantic search + keyword + entity graph traversal + rerank. Returns ranked result objects. Fast. Default prefetch method.

**`reflect`** — synthesis: an LLM pass over all retrieved relevant memories to produce a coherent answer. Returns a paragraph, not a list of facts. Expensive. Use for "what do you know about X" questions.

`recall_budget` (`low`/`mid`/`high`) controls server-side thoroughness — depth of entity graph traversal, number of strategies used, reranking model quality.

### Three Memory Modes

- `hybrid` — auto-inject into context AND expose tools (default)
- `context` — auto-inject only, no tools exposed
- `tools` — tools only, no auto-inject; LLM must explicitly call

In `context` mode `get_tool_schemas()` returns `[]` — no tools registered with the agent.

### Threading Architecture

Three concurrent execution contexts per provider instance:

```
main thread      → sync_turn() enqueues jobs, prefetch() reads cached result
writer thread    → drains _retain_queue serially, one aretain_batch at a time
prefetch thread  → fires arecall/areflect per turn, writes to _prefetch_result
```

Plus one **module-global async event loop** shared across all provider instances in the process (see `asyncio.md`):

```python
_loop: asyncio.AbstractEventLoop | None = None   # one per process
_loop_thread: threading.Thread                   # runs loop.run_forever()
```

All async calls go through `asyncio.run_coroutine_threadsafe(coro, loop).result(timeout=N)`.

The loop is intentionally NOT stopped on per-instance shutdown — other provider instances (one per chat session in the gateway) still need it.

### The Writer Queue

`sync_turn()` can't block the main thread waiting 30-40s for an HTTP retain call. It enqueues a closure instead:

```
sync_turn()
    → builds content snapshot (immutable at enqueue time)
    → puts _do_retain closure onto _retain_queue
    → returns immediately

writer thread (daemon)
    → blocks on queue.get(timeout=1.0)
    → calls job() → aretain_batch() → HTTP call
    → task_done()
    → repeat
```

The snapshot at enqueue time is critical — `_session_turns` keeps growing, so capturing by reference would send future turns' data under past document IDs. The closure captures immutable values: `content`, `metadata_snapshot`, `document_id`, `bank_id`.

**Shutdown sequence:**
```
_shutting_down.set()          ← stops new enqueues
_retain_queue.put(SENTINEL)   ← wakes writer for clean exit
writer.join(timeout=10s)      ← waits for in-flight work
client.aclose() on shared loop ← releases aiohttp sessions
```

### Retain Every N Turns — The Accumulation Model

`retain_every_n_turns` is user-set via config (default 1). The provider reads it in `initialize()` and checks it internally in `sync_turn()` — the MemoryManager has no involvement in this decision.

With N=3:
```
turn 1: _session_turns=[t1],         _turn_counter=1  → 1%3≠0, buffer only
turn 2: _session_turns=[t1,t2],      _turn_counter=2  → 2%3≠0, buffer only
turn 3: _session_turns=[t1,t2,t3],   _turn_counter=3  → 3%3=0, RETAIN
    content sent = "[t1, t2, t3]"   ← all accumulated turns, same document_id

turn 6: _turn_counter=6  → RETAIN
    content sent = "[t1,t2,t3,t4,t5,t6]"   ← same document_id, upserted
```

The document grows cumulatively — each retain upserts the same `document_id` with a longer transcript. Not incremental deltas. The server always sees the full conversation.

**Data-loss risk:** turns buffered between retains are lost on crash. N=1 loses nothing. N=3 can lose up to 2 turns. `on_session_switch()` flushes the buffer at session boundaries to prevent silent loss across `/new` or `/resume`.

**Why N>1?** Retain involves an HTTP call and server-side LLM extraction. N=1 on every turn means many API calls on high-frequency chats. N=3–5 trades some crash durability for fewer server calls and less extraction noise from very short exchanges.

### Auto Injection — Two Distinct Injection Points

There is a static layer and a dynamic layer:

```
system_prompt_block()   ← injected ONCE at agent init, frozen for the session
                           For Hindsight: just a header ("Active. Bank: hermes, budget: mid")

prefetch() result       ← injected fresh EVERY turn, before LLM sees the message
                           For Hindsight: actual recalled memories as bullet points
```

The per-turn flow:
```
user message arrives
        ↓
MemoryManager.queue_prefetch_all(user_message)
    → Hindsight fires background thread: arecall(query=user_message[:800])
    → result cached in _prefetch_result
        ↓
[LLM starts loading...]
        ↓
MemoryManager.prefetch_all()
    → calls prefetch() on each provider
    → Hindsight: joins prefetch thread (3s timeout), reads _prefetch_result
    → returns:
        "# Hindsight Memory (persistent cross-session context)
         - Parth prefers TypeScript
         - deploy process uses GitHub Actions..."
        ↓
injected into context above the conversation turn
        ↓
LLM sees: [system prompt] + [memory block] + [conversation history] + [user message]
```

`recall_max_input_chars=800` caps the query — long user messages get truncated before being sent as the recall query. The first 800 chars are assumed to capture intent well enough.

### Prefetch Two-Phase

```
queue_prefetch(user_message)
    → spawns prefetch thread
    → thread calls arecall() or areflect() on shared loop
    → result cached in _prefetch_result under _prefetch_lock

[LLM is loading / generating first tokens...]

prefetch()   ← MemoryManager calls this
    → join(timeout=3s) on prefetch thread
    → read and clear _prefetch_result
    → return formatted block
```

The prefetch thread races the LLM startup time. 3s join timeout — if Hindsight is slow, the turn proceeds without memory rather than blocking indefinitely.


### Session Switch Flush

On `/resume`, `/new`, `/reset`, `/branch`:

```
1. If _session_turns non-empty:
       snapshot all state (old document_id, old session_id, turns)
       enqueue flush job → aretain_batch under OLD document_id
       (serialized through same writer queue — FIFO, no race)

2. Drain + clear prefetch thread and cached result

3. Rotate:
       _session_id = new_id
       _document_id = "{new_id}-{timestamp}"   ← fresh document
       _session_turns = []
       _turn_counter = 0
```

Without step 1: buffered turns lost when `retain_every_n_turns > 1`. Without step 3's new `document_id`: new session's first retain overwrites old session's document.

