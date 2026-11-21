# Quiz 1 — Productionizing Agents

Knowledge check for [mod-207](../README.md). Answers are at the bottom; try each question before scrolling. Covers all four chapters.

## Questions

### 1. Picking an endpoint shape

Your agent typically takes 3–8 minutes end to end, with many model and tool calls along the way. Clients are web UIs that can't hold an HTTP request open that long. Which endpoint shape from Chapter 1 fits best?

- A. A synchronous `POST /agent/run` with a very long server timeout.
- B. A streaming SSE endpoint, so the connection is kept warm for the whole run.
- C. A background-job endpoint that returns a `job_id` immediately and lets the client poll or subscribe.
- D. A WebSocket per request so the server can push the final answer when it's ready.

### 2. The `BackgroundTasks` footgun

Why does Chapter 1 warn that FastAPI's `BackgroundTasks` is a poor choice for a run you "can't afford to lose"?

- A. `BackgroundTasks` is synchronous and blocks the event loop.
- B. It runs in the same process as the API, so it's lost on a crash or deploy — it is not a durable job store.
- C. It silently swallows exceptions, so failures never surface.
- D. It can only schedule CPU-bound work, not async model calls.

### 3. Operational concerns the endpoint owns

A traffic spike causes your agent service to exhaust the provider's rate limit and run out of memory because dozens of multi-step runs are in flight at once. Which Chapter 1 technique most directly addresses this?

- A. Switch from streaming to synchronous responses to reduce memory use.
- B. Cap concurrent in-flight runs with a semaphore or a queue.
- C. Move the API key into the container image to speed up startup.
- D. Replace Uvicorn with the dev server's reload mode to recycle workers faster.

### 4. Container hygiene

Which of the following is **inconsistent** with the Chapter 1 Dockerfile guidance?

- A. Pinning the base image (e.g., `python:3.12-slim`) rather than using `:latest`.
- B. Copying `requirements.txt` and installing dependencies before copying source, so the dependency layer caches.
- C. Baking `ANTHROPIC_API_KEY` into the image at build time so the container starts without `-e`.
- D. Creating a non-root user and `USER appuser`-ing before `CMD`.

### 5. Workflows vs. activities

In a Temporal-style split, which of the following belongs **inside the workflow function** and not an activity?

- A. The HTTP call to the model provider.
- B. The decision of which step to run next, given the recorded results of previous steps.
- C. Writing a record to the database.
- D. Reading the current wall-clock time via `datetime.now()`.

### 6. What replay buys you

Chapter 2's worker dies after step 7 of 10 completes. The run is resumed on a fresh worker. What happens to the model calls made during steps 1–7?

- A. They are re-executed and re-billed, which is why durable execution is often a net loss.
- B. Their recorded results are handed back from the history — the model calls do **not** run or get billed a second time.
- C. Only the last completed step's result is replayed; earlier steps run again.
- D. The workflow starts from step 7 with no replay; steps 1–6 are skipped entirely without feeding results forward.

### 7. Idempotency under retry

Replay protects *completed* activities. An activity that was interrupted mid-flight gets retried, so it can run twice. Which of your tool activities most urgently needs an idempotency key?

- A. A pure model call that just returns a summary string.
- B. An activity that charges a customer's card.
- C. An activity that reads a row from a database.
- D. An activity that computes a hash of some in-memory data.

### 8. Prefix ordering for the cache

You want prompt caching to pay off across a burst of related turns. How should you order the content of the prompt?

- A. Put the latest user turn first so the most important content is early.
- B. Interleave the stable system prompt with the latest user turn so both are near the cache breakpoint.
- C. Put the stable content (system, tools, retrieved context) first and the volatile user turn last, since the cache matches a prefix.
- D. The order doesn't matter — the cache hashes the whole prompt regardless of position.

### 9. Verifying the cache is working

You added `cache_control: {"type": "ephemeral"}` to your system prefix and want to know if the cache is actually paying off. Which signal from the API response is Chapter 3's direct proof?

- A. The wall-clock latency of the second call is lower than the first.
- B. `usage.cache_read_input_tokens` is non-zero on calls after the first — the cache is being read.
- C. `usage.output_tokens` drops relative to the uncached run.
- D. The model returns a `cached: true` field on the top-level response.

