# Quiz 1 — Building a Custom Agent Harness

Knowledge check for [mod-209](../README.md). Answers are at the bottom; try each question before scrolling. Covers all nine chapters.

## Questions

### 1. When to build your own

Your team is adding an agent that triages customer-support tickets inside your product, with its own domain tools (CRM, ticketing, knowledge base) and an audit requirement. Chapter 1 lists the cases where building a custom harness is warranted. Which choice best matches the chapter's guidance?

- A. Always build from scratch — using an SDK means you can't own the loop.
- B. Always configure Claude Code or Codex — teams never need a custom harness.
- C. The agent lives inside your product with domain tools and audit needs, which Chapter 1 lists as reasons to build. Even so, look at the Claude Agent SDK first; start from it and replace parts as your needs diverge.
- D. Build a harness only after you have shipped three generations of configured ones.

### 2. The design rule you'll feel most

A reviewer insists "we told the model in the prompt not to write outside the workspace, so we're fine." Chapter 1 pushes back with one of its seven design rules. Which one, and what does it imply?

- A. "Everything is an event" — emit a `workspace_violation` event and move on.
- B. "The harness enforces; the prompt requests." Rules that must hold belong in code or at the OS level; the system prompt can explain the rule so the model doesn't waste turns running into it.
- C. "Pure core, pluggable edges" — swap in a different model and the violations will stop.
- D. "Deterministic tests for a non-deterministic system" — add a replay test and the rule is covered.

### 3. Errors as observations

Your loop currently raises on a `KeyError` when the model calls a tool with a misspelled argument, aborting the run. Chapter 2 argues this is the wrong shape. What does the chapter say the loop should do instead?

- A. Silently retry the same tool call with corrected input the harness guesses.
- B. Return the exception message as a tool result marked `is_error`, so the model can read it on the next turn and correct itself. Every failure path — unknown tool, schema-invalid input, denial, exception, timeout — becomes an observation the model acts on.
- C. End the run with `stop_reason="error"` so the user can restart.
- D. Strip the bad argument and call the tool anyway; the model will figure it out from the result.

### 4. Stop conditions

You're reviewing a loop that only stops when the model emits no tool calls. Chapter 2 lists the four stop conditions a loop needs, and warns that running forever is more expensive than you think. Which set matches the chapter?

- A. No tool calls; `max_turns`; a token budget; and detection of repeated identical tool calls — with the stop reason returned alongside the result.
- B. Only `max_turns` — anything else is premature optimization.
- C. No tool calls, plus a magic keyword in the final text like "DONE".
- D. A single wall-clock timeout, so the user can tune a single knob.

### 5. Exact-match edits

Chapter 3 compares three edit-tool designs: whole-file writes, line-number edits, and exact-match replacement. Which claim about **exact-match replacement** matches the chapter?

- A. It's slower because the model has to re-send the file.
- B. It works because the edit is anchored to position, not content.
- C. It's self-checking (the text has to be there), it forces a read first (you can't produce an exact match without seeing current content), and edits are small (only what changes, plus enough context to make it unique).
- D. It only works for single-line edits; multi-line changes need a patch format.

### 6. Read-before-edit

Your `edit_file` tool rejects a request with "src/app.py changed since you last read it." Chapter 3 says the read ledger plus a modification-time check catches three real problems. Which set matches?

- A. A hostile MCP server; a compromised dependency; a model that lies about having read the file.
- B. The model "remembering" a file from training data or an earlier session; a human or another process changing the file after the read; a compaction that dropped the file's content so the model edits from a summary.
- C. Race conditions between threads; filesystem corruption; stale OS caches.
- D. None of the above — the check exists mainly to slow the model down.

### 7. Where the real boundary is

A teammate proposes that path containment inside the file tool is the security boundary for your harness. Chapter 4 argues otherwise. Which statement matches the chapter?

