# Quiz 1 — Multi-Agent Systems Implementation

Knowledge check for [mod-204](../README.md). Answers are at the bottom; try each question before scrolling. Covers all five chapters.

## Questions

### 1. When orchestrator-worker beats a single agent

Chapter 1 argues for orchestrator-worker over a single agent in which of these situations?

- A. A five-step refactor where each step reads the compiler output from the previous step before deciding what to do next.
- B. "Compare the managed Kubernetes offerings of AWS, GCP, and Azure on cost, autoscaling, and GPU support" — three independent profiles that would each read a handful of docs.
- C. A short QA task the model can answer in two tool calls with no cross-source comparison.
- D. A support ticket whose answer depends on the user's most recent message and their billing status — and nothing else.

### 2. Why decomposition must return structured output

The orchestrator's `decompose` step is told to return a list of `{worker_role, instruction}` objects via a response schema rather than prose. Why, per Chapter 1?

- A. JSON parses faster than prose at runtime.
- B. The provider refuses to dispatch workers unless assignments are JSON.
- C. Vague or overlapping prose assignments are where most orchestrator-worker systems fail; a structured schema forces the orchestrator to commit to concrete, parseable assignments and makes the fan-out bound enforceable.
- D. Only structured outputs can be run concurrently with `asyncio.gather`.

### 3. What a worker gets in its context

Chapter 1 and Chapter 4 both say a worker's agent loop should be seeded with only its assignment — not the orchestrator's conversation, not its siblings' work. The main reason is:

- A. The provider API rejects messages longer than ~8k tokens.
- B. Context economics: the worker can burn many tokens internally while the orchestrator's window stays small, and leaking siblings' work breaks the isolation that makes the pattern worth paying for.
- C. Workers run on cheaper models that physically cannot read the orchestrator's history.
- D. Passing the full conversation would violate the MCP specification.

### 4. The payload of a handoff

Chapter 2's `handoff` tool carries a `{to, summary}` payload, and the runtime seeds the next agent with the `summary` only. What problem does this specifically avoid?

- A. The receiving agent silently inheriting a larger, noisier context and re-parsing a transcript it doesn't need, while also stripping the sender's reasoning that could rationalize a bad decision.
- B. The receiving agent being unable to use any tools at all.
- C. Loops — the summary cannot be forwarded further.
- D. OpenAI rate-limiting the second agent for sending a long prompt.

### 5. Preventing handoff loops

A support flow has `billing → technical → billing → technical → …` ping-ponging forever. Which combination does Chapter 2 prescribe to stop it?

- A. Only a `max_hops` ceiling — a hard cap is enough on its own.
- B. Only a system-prompt rule that tells each specialist not to re-hand-off to a prior agent.
- C. A hop limit **and** a visited-set rule in each specialist's system prompt ("Do not hand off to an agent that has already handled this request — answer or escalate"), with the visited set tracked and passed along.
- D. Deleting one of the specialists so no cycle is possible.

### 6. Routers vs. orchestrators

Which pair of properties correctly describes a **router** (as distinct from an orchestrator) in Chapter 2's taxonomy?

- A. Exactly one agent is active at any moment; the work is sequential or branching, not parallel — a lightweight classifier dispatches to one specialist and does no work itself.
- B. Many agents are active at once; a lead agent fans work out in parallel.
- C. The router executes tool calls on behalf of each specialist and merges their output.
- D. Routers and orchestrators are the same thing under two names.

### 7. A2A vs. MCP

How does Chapter 3 distinguish A2A from MCP?

- A. MCP is a newer rewrite of A2A; they solve the same problem.
- B. A2A is agent ↔ agent (horizontal: typed inter-agent messages); MCP is agent → capability (vertical: how an agent talks to shared tools and resources). A typical system uses both.
- C. A2A is for cloud deployments and MCP is for local ones.
- D. A2A only works over HTTP while MCP only works over stdio.

### 8. Why `error` is a first-class intent in A2A messages

Chapter 3 says `intent` on an `AgentMessage` should include `error` alongside `request` and `result`. The main reason given is:

- A. Error messages cost fewer tokens than results and save money.
- B. Without an explicit `error` intent, a failing worker tends to return a hallucinated success that downstream agents can't distinguish from a real one; an explicit error lets downstream agents branch on `intent` instead of trying to parse prose.
- C. The MCP specification mandates it.
- D. It lets the orchestrator retry the worker automatically with no code change.

### 9. Per-agent MCP tool scoping

Your fleet shares a database MCP server. The `writer` agent should be able to read records; the `executor` agent should be able to write them. Chapter 3's guidance on this is:

- A. Run two copies of the server, one per agent.
- B. Expose every tool to every agent and rely on the system prompt to keep the writer from calling writes.
- C. Run one server; give each agent a different **allowlist** over the same server's tools (same source of truth, least privilege per agent).
- D. Encode the permission in the `AgentMessage` `payload`; the server will enforce it.

### 10. Distilled returns

A worker finishes an investigation after 18 tool calls and ~80k internal tokens. What should it hand back to the orchestrator, per Chapter 4?

- A. The entire transcript so the orchestrator can audit the reasoning.
- B. The answer, a list of source *references* (ids/urls/paths — not contents), a confidence level, and any open questions. The search process and raw tool output are omitted.
- C. Only the answer — references, confidence, and open questions are noise.
- D. A compressed `.gz` of the transcript the orchestrator can decompress on demand.

### 11. When isolation is the wrong call

Chapter 4 names cases where spinning up an isolated sub-agent is *not* the right design. Which of the following is **not** one of them?

