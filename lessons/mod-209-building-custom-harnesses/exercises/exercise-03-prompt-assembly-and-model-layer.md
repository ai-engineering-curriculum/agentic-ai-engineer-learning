# exercise-03: Prompt Assembly and a Resilient Model Layer

**Estimated effort:** 3 hours

## Objective

Make your harness build its own system prompt from the environment, as production harnesses do. Discover instruction files up the directory tree, index skills without loading them, and describe the environment. Then harden the model layer: retries with backoff for rate limits and overloads, prompt-cache breakpoints on the stable prefix, streaming events for a live UI, and parallel execution of independent read-only tool calls. By the end, your harness's startup and model calls will behave much like Claude Code's.

## Background

This exercise covers material from:

- [Chapter 6 — Prompt Assembly and the Model Layer](../06-prompt-assembly-and-model-layer.md)

Build on your [exercise-01](exercise-01-build-a-minimal-harness.md) harness or its reference solution.

## Prerequisites

- A working exercise-01 harness.
- Python 3.11+, `pytest`.
- The `anthropic` SDK and a key for the live check in task 6. Everything else is tested with fakes.

## Tasks

### 1. Instruction-file discovery

- Starting from the workspace root, collect `AGENTS.md` (and `CLAUDE.md`, if present) from the root and every parent directory up to a stop point (the git root, or the filesystem root if there's no repo), plus a user-level file (`~/.config/<your-harness>/AGENTS.md`).
- Order them from most general (user-level) to most specific (workspace root), so specific rules come last.
- Support `@relative/path.md` import lines: replace the line with that file's contents, resolving relative to the importing file. Limit import depth (say, 5) and detect cycles.
- Wrap each file in a labeled block naming its path, so the model knows where each rule came from.
- Cap total instruction size. If over the cap, keep the most specific files and say what was dropped.

### 2. Skills index and a `load_skill` tool

- Scan `<workspace>/.skills/*/SKILL.md` and a user-level skills directory. Parse YAML-style frontmatter for `name` and `description` (a minimal parser for `key: value` lines is enough).
- Put only the index in the system prompt: one line per skill, with name and description.
- Add a `load_skill(name)` tool that returns the skill body. Reject names that aren't in the index.

### 3. The system prompt

- Assemble in a fixed order: a base prompt (role, tool-use rules, how to report being done), an environment block (working directory, platform, date, whether it's a git repo and the current branch), the skills index, and the instruction files.
- Return it as a list of blocks rather than one string, and mark the **last block that rarely changes** as a cache breakpoint. Put anything that changes per session (date, branch) after the breakpoint.

### 4. Retries with backoff

- Wrap the model client so it retries on rate-limit (429), overloaded (529), and 5xx errors, and on connection timeouts. Don't retry on 400, 401, or 403.
- Use exponential backoff with full jitter, a maximum number of attempts, and a maximum total wait. Honor a `retry-after` header when the error carries one.
- Make the sleep function injectable so tests don't actually wait.
- Emit an event on every retry (attempt, delay, reason).

### 5. Streaming events and parallel tools

- Add an event callback to the harness. Emit `text_delta`, `tool_call_started`, `tool_call_finished`, `turn_finished`, and `retry` events. A CLI can print them live; tests can collect them.
- When a single model turn requests several tool calls that are all marked `read_only` in the registry (`read_file`, `glob`, `grep`, `load_skill`), run them concurrently with a thread pool. Run any batch containing a write or command one call at a time, in order.
- Results must go back to the model in the same order as the calls, whatever order they finished in.

### 6. Live check

- With the real `AnthropicModel`, run a short task twice within the cache lifetime and show the second run's `cache_read_input_tokens` is above zero.

## Starter guidance

```python
RETRYABLE_STATUS = {429, 500, 502, 503, 504, 529}

def backoff_delays(base: float, cap: float, attempts: int, rng: random.Random) -> list[float]:
    """Full jitter: sleep a random amount between 0 and min(cap, base * 2**n)."""
    return [rng.uniform(0, min(cap, base * 2**n)) for n in range(attempts)]
```

```python
@dataclass(frozen=True)
class PromptBlock:
    label: str
    text: str
    cache_breakpoint: bool = False
```

## Acceptance criteria

You can demonstrate that, in `pytest`:

- Instruction discovery finds files in parents, orders general to specific, resolves `@imports`, stops cycles, and enforces the size cap with a note.
- The system prompt contains the skills index but no skill bodies, and `load_skill` returns a body only for indexed names.
- The cache breakpoint sits on the last stable block, and per-session values come after it.
- Retries happen on 429/529/5xx with growing jittered delays and respect `retry-after`; 400/401 fail immediately; the attempt cap holds.
- A turn with three read-only calls runs them concurrently (measurably faster than sequential with a slow fake tool) and returns results in call order. A turn mixing a read and a write runs sequentially.
- The event stream for a scripted run contains the expected events in order.
- The live run shows a cache hit on the second call.

## Reflection

In `NOTES.md`:

1. Why does the order of instruction files matter, and why do specific files go last?
2. What would happen to your cache hit rate if the date were in the first prompt block?
3. Parallel tool execution saves time but complicates things. What could go wrong if you ran two `edit_file` calls on the same file concurrently?

## Stretch goals

- Detect and warn when an instruction file is very large or hasn't changed in a long time while the code around it has.
- A second provider (OpenAI or a local model through an OpenAI-compatible server) behind the same `ModelClient` interface, and a config switch between them.
- A circuit breaker that pauses all calls for a cool-down after repeated overload errors, shared across concurrent sessions.
