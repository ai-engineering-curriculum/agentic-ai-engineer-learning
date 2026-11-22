# Lab 3 — The JSON Feature List Pattern for Long-Running Work

**Duration:** ~45 minutes
**Covers:** [Chapter 4 — Sensors: Feedback Loops and Harness Evals](../04-sensors-feedback-and-evals.md)

## Objective

Chapter 4 describes a long-running-session pattern from Anthropic's write-up: an initializer session, a **JSON** feature list of end-to-end behaviors, a progress file, one feature per session, and git as the checkpoint. Exercise 03 covers the *evals* half of Chapter 4; this lab covers the *session state* half. Spend 45 minutes building the pattern against a tiny mock project and a scripted "agent session" so you can see why the chapter picks JSON over Markdown and why a hook enforces the shape.

By the end of the lab you will:

- Build a toy project with an `init.sh`, a `features.json` of failing features, and a `PROGRESS.md`.
- Run three scripted "sessions" that each read progress, pick one failing feature, mark it passing, append a progress line, and commit.
- Watch a `PreToolUse`-style hook refuse an "agent" that tries to **rewrite** `features.json` wholesale — the exact failure mode Chapter 4 names.
- Feel why Markdown checklists drift and why one-feature-per-session stops the "started five things, finished none" pathology.

Scripted, offline, pure stdlib. The point is to feel the shape of the pattern before you apply it to a real multi-day task.

## Prerequisites

- Python 3.11+ and `git`.
- [Chapter 4](../04-sensors-feedback-and-evals.md) read — especially the "Long-running work across sessions" section.

## Setup

```bash
mkdir lab-03 && cd lab-03
git init -q
git config user.email "lab@local" && git config user.name "lab"
```

## Step 1 — The initializer session

Chapter 4 starts a long-running project with an initializer that sets up everything once: the environment, a feature list of end-to-end behaviors, a progress file, and a first commit. Model it.

Save as `init.sh` and make it executable (`chmod +x init.sh`):

```bash
#!/usr/bin/env bash
# Initializer session: idempotent, sets up the working tree so every later session starts from the same floor.
set -euo pipefail

mkdir -p src tests

cat > src/report.py <<'PY'
def summarize(items):  # unimplemented; features start failing
    raise NotImplementedError
PY

cat > tests/check_e2e.sh <<'PY'
#!/usr/bin/env bash
# End-to-end check: each feature maps to a line here, non-zero on first fail.
set -e
python -c "from src.report import summarize; summarize([])"
PY
chmod +x tests/check_e2e.sh

if [ ! -f features.json ]; then
  cat > features.json <<'JSON'
[
  {"id": "sum-empty",      "desc": "summarize([]) returns \"no items\"",              "passes": false},
  {"id": "sum-one",        "desc": "summarize([{'n':1}]) returns \"1 item, total 1\"", "passes": false},
  {"id": "sum-many",       "desc": "summarize([{'n':1},{'n':2}]) returns \"2 items, total 3\"", "passes": false},
  {"id": "sum-non-ints",   "desc": "summarize rejects non-integers with a ValueError", "passes": false}
]
JSON
fi

if [ ! -f PROGRESS.md ]; then
  echo "# Progress" > PROGRESS.md
fi

echo "init: ok"
```

Three details worth noticing:

- **Idempotent.** `init.sh` can be re-run at the start of every session without clobbering `features.json` or `PROGRESS.md`. Chapter 4's rule — every later session runs the initializer — only works if the initializer is safe to re-run.
- **End-to-end check script.** `tests/check_e2e.sh` is what the session's final verification runs. Chapter 4: *"Unit tests can pass while the feature is broken."* This check is written against the behaviors in `features.json`, not against private helpers.
- **The feature list starts entirely failing.** The whole point of the pattern is a visible floor: *N* features, 0 passing. Progress is measured by lines in `features.json` flipping to `"passes": true`.

Run it and commit:

```bash
./init.sh
git add . && git commit -q -m "init: scaffolding, failing features, progress file"
```

## Step 2 — A scripted session

Chapter 4: *"Every later session starts the same way: read the progress file, the feature list, and the recent git log; run `init.sh`; run a basic end-to-end check to confirm nothing is broken; then pick one failing feature."* Model it in code.

Save as `session.py`:

