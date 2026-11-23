# Lab 3 — Clear Tool Results Without Breaking Pairing

**Duration:** ~45 minutes
**Covers:** [Chapter 5 — Context, Compaction, and Session State](../05-context-compaction-state.md)

## Objective

Chapter 5 says tool-result clearing is the cheapest context-management technique: it reclaims most of the window, loses little, and is the first thing to try. It also has two subtle invariants — keep the tool call, clear only its result; and keep every call paired with a result — that provider APIs reject when you get them wrong. Before [Exercise 04](../exercises/exercise-04-context-and-session-state.md) asks you to wire clearing, compaction, and session state files together inside your harness, spend 45 minutes building the clearing function in isolation and watching both invariants fire.

By the end of the lab you will:

- Hand-build a provider-shape history with a `system` message, a `user` turn, an `assistant` turn carrying three `tool_use` blocks, and a `user` turn carrying their three `tool_result` blocks.
- Write a 20-line `clear_old_tool_results` that replaces old result *content* with a placeholder, keeps the most recent N results intact, and never deletes a message outright.
- Watch three wrong implementations break the shape in different ways — deleting whole messages, clearing calls instead of results, and clearing from the most recent backward — and feel why Chapter 5 is specific about each rule.
- Count tokens with the Chapter 5 estimator (characters ÷ 4) and verify clearing dropped context under threshold without starting a model turn.

Scripted, offline, pure stdlib. The point is to feel the message-shape invariants before you hit them against a real provider.

## Prerequisites

- Python 3.11+.
- [Chapter 5](../05-context-compaction-state.md) read, especially the "Clear old tool results" section.
- No external packages.

## Setup

```bash
mkdir lab-03 && cd lab-03
```

No venv needed.

## Step 1 — The provider-shape history

Chapter 1 says the loop keeps two histories: `messages` (what the model sees next) and `transcript` (everything that happened, append-only). Clearing operates on `messages`. The shape is the Anthropic Messages API's — one `assistant` message with `tool_use` blocks, followed by one `user` message with `tool_result` blocks, each `tool_use_id` matching a `tool_use.id`. Save as `fixtures.py`:

```python
"""A realistic messages list with three old tool results and two recent ones."""


def big_file(nlines: int, prefix: str) -> str:
    return "\n".join(f"{i:5d}\t{prefix}_line_{i}" for i in range(1, nlines + 1))


def make_messages() -> list[dict]:
    """Five tool calls across two turns. Content sizes are chosen so the estimator
    reports ~thousands of tokens on the old ones and ~tens on the recent ones."""
    return [
        {"role": "user", "content": "Explore the codebase and find the retry logic."},

        # Turn 1: three big reads (old; good candidates for clearing).
        {"role": "assistant", "content": [
            {"type": "text", "text": "I'll read three files to orient."},
            {"type": "tool_use", "id": "tu_01", "name": "read_file",
             "input": {"path": "src/billing/proration.py"}},
            {"type": "tool_use", "id": "tu_02", "name": "read_file",
             "input": {"path": "src/billing/invoice.py"}},
            {"type": "tool_use", "id": "tu_03", "name": "read_file",
             "input": {"path": "src/http/client.py"}},
        ]},
        {"role": "user", "content": [
            {"type": "tool_result", "tool_use_id": "tu_01",
             "content": big_file(400, "proration")},
            {"type": "tool_result", "tool_use_id": "tu_02",
             "content": big_file(300, "invoice")},
            {"type": "tool_result", "tool_use_id": "tu_03",
             "content": big_file(500, "client")},
        ]},

        # Turn 2: narrower follow-up (recent; keep).
        {"role": "assistant", "content": [
            {"type": "text", "text": "Grep for `retry` in http/."},
            {"type": "tool_use", "id": "tu_04", "name": "grep",
             "input": {"pattern": "def _retry", "path": "src/http"}},
            {"type": "tool_use", "id": "tu_05", "name": "read_file",
             "input": {"path": "src/http/retry.py"}},
        ]},
        {"role": "user", "content": [
            {"type": "tool_result", "tool_use_id": "tu_04",
             "content": "src/http/retry.py:12:def _retry(fn, *, attempts=5):"},
            {"type": "tool_result", "tool_use_id": "tu_05",
             "content": "def _retry(fn, *, attempts=5):\n    for n in range(attempts): ...\n"},
        ]},
    ]
```