### 10. Routing needs a safety net

Chapter 3 says routing each turn to the smallest capable model is a big cost lever, but it names a trap you must defend against. Which defense does the chapter recommend?

- A. Always route through the largest model first, then re-route to the smallest for the final answer.
- B. Pair routing with an escalation path: when the cheap model's output fails a check, retry on the larger model, and measure the blend against an eval set.
- C. Never route a turn to anything but your most capable model; use caching alone for cost savings.
- D. Use a model-based router as the first routing strategy you ship, before any static rules.

### 11. HITL that survives a restart

You add a LangGraph `interrupt()` for a human approval and compile the graph with an **in-memory** checkpointer. During a demo the server restarts while a run is paused for approval. What goes wrong — and what does Chapter 4 recommend instead?

- A. Nothing — in-memory checkpointers are persisted to disk automatically; keep using it.
- B. The paused run is lost on restart. Use a database-backed checkpointer (e.g., Postgres) so paused state survives restarts and can be resumed from a different process.
- C. The run survives but the `thread_id` is scrambled; name threads manually to avoid this.
- D. The agent silently resumes from the beginning; use a longer interrupt payload to prevent re-entry.

### 12. The gate is also a security boundary

Chapter 4 calls the HITL approval gate a correctness *and* security boundary. Which pair of design rules speak to the "security" half?

- A. Surface enough context to decide; support edits as well as approve/reject.
- B. Make the gated action idempotent; keep state in memory for speed.
- C. Authorize the approver (verify they're allowed to approve) and make the gated action idempotent so a double-click or redelivered webhook fires the effect exactly once.
- D. Persist to a real store; use the same model for the agent and for the approval judge.

## Answer key

1. **C** — Multi-minute runs belong on a background-job endpoint; sync and streaming both run into client/proxy timeouts over that duration ([Chapter 1](../01-agent-apis-and-containers.md)).
2. **B** — `BackgroundTasks` runs in the same process; a restart or deploy loses the work, so it's only suitable for runs you can afford to lose ([Chapter 1](../01-agent-apis-and-containers.md)).
3. **B** — Concurrency limits (a semaphore or queue) cap in-flight runs so a spike doesn't exhaust rate limits or memory; this is one of the operational concerns the endpoint itself owns ([Chapter 1](../01-agent-apis-and-containers.md)).
4. **C** — The API key comes from the environment at runtime (`-e ANTHROPIC_API_KEY=...`), never baked into the image. The other three practices are explicitly recommended ([Chapter 1](../01-agent-apis-and-containers.md)).
5. **B** — The workflow is the deterministic orchestration layer; side-effecting I/O (model calls, DB writes, wall-clock reads) must live in activities ([Chapter 2](../02-durable-execution.md)).
6. **B** — On recovery the engine replays the workflow and hands back each completed activity's recorded result instantly; the steps' side effects (including model billing) are not re-run ([Chapter 2](../02-durable-execution.md)).
7. **B** — A card charge is the dangerous case: an interrupted activity may retry, so the downstream system must dedupe on a stable idempotency key. Pure/read-only activities are naturally safe to retry ([Chapter 2](../02-durable-execution.md)).
8. **C** — The cache matches a prefix; one changed token early invalidates everything after it, so stable content goes first and the volatile user turn last ([Chapter 3](../03-caching-and-routing.md)).
9. **B** — `cache_read_input_tokens` rising on subsequent calls is the direct proof the cache is being read; if it stays zero, the prefix is changing or too small ([Chapter 3](../03-caching-and-routing.md)).
10. **B** — The silent-quality-loss trap is defused by an escalation path (retry on a larger model when the cheap output fails a check) and by measuring the blend against an eval set ([Chapter 3](../03-caching-and-routing.md)).
11. **B** — An in-memory checkpointer loses every paused run on restart; a database-backed checkpointer lets a different process resume from the persisted `thread_id` ([Chapter 4](../04-hitl-with-persistence.md)).
12. **C** — Authorize the approver (security) and make the gated action idempotent (correctness under a retried resume). In-memory persistence and model reuse are not Chapter 4's rules ([Chapter 4](../04-hitl-with-persistence.md)).
