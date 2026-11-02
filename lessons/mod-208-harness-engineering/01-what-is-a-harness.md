# Chapter 1 — What a Harness Is

In earlier modules you wrote the reason-act loop: send messages to a model, run the tools it asks for, append the results, repeat. That loop is the smallest possible harness. Production coding agents like Claude Code and Codex CLI wrap the same loop in much more: a system prompt, a tool set, a permission system, hooks, subagents, context compaction, and a way to load project instructions. This chapter names those parts and gives you a way to think about improving them.

## Agent = model + harness

A useful definition: the harness is everything in an agent except the model. Two agents running the same model can behave very differently because of their harnesses. The same is true of one agent in two repositories. Codebase A has a clear `AGENTS.md`, one check command, and fast tests. Codebase B has none of these. The agent will do much better in A, and the model has nothing to do with it.

This matters because the harness is the part you control. You can't retrain the model this afternoon. You can add a line to an instruction file, a hook that blocks a mistake, or a linter rule whose error message tells the agent how to fix the problem.

It helps to split the harness in two:

- **Inner harness.** What the tool ships with: its system prompt, built-in tools, loop, compaction strategy, and default permissions. You mostly configure it rather than write it. [mod-209](../mod-209-building-custom-harnesses/README.md) has you build one so you know what's inside.
- **Outer harness.** What your team adds for its codebase: instruction files, skills, MCP servers, permission rules, hooks, check commands, test suites, and conventions. This is where most teams get most of their improvement, and it's what this module covers.

## The harness surfaces

| Surface | What it controls | Example |
|---|---|---|
| Instruction files | What the agent knows about the project at the start of a session | `AGENTS.md`, `CLAUDE.md`, nested per-directory files |
| Skills | Procedures loaded on demand when a task matches | `.claude/skills/add-migration/SKILL.md` |
| Tools and MCP servers | What the agent can do and see | A database MCP server, a `run_tests` tool |
| Permissions and sandbox | What it's allowed to do without asking, and what it can't do at all | Allow `pytest`; deny `git push --force`; no network |
| Hooks | Deterministic code that runs at lifecycle points | Block writes to `.env`; format after every edit; run checks before stopping |
| Subagents | Separate contexts for side tasks | A reviewer agent that sees only the diff |
| Verification | How the agent and you know the work is done | `make check`, end-to-end tests, screenshots |
| Session state | What carries over between context windows | A progress file, a JSON feature list, git history |

Each surface is a lever. Part of the skill is picking the right one for a given problem.

## Guides and sensors

Birgitta Böckeler's framing on martinfowler.com sorts harness controls by when they act:

- **Guides** act *before* the agent does something. They raise the odds of a good first attempt. Instruction files, skills, code templates, and type signatures are guides.
- **Sensors** act *after*. They observe the result and give the agent a signal it can correct from. Tests, linters, type checkers, and review agents are sensors.

You need both. Guides alone never check whether the agent followed them. Sensors alone let the agent make the same mistake every session and burn turns fixing it. A good harness turns a sensor that keeps firing into a guide. If the linter flags the same thing every day, add a line to the instruction file, or a template that avoids the problem in the first place.

A second axis is how a control decides:

- **Computational controls** are deterministic programs: tests, linters, type checkers, hooks, structural checks. They're fast, cheap, and give the same answer every time. Run them on every change.
- **Inferential controls** use a model: an LLM reviewer, a judge that checks the diff matches the intent. They catch things no rule can express, but they're slower, cost money, and can be wrong. Run them less often and later.

Prefer computational controls wherever a rule can be written down. Save inferential ones for questions of judgment.

## The steering loop

The core habit of harness engineering is simple: **treat every repeated agent mistake as a bug in the harness.** When the agent gets something wrong, don't just correct it in chat. Ask which surface would have prevented it, and change that surface.

| The agent... | Likely fix | Surface |
|---|---|---|
| Didn't know how to run the tests | Put the exact command in the instruction file | Guide |
| Ran the tests but missed a type error | Add the type checker to the check command | Sensor |
| Edited a generated file | Deny writes to that path; say why in the denial message | Permission or hook |
| Wrote code in the wrong layer | A structural test that fails with "core/ must not import from api/" | Sensor |
| Claimed "done" with failing tests | A stop hook that runs the check command | Hook |
| Lost track of the plan in a long task | A progress file it reads at the start of each session | Session state |
| Did a multi-step procedure differently each time | Write it as a skill | Guide |

Over time, this loop moves your attention from fixing the agent's output to improving the environment. OpenAI's February 2026 write-up of a team building a product with Codex describes the same pattern. Early progress was slow because the environment was underspecified, not because the model couldn't do the work, and the fixes were changes to the repository and tooling.

## Harnessability

Some codebases are easier to harness than others. Strong types, clear module boundaries, fast and reliable tests, one obvious way to build, and consistent conventions all give the agent more to go on and give you more to check against. A codebase that's hard for a new engineer to understand is usually hard for an agent too. Often the best harness change is a codebase change: split a 3,000-line file, make the tests run in ten seconds instead of ten minutes, or delete the second build system.

## Key takeaways

- The harness is everything except the model. It's the part you control, and it often matters as much as the model.
- The inner harness ships with the tool; the outer harness is what your team builds around its codebase.
- Guides steer before the agent acts; sensors give feedback after. You need both, and a sensor that keeps firing should become a guide.
- Prefer computational controls (fast, deterministic) and use inferential ones (model-based) for judgment calls.
- Treat every repeated mistake as a harness bug: find the surface that would have prevented it and change that.
