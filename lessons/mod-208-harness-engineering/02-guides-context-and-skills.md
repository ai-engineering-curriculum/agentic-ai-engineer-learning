# Chapter 2 — Guides: Context, Instructions, and Skills

Guides shape what the agent does before it does it. The main guide is the project instruction file, loaded at the start of every session. Skills come next: procedures the agent loads only when a task calls for them. This chapter covers how to write both so they help, rather than filling the context window with text the agent ignores.

## The instruction file is a map, not an encyclopedia

Every coding agent reads a project instruction file at startup. `AGENTS.md` has become the shared convention across tools: Codex, Cursor, Copilot, Gemini CLI, and others read it. Claude Code reads `CLAUDE.md`, which can import `AGENTS.md` with a single `@AGENTS.md` line, so one file serves every tool.

The common mistake is to treat this file as the place for all project knowledge. It gets loaded into every session and uses context every turn, and models follow a short list of specific rules better than a long list of general ones. A 600-line instruction file costs tokens and gets followed less reliably.

Write it as a map. It should tell the agent where things are and how to check its work, and point to deeper documents it can read when needed:

- **What the project is**, in two sentences.
- **Exact commands**: setup, the one check command, how to run a single test. Copy them so they can be pasted. "Run the tests" is not a command.
- **A directory map**: the five to ten places that matter and what lives in each.
- **The few conventions the agent actually gets wrong.** Not every style rule; the linter handles those.
- **Pointers**: "Architecture: `docs/architecture.md`", "Adding an endpoint: use the `add-endpoint` skill".

A test for every line: *would the agent get this wrong without it?* If not, delete it. Revisit the file whenever the agent keeps making a mistake (add a line) or the project changes (fix stale lines). Stale instructions are worse than none, because the agent trusts them.

### Specific beats general

| Weak | Strong |
|---|---|
| Write clean, well-tested code. | Every new function in `core/` needs a test in `tests/core/`. Run `make check` before saying you're done. |
| Follow our API conventions. | Handlers return `Result[T, ApiError]`; never raise from a handler. See `api/users.py` for the pattern. |
| Be careful with the database. | Never edit files in `migrations/` by hand. Create new ones with `make migration name=<slug>`. |

The strong versions name files, commands, and checkable rules. Pointing to an example file in the repo is often the most effective instruction there is. The agent can read the real pattern instead of guessing from a description.

### Nesting and scope

Most tools support instruction files in subdirectories that apply only when the agent works there. Use them for rules that only matter in one area (say, the frontend's component conventions) so they don't use context in every session. Claude Code also supports path-scoped rules files and a user-level file for personal preferences that shouldn't be in the repo.

## Progressive disclosure

The idea behind the map approach is **progressive disclosure**: give the agent a small amount of always-loaded context that tells it where to find more, and let it pull details when the task needs them. This is how a good engineer onboards someone: here's the layout, here's how to build, here's where the design doc lives. Not a two-day lecture before they touch the code.

The repository itself is part of this. Docs in the repo, kept current and linked from the instruction file, are part of the harness. Knowledge that lives only in a wiki, a chat thread, or someone's head doesn't exist for the agent. If a decision matters, write it into the repo where the agent can find it: an architecture decision record, a `docs/` page, a comment at the place where the rule applies.

## Skills

A **skill** is a packaged procedure the agent loads on demand. In Claude Code, a skill is a directory with a `SKILL.md` file. The frontmatter `description` stays in context all the time. The body loads only when the agent decides the skill applies, so you can have dozens of skills without paying for them in every session. Codex and other tools have similar mechanisms, and where they don't, a prompt file referenced from `AGENTS.md` does the same job.

Good candidates for skills:

- Multi-step procedures the agent does inconsistently: adding a migration, creating a new service, cutting a release.
- Tasks that need a specific checklist: a security review of a diff, updating the changelog.
- Domain knowledge needed only sometimes: how the billing system computes proration.

```markdown
---
name: add-cli-subcommand
description: Use when adding a new subcommand to the CLI. Covers registration, argument parsing, tests, and help text.
---

1. Create `src/tool/commands/<name>.py` with a `register(subparsers)` function. Copy the shape of `commands/export.py`.
2. Add the import and `register` call in `src/tool/cli.py`, keeping the list alphabetical.
3. Add `tests/commands/test_<name>.py` covering: happy path, a missing required argument, `--help` output.
4. Run `make check`. The subcommand must appear in `tool --help`.
```

The `description` decides when the skill loads, so write it as a trigger ("Use when..."), not a title. Keep the body as steps plus a verification command. Background explanation belongs in a linked doc the agent can read if it needs to.

## Making the codebase easy to read

Guides aren't only text written for the agent. The codebase is the largest guide there is, and the agent reads it constantly. Things that help:

- **Consistent patterns.** If there are three ways to write a handler, the agent will pick one at random. Pick one and migrate the others.
- **Names that say what things do**, and files small enough to read whole.
- **Types.** A function signature is an instruction the type checker enforces.
- **Examples in the repo.** One good reference implementation is worth a page of rules.

## Key takeaways

- The instruction file is loaded every session. Keep it short: exact commands, a directory map, the conventions the agent gets wrong, and pointers to more.
- Specific, checkable rules that name files and commands work much better than general advice.
- Use progressive disclosure: a small always-loaded map, with details in docs, nested files, and skills the agent pulls in when needed.
- Skills hold procedures the agent loads only when a task matches. Write the description as a trigger.
- Knowledge that isn't in the repo doesn't exist for the agent. The codebase itself is the biggest guide.
