# Lab 2 — Exact-Match Edit and the Read Ledger

**Duration:** ~45 minutes
**Covers:** [Chapter 3 — Tools for a Coding Agent](../03-coding-tools.md)

## Objective

Chapter 3 calls the edit tool "the most consequential tool decision in a coding harness" and converges on exact-match replacement with a read-before-edit rule. Before [Exercise 02](../exercises/exercise-02-coding-tool-suite.md) asks you to build the full tool suite (paged reads, `glob`, `grep`, a Docker runner, git checkpoints) and wire it into the harness, spend 45 minutes driving the **edit tool and read ledger in isolation** — pure functions, hand-crafted calls, no model, no registry.

By the end of the lab you will:

- Build `apply_edit` as a 10-line pure function and drive it through zero-match, multi-match, and successful-unique cases from the shell.
- Build a `ReadLedger` keyed on modification time, and watch it refuse an edit to an unread file and to a file changed since it was read — Chapter 3's three "real problems" the check catches.
- Feel why the chapter insists the failure *messages* matter as much as the success path: swap a bare `"not found"` for the chapter's suggestion and see what the next agent turn would have to guess.
- Weigh exact-match replacement against the two designs the chapter rules out (whole-file write, line-number edits) by trying the same change under each.

Scripted, offline, pure stdlib. The point is to feel the shape of the design decisions before the full tool meets a model.

## Prerequisites

- Python 3.11+.
- [Chapter 3](../03-coding-tools.md) read, especially the "Editing files" and "Read before edit" sections.
- No external packages.

## Setup

```bash
mkdir lab-02 && cd lab-02
mkdir workspace
cat > workspace/app.py <<'PY'
def greet(name):
    return f"hello, {name}"

def farewell(name):
    return f"goodbye, {name}"
PY
```

## Step 1 — `apply_edit` as a pure function

Chapter 3 is specific about what the three failure messages should contain. Encode them as a pure function. Save as `edit.py`:

```python
"""Exact-match replace, as in Claude Code's Edit and the str_replace command."""


class EditError(ValueError):
    """Why the model's edit can't apply yet. The message goes back as the tool result."""


def apply_edit(text: str, old: str, new: str, replace_all: bool = False) -> str:
    count = text.count(old)
    if count == 0:
        raise EditError(
            "old_string not found. The file may have changed, or whitespace differs. "
            "Read the file again and copy the exact text, without line-number prefixes."
        )
    if count > 1 and not replace_all:
        # Report line numbers; models work from line numbers in Read output.
        lines = text.splitlines(keepends=True)
        hits = []
        pos = 0
        for i, line in enumerate(lines, start=1):
            if old in text[pos:pos + len(line) + len(old)] and old in line:
                hits.append(i)
            pos += len(line)
        raise EditError(
            f"old_string matches {count} places in the file"
            + (f" (lines {hits})" if hits else "")
            + ". Include more surrounding lines so it matches one place, or set replace_all=true."
        )
    return text.replace(old, new) if replace_all else text.replace(old, new, 1)
```

Three details worth noticing:

- **The function is pure.** `(text, old, new, replace_all) -> text`. It doesn't touch the filesystem, doesn't care about the ledger yet, doesn't know what a tool result is. Chapter 1's "pure core" rule is easier to follow in small pieces.
- **The error messages are the whole value.** Chapter 3: *"Each tells the model exactly what to do next."* Bare `raise ValueError("not found")` is a wasted turn.
- **`replace_all=False` by default.** If the model is uncertain, make it be explicit. The multi-match error even suggests the escape hatch (`set replace_all=true`) in the same message.

## Step 2 — Drive the three cases from the shell

Save as `drive_edit.py`:

```python
from pathlib import Path
from edit import apply_edit, EditError

app_py = (Path("workspace") / "app.py").read_text()


def show(label: str, fn):
    print(f"\n--- {label} ---")
    try:
        result = fn()
        preview = result.splitlines()[:2]
        print(f"OK (first 2 lines: {preview})")
    except EditError as exc:
        print(f"ERR: {exc}")


# Case 1: unique match — succeeds.
show("unique match",
     lambda: apply_edit(app_py, 'return f"hello, {name}"',
                                'return f"Hello, {name.title()}"'))

# Case 2: zero match — fails with Chapter 3's wording.
show("zero match",
     lambda: apply_edit(app_py, 'return f"HELLO, {name}"', 'nope'))

# Case 3: ambiguous match — fails with the count.
show("ambiguous match",
     lambda: apply_edit(app_py, 'return f"', 'return f"PREFIX_'))

# Case 4: ambiguous but opt-in — succeeds under replace_all.
show("ambiguous + replace_all",
     lambda: apply_edit(app_py, 'return f"', 'return f"PREFIX_', replace_all=True))
```