Three details worth noticing:

- **The shape is exactly what Anthropic's Messages API requires.** Each `tool_use` has a unique `id`; each `tool_result` has a `tool_use_id` that points back. Chapter 5: *"Provider APIs reject a tool result without its call, or a call without its result."* Both invariants are already visible in this fixture.
- **Three old, two recent.** Chapter 5 says keep "the most recent N results intact"; `N = 2` is a tiny budget that makes the rule easy to test. Exercise 04's default is 3.
- **File reads are bulky by design.** A real `read_file` on a 500-line source file is thousands of tokens; grep output on a specific pattern is tens. That ratio is why clearing reclaims most of the window.

## Step 2 — An estimator and an invariant-checker

Chapter 5: *"About four characters per token is a common rule of thumb... Put it behind one function so you can swap in a real tokenizer later."* Save as `budget.py`:

```python
"""Token estimation and the pairing invariant."""


def estimate_tokens(value) -> int:
    """Characters divided by 4. Rough, but the shape is what matters at this stage."""
    if isinstance(value, dict):
        return sum(estimate_tokens(v) for v in value.values())
    if isinstance(value, list):
        return sum(estimate_tokens(v) for v in value)
    if isinstance(value, str):
        return max(1, len(value) // 4)
    return 0


def context_tokens(messages: list[dict]) -> int:
    return sum(estimate_tokens(m) for m in messages)


def check_pairing(messages: list[dict]) -> list[str]:
    """Return a list of violations. An empty list means the history is valid."""
    problems = []
    pending: dict[str, str] = {}
    for mi, m in enumerate(messages):
        content = m.get("content")
        if not isinstance(content, list):
            continue
        for bi, block in enumerate(content):
            btype = block.get("type")
            if btype == "tool_use":
                tid = block["id"]
                if tid in pending:
                    problems.append(f"msg {mi}[{bi}]: duplicate tool_use id {tid!r}")
                pending[tid] = block["name"]
            elif btype == "tool_result":
                rid = block["tool_use_id"]
                if rid not in pending:
                    problems.append(
                        f"msg {mi}[{bi}]: tool_result {rid!r} without a matching tool_use"
                    )
                else:
                    del pending[rid]
    for tid, name in pending.items():
        problems.append(f"tool_use {tid!r} ({name}) has no tool_result")
    return problems
```

Three details:

- **`estimate_tokens` works on any JSON value.** You don't want the estimator to care whether it's measuring the whole `messages` list, one message, or one block — recursive sum keeps the call site boring.
- **`check_pairing` walks blocks inside messages**, because the real shape is "N blocks per message," not "one message per call." A clearing implementation that compresses one block but misorders the rest would silently break this check.
- **Both functions are pure.** Same `messages`, same answer every time; that's what lets the drive script in Step 4 assert on exact numbers.

## Step 3 — The right implementation

Chapter 5's three rules for clearing: **keep the tool call**, replace only the result content; **keep calls and results paired**; **keep the most recent N results intact**. Save as `clear.py`:

```python
"""Replace old tool_result content with a placeholder. Keep calls and pairing intact."""
from budget import estimate_tokens


PLACEHOLDER = "[cleared: {name}({args}) — {tokens} tokens. Re-read if needed.]"


def _name_and_args(messages: list[dict], tool_use_id: str) -> tuple[str, str]:
    """Look up the tool_use block that matches this tool_use_id, so the placeholder
    can name what was cleared ('read_file src/app.py') rather than just an opaque id."""
    for m in messages:
        if not isinstance(m.get("content"), list):
            continue
        for b in m["content"]:
            if b.get("type") == "tool_use" and b.get("id") == tool_use_id:
                args = ", ".join(f"{k}={v!r}" for k, v in b.get("input", {}).items())
                return b.get("name", "?"), args
    return "?", ""


def clear_old_tool_results(messages: list[dict], keep_recent: int = 2) -> list[dict]:
    """Replace the content of every tool_result older than the last `keep_recent`."""
    # Index each (message_index, block_index) that is a tool_result, in order.
    positions: list[tuple[int, int]] = []
    for mi, m in enumerate(messages):
        if not isinstance(m.get("content"), list):
            continue
        for bi, b in enumerate(m["content"]):
            if b.get("type") == "tool_result":
                positions.append((mi, bi))

    # Clear all but the most recent `keep_recent`.
    to_clear = positions[:-keep_recent] if keep_recent > 0 else positions

    cleared = [dict(m) for m in messages]  # shallow copy — we only replace the block
    for mi, bi in to_clear:
        new_content = list(cleared[mi]["content"])
        block = dict(new_content[bi])
        name, args = _name_and_args(messages, block["tool_use_id"])
        before = estimate_tokens(block.get("content", ""))
        block["content"] = PLACEHOLDER.format(name=name, args=args, tokens=before)
        new_content[bi] = block
        cleared[mi]["content"] = new_content
    return cleared
```

