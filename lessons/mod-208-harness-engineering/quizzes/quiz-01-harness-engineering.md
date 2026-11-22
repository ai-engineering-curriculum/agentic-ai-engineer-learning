# Quiz 1 — Harness Engineering

Knowledge check for [mod-208](../README.md). Answers are at the bottom; try each question before scrolling. Covers all four chapters.

## Questions

### 1. Inner vs. outer harness

A teammate says "we can't improve how our agent behaves in this repo — that would mean retraining the model or rewriting Claude Code." Which part of Chapter 1's framing most directly rebuts this?

- A. The inner harness (system prompt, built-in tools, default permissions) is the main lever; your team edits it to fit the project.
- B. The outer harness — instruction files, skills, MCP servers, permission rules, hooks, check commands, conventions — is what your team adds for *its* codebase, and is where most teams get most of their improvement.
- C. "Agent = model + harness" means improvements require a new model release; there is nothing the team can do until then.
- D. The system prompt is the only surface worth tuning; everything else is cosmetic.

### 2. Guides vs. sensors

Your linter flags the same "handlers must return `Result[T, ApiError]`" violation in almost every session, and the agent fixes it after the linter fires. Chapter 1 argues this is working, but still suboptimal. What's the recommended next move?

- A. Leave it alone — the sensor is doing its job, the agent is correcting itself.
- B. Remove the linter rule; a sensor that keeps firing is wasteful.
- C. Turn the recurring sensor into a guide: add a line to the instruction file (or a template / example file) that prevents the mistake on the first attempt.
- D. Replace the linter with an inferential LLM reviewer so the feedback is more nuanced.

### 3. Computational vs. inferential controls

Chapter 1 contrasts computational and inferential controls. Which assignment matches the chapter's advice?

- A. Run an LLM reviewer on every file edit; run unit tests only before release.
- B. Use the LLM reviewer for layer-boundary rules ("`core/` must not import from `api/`") and leave judgment calls to a unit test.
- C. Run deterministic computational controls (tests, linters, type checkers, hooks, structural checks) on every change; save inferential controls for questions of judgment, and run them less often.
- D. Replace all computational controls with inferential ones so the harness stays flexible.

### 4. The steering loop

Across three sessions the agent has edited the generated `openapi_client/` directory and broken the build. Chapter 1's steering loop says to fix the harness, not the transcript. Which surface is the best primary fix?

- A. Add a line to `AGENTS.md` asking the agent "please be careful around generated files".
- B. Deny writes to `openapi_client/` via permissions or a `PreToolUse` hook, with a block message that points to the regeneration command.
- C. Open a chat with the agent and correct it each time it happens.
- D. Switch to a different model; this is a model capability problem.

### 5. Instruction file as a map

You're reviewing a 600-line `AGENTS.md` that copies the architecture doc, lists every lint rule, and restates the testing framework's docs. By Chapter 2, what is wrong with this file *as an instruction file*?

- A. Nothing — more context is always better for the agent.
- B. Instruction files are loaded every session and use context every turn; long, general files cost tokens and get followed less reliably than short, specific ones. Make it a map that points to the deeper docs instead of copying them.
- C. `AGENTS.md` is deprecated; only `CLAUDE.md` is read by current tools.
- D. The problem is the format — rewrite it as JSON so the agent parses it more reliably.

### 6. Specific beats general

Chapter 2 contrasts weak and strong instructions. Which of these is a **strong** instruction by the chapter's criteria?

- A. "Write clean, well-tested code."
- B. "Follow our API conventions."
- C. "Be careful with the database."
- D. "Handlers return `Result[T, ApiError]`; never raise from a handler. See `api/users.py` for the pattern."

### 7. Skill descriptions

You're writing a Claude Code skill for the multi-step "add a database migration" procedure. Chapter 2 says the frontmatter `description` stays in context all the time, but the body loads only when the skill matches. What should the `description` be optimized for?