- A. Path containment in the tool is the real boundary; sandboxes are a performance optimization.
- B. In-process checks give fast, clear feedback but aren't a boundary — `python -c`, a script file, or a symlink built across two commands can get past them. The boundary is the OS: sandbox profile, container, or VM. Use both: defense in depth.
- C. Containers aren't needed if your allowlist is strict enough.
- D. Permission rules and sandboxes are two names for the same thing.

### 8. Checkpoints without touching the user's git state

Chapter 4 describes checkpointing the workspace before each turn using git, without disturbing the user's branch, index, or `HEAD`. Which technique matches the chapter?

- A. `git stash push --include-untracked` before each turn and `git stash pop` to rewind.
- B. A new branch per turn, with `git commit --amend` on failure.
- C. `GIT_INDEX_FILE` pointing at a temp index, `git add -A`, `git write-tree` for the tree, `git commit-tree` for a parent-less commit, and `git update-ref refs/harness/checkpoints/<n>` to keep it alive. The user's branch, index, and `HEAD` are untouched.
- D. Copy the whole working tree to `/tmp` on every turn.

### 9. Clearing old tool results

You're adding tool-result clearing from Chapter 5 and considering just deleting the oldest messages. Which rule matches the chapter?

- A. Delete the whole message, call and result together. Shorter history is always better.
- B. Keep the tool call and clear only the result's content, replacing it with a placeholder. Keep calls and results paired — most provider APIs reject a result without its call or a call without its result — and keep the most recent results intact.
- C. Clear from the most recent backward, since those are most likely to have been "used up."
- D. Drop the tool calls but keep the results so the model doesn't repeat work.

### 10. Enforcing the feature-list rule

Chapter 5 says the long-running pattern's `features.json` only works if the agent can't quietly rewrite it. Which approach best matches the chapter's recommendation?

- A. Add a sentence to `AGENTS.md` asking the agent not to add, remove, or reword features.
- B. A pre-tool hook that loads the current `features.json` and the proposed new one and rejects the write unless the only differences are `passes` values on existing features. "A rule the agent kept breaking moved from an instruction into a mechanical check."
- C. Make `features.json` read-only on disk at the start of every session.
- D. Rely on compaction to summarize the file so the agent forgets its structure.

### 11. Where the cache breakpoint goes

You're laying out your system prompt for prompt caching. Chapter 6 gives explicit rules. Which layout follows them?

- A. One timestamp at the top of the system prompt, then tools, then base prompt, then instruction files.
- B. Volatile first (date, branch, git status), then stable content. Models weight later instructions more heavily.
- C. Tools (sorted in a stable order), base prompt, skills index, instruction files — cache breakpoint here — then the environment block (cwd, platform, date, branch, git status). The prefix is cached exactly, so any change near the start invalidates everything after.
- D. Put the cache breakpoint after the most recent user message so every new turn is a fresh cache entry.

### 12. Parallel tool calls

The model requests three tools in one turn: two `read_file` calls and one `edit_file`. Chapter 6 gives a simple safe rule for whether to run them concurrently. Which answer matches?

- A. Run them all concurrently; the provider will reorder results.
- B. Run them all sequentially; concurrency inside a turn breaks the transcript.
- C. If every call in a turn is marked `read_only`, run concurrently; if any call writes or runs a command, run the whole batch in order. Return results in call order regardless of finish order.
- D. Run read-only calls first, then writes, interleaving by `id`.

### 13. Subagent delegation

A teammate suggests giving the child subagent its own permission policy so it can edit files the parent can't. Chapter 7 says this misunderstands the point. Which rule applies?

- A. Subagents are only for parallelism; they should share the parent's policy by accident.
- B. The child inherits the parent's permission policy and hooks, uses a subset of the parent's tools, counts against the parent's token budget, and can't recurse past a depth limit. Otherwise delegation is a way around the rules.
- C. The child sees the parent's full history for context, otherwise it can't help.
- D. The child's `stop_reason` is hidden from the parent so the parent isn't distracted.