Run it:

```bash
python3 drive_edit.py
```

**Expected output:**

```
--- unique match ---
OK (first 2 lines: ['def greet(name):', '    return f"Hello, {name.title()}"'])

--- zero match ---
ERR: old_string not found. The file may have changed, or whitespace differs. Read the file again and copy the exact text, without line-number prefixes.

--- ambiguous match ---
ERR: old_string matches 2 places in the file (lines [2, 5]). Include more surrounding lines so it matches one place, or set replace_all=true.

--- ambiguous + replace_all ---
OK (first 2 lines: ['def greet(name):', '    return f"PREFIX_hello, {name}"'])
```

Four things to notice:

- **The unique case succeeds silently** — Chapter 3's success path: *"return a few lines around the change, not the whole file."* In a full tool you'd enclose `result` in a small confirmation block; here the output lets you see which lines changed.
- **Zero match tells the model why the match might have missed** (file changed, whitespace, line-number prefixes). Three plausible reasons — one is almost always the right one, and the model's next turn can pick.
- **Ambiguous match reports the count and the line numbers.** Chapter 3: *"Include more surrounding lines so it matches one place, or set replace_all=true."* Both escape hatches are in the message; the model doesn't have to guess which it's allowed to use.
- **`replace_all=True` opts out of self-checking** — use it when the intent *is* a global rename. Chapter 3 keeps it as an input flag rather than a separate tool so the model's calling shape stays simple.

## Step 3 — Weaken a message and see the next turn suffer

Chapter 3: *"The failure messages matter as much as the success path."* Temporarily edit `edit.py` to weaken the zero-match message:

```python
if count == 0:
    raise EditError("not found")
```

Re-run:

```bash
python3 drive_edit.py 2>&1 | grep "zero match" -A1
```

**Expected:**

```
--- zero match ---
ERR: not found
```

Now stand in for the model. You asked to replace `return f"HELLO, {name}"`. You now know the harness refused. You do not know:

- Whether the problem is that *that text* isn't in the file — maybe you misread the content.
- Whether the problem is *whitespace* — maybe you read `    return f"hello"` but typed `	return f"hello"` with a tab.
- Whether the problem is that *the file has changed* since you read it — someone or something else ran an edit.
- Whether you should *read the file again* (yes), *copy the exact bytes* (yes), or *strip line-number prefixes* (yes, if your read tool added them).

The chapter's point: without the "do this instead" sentence, the model's next turn is a lottery. Revert the change before continuing:

```python
if count == 0:
    raise EditError(
        "old_string not found. The file may have changed, or whitespace differs. "
        "Read the file again and copy the exact text, without line-number prefixes."
    )
```

## Step 4 — A read ledger with modification times

Chapter 3: *"you can't edit a file you haven't read, or one that changed since you read it."* Save as `ledger.py`:

```python
"""Per-session record of which files the model has read, and when."""
from dataclasses import dataclass
from pathlib import Path


@dataclass(frozen=True)
class ReadRecord:
    path: Path
    mtime_ns: int


class ReadLedger:
    def __init__(self) -> None:
        self._records: dict[Path, ReadRecord] = {}

    def record(self, path: Path) -> None:
        """Called by the read tool after a successful read."""
        resolved = path.resolve()
        self._records[resolved] = ReadRecord(resolved, resolved.stat().st_mtime_ns)

    def check_fresh(self, path: Path) -> str | None:
        """Return None if safe to edit, else a message for the model."""
        resolved = path.resolve()
        rec = self._records.get(resolved)
        if rec is None:
            return (f"{path} is not in the read ledger. "
                    "Read it first so I know you've seen the current content.")
        try:
            current_mtime = resolved.stat().st_mtime_ns
        except FileNotFoundError:
            return f"{path} no longer exists. Read the current state before editing."
        if current_mtime != rec.mtime_ns:
            return (f"{path} changed since you last read it "
                    f"({rec.mtime_ns} → {current_mtime}). Read it again and re-plan the edit.")
        return None

    def update_after_edit(self, path: Path) -> None:
        """Called by the edit tool after a successful write, since the harness made the change."""
        self.record(path)
```