Four details worth noticing:

- **We clear `content`, not the block.** The block's `type` stays `tool_result` and its `tool_use_id` stays the same. Chapter 5: *"Keep the tool call; clear only the result."*
- **We keep messages structurally intact.** No `del messages[i]`, no reordering. The provider sees the same message count, in the same order; only the `content` string inside each old `tool_result` block got shorter.
- **The placeholder names the tool and arguments.** `[cleared: read_file(path='src/billing/proration.py') — 3850 tokens. Re-read if needed.]` tells the model *what* was cleared and *how* to recover (read again). Chapter 5: *"Twenty turns later, the 400-line file it read at the start is dead weight. It can read it again if it needs to."*
- **`keep_recent` is a parameter.** Chapter 5 says "the most recent N results"; the right N depends on the task. Exercise 04 wires it to a config field.

## Step 4 — Drive the right implementation

Save as `drive_clear.py`:

```python
from fixtures import make_messages
from budget import check_pairing, context_tokens
from clear import clear_old_tool_results


def show(label: str, messages: list[dict]) -> None:
    print(f"\n--- {label} ---")
    print(f"tokens: {context_tokens(messages):,}")
    pairing = check_pairing(messages)
    print(f"pairing: {'ok' if not pairing else pairing}")
    # Show the first tool_result's content (truncated) in each user message.
    for m in messages:
        if not isinstance(m.get("content"), list):
            continue
        for b in m["content"]:
            if b.get("type") == "tool_result":
                preview = (b["content"][:70] + "…") if len(b["content"]) > 70 else b["content"]
                print(f"  tool_result {b['tool_use_id']}: {preview}")


messages = make_messages()
show("before", messages)
cleared = clear_old_tool_results(messages, keep_recent=2)
show("after (keep_recent=2)", cleared)
```

Run it:

```bash
python3 drive_clear.py
```

**Expected output (abbreviated):**

```
--- before ---
tokens: 6,...
pairing: ok
  tool_result tu_01:     1	proration_line_1
     2	proration_line_2
     3	proration_lin…
  tool_result tu_02:     1	invoice_line_1
     2	invoice_line_2
     3	invoice_line_3…
  tool_result tu_03:     1	client_line_1
     2	client_line_2
     3	client_line_3
   …
  tool_result tu_04: src/http/retry.py:12:def _retry(fn, *, attempts=5):
  tool_result tu_05: def _retry(fn, *, attempts=5):
    for n in range(attempts): ...

--- after (keep_recent=2) ---
tokens: 1,...
pairing: ok
  tool_result tu_01: [cleared: read_file(path='src/billing/proration.py') — ... tokens.…
  tool_result tu_02: [cleared: read_file(path='src/billing/invoice.py') — ... tokens. …
  tool_result tu_03: [cleared: read_file(path='src/http/client.py') — ... tokens. Re-r…
  tool_result tu_04: src/http/retry.py:12:def _retry(fn, *, attempts=5):
  tool_result tu_05: def _retry(fn, *, attempts=5):
    for n in range(attempts): ...
```

Four things to notice:

- **Token count drops by roughly 5-6×** between "before" and "after." Three bulky reads replaced by three one-line placeholders is the cheapest reclamation available, which is why Chapter 5 says to try it first.
- **`pairing: ok` in both cases.** Every `tool_use` still has a matching `tool_result`; the harness can hand this to the Messages API unchanged.
- **The three oldest results were cleared** (`tu_01`, `tu_02`, `tu_03`); the last two (`tu_04`, `tu_05`) are intact. Chapter 5: *"Keep the most recent results. They're what the model is working with right now."*
- **The placeholder names the tool and path.** The model knows *what* it has forgotten and how to recover ("re-read if needed"). A bare `[cleared]` would leave it wondering whether it had ever read the file.