- A. The sub-task is small — the orchestrator could handle it in two tool calls, so isolation only adds a round-trip and a summarization step.
- B. Workers B and C genuinely need to see worker A's raw output (not a summary) — they aren't independent, and the decomposition is wrong.
- C. The distilled schema keeps losing load-bearing detail, forcing the orchestrator to keep asking the sub-agent to re-explain itself.
- D. The sub-task involves a large corpus the orchestrator shouldn't drag into its own window — exactly the case isolation is for.

### 12. Why two agents for the evaluator-optimizer loop

Chapter 5 argues that a generator + adversarial critic beats a single agent doing both roles. The strongest reason given is:

- A. Two models always agree faster than one model disagreeing with itself.
- B. The critic carries a different system prompt ("find what's wrong"), sees the artifact fresh without the generator's rationalizing reasoning, and can be given different tools (run the tests, fetch the spec) so judgment is grounded in evidence.
- C. The critic is cheaper than the generator by construction.
- D. Only an evaluator agent can emit structured output.

### 13. Not looping forever

Chapter 5 lists three defenses against the loop never terminating. Which set matches?

- A. A hard `max_rounds` cap; a concrete, reachable pass condition (e.g., `score ≥ 8 and not must_fix`); and a no-progress exit that stops when the score doesn't improve for two rounds.
- B. A timeout in seconds; a retry-on-failure counter; and an exponential backoff.
- C. Caching the critic's verdict; sharing the generator and critic context windows; and lowering the model temperature.
- D. A soft budget warning; a human-in-the-loop approval; and switching providers mid-loop.

### 14. Prefer a deterministic check

Your task is "write a Python function that passes this test suite." Chapter 5's verdict on whether to use an LLM critic here:

- A. Always use an LLM critic — tests cannot judge code quality.
- B. If a deterministic check (pytest, a validator) can be the evaluator, use *that* — it is cheaper and more reliable than a critic agent. Save the LLM critic for tasks where no such check exists.
- C. Use both in parallel every round and majority-vote.
- D. Skip the evaluator entirely; one generator pass is enough when tests exist.

## Answer key

1. **B** — Orchestrator-worker is for independent, broad work where sub-tasks benefit from running concurrently in their own windows. A is sequential (handoffs/reactive loop), C is a single-agent task, D is a thin workflow ([Chapter 1](../01-orchestrator-worker.md)).
2. **C** — Chapter 1 calls the decomposition step the single biggest source of failures; a structured schema forces concrete, non-overlapping assignments and makes the N-cap enforceable. The other options invent constraints the chapter doesn't make ([Chapter 1](../01-orchestrator-worker.md)).
3. **B** — The whole point of isolation is context economics: a sub-agent spends tokens in its own window and returns a small distilled result. Seeding it with the parent transcript defeats the pattern ([Chapter 1](../01-orchestrator-worker.md), [Chapter 4](../04-subagent-isolation.md)).
4. **A** — Chapter 2 emphasizes that the next agent "starts fresh with just that, not the whole transcript." A thin summary is a *different* failure mode (context loss); a thick transcript is the one the summary-only design is defending against ([Chapter 2](../02-handoffs-and-routing.md)).
5. **C** — Chapter 2 prescribes both a hop limit *and* a visited-set rule in each system prompt, with the visited set tracked and passed along. Either alone is weaker ([Chapter 2](../02-handoffs-and-routing.md)).
6. **A** — A router is a lightweight, one-shot, no-tools classifier that dispatches to one specialist. Only one agent is active at a time, and the shape is sequential/branching — the opposite of orchestrator-worker's parallel fan-out ([Chapter 2](../02-handoffs-and-routing.md)).
7. **B** — MCP is vertical (agent → capability); A2A is horizontal (agent ↔ agent). Chapter 3 explicitly calls them orthogonal, and typical systems use both ([Chapter 3](../03-inter-agent-communication.md)).
8. **B** — Chapter 3 says a worker that fails "should return a structured error, not a hallucinated success" so downstream agents can branch on `intent` rather than trying to parse prose ([Chapter 3](../03-inter-agent-communication.md)).
9. **C** — "Same servers, different allowlists — least privilege across the fleet" is Chapter 3's per-agent tool scoping. One source of truth for the capability, per-agent scopes on top ([Chapter 3](../03-inter-agent-communication.md)).
10. **B** — The distilled return in Chapter 4 is answer + sources (as references, not contents) + confidence + open questions. Transcripts defeat isolation; returning only the bare answer drops the provenance the orchestrator needs to trust it ([Chapter 4](../04-subagent-isolation.md)).
11. **D** — D describes exactly when isolation is *right*. A (small task), B (tightly coupled sub-tasks), and C (schema too lossy) are the three cases Chapter 4 says isolation hurts ([Chapter 4](../04-subagent-isolation.md)).
12. **B** — A single agent "tends to rubber-stamp its own work." Separating the roles means different prompt, fresh view of the artifact, and potentially different tools so the critic's judgment is grounded in evidence, not vibes ([Chapter 5](../05-evaluator-optimizer-loop.md)).
13. **A** — Chapter 5's three axes are a hard round cap, a reachable pass condition, and a no-progress exit when the score stops improving. The other options are generic knobs the chapter doesn't endorse ([Chapter 5](../05-evaluator-optimizer-loop.md)).
14. **B** — "If a linter or test suite can be the 'critic,' use *that* as the evaluator and save a model call." Chapter 5 explicitly prefers deterministic checks when they exist ([Chapter 5](../05-evaluator-optimizer-loop.md)).