```python
"""One scripted agent-ish session: pick one failing feature, implement it, verify, record."""
import json
import re
import subprocess
import sys
from pathlib import Path


FEATURES = Path("features.json")
PROGRESS = Path("PROGRESS.md")


def read_features() -> list[dict]:
    return json.loads(FEATURES.read_text())


def flip_passing(feature_id: str) -> None:
    """The only mutation the session is allowed: set one feature's `passes` to true.
    In a real harness this is enforced by a PreToolUse hook — see Step 4."""
    features = read_features()
    found = False
    for f in features:
        if f["id"] == feature_id:
            f["passes"] = True
            found = True
    if not found:
        raise SystemExit(f"session: no feature with id={feature_id!r}")
    FEATURES.write_text(json.dumps(features, indent=2) + "\n")


def implement(feature_id: str) -> None:
    """Scripted implementations keyed by feature id. A real agent would reason here."""
    src = Path("src/report.py")
    if feature_id == "sum-empty":
        src.write_text(
            'def summarize(items):\n'
            '    if not items: return "no items"\n'
            '    raise NotImplementedError\n'
        )
    elif feature_id == "sum-one":
        src.write_text(
            'def summarize(items):\n'
            '    if not items: return "no items"\n'
            '    if len(items) == 1: return f"1 item, total {items[0][\'n\']}"\n'
            '    raise NotImplementedError\n'
        )
    elif feature_id == "sum-many":
        src.write_text(
            'def summarize(items):\n'
            '    if not items: return "no items"\n'
            '    if len(items) == 1: return f"1 item, total {items[0][\'n\']}"\n'
            '    total = sum(i["n"] for i in items)\n'
            '    return f"{len(items)} items, total {total}"\n'
        )
    elif feature_id == "sum-non-ints":
        src.write_text(
            'def summarize(items):\n'
            '    for i in items:\n'
            '        if not isinstance(i.get("n"), int):\n'
            '            raise ValueError(f"non-integer n: {i!r}")\n'
            '    if not items: return "no items"\n'
            '    if len(items) == 1: return f"1 item, total {items[0][\'n\']}"\n'
            '    total = sum(i["n"] for i in items)\n'
            '    return f"{len(items)} items, total {total}"\n'
        )


def end_to_end_passes(feature_id: str) -> bool:
    """Run the behavior for this feature against the real module."""
    try:
        from importlib import reload
        import src.report as r
        reload(r)
        if feature_id == "sum-empty":
            return r.summarize([]) == "no items"
        if feature_id == "sum-one":
            return r.summarize([{"n": 1}]) == "1 item, total 1"
        if feature_id == "sum-many":
            return r.summarize([{"n": 1}, {"n": 2}]) == "2 items, total 3"
        if feature_id == "sum-non-ints":
            try:
                r.summarize([{"n": "x"}])
            except ValueError:
                return True
            return False
    except Exception:
        return False
    return False


def append_progress(line: str) -> None:
    with PROGRESS.open("a") as f:
        f.write(line + "\n")


def commit(message: str) -> None:
    subprocess.run(["git", "add", "-A"], check=True)
    subprocess.run(["git", "commit", "-q", "-m", message], check=True)


def main() -> int:
    # 1. Read prior state (Chapter 4: progress, features, recent git log).
    features = read_features()
    done = [f["id"] for f in features if f["passes"]]
    todo = [f for f in features if not f["passes"]]
    if not todo:
        print("session: all features passing; nothing to do")
        return 0

    # 2. Pick exactly one feature.
    feature = todo[0]
    print(f"session: picked {feature['id']}  (done so far: {done})")

    # 3. Implement + check end to end.
    implement(feature["id"])
    if not end_to_end_passes(feature["id"]):
        print(f"session: end-to-end failed for {feature['id']}; NOT marking passing")
        return 1

    # 4. Flip only this feature's bit, append progress, commit.
    flip_passing(feature["id"])
    append_progress(f"- {feature['id']}: {feature['desc']}")
    commit(f"{feature['id']}: {feature['desc']}")
    print(f"session: {feature['id']} is now passing")
    return 0


if __name__ == "__main__":
    sys.exit(main())
```

Four details that mirror Chapter 4:

- **Read before you write.** Each session reads `features.json` and `PROGRESS.md` first; the state is the source of truth, not the context window of a previous session.
- **One feature per session.** `todo[0]` picks exactly one; the session refuses to start a second one even if context allowed it. Chapter 4: *"One feature per session stops the agent from starting five things and finishing none."*
- **Mark passing only on green.** The session runs the end-to-end check and refuses to flip the bit if it fails. "Claimed done" without passing is the failure mode the pattern exists to prevent.
- **Git as checkpoint.** Every session ends in a commit whose message names the feature. A later session can read `git log --oneline -10` and see what happened; a bad change can be reverted.

## Step 3 — Three sessions in a row

```bash
python session.py
python session.py
python session.py
cat features.json
echo "---"
cat PROGRESS.md
echo "---"
git log --oneline
```

**Expected shape:**

```
session: picked sum-empty  (done so far: [])
session: sum-empty is now passing
session: picked sum-one  (done so far: ['sum-empty'])
session: sum-one is now passing
session: picked sum-many  (done so far: ['sum-empty', 'sum-one'])
session: sum-many is now passing
---
# Progress
- sum-empty: summarize([]) returns "no items"
- sum-one: summarize([{'n':1}]) returns "1 item, total 1"
- sum-many: summarize([{'n':1},{'n':2}]) returns "2 items, total 3"
---
<sha> sum-many: summarize...
<sha> sum-one: summarize...
<sha> sum-empty: summarize...
<sha> init: scaffolding...
```

Three things to notice:

- **The next session has everything it needs to catch up.** `features.json` says which features are done; `PROGRESS.md` is the human-readable breadcrumb; `git log` says *when* each feature closed. None of it requires context from the previous session's window.
- **A fresh clone of this repo at this commit is resumable.** That's the real test of the pattern — not "the current session has the context" but "a completely new agent on a fresh machine can pick up where we left off."
- **The feature list is append-only in spirit.** The session only flips `passes: false → true`, never deletes or reorders entries. Which brings us to Step 4.

## Step 4 — The hook that keeps `features.json` honest

Chapter 4: *"A harness can enforce [that agents may only flip each feature's `passes` field] with a hook, which is what mod-209's exercise-04 has you do."* Model it.

Save as `features_json_guard.py`:

```python
#!/usr/bin/env python3
"""PreToolUse-style guard: refuse rewrites of features.json that change anything but `passes` fields."""
import json
import sys
from pathlib import Path


def is_legal_change(before: list[dict], after: list[dict]) -> tuple[bool, str]:
    if len(before) != len(after):
        return False, (
            "features.json: entry count changed. Agents must not add, remove, "
            "or reorder features — flip only the `passes` field of an existing entry."
        )
    for b, a in zip(before, after):
        if b["id"] != a["id"] or b["desc"] != a["desc"]:
            return False, (
                f"features.json: entry {b['id']!r} id or desc changed. "
                "Agents must not rename or rewrite features."
            )
        for key in set(b) | set(a):
            if key == "passes":
                continue
            if b.get(key) != a.get(key):
                return False, (
                    f"features.json: non-`passes` field {key!r} changed on "
                    f"{b['id']!r}. Agents may only flip `passes`."
                )
        if b.get("passes") and not a.get("passes"):
            return False, (
                f"features.json: {b['id']!r} was passing and would be set back to "
                "failing. If regressions are real, revert the commit instead."
            )
    return True, ""


def main() -> int:
    event = json.load(sys.stdin)
    if event.get("tool_name") not in {"Write", "Edit"}:
        return 0
    path = event.get("tool_input", {}).get("file_path", "")
    if not path.endswith("features.json"):
        return 0

    proposed = event.get("tool_input", {}).get("content")
    if proposed is None:
        print("guard: features.json edit without a proposed content payload; blocking", file=sys.stderr)
        return 2

    try:
        before = json.loads(Path(path).read_text())
        after = json.loads(proposed)
    except Exception as e:
        print(f"guard: could not parse features.json before/after: {e}; blocking", file=sys.stderr)
        return 2

    ok, reason = is_legal_change(before, after)
    if not ok:
        print(reason, file=sys.stderr)
        return 2
    return 0


if __name__ == "__main__":
    sys.exit(main())
```

Try two events through it. First a legal change (the kind `session.py` makes):