## Step 5 — Watch three wrong implementations break

Each of these is a tempting shortcut. Each breaks exactly one of Chapter 5's rules. Save as `drive_wrong.py`:

```python
from fixtures import make_messages
from budget import check_pairing


def wrong_delete_messages(messages: list[dict], drop_first_n_user_turns: int = 1) -> list[dict]:
    """Shortcut: 'just delete the oldest user messages (they hold tool_result blocks).'"""
    # Find all user messages whose content is a list of tool_result blocks.
    kept = []
    dropped = 0
    for m in messages:
        looks_like_results = (m["role"] == "user" and isinstance(m.get("content"), list)
                              and all(b.get("type") == "tool_result" for b in m["content"]))
        if looks_like_results and dropped < drop_first_n_user_turns:
            dropped += 1
            continue
        kept.append(m)
    return kept


def wrong_clear_tool_use_too(messages: list[dict]) -> list[dict]:
    """Shortcut: 'drop the tool_use blocks too, since we're saving tokens.'"""
    out = []
    for m in messages:
        if isinstance(m.get("content"), list):
            m = dict(m)
            m["content"] = [b for b in m["content"] if b.get("type") != "tool_use"]
        out.append(m)
    return out


def wrong_clear_most_recent(messages: list[dict], clear_last_n: int = 2) -> list[dict]:
    """Shortcut: 'clear the last N results — they're the biggest since they're on screen.'"""
    out = [dict(m) for m in messages]
    positions = []
    for mi, m in enumerate(out):
        if not isinstance(m.get("content"), list):
            continue
        for bi, b in enumerate(m["content"]):
            if b.get("type") == "tool_result":
                positions.append((mi, bi))
    for mi, bi in positions[-clear_last_n:]:
        blocks = list(out[mi]["content"])
        b = dict(blocks[bi]); b["content"] = "[cleared]"
        blocks[bi] = b; out[mi]["content"] = blocks
    return out


messages = make_messages()


def show(label, result):
    pairing = check_pairing(result)
    print(f"\n--- {label} ---")
    print(f"pairing: {'ok' if not pairing else pairing}")


show("wrong: delete whole messages", wrong_delete_messages(messages))
show("wrong: clear tool_use too",   wrong_clear_tool_use_too(messages))
show("wrong: clear most recent",    wrong_clear_most_recent(messages))
```

Run it:

```bash
python3 drive_wrong.py
```

**Expected output:**

```
--- wrong: delete whole messages ---
pairing: ["tool_use 'tu_01' (read_file) has no tool_result", "tool_use 'tu_02' (read_file) has no tool_result", "tool_use 'tu_03' (read_file) has no tool_result"]

--- wrong: clear tool_use too ---
pairing: ["msg 2[0]: tool_result 'tu_01' without a matching tool_use", "msg 2[1]: tool_result 'tu_02' without a matching tool_use", "msg 2[2]: tool_result 'tu_03' without a matching tool_use", "msg 4[0]: tool_result 'tu_04' without a matching tool_use", "msg 4[1]: tool_result 'tu_05' without a matching tool_use"]

--- wrong: clear most recent ---
pairing: ok
```

Three things to notice:

- **Shortcut 1 ("delete the user message with the old results") breaks pairing in one direction** — the `tool_use` blocks are now orphans. This is the error the Anthropic Messages API rejects with "`tool_use` without matching `tool_result`." Chapter 5: *"Clear content; don't delete messages."*
- **Shortcut 2 ("drop the tool_use blocks too, since we're saving tokens") breaks pairing in the other direction** — now the `tool_result` blocks are orphans. Same rejection, opposite side. Chapter 5: *"The model needs to see that it read the file, so it doesn't wonder whether it did."*
- **Shortcut 3 ("clear the most recent") looks structurally valid** (`pairing: ok`) but is strategically wrong — the model is actively using those results. Chapter 5 is explicit: "oldest first, keeping the last few intact." The invariant checker won't catch this; the eval will, when the agent starts re-doing the work it just did. This is a reminder that "pairing: ok" is necessary but not sufficient.