- A. A faithful title of the procedure, so humans reading `SKILL.md` understand what it is.
- B. A trigger that tells the agent *when* to load the skill (e.g., "Use when adding a new database migration; covers naming, up/down, and tests").
- C. A full summary of every step, so the agent can act without loading the body.
- D. A list of maintainers and version numbers.

### 8. Hook or instruction?

Chapter 3 gives a decision table for whether a rule belongs in a hook or in the instruction file. Which rule most clearly belongs in a **hook**?

- A. "Prefer dependency injection in the service layer" — a stylistic preference.
- B. "Writes to `.env*`, `.git/`, and lockfiles must be blocked, with a message telling the agent how to make the change properly instead" — mechanical, irreversible if it slips through, and the agent has kept breaking it.
- C. "When refactoring, consider whether a smaller change would do" — a judgment call.
- D. "Favor table-driven tests for the parsers in `parsers/`" — a convention that only matters in one area.

### 9. Permissions are not a security boundary

You denied `rm -rf` and `curl ... | sh` in `.claude/settings.json`. Chapter 3 calls this a helpful guardrail but not a security boundary. Why?

- A. Permission rules match patterns on tool calls, which can be bypassed — put the command in a script, use `python -c`, or build the string from pieces. The real boundary is the sandbox: what the process can reach at the OS level (filesystem scope, network, credentials).
- B. Permissions are only advisory in Claude Code; the agent can override them at will.
- C. `rm -rf` is actually safe; the warning in the chapter is historical.
- D. Pattern rules are the security boundary; sandboxing is only a performance optimization.

### 10. Block messages

A `PreToolUse` hook correctly blocks an edit to `migrations/0007_add_index.sql`. Chapter 3 is specific about what the stderr message should contain. Which message follows the chapter's advice?

- A. `denied`.
- B. `error: protected path`.
- C. `migrations/0007_add_index.sql is protected. To add a migration, run \`make migration name=<slug>\`. To edit an existing migration, revert the commit and create a new one.`
- D. `Policy violation: PATH_RULE_42 triggered. See compliance docs.`

### 11. A fast check command is the headline sensor

Chapter 4 opens with the claim that the single most valuable thing in a coding harness is a fast, reliable way for the agent to check its own work. Which of the following is **inconsistent** with that chapter's advice?

- A. Point the agent at one entry point (`make check`, `just check`, `scripts/check.sh`) that runs format, lint, type check, and tests, and exits non-zero on failure.
- B. Name that command in the instruction file as the definition of done.
- C. Keep flaky tests around; they're better than no tests at all.
- D. For user-facing work, give the agent an end-to-end way to check like a user (browser automation, screenshots, a request against a running server).

### 12. The JSON feature list

Chapter 4 describes the long-running session pattern from Anthropic's write-up. Why is the feature list specifically a **JSON** file and not a Markdown checklist?

- A. JSON is faster to parse, saving tokens on every read.
- B. Markdown doesn't support boolean values.
- C. The write-up reports that models were less likely to rewrite or delete entries in a JSON file than in a Markdown checklist; a hook can further enforce that agents may only flip each feature's `passes` field.
- D. JSON is required by Claude Code; Codex supports both.

### 13. Measuring a harness change

You changed `AGENTS.md` and want to know if it helped. Ten sessions on one task show 7 passes with the new file vs. 5 with the old. By Chapter 4's rules, which conclusion is *best supported*?

- A. The new harness is clearly better; a 40% relative improvement is large.
- B. Agent runs are noisy; one task with a small sample may not clear the noise floor. Report the pass-rate difference with an uncertainty interval (e.g., a bootstrap 95% CI) and if it includes zero, say "no clear difference" — and run several tasks, not one.
- C. A single trial per configuration is enough as long as the task is realistic.
- D. The right comparison is average turns, not pass rate; whichever harness used fewer turns wins.

### 14. Reward hacking

