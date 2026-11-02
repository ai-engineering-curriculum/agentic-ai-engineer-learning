# exercise-01: Harness a Repository

**Estimated effort:** 3 hours

## Objective

Take a small, real repository and give a coding agent (Claude Code or Codex CLI) a harness for it: an instruction file that works as a map, one skill, a permission policy, and a single verification command the agent can run. Then run the same three tasks before and after, and record what changed. By the end you'll have a harness checked into the repo and evidence of whether it helped.

## Background

This exercise covers material from:

- [Chapter 1 — What a Harness Is](../01-what-is-a-harness.md)
- [Chapter 2 — Guides: Context, Instructions, and Skills](../02-guides-context-and-skills.md)

Pick a repository you know: a personal project, a lab from an earlier module, or a small open-source project. It should have tests (or you should be willing to add a few) and be under about 10k lines, so the agent can explore it in one session.

## Prerequisites

- Claude Code or Codex CLI installed and signed in. Any agent with a project instruction file, skills or equivalent, and a permission policy works; the tasks name both tools.
- A repository with a working test command.
- A small spend cap or plan limit. You'll run six agent sessions.

## Tasks

### 1. Baseline (no harness)

- Move any existing `AGENTS.md`, `CLAUDE.md`, `.claude/`, or `.codex/` out of the way on a scratch branch.
- Write three tasks in `TASKS.md`. Make them realistic and checkable: one bug fix, one small feature, one refactor. Each task needs a concrete done condition ("`pytest tests/test_parser.py` passes and the new flag appears in `--help`").
- Run each task in a fresh session. For each, record in `RESULTS.md`: did it finish, did the done condition pass, number of turns or tool calls, rough cost or tokens, and every place you had to step in.

### 2. Write the instruction file as a map

- Write `AGENTS.md` at the repo root (and a one-line `CLAUDE.md` that imports it with `@AGENTS.md` if you use Claude Code).
- Keep it under 80 lines. It should cover: what the project is (two sentences), how to set up, build, and test (exact commands), the directory map, the three or four conventions the agent got wrong in the baseline, and pointers to deeper docs rather than their contents.
- Every line must earn its place. For each line, ask "would the agent get this wrong without it?" If not, delete it.

### 3. Add one skill

- Write a skill for a repeated, multi-step procedure in this repo, such as "add a new CLI subcommand" or "add a database migration".
- Claude Code: `.claude/skills/<name>/SKILL.md` with a `description` in the frontmatter that says when to use it. Codex: the equivalent skill or a prompt file referenced from `AGENTS.md`.
- The body should be steps plus the verification command, not background.

### 4. Set a permission policy

- Allow the read-only and verification commands the agent needs (test runner, linter, `git status`, `git diff`) so it doesn't stop to ask.
- Deny what it should never do here: force-push, `rm -rf` outside the build directory, reading `.env` or secrets files.
- Claude Code: `.claude/settings.json` `permissions.allow` / `permissions.deny`. Codex: sandbox mode and approval policy in `config.toml`.

### 5. One verification command

- Add a single entry point (a `make check`, `just check`, or `scripts/check.sh`) that runs format check, lint, type check (if any), and tests, and exits non-zero on failure.
- Reference it from `AGENTS.md` as the definition of done.

### 6. Re-run and compare

- Run the same three tasks in fresh sessions with the harness in place. Record the same fields in `RESULTS.md`.

## Starter guidance

A skeleton for the instruction file:

```markdown
# <project name>

<Two sentences: what it is and who uses it.>

## Commands
- Setup: `uv sync`
- Check (run before you say you're done): `make check`
- Single test: `uv run pytest tests/test_x.py -k name`

## Map
- `src/<pkg>/cli.py` — argument parsing; subcommands register here
- `src/<pkg>/core/` — pure logic, no I/O
- `tests/` — mirrors `src/`

## Conventions
- <Rule the agent broke in the baseline, stated as an instruction.>

## More
- Architecture: `docs/architecture.md`
- Adding a subcommand: use the `add-subcommand` skill
```

## Acceptance criteria

You can demonstrate that:

- `AGENTS.md` is under 80 lines, contains the exact build and test commands, and points to deeper docs instead of copying them.
- The skill loads when the agent hits the task it's for (show the transcript line or the skill being invoked).
- The permission policy runs the check command without prompting and blocks at least one denied action (show the denial).
- `make check` (or equivalent) exits non-zero on a deliberately broken change and zero on a clean tree.
- `RESULTS.md` has before-and-after rows for all three tasks with the same fields.

## Reflection

In `NOTES.md`:

1. Which harness change made the biggest difference, and how do you know it was that change and not run-to-run noise?
2. Name one line you cut from `AGENTS.md` and why it wasn't earning its place.
3. Three tasks run once each is a tiny sample. What would you need to claim the harness "helped" with any confidence? (Preview of exercise-03.)

## Stretch goals

- Add a nested `AGENTS.md` in one subdirectory with rules that only apply there, and confirm the agent picks it up when working in that directory.
- Make the harness tool-agnostic: get the same `AGENTS.md` working in both Claude Code and Codex.