Three details worth noticing:

- **`mtime_ns`, not `mtime`.** POSIX mtime rounds to the second; a formatter hook running inside the same second as a read can slip past a second-resolution check. Nanoseconds are the smallest unit the stdlib exposes; Exercise 02 keeps this field.
- **Three failure messages, three distinct problems.** Not in ledger ("read first"), file deleted ("read current state"), mtime changed ("read again and re-plan"). Each tells the model what to do next.
- **`update_after_edit` closes the loop.** The harness just changed the file, so the ledger records the new mtime; without this, every edit except the first would fail the freshness check.

## Step 5 — Drive the three "real problems" from Chapter 3

Save as `drive_ledger.py`:

```python
import os
import time
from pathlib import Path
from edit import apply_edit, EditError
from ledger import ReadLedger

APP = Path("workspace/app.py")
ledger = ReadLedger()


def read_tool(path: Path) -> str:
    text = path.read_text()
    ledger.record(path)
    return text


def edit_tool(path: Path, old: str, new: str) -> str:
    check = ledger.check_fresh(path)
    if check:
        raise EditError(check)
    text = path.read_text()
    new_text = apply_edit(text, old, new)
    path.write_text(new_text)
    ledger.update_after_edit(path)
    return f"edited {path}"


def show(label: str, fn):
    print(f"\n--- {label} ---")
    try:
        print("OK:", fn())
    except EditError as exc:
        print(f"ERR: {exc}")


# Case 1: edit without reading — simulates the model editing "from memory."
show("edit before read",
     lambda: edit_tool(APP, 'hello', 'HELLO'))

# Case 2: read, then edit — the happy path.
read_tool(APP)
show("read then edit",
     lambda: edit_tool(APP, 'hello', 'HELLO'))

# Case 3: outside process changes the file after the read (a formatter hook,
# a human, another agent) — simulated by writing a new mtime.
time.sleep(0.01)  # ensure a different mtime_ns on fast filesystems
APP.write_text(APP.read_text() + "\n# touched by someone else\n")

show("outside change after read",
     lambda: edit_tool(APP, 'HELLO', 'howdy'))

# Case 4: re-read to recover.
read_tool(APP)
show("re-read then edit",
     lambda: edit_tool(APP, 'HELLO', 'howdy'))
```

Run it:

```bash
python3 drive_ledger.py
```

**Expected output:**

```
--- edit before read ---
ERR: workspace/app.py is not in the read ledger. Read it first so I know you've seen the current content.

--- read then edit ---
OK: edited workspace/app.py

--- outside change after read ---
ERR: workspace/app.py changed since you last read it (...→...). Read it again and re-plan the edit.

--- re-read then edit ---
OK: edited workspace/app.py
```

Four things to notice:

- **The three scenarios map one-to-one onto Chapter 3's list of problems.** "Edit before read" is the model editing from training-data memory or an earlier session; "outside change" is a human, a formatter hook, or another agent; the re-read recovery is the compaction case from Chapter 5 (the file left context; reading it brings the current state back).
- **The messages name what changed.** "Not in the ledger" is a distinct instruction from "changed since you last read it." A single "edit refused" message would make the next-turn recovery guesswork — one right answer (read, then retry) in one case; a different right answer (read *and re-plan*, since the content shifted) in the other.
- **The ledger is a thin wrapper around `stat().st_mtime_ns`.** Chapter 3's claim isn't that you need a complicated system; it's that you need *any* record of what the model has seen. The data model is one `dict[Path, ReadRecord]`.
- **The ledger is per-session.** Across sessions, Chapter 5's session state and compaction take over; across concurrent sessions, Chapter 4's worktrees mean each agent has its own ledger because each has its own workspace.

## Step 6 — Feel why exact-match beats the alternatives

Chapter 3 compares exact-match against two designs it rules out. Try the same change under each and feel the cost. First, **whole-file write**:

```python
# Whole-file write — the model sends the entire new file.
new_contents = '''def greet(name):
    return f"howdy, {name}"

def farewell(name):
    return f"goodbye, {name}"
'''

APP.write_text(new_contents)
print("whole-file write: no self-check; if the model forgets `farewell`, it silently disappears")
```

The chapter's problem: the write is positional, not anchored. If the model forgot the `farewell` block when constructing `new_contents`, the file loses `farewell` and nothing complains. Expensive for large files, easy to drop unrelated lines, **silent damage**.

Now **line-number edits**:

```python
# Line-number edits — "Replace lines 2-2 with ..."
lines = APP.read_text().splitlines(keepends=True)
# The agent wants to edit the `greet` return. Line 2 — but that's only true until
# some other edit shifts it. On the second edit of the same file in a session, the
# line number is already stale.
new_line = '    return f"howdy, {name}"\n'
lines[1] = new_line
APP.write_text("".join(lines))
print("line-number edits: off-by-one on index 1 vs 2; shifts after each prior edit")
```

The chapter's problem: line numbers shift after each edit. The agent's picture of line 2 is only true until something else changes the file above line 2; chained edits are a classic source of off-by-one bugs that look right for one call and corrupt the file on the second.

**Exact-match replacement sits between these two.** It's anchored to content (unlike line numbers), so it survives a prior edit above it; and it fails loudly when the anchor isn't there (unlike whole-file writes), so the model can't silently destroy unrelated content. The read ledger adds one more rail: you can't produce an exact match without having seen the current content.

Revert `workspace/app.py` and move on:

```bash
cat > workspace/app.py <<'PY'
def greet(name):
    return f"hello, {name}"

def farewell(name):
    return f"goodbye, {name}"
PY
```

## Step 7 — Checkpoint

Without peeking, you can state:

1. The three error cases `apply_edit` distinguishes and what each error message tells the model to do next.
2. The three "real problems" the read ledger's mtime check catches, in the exact order they appear in Chapter 3.
3. Why a bare `"edit failed"` message is worse than an eight-word "the file may have changed, or whitespace differs" — in one sentence that would fit inside `EditError`.

## Wrap-up

Carry `apply_edit`, `ReadLedger`, and the four failure messages into [Exercise 02](../exercises/exercise-02-coding-tool-suite.md), which asks you to:

- Wrap `apply_edit` as a registered `edit_file` tool with a JSON Schema, slot it behind the loop's `execute()` from Exercise 01, and route a short confirmation ("3 lines changed near `greet`") back as the tool result on success.
- Pair it with `read_file` that returns `cat -n`-style line-numbered output with paging (`offset`, `limit`), truncates very long lines, describes binary files, and records each read in the ledger before returning.
- Add `glob` and `grep` with ripgrep underneath and a pure-Python fallback, both skipping `.git/`, `node_modules/`, `.venv/`, and `__pycache__/`.
- Pluggable commands behind a `CommandRunner` protocol, with a `LocalRunner` (scrubbed env, no shell) and a `DockerRunner` (no network, read-only root, `--user`).
- Git checkpoints per turn, using a private index (`GIT_INDEX_FILE`) and private refs (`refs/harness/checkpoints/*`) so the user's branch, index, and `HEAD` are untouched.

What this lab closes: the *semantics* of the edit tool — why exact match, why read-before-edit, why these four failure messages — are no longer a mystery. Exercise 02 is now a question of wrapping these into a registered tool and placing a sandbox underneath, not reverse-engineering the design.

### Stretch

- Add a fourth failure case to `apply_edit`: when `old_string` is the empty string. (Pass `old=""` to `str.replace` and see what happens; think about what a model asking to replace an empty string most likely meant.) Compare your refusal message to Claude Code's current behavior.
- Add a `multi_edit` function that takes a list of `(old, new)` pairs and applies them atomically — all or none. Decide whether the ledger check runs once per file or once per edit, and defend your choice in two sentences.
- Instrument the ledger with an in-memory event log: every `record`, `check_fresh` result, and `update_after_edit` becomes one dict. Then run `drive_ledger.py` and inspect the log — this is the shape Exercise 06's JSONL run log will carry for real tools.