Chapter 4 warns about reward hacking in harness evals. Which pair of defenses does the chapter explicitly recommend?

- A. Delete the check command so the agent can't game it; rely on model self-report.
- B. Trust pass rate alone; it is objective by construction.
- C. Protect or hold out tests from the agent (deny writes, or keep them outside the workspace) *and* review a sample of diffs by hand (optionally with an inferential sensor that compares the diff to the task intent).
- D. Increase the timeout; most reward hacking is caused by the agent running out of time.

## Answer key

1. **B** — The outer harness (instruction files, skills, MCP servers, permissions, hooks, checks, conventions) is what your team adds around the inner harness, and is where most teams get most of their improvement ([Chapter 1 §Agent = model + harness](../01-what-is-a-harness.md)).
2. **C** — A sensor that keeps firing on the same thing should become a guide; add an instruction, template, or type signature that prevents the mistake on the first attempt ([Chapter 1 §Guides and sensors](../01-what-is-a-harness.md)).
3. **C** — Prefer computational (deterministic, fast, cheap, same answer every time) wherever a rule can be written down; save inferential controls for judgment and run them less often ([Chapter 1 §Guides and sensors](../01-what-is-a-harness.md)).
4. **B** — The steering-loop table maps "edited a generated file" to a permission-or-hook denial with a message that explains the alternative; chat corrections don't fix the next session ([Chapter 1 §The steering loop](../01-what-is-a-harness.md)).
5. **B** — Instruction files load every session; keep them short — exact commands, directory map, conventions the agent gets wrong, pointers to more. A 600-line file is an encyclopedia, not a map ([Chapter 2 §The instruction file is a map](../02-guides-context-and-skills.md)).
6. **D** — The strong column names files, commands, and checkable rules and points to an example in the repo; the other three are vague advice the agent can't verify against ([Chapter 2 §Specific beats general](../02-guides-context-and-skills.md)).
7. **B** — The `description` is the trigger that decides when the skill loads, so write it as "Use when...", not as a title; the body of the skill holds the steps ([Chapter 2 §Skills](../02-guides-context-and-skills.md)).
8. **B** — Hooks belong where breaking the rule is costly or irreversible, the rule is mechanically checkable, and the agent has kept breaking it despite instructions. Protected-path writes satisfy all three ([Chapter 3 §Hook or instruction?](../03-tools-permissions-hooks.md)).
9. **A** — Pattern rules can be bypassed by rephrasing or wrapping the command; the sandbox (filesystem scope, network allowlist, limited credentials) is the real boundary ([Chapter 3 §Sandboxing](../03-tools-permissions-hooks.md)).
10. **C** — Block messages should tell the model what to do instead, so the next attempt is likely to be right; bare "denied" leads to the agent retrying slight variations ([Chapter 3 §Hooks](../03-tools-permissions-hooks.md)).
11. **C** — Flaky tests teach the agent that failures can be ignored; the chapter says to fix or quarantine them. The other three are the chapter's explicit recommendations ([Chapter 4 §Give the agent a way to check its work](../04-sensors-feedback-and-evals.md)).
12. **C** — Chapter 4 cites the write-up's finding that models were less likely to mangle a JSON file than a Markdown checklist, and notes a hook can enforce that only the `passes` field flips ([Chapter 4 §Long-running work across sessions](../04-sensors-feedback-and-evals.md)).
13. **B** — Agent runs are noisy; evaluate with a fixed task set, repeated trials, several metrics, and an honest interval. "7 vs. 5 on one task" plausibly doesn't clear the noise floor ([Chapter 4 §Evaluate the harness, not just the agent](../04-sensors-feedback-and-evals.md)).
14. **C** — The named defenses are tests the agent can't edit (deny writes or keep held-out tests outside the workspace), by-hand review of a sample of diffs, and an inferential sensor that compares the diff to the task's intent ([Chapter 4 §Evaluate the harness](../04-sensors-feedback-and-evals.md)).
