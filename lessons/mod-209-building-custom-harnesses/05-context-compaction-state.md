# Chapter 5 — Context, Compaction, and Session State

The context window is the harness's scarcest resource. Every file read, every command output, and every turn stays in history and gets sent again on the next call. A coding agent can fill a large window in an hour of work. Long before it's full, quality drops: details buried in the middle of a long context get missed, and the model starts repeating work it has already done. This chapter covers the techniques harnesses use to manage context, from cheap to expensive, and the on-disk state that lets work continue past any single window.

## Count what you're spending

You can't manage what you don't measure. The harness should know at every turn roughly how many tokens the next call will send:

- **Use the provider's numbers when you have them.** Each response reports input and output tokens. The input count for the last call is the best measure of current context size.
- **Estimate when you don't.** Before a call, or for a message you're deciding whether to keep, a rough estimate works. About four characters per token is a common rule of thumb for English and code. Put it behind one function so you can swap in a real tokenizer later.
- **Set thresholds as fractions of the limit**, not absolute numbers, so the same harness works across models with different windows.

## Three levels of reduction

From cheapest and least lossy to most:

### 1. Clear old tool results

Most context in a coding session is tool output: file contents, test logs, search results. The model usually needed them once, to decide what to do next. Twenty turns later, the 400-line file it read at the start is dead weight. It can read it again if it needs to.

**Tool-result clearing** replaces old tool results with a short placeholder, oldest first, keeping the last few intact:

```text
[cleared: read_file src/billing/proration.py — 3,850 tokens. Re-read if needed.]
```

Rules that matter:

- **Keep the tool call; clear only the result.** The model needs to see *that* it read the file, so it doesn't wonder whether it did.
- **Keep calls and results paired.** Provider APIs reject a tool result without its call, or a call without its result. Clear content; don't delete messages.
- **Keep the most recent results.** They're what the model is working with right now.

Clearing loses little, because the information can be fetched again, and it often reclaims most of the window. Do it first. Some providers now offer this as a server-side feature. Building it yourself shows you what it does.

### 2. Compact by summarizing

When clearing isn't enough, **compaction** asks the model to summarize the session so far, then replaces the history with that summary plus the most recent turns. This is what Claude Code's `/compact` and its automatic compaction do.

Compaction loses information, so the summary prompt matters. Ask for structure, not narrative:

```text
Summarize this session for an engineer taking over.
Headings: Goal, Decisions (and why), Files changed, Current state, Next step.
Be specific: file paths, function names, failing test names, exact error messages.
```

Things to get right:

- **Make it a separate call** with its own small budget, not a turn in the main loop.
- **Keep the system prompt and the last few turns verbatim.** The summary replaces the middle, not the whole history.
- **Check you're actually under the threshold afterward.** If you aren't, clear more aggressively or end the session and start fresh.
- **Know what it loses.** Small details that turn out to matter later, like an exact error string or why an approach was rejected, are what summaries drop. When a compacted agent repeats a dead end, that's usually the reason.

### 3. Start a fresh session with state on disk

The most reliable way to reset context is a new session. The cost is that the new session knows nothing, unless the harness gives it a way to catch up quickly. That's what session state files are for.

## Session state on disk

The long-running pattern from [mod-208 Chapter 4](../mod-208-harness-engineering/04-sensors-feedback-and-evals.md) depends on a few files that act as the agent's memory between sessions:

| File | Holds | Who writes it |
|---|---|---|
| `features.json` | Every feature to build, each with `passes: true/false` | Initializer creates it; sessions only flip `passes` |
| `progress.md` | What each session did, what it learned, what's next | Appended at the end of each session |
| git history | The actual changes, with descriptive messages | A commit at the end of each session |
| `init.sh` | How to start the app and run a smoke check | Initializer |

The startup routine for every session reads these first: `progress.md`, `features.json`, `git log --oneline -20`. Then it runs `init.sh` and a smoke check, and picks one failing feature. A new session with a clean context and this routine often does better than a compacted one, because the files hold exactly what's needed and nothing else.

### Enforce the rules in the harness

The feature list only works if the agent can't quietly rewrite it: delete a hard feature, reword one to match what it built, or mark one passing without testing. Telling the agent not to helps. Enforcing it in the harness is better. A pre-tool hook can load the current file and the proposed new one, and reject the write unless the only differences are `passes` values on existing features:

```python
def features_guard(call: ToolCall) -> str | None:
    if not call.input.get("path", "").endswith("features.json"):
        return None
    old = {f["id"]: f for f in json.loads(FEATURES.read_text())}
    new = {f["id"]: f for f in json.loads(call.input["content"])}
    if old.keys() != new.keys():
        return "features.json: you may not add or remove features. Only flip `passes`."
    for fid, feat in new.items():
        if {k: v for k, v in feat.items() if k != "passes"} != {k: v for k, v in old[fid].items() if k != "passes"}:
            return f"features.json: feature {fid} changed beyond `passes`. Only flip `passes`."
    return None
```

This is the harness-engineering habit again. A rule the agent kept breaking moved from an instruction into a mechanical check.

## Memory beyond one task

Session state files are memory for one project or task. Some harnesses also give the agent a general memory store: a directory of notes it can write and read across tasks (Claude Code's memory files are one example). The same rules apply as for instruction files. Keep entries short and specific, keep them current, and remember that everything loaded at startup costs context in every session.

## Key takeaways

- Measure context every turn, using provider usage when available and an estimate behind one function otherwise.
- Clear old tool results first. It's cheap, loses little, and must keep calls and results paired.
- Compact by summary when clearing isn't enough. Ask for structure, keep recent turns verbatim, and know what summaries drop.
- For long work, prefer fresh sessions with state on disk: a JSON feature list, a progress file, git history, and a startup routine.
- Enforce state-file rules in the harness, not just in the instructions.