```bash
LEGAL=$(python -c '
import json, pathlib
before = json.loads(pathlib.Path("features.json").read_text())
after = [{**f, "passes": True} if f["id"]=="sum-non-ints" else f for f in before]
print(json.dumps({"tool_name": "Write",
                  "tool_input": {"file_path": "features.json",
                                 "content": json.dumps(after, indent=2)}}))
')
echo "$LEGAL" | python3 features_json_guard.py ; echo "exit=$?"
```

**Expected:** `exit=0`.

Now an illegal change — an "agent" that tries to shorten the feature list, or rename a feature, or wipe one back to failing:

```bash
ILLEGAL=$(python -c '
import json, pathlib
before = json.loads(pathlib.Path("features.json").read_text())
after = [{"id": f["id"], "desc": "rewritten!", "passes": True} for f in before[:2]]
print(json.dumps({"tool_name": "Write",
                  "tool_input": {"file_path": "features.json",
                                 "content": json.dumps(after)}}))
')
echo "$ILLEGAL" | python3 features_json_guard.py ; echo "exit=$?"
```

**Expected:**

```
features.json: entry count changed. Agents must not add, remove, or reorder features — flip only the `passes` field of an existing entry.
exit=2
```

Three things to notice:

- **The hook lets `session.py`'s diff through.** A legal change — one `passes` field flipped — is exactly what the pattern wants. Everything else is blocked.
- **The block message explains the alternative.** Chapter 3's rule carried into Chapter 4's pattern: *"Agents must not add, remove, or reorder features — flip only the `passes` field."* The next attempt is much more likely to be correct.
- **A Markdown checklist couldn't be guarded this way.** Chapter 4's write-up found models rewrote or deleted entries in Markdown checklists more often; a hook on a free-text file would have to parse it tolerantly and probably miss edits anyway. JSON gives you a diffable schema; a hook gives you an enforcer for that schema.

<!-- needs-research: confirm whether current Claude Code `Write`/`Edit` events include the full proposed content (field name "content" vs "new_string"); the guard's `proposed` field may need to combine old_string+new_string for `Edit` events. -->

## Step 5 — Checkpoint

Without peeking, you can state:

1. The three artefacts the pattern writes to disk at the end of a session, and why *all three* are necessary to make the next session resumable.
2. One specific failure mode Chapter 4's "one feature per session" rule prevents, in the chapter's own words.
3. Why the chapter prefers JSON to a Markdown checklist for the feature list, and the role a hook plays on top of that choice.

## Wrap-up

This pattern is designed for *outside* the exercise set: a multi-day effort that doesn't fit in one context window. Carry it forward when:

- You're building a feature-by-feature project with a coding agent (a mini-app, a migration, a rewrite).
- The user-visible behavior list is longer than the number of features you can keep in head between sessions.
- A single long session would otherwise drift, start several things, or declare victory on none of them.

What's not in this lab, on purpose:

- **Real evals of whether the pattern helped.** That's [Exercise 03](../exercises/exercise-03-harness-eval.md): fixed task set, repeated trials, pass rate / cost / turns, honest uncertainty. The lab you just ran gives you *one* trajectory; the exercise gives you numbers.
- **A real `PreToolUse` wire-up in `.claude/settings.json`.** The guard here is piped from the shell exactly like [Lab 02](lab-02-pipe-hook-events-to-feel-the-contract.md); wiring into a live session is [Exercise 02](../exercises/exercise-02-guardrail-hooks.md)'s territory.
- **A full `init.sh` that starts the app and runs an end-to-end browser check.** For UI work, Chapter 4 reports browser automation moved the needle; the `tests/check_e2e.sh` here is a stand-in.

### Stretch

- Re-run the three sessions after replacing `features.json` with `features.md`, a Markdown checklist (`- [ ] sum-empty: ...`). Write a `session_md.py` that tries to tick a box. Try an "agent rewrite" that renames an item. Compare how hard it is to detect — Chapter 4's JSON-vs-Markdown choice, felt.
- Add a `SessionStart`-style injection: a tiny script that prints the current `features.json`, the last 3 progress lines, and `git log --oneline -5` to stdout. That's the shape of what Chapter 3's `SessionStart` row of the hook table does — primes the agent with "here's where you left off" instead of letting it rebuild context from the filesystem.
- Inject a bug: have one of `session.py`'s implementations *almost* work but fail the end-to-end check. Confirm the session refuses to flip the feature's bit and that the next session re-picks the same feature. The "mark passing only on green" rule in action.
