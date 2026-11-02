# mod-208-harness-engineering: Harness Engineering

**Estimated effort:** 10 hours

An agent is a model plus a harness. The harness is everything that isn't the model: the loop that calls it, the tools it can use, the instructions it reads, the permissions that limit it, the hooks that check its work, and the files that carry state between sessions. When a coding agent fails, the cause is usually in the harness. The environment didn't tell it how to run the tests. Nothing stopped it from editing the lockfile. No check caught the broken build before it said "done". Harness engineering means fixing those causes in the environment, so the same mistake doesn't happen again.

This module works from the user's side: you configure an existing harness (Claude Code or Codex CLI) for a real repository, with instruction files, skills, tools, permissions, and hooks, and you build an eval that tells you whether a change helped. Its companion, [mod-209: Building a Custom Agent Harness](../mod-209-building-custom-harnesses/README.md), goes inside: you build a full harness from scratch, with the loop, coding tools, sandboxing, context management, subagents, MCP, and tracing.

> **Fix the environment, not the transcript.** When an agent makes a mistake, correcting it in chat fixes one session. Changing the harness fixes every session after it. The question to ask is "what would have made this mistake impossible, or at least obvious?"

## Learning objectives

- **Explain the harness model**: the inner harness a tool ships with vs. the outer harness a team builds, guides (feedforward) vs. sensors (feedback), and computational vs. inferential controls.
- **Write instruction files and skills that work as a map**: short, specific, and pointing to deeper docs instead of copying them.
- **Enforce rules deterministically** with tool permissions, sandboxing, and lifecycle hooks, and know which rules belong in a hook and which belong in prose.
- **Evaluate harness changes** with a fixed task set, repeated trials, and pass rate, cost, and turns, so you can tell a real improvement from noise.
- **Structure long-running work** across sessions with an initializer, a JSON feature list, a progress file, and git checkpoints.

## Lecture chapters

1. [What a Harness Is](01-what-is-a-harness.md) — model vs. harness, the harness surfaces, guides and sensors, and the steering loop.
2. [Guides: Context, Instructions, and Skills](02-guides-context-and-skills.md) — instruction files as a map, progressive disclosure, skills, and making a codebase easy for an agent to read.
3. [Tools, Permissions, and Hooks](03-tools-permissions-hooks.md) — designing tools for agents, permission policies and sandboxes, hooks as deterministic enforcement, and subagents for context isolation.
4. [Sensors: Feedback Loops and Harness Evals](04-sensors-feedback-and-evals.md) — verification commands, error messages written for a model, long-running session patterns, and measuring whether a harness change helped.

## Exercises

Hands-on practice. Reference solutions live in the paired [solutions repo](https://github.com/ai-engineering-curriculum/agentic-ai-engineer-solutions).

- [exercise-01: Harness a repository](exercises/exercise-01-harness-a-repository.md) — write an instruction file, a skill, a permission policy, and a check command, then compare three tasks before and after.
- [exercise-02: Deterministic guardrails with hooks](exercises/exercise-02-guardrail-hooks.md) — a pre-tool guard, a post-edit formatter, and a stop gate, each with offline tests.
- [exercise-03: Evaluate a harness change](exercises/exercise-03-harness-eval.md) — an eval runner that compares two harness configurations across repeated trials.

## Prerequisites

- [mod-201: Agent Fundamentals](../mod-201-agent-fundamentals/README.md) — the reason-act loop and tool calling.
- [mod-205: Evaluation & Observability](../mod-205-evaluation-observability/README.md) — exercise-03 applies the same eval ideas to the harness.
- [mod-206: Guardrails & Safety](../mod-206-guardrails-implementation/README.md) — least-privilege tools and sandboxing, which Chapter 3 applies to coding agents.
- Claude Code or Codex CLI installed, and a repository you know well.

Next: [mod-209: Building a Custom Agent Harness](../mod-209-building-custom-harnesses/README.md). For a shorter, tool-by-tool reference, see the guidebook's [Harness Engineering](https://github.com/ai-infra-curriculum/ai-agent-guidebook/blob/main/best-practices/harness-engineering.md) page. See [resources.md](resources.md) for primary references.
