# exercise-02: A Coding Tool Suite with Isolated Execution

**Estimated effort:** 4 hours

## Objective

Replace the four basic tools from exercise-01 with a tool suite designed for a coding agent: line-numbered reads with paging, exact-match edits that fail loudly when they're ambiguous, file search by pattern and by content, and command execution behind a pluggable sandbox. Add git checkpoints so every agent turn can be rolled back. By the end your harness's tools will behave like the ones in production coding agents, and you'll know why each design choice is there.

## Background

This exercise covers material from:

- [Chapter 3 — Tools for a Coding Agent](../03-coding-tools.md)
- [Chapter 4 — Execution Isolation and Checkpoints](../04-execution-isolation.md)

Build on your [exercise-01](exercise-01-build-a-minimal-harness.md) harness or its reference solution. Keep the registry, permission gate, and hooks; this exercise changes the tools and what runs them.

## Prerequisites

- A working exercise-01 harness.
- Python 3.11+, `pytest`, and `git`.
- Docker for the container runner (task 5). The tests for everything else run without it.

## Tasks

### 1. `read_file` with line numbers and paging

- Return content with line numbers (a right-aligned number, a tab, then the line, like `cat -n`), so the model can refer to exact lines.
- Accept `offset` and `limit` (default limit 2,000 lines). When a file is cut, end with a note saying how to read the rest.
- Truncate very long single lines (say, over 2,000 characters) with a marker.
- Detect binary files and return a short description instead of their bytes.
- Record each file read and its modification time in a per-session **read ledger**.

### 2. `edit_file` with exact-match replacement

- Input: `path`, `old_string`, `new_string`, optional `replace_all` (default `false`).
- Fail with a clear error if `old_string` isn't found, or if it appears more than once and `replace_all` is false. Say how many matches there were and suggest adding surrounding lines to make it unique.
- **Read-before-edit:** refuse to edit a file that isn't in the read ledger, or whose modification time has changed since it was read ("file changed since you read it; read it again").
- Return a short confirmation with a few lines of context around the change, not the whole file.
- Keep `write_file` for new files only. Overwriting an existing file also requires a prior read.

### 3. `glob` and `grep`

- `glob(pattern, path=".")`: return matching paths inside the workspace, newest first, capped at a limit with a "more results" note.
- `grep(pattern, path=".", glob=None, output="files"|"content"|"count", context=0)`: regex search. Use `rg` (ripgrep) when it's installed and a pure-Python fallback when it isn't. Both must give the same results on the test fixtures.
- Both skip `.git/`, and both respect a simple ignore list (`node_modules/`, `.venv/`, `__pycache__/`).

### 4. `bash` behind a `CommandRunner`

- Define a `CommandRunner` protocol: `run(argv, cwd, timeout, env) -> CommandResult` (`exit_code`, `stdout`, `stderr`, `timed_out`, `duration_s`).
- `LocalRunner`: `subprocess.run` with no shell, a scrubbed environment (only an allowlist of variables such as `PATH`, `HOME`, `LANG`; never the API key), and the head-and-tail truncation from exercise-01.
- The `bash` tool accepts a command string, splits it with `shlex`, applies the program allowlist from exercise-01, and hands the result to the runner. Report non-zero exits as normal results (the model needs to see test failures), and report timeouts as errors.

### 5. `DockerRunner`

- Runs each command in a container: `--network none`, workspace mounted at `/workspace`, read-only root filesystem with a writable `/tmp`, CPU and memory limits, `--user` set to your UID, and `--rm`.
- Same interface as `LocalRunner`; the harness picks one from config.
- Skip its tests automatically when Docker isn't available.

### 6. Git checkpoints

- Before each model turn that might change files, record a checkpoint: commit the working tree to a private ref (`refs/harness/checkpoints/<n>`) without touching the user's branch or index. (Hint: a temporary index via `GIT_INDEX_FILE`, then `git write-tree` and `git commit-tree`.)
- Add `rewind(n)` to restore the working tree to checkpoint `n`.
- If the workspace isn't a git repo, checkpoints are disabled with a warning, not a crash.

## Starter guidance

```python
@dataclass(frozen=True)
class ReadRecord:
    path: Path
    mtime_ns: int

class ReadLedger:
    """Which files the model has seen this session, and when."""
    def __init__(self) -> None:
        self._records: dict[Path, ReadRecord] = {}

    def record(self, path: Path) -> None: ...
    def check_fresh(self, path: Path) -> str | None:
        """None if safe to edit, else an error message for the model."""
```

```python
def apply_edit(text: str, old: str, new: str, replace_all: bool) -> str:
    count = text.count(old)
    if count == 0:
        raise EditError("old_string not found. Read the file again and copy the exact text, including whitespace.")
    if count > 1 and not replace_all:
        raise EditError(f"old_string matches {count} places. Include more surrounding lines so it matches one, or set replace_all.")
    return text.replace(old, new) if replace_all else text.replace(old, new, 1)
```

## Acceptance criteria

You can demonstrate that, in `pytest`:

- `read_file` numbers lines, pages with `offset`/`limit`, flags truncation, and describes binary files instead of returning them.
- `edit_file` rejects zero matches and ambiguous matches with actionable messages, and applies unique and `replace_all` edits correctly.
- `edit_file` refuses edits to unread files and to files modified after they were read.
- `glob` and `grep` stay inside the workspace, skip ignored directories, and give identical results with `rg` and the Python fallback.
- `LocalRunner` doesn't pass the API key or other non-allowlisted environment variables to the child process.
- Checkpoints restore a modified, a deleted, and a newly created file, and leave the user's branch, index, and `HEAD` unchanged.
- With Docker present: a `DockerRunner` command can't reach the network and can't write outside `/workspace` and `/tmp`.

## Reflection

In `NOTES.md`:

1. Why does exact-match replacement work better for a model than "rewrite the whole file" or line-number-based edits?
2. What does read-before-edit protect against, in a harness where a human or another process may also be editing files?
3. Your `LocalRunner` scrubs the environment but still runs on the host. List what a malicious or confused command could still do, and which of those `DockerRunner` stops.

## Stretch goals

- A `multi_edit` tool that applies several replacements to one file atomically: all of them or none.
- A persistent shell session (one long-lived `bash` process, commands sent over stdin with a sentinel to detect completion), so `cd` and exported variables carry over between calls.
- A `--sandbox` mode using `bwrap` (Linux) or `sandbox-exec` (macOS) as a lighter alternative to Docker.
