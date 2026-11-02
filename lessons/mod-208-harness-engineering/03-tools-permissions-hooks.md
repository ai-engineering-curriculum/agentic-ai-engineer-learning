# Chapter 3 — Tools, Permissions, and Hooks

Instruction files ask. Tools, permissions, and hooks decide. This chapter covers the parts of the harness that set what the agent can do, what it may do without asking, and what happens around every action whatever the model chooses. These are where you put rules that must hold every time.

## Tools are an interface for a model

Every tool is an interface whose user is a model. The design rules from [mod-201](../mod-201-agent-fundamentals/README.md) and [mod-202](../mod-202-frameworks/README.md) still apply. For a coding harness, a few matter most:

- **Fewer, higher-level tools beat many thin ones.** A `run_tests(path)` tool that returns only the failures is better than handing the agent a shell and hoping it remembers the flags. Each tool adds to the decision the model makes on every turn.
- **Return what the model needs, not everything.** A test run that dumps 4,000 lines of passing output wastes context. Return the failures, a count, and how to rerun one test.
- **Errors should say what to do next.** "File not found: `src/app.py`. Did you mean `src/tool/app.py`?" gets a better next step than a stack trace.
- **MCP servers are tools too.** Connecting a database, issue tracker, or browser over the Model Context Protocol widens what the agent can see and do. Add servers for a reason, scope them (read-only by default), and remember that every tool description uses context.

## Permissions: what runs without asking

Coding agents sort tool calls into three outcomes: allowed, denied, or ask the human. The permission policy is how you choose which is which.

- **Allow** the things the agent needs to do constantly and that are safe to repeat: reading files, running the test suite, the linter, `git status`, `git diff`. Every unnecessary prompt slows the human down and teaches them to approve without reading.
- **Deny** things that should never happen in this repo: reading `.env` and credential files, force-pushing, deleting outside the build directory, publishing a package.
- **Ask** for the rest: anything with external side effects, anything irreversible.

In Claude Code this is `permissions.allow` / `permissions.deny` / `permissions.ask` in `.claude/settings.json` (checked in for the team) and `.claude/settings.local.json` (personal). Codex combines a sandbox mode (read-only, workspace-write, or full access) with an approval policy in `config.toml`. Check each tool's docs for the current syntax. These settings change often.

## Sandboxing: the real boundary

Permission rules match patterns on tool calls. Patterns can be got around: a denied command can be put in a script and run, written as `python -c "..."`, or built from pieces. A rule that says "deny `rm -rf`" is a helpful guardrail, not a security boundary.

The boundary is the sandbox: what the process can reach at the operating-system level. Run agents with filesystem access limited to the workspace, no network or an allowlist of hosts, and no credentials they don't need. Claude Code and Codex both ship OS-level sandboxing (Seatbelt on macOS, bubblewrap or Landlock on Linux). Containers and VMs are the stronger option for anything unattended. The rule of thumb from [mod-206](../mod-206-guardrails-implementation/README.md) still holds: assume the model can be talked into anything, and make sure the environment limits the damage.

## Hooks: deterministic code at lifecycle points

A hook is a program the harness runs at a set point in the agent's lifecycle. Whatever the model decides, the hook runs. That makes hooks the right place for rules that must hold every time.

Claude Code's main events:

| Event | Fires | Typical use |
|---|---|---|
| `SessionStart` | When a session begins | Inject the current branch, recent commits, open issue |
| `UserPromptSubmit` | Before the model sees a prompt | Add context, block prompts with secrets |
| `PreToolUse` | Before a tool call runs | Block dangerous commands or protected paths |
| `PostToolUse` | After a tool call succeeds | Format or lint the edited file; feed results back |
| `Stop` | When the agent tries to finish | Run the check command; send it back if it fails |
| `SubagentStop` | When a subagent finishes | Validate the subagent's output |

The contract is simple. The hook gets a JSON event on stdin describing what's happening. Its exit code and output decide what happens next. Exit `0` means continue. Exit `2` means block, and whatever the hook wrote to stderr goes to the model as the reason. Hooks can also print structured JSON for finer control, such as a `Stop` hook returning `{"decision": "block", "reason": "..."}` to keep the agent working.

```python
#!/usr/bin/env python3
"""PreToolUse: block edits to protected paths and tell the model why."""
import json, sys
from fnmatch import fnmatch

PROTECTED = [".env*", ".git/*", "*.lock", "migrations/*"]

event = json.load(sys.stdin)
path = event.get("tool_input", {}).get("file_path", "")
if event.get("tool_name") in {"Write", "Edit"} and any(fnmatch(path, p) or fnmatch(path, f"*/{p}") for p in PROTECTED):
    print(f"{path} is protected. Migrations: run `make migration name=<slug>`. "
          "Lockfiles: change the manifest and run the package manager.", file=sys.stderr)
    sys.exit(2)
sys.exit(0)
```

Note the block message. It says what to do instead, so the model's next attempt is likely to be right. A bare "denied" leads to the agent retrying slight variations.

### Hook or instruction?

| Put it in a hook when... | Put it in the instruction file when... |
|---|---|
| Breaking it is costly or irreversible | It's a preference or a default |
| It can be checked mechanically | It needs judgment to apply |
| The agent keeps breaking it despite instructions | The agent follows it reliably once told |
| It applies to every session, whatever the task | It only matters for some tasks (then maybe a skill) |

### Hooks have costs

Hooks run on every matching event, so a slow hook slows every action. Keep `PreToolUse` and `PostToolUse` hooks to milliseconds, and save expensive checks for `Stop`. Hooks are code that runs with your permissions: review them like any other code, and test them. Exercise-02 has you test hooks by piping events to them. A `Stop` hook that always blocks traps the agent in a loop; check the `stop_hook_active` flag and let it stop on the second try.

## Subagents: context isolation

A subagent is a nested agent run with its own fresh context window, its own system prompt, and often a restricted tool set. The parent hands it a task and gets back only a summary. Subagents help the harness in two ways:

- **They keep the main context clean.** A broad search through a large codebase might read 50 files. Done in a subagent, the main session gets back a few paragraphs, not 50 files of content.
- **They give an independent view.** A reviewer subagent that sees only the diff and the task, not the reasoning that produced it, is more likely to catch mistakes than the agent that wrote the code. This is a common inferential sensor.

In Claude Code, subagents are Markdown files under `.claude/agents/` with a description, a tool list, and a system prompt. The cost is latency and tokens, plus the loss of detail in the summary. Use them for work that is broad, independent, or benefits from fresh eyes.

## Key takeaways

- Tools are interfaces for a model: few and high-level, returning what's needed, with errors that say what to do next.
- Permission policies decide allow, deny, or ask. Allow the safe, frequent calls so prompts mean something.
- Pattern-based rules can be got around. The sandbox (filesystem, network, credentials) is the real boundary.
- Hooks run whatever the model decides. Use them for rules that must hold, make block messages explain the alternative, keep them fast, and test them.
- Subagents isolate context and provide independent review, at the cost of latency, tokens, and detail.