## Step 6 — A threshold check loops it back to Chapter 5's thresholds

Chapter 5 sets two thresholds: clear at 60% of the limit, compact at 85%. Model the first one. Add to `drive_clear.py`:

```python
LIMIT = 10_000       # pretend this is the context window
CLEAR_AT = 0.60      # Chapter 5's suggested threshold
KEEP = 2


def manage(messages: list[dict]) -> list[dict]:
    if context_tokens(messages) >= int(LIMIT * CLEAR_AT):
        managed = clear_old_tool_results(messages, keep_recent=KEEP)
        if context_tokens(managed) < int(LIMIT * CLEAR_AT):
            print(f"  clearing dropped context under the 60% threshold")
        else:
            print(f"  clearing was not enough — compaction would run next (Chapter 5 §Compact by summarizing)")
        return managed
    print("  under threshold; no action")
    return messages


print("\n--- threshold check ---")
print(f"limit={LIMIT}, clear_at={int(LIMIT*CLEAR_AT)}, actual={context_tokens(messages):,}")
managed = manage(messages)
print(f"  after: actual={context_tokens(managed):,}")
```

Re-run:

```bash
python3 drive_clear.py
```

**Expected output (new section at the end):**

```
--- threshold check ---
limit=10000, clear_at=6000, actual=6,... (over)
  clearing dropped context under the 60% threshold
  after: actual=1,...
```

Two things to notice:

- **Clearing brought context under the threshold by itself.** On a read-heavy session this is common — Chapter 5: *"it often reclaims most of the window. Do it first."* Exercise 04 instruments the same path with a `context_managed` event so you can see how often clearing is enough and when compaction has to run.
- **The fallback message is where the story continues.** When clearing *isn't* enough, Chapter 5's next move is compaction (a summary call), and after that, a fresh session with `features.json` and `progress.md`. Exercise 04 builds both.

## Step 7 — Checkpoint

Without peeking, you can state:

1. The two message-shape invariants a clearing function must preserve, in the words the provider's error message would use.
2. Why the placeholder names the tool and its input ("read_file src/billing/proration.py") instead of being bare — in one sentence that would fit above `PLACEHOLDER`.
3. Why clearing from the most recent backward passes the structural check but fails the point of clearing — in one sentence that would fit above `clear_old_tool_results`.

## Wrap-up

Carry `clear_old_tool_results`, the `check_pairing` invariant, and the estimator into [Exercise 04](../exercises/exercise-04-context-and-session-state.md), which asks you to:

- Wire token accounting into the loop: use provider usage when available and the estimator as a fallback, and expose `context_tokens()` plus a configurable `context_limit`.
- Trigger clearing at the first threshold (around 60%) using the function from this lab, with a configurable `KEEP_RECENT_RESULTS`.
- Add compaction at the second threshold (around 85%): a separate model call with a small budget, structured headings (Goal, Decisions, Files changed, Current state, Next step), and a replacement history of system + summary + recent turns.
- Build the session-state files from Chapter 5: `features.json` (with a pre-tool hook that rejects any change beyond flipping `passes`), `progress.md`, and git history.
- Prove resume: kill a run partway, start a fresh process, and show it picks up at the next failing feature from the state on disk.

What this lab closes: the *mechanics* of tool-result clearing — the two invariants, the placeholder shape, the right direction, the threshold loop — are no longer a mystery. Exercise 04 is now a question of wiring clearing, compaction, and state files together, not reverse-engineering the clearing function.

### Stretch

- Replace the characters-÷-4 estimator with `anthropic.Anthropic().messages.count_tokens(...)` behind the same `estimate_tokens` function, and compare the two numbers on the fixture. Chapter 5's point that the estimator is "behind one function so you can swap in a real tokenizer later" is real; try it.
- Add a `freeze` block type: a message-level flag that `clear_old_tool_results` never clears, regardless of age. Try it on a result that holds the current `features.json` — the kind of "small detail that turns out to matter later" that Chapter 5 says summaries drop.
- Build a second pass that detects when two recent `tool_result` blocks are identical (same tool, same input, same output) and collapses the older one. That's one way to handle the "model re-ran the test suite three times in a row" shape without touching compaction.