### 14. MCP as untrusted code

Your harness just connected to an MCP server for issue tracking. Chapter 7 tells you how to treat its tools. Which statement matches?

- A. MCP servers are trusted code from the registry; register their tools without special handling.
- B. Tool descriptions and results from an MCP server are untrusted input that enters the model's context — a malicious server can put instructions in a tool description or a result. Cap description length, default MCP tools to `ask`, and route every MCP call through the same gate and hooks as built-in tools.
- C. MCP calls should bypass your hooks because the server already filters content.
- D. Trust remote MCP servers over stdio servers; HTTP is harder to spoof.

### 15. Replay tests

You added a replay test suite, renamed a tool in one place but not another, and the test caught it. Chapter 8 names what replay tests do and don't catch. Which statement matches?

- A. Replay tests prove the agent will still succeed on the task.
- B. Replay tests catch unintended changes to what the harness *sends* the model — prompt or tool-schema reorderings, renamed tools, broken cache prefixes — by checking each request against a cassette. They don't tell you whether the agent will still succeed; a real model may respond differently. Live evals cover that.
- C. Replay tests replace scripted-model unit tests entirely.
- D. Replay tests record both model calls and shell commands, so they double as end-to-end tests.

### 16. Headless defaults

You're about to run your harness unattended in CI. Chapter 8 is specific about how `ask` decisions should resolve in headless mode. Which policy matches?

- A. `ask` resolves to allow in headless mode, since no human is there to answer.
- B. `ask` resolves to deny by default, with an explicit flag to allow. An unattended agent should never get more power than an attended one by default.
- C. `ask` prompts are queued until the next interactive session resumes the run.
- D. Headless mode disables the permission gate entirely.

### 17. Gate maps onto the permission gate

Chapter 9 describes integrating VeriSwarm's Gate as one more permission policy, not a replacement. Which mapping and design matches?

- A. VeriSwarm's `allow` → ask, `review` → allow, `deny` → deny; call on every tool and cache nothing.
- B. VeriSwarm's `allow` → allow, `review` → ask (approver or deny when headless), `deny` → deny with the `reason_code` in the model's message. Govern only side-effecting tools, cache decisions for a short TTL, and keep local `deny` authoritative.
- C. Gate replaces your local policy, so your local hooks don't run for governed tools.
- D. `review` is treated as `allow` since Gate has already checked the agent's score.

### 18. Fail modes for the trust service

Chapter 9 says every remote control needs an answer to "what if the service is down?" and that the SDK's framework adapters fail open on API errors. For a harness, which approach matches the chapter?

- A. Fail open for everything — the agent should keep working even when telemetry is dark.
- B. Fail closed for everything — pause the harness until the service is reachable again.
- C. Decide per tool, write it down, and test with a simulated outage. Fail closed for anything that matters (writes, commands, payments, outbound messages). Remember that an attacker who can take the service offline can otherwise force every check to "allow."
- D. Fail mode doesn't matter if events are batched, because the record will catch up later.

## Answer key

