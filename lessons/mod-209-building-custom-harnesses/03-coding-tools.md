# Chapter 3 — Tools for a Coding Agent

The four tools from exercise-01 (`read_file`, `write_file`, `list_dir`, `run_command`) are enough to work, but not enough to work well. Production coding harnesses converge on a similar tool set, and the details of each tool, such as how it numbers lines, how it reports a failed edit, and what it returns from a search, have a large effect on how reliably the agent works. This chapter goes through that tool set and the reasons behind each design choice.

## Tool descriptions are prompts

A tool's name, description, and input schema go into the model's context on every call. They're the model's only documentation for the tool, so write them like a short manual page:

- **Say when to use it and when not to.** "Use `grep` to search file contents. Don't use `bash` with `grep` or `rg`; this tool is faster and respects ignore files."
- **State the contract.** "`old_string` must match exactly once, including whitespace and indentation."
- **Describe the output.** "Returns matching file paths, newest first, at most 100."
- **Name each parameter clearly and describe it.** `path`, not `p`. Mark required fields as required.

Keep descriptions accurate as the tool changes. A description that promises behavior the tool doesn't have wastes turns as the model works out the difference.

## Reading files

```text
     1	import json
     2	from pathlib import Path
     3
     4	def load(path: str) -> dict:
```

- **Line numbers.** They let the model refer to exact places ("the bug is on line 48"), check that an edit landed where it meant, and page through large files. Use a fixed-width number and a tab, so the numbers are clearly not part of the content. Tell the model in the edit tool's description to leave them out of `old_string`.
- **Paging.** Return a default window (for example 2,000 lines), accept `offset` and `limit`, and end a cut-off read with a note: "File has 5,210 lines; showing 1–2000. Use offset=2001 to continue." Without the note, the model assumes it has seen everything.
- **Long lines.** A minified bundle or a data file can have one line of 500k characters. Truncate each line past a limit, with a marker.
- **Binary files.** Detect them (a null byte in the first few KB is a good heuristic) and return a description such as "binary file, 48 KB, PNG image" instead of bytes. Images can go to multimodal models as image blocks; that's a stretch goal.
- **The read ledger.** Record every file read and its modification time. The edit tool needs it.

## Editing files

How the model changes files is the most consequential tool decision in a coding harness. There are three common designs:

| Design | How it works | Problem |
|---|---|---|
| Whole-file write | Model sends the full new file | Expensive for large files; easy to drop unrelated lines; silent damage |
| Line-number edits | "Replace lines 40–52 with ..." | Line numbers shift after each edit; off-by-one errors |
| Exact-match replace | "Replace this exact text with that text" | Needs the text to be unique, which the tool enforces |

Production harnesses mostly use **exact-match replacement** (Claude Code's `Edit`, and the `str_replace` command in Anthropic's text-editor tool), with a whole-file write for new files. Codex uses a patch format with context lines, which shares the key property: the change is anchored to content, not position.

Why exact match works well:

- **It's self-checking.** If the text isn't there, the model's picture of the file is wrong, and the edit fails before doing damage.
- **It forces a read first.** The model can't produce an exact match without having seen the current content.
- **Edits are small.** The model sends only what changes plus enough context to make it unique.

The failure messages matter as much as the success path:

```text
old_string not found in src/app.py. The file may have changed, or whitespace differs.
Read the file again and copy the exact text, without line-number prefixes.

old_string matches 3 places in src/app.py (lines 14, 88, 140).
Include more surrounding lines so it matches one place, or set replace_all=true.
```

Each tells the model exactly what to do next. On success, return a few lines around the change, not the whole file. The model gets confirmation that the edit landed where it meant, at little cost.

### Read before edit

The read ledger enables a simple, important rule: **you can't edit a file you haven't read, or one that changed since you read it.** This catches three real problems:

- The model "remembers" a file from its training data or an earlier session and edits it from memory.
- A human, a formatter hook, or another agent changed the file after the model read it.
- A compaction removed the file's content from context, and the model edits from a summary.

Check modification time (nanoseconds) at edit time against the ledger. On a mismatch, refuse with "src/app.py changed since you last read it; read it again." After a successful edit, update the ledger with the new modification time, since the harness made the change.

## Searching

An agent in an unfamiliar codebase spends much of its time finding things. Give it purpose-built search tools rather than relying on `find` and `grep` through a shell:

- **`glob(pattern)`**: find files by name. Return paths, newest first, since recently changed files are more often relevant. Cap the count and say when results were cut.
- **`grep(pattern)`**: search contents with a regex. Offer output modes: just file names (cheap, good first step), matching lines with optional context, or counts. Allow a file filter (`glob="*.py"`).

Use ripgrep (`rg`) underneath when it's available: it's fast and respects `.gitignore`. Keep a pure-Python fallback, so the harness works anywhere. Both skip `.git/` and common dependency and build directories. Searching `node_modules/` floods context with useless matches.

Why not just let the agent use `bash` for search? You can, and agents will. Dedicated tools give you control over output size, ignore rules, and workspace boundaries, and they let you mark search as read-only, so the harness can run several searches in parallel (Chapter 6) and allow them without permission prompts.

## Running commands

The shell tool is where the agent runs tests, builds, linters, and git. It needs a few properties beyond exercise-01's version:

- **Non-zero exit is a result, not an error.** A failing test suite is the most useful output the agent gets. Return the exit code, stdout, and stderr as a normal result. Reserve the error flag for the harness failing to run the command at all, or a timeout.
- **A scrubbed environment.** Start from an empty environment and add an allowlist (`PATH`, `HOME`, `LANG`, maybe `TERM`). Never pass through the harness's own API key or cloud credentials. A model that runs `env` will print whatever it finds.
- **Bounded output.** Keep the head and tail, and say how many lines were dropped. Test runners and compilers put the summary at the end.
- **A pluggable runner.** The tool builds the command; a `CommandRunner` decides where it runs: locally, in a container, or in a remote sandbox. Chapter 4 covers runners.

### Persistent shells

Claude Code's Bash tool keeps a persistent shell session, so `cd` and exported variables carry over between calls. That's convenient, and it matches how humans work. It also adds complexity: you have to detect when a command has finished in a long-lived process (usually by echoing a unique sentinel and the exit code), handle commands that never finish, and decide what happens to background processes. A stateless runner, a new process per call with an explicit `cwd`, is simpler and easier to sandbox. Start stateless; add persistence if your users need it (exercise-02 stretch goal).

## Tool output budget

Every tool result enters context and stays there until it's cleared (Chapter 5). Set a size limit for every tool, and when output is cut, say so and say how to get more. A useful habit is to log the token size of each tool result and look at the largest ones from a week of runs. The top offenders are usually easy fixes: a test runner in verbose mode, a search without a file filter, a file read that should have used `offset`.

## Key takeaways

- Tool descriptions are prompts: say when to use the tool, state the contract, describe the output.
- Read with line numbers, paging, long-line truncation, and binary detection, and record reads in a ledger.
- Edit with exact-match replacement that fails loudly on zero or multiple matches, and require a fresh read before every edit.
- Give the agent dedicated `glob` and `grep` tools with ignore rules, output modes, and limits.
- Run commands with exit codes as results, a scrubbed environment, bounded output, and a pluggable runner.
- Budget every tool's output, say when it was cut, and look at the largest results regularly.