1. **C** — Chapter 1's "When to build your own" lists the case (agent lives in your product, domain tools, audit) and advises looking at the Claude Agent SDK first, starting from an SDK and replacing parts as needs diverge ([Chapter 1 §When to build your own](../01-harness-architecture.md)).
2. **B** — "The harness enforces; the prompt requests" is one of the seven design rules: if a rule must hold, enforce it in code or at the OS level; the system prompt explains it so the model doesn't waste turns hitting it ([Chapter 1 §Design rules](../01-harness-architecture.md)).
3. **B** — Errors are observations. Every failure path returns a result the model can read and react to; a loop that crashes on bad tool input breaks the first time a model misspells an argument ([Chapter 2 §Tool dispatch](../02-the-agent-loop.md)).
4. **A** — The four conditions are no tool calls (done), `max_turns`, a token budget, and repeated identical tool calls, with the stop reason returned alongside the final text ([Chapter 2 §Stop conditions](../02-the-agent-loop.md)).
5. **C** — Why exact match works well: self-checking (if the text isn't there, the model's picture is wrong, and the edit fails before doing damage), forces a read first, and keeps edits small ([Chapter 3 §Editing files](../03-coding-tools.md)).
6. **B** — The chapter lists exactly these three cases for the read-before-edit check: model memory from training or earlier sessions, a human or other process changing the file, and compaction dropping the file content so the model edits from a summary ([Chapter 3 §Read before edit](../03-coding-tools.md)).
7. **B** — In-process checks "give feedback, not protection" — `python -c`, a script file, or a symlink built across two commands get past them. The real boundary is the OS, used together with the in-process checks for defense in depth ([Chapter 4 §Layers](../04-execution-isolation.md)).
8. **C** — The chapter's exact technique: a private index via `GIT_INDEX_FILE`, `git write-tree` and `git commit-tree`, then `git update-ref refs/harness/checkpoints/<n>` to keep it alive without touching the user's branch, index, or `HEAD` ([Chapter 4 §Checkpoints](../04-execution-isolation.md)).
9. **B** — Keep the tool call; clear only the result. Keep calls and results paired (APIs reject either one alone). Keep the most recent results intact — they're what the model is working with right now ([Chapter 5 §Clear old tool results](../05-context-compaction-state.md)).
10. **B** — The chapter shows a pre-tool `features_guard` that loads the current and proposed file and rejects the write unless the only differences are `passes` values. "A rule the agent kept breaking moved from an instruction into a mechanical check." ([Chapter 5 §Enforce the rules in the harness](../05-context-compaction-state.md)).
11. **C** — Prefix order is tools, then system, then messages, sorted stably; the cache breakpoint sits after the last stable block, with volatile values (date, branch, git status) after it. A timestamp at the top means no cache hits ever ([Chapter 6 §Prompt caching](../06-prompt-assembly-and-model-layer.md)).
12. **C** — The chapter's simple, safe rule: mark each tool `read_only` or not; concurrent only when every call in the turn is read-only; always return results in call order ([Chapter 6 §Parallel tool calls](../06-prompt-assembly-and-model-layer.md)).
13. **B** — The table of rules the harness must enforce: inherit parent's policy and hooks, subset of parent's tools, count against parent's budget, depth limit (usually 1), stop reason reaches the parent. Otherwise delegating is a way around the rules ([Chapter 7 §Rules the harness must enforce](../07-subagents-skills-mcp.md)).
14. **B** — An MCP server is code from someone else that can put text into the model's context, through descriptions and results. Cap description length, default to `ask`, and route every MCP call through the same gate and hooks ([Chapter 7 §MCP safety](../07-subagents-skills-mcp.md)).
15. **B** — Replay tests check that the harness sends the same requests; renaming a tool in one place but not another is exactly the kind of bug they catch. They don't tell you whether the agent will still succeed — that's what live evals are for ([Chapter 8 §Record and replay](../08-observability-testing-shipping.md)).
16. **B** — In headless mode no human is there to answer `ask`. Resolve to deny by default, with an explicit flag to allow; an unattended agent should never get more power than an attended one by default ([Chapter 8 §Headless defaults](../08-observability-testing-shipping.md)).
17. **B** — The mapping table: `allow`→allow, `review`→ask (approver or deny when headless), `deny`→deny with `reason_code`. Govern only side-effecting tools, cache decisions, and keep local `deny` authoritative ([Chapter 9 §Gate as a permission source](../09-trust-layer-veriswarm.md)).
18. **C** — Decide per tool, write it down, test with a simulated outage. Fail closed for anything that matters; an attacker who can take the service offline can otherwise force every check to "allow." The SDK's framework adapters fail open on API errors, which is why the chapter integrates through the core client ([Chapter 9 §Fail open or fail closed?](../09-trust-layer-veriswarm.md)).
