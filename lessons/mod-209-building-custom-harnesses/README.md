# mod-209-building-custom-harnesses: Building a Custom Agent Harness

**Estimated effort:** 27 hours

In [mod-208](../mod-208-harness-engineering/README.md) you configured a harness someone else built. In this module you build one, with the same parts as Claude Code or Codex CLI. You start from the loop and add a coding tool suite, an OS-level sandbox with undoable checkpoints, context management, prompt assembly, a resilient model layer, subagents, MCP, declarative config, tracing, replay tests, a headless CLI, and an external trust layer. Each exercise adds one layer to the same codebase. By the end you'll have a harness you could ship, and you'll understand the harnesses you use every day well enough to configure, debug, and extend them.

> **The harness enforces; the prompt requests.** Every chapter comes back to this. A rule the model must follow belongs in code or at the OS level. The prompt explains the rule so the model doesn't waste turns running into it.

## Learning objectives

- **Decide when to build** a custom harness vs. configuring one or starting from an SDK, and lay out its architecture: core types, event bus, and pluggable model, sandbox, and front end.
- **Build the agent loop** with schema validation, errors as observations, hooks, a permission gate, and stop conditions.
- **Design coding tools for a model**: line-numbered paged reads, exact-match edits with read-before-edit, `glob` and `grep`, and a command tool with a scrubbed environment.
- **Isolate execution** with a threat model, a pluggable command runner, containers with no network, and git checkpoints that never touch the user's branch or index.
- **Manage context and sessions**: token accounting, tool-result clearing, compaction, and on-disk state for resuming.
- **Assemble the system prompt** from instruction files, skills, and the environment, laid out for prompt caching; and **harden the model layer** with retries, streaming, and parallel read-only tool calls.
- **Make the harness extensible** with subagents, MCP servers, and validated layered config.
- **Operate and ship it**: event logs with redaction, OpenTelemetry spans, record-and-replay tests, and a headless CLI with JSON output and meaningful exit codes.
- **Add a trust layer**: route side-effecting tool decisions through an external policy service, keep PII out of the model's context, scan untrusted output, and stream behavioral events for scoring and audit, with deliberate fail-open and fail-closed choices.

## Lecture chapters

1. [Harness Architecture](01-harness-architecture.md) — when to build, the parts and how they connect, seven design rules, and the core data model.
2. [The Agent Loop](02-the-agent-loop.md) — the loop, tool dispatch, errors as observations, path containment, permissions and hooks as middleware, and stop conditions.
3. [Tools for a Coding Agent](03-coding-tools.md) — descriptions as prompts, reading, exact-match editing, read-before-edit, search, commands, and output budgets.
4. [Execution Isolation and Checkpoints](04-execution-isolation.md) — threat model, sandbox layers, containers, egress and credentials, the runner interface, git checkpoints, and worktrees.
5. [Context, Compaction, and Session State](05-context-compaction-state.md) — token budgets, clearing old tool results, compaction, and state files that let a fresh session resume.
6. [Prompt Assembly and the Model Layer](06-prompt-assembly-and-model-layer.md) — building the system prompt, instruction discovery, skills index, prompt caching, retries, streaming, and parallel tools.
7. [Subagents, Skills, and MCP](07-subagents-skills-mcp.md) — delegation with isolation, agent definitions, MCP client integration and its risks, and declarative config.
8. [Observability, Testing, and Shipping](08-observability-testing-shipping.md) — events, run logs, tracing, three levels of testing, front ends, headless defaults, exit codes, cost, and packaging.
9. [Adding a Trust Layer: VeriSwarm](09-trust-layer-veriswarm.md) — what a harness can't do alone, Gate decisions as a permission source, fail modes, events for scoring and audit, PII tokenization and injection scanning, identity, and the vendor-neutral pattern.

## Exercises

Every exercise builds on [exercise-01](exercises/exercise-01-build-a-minimal-harness.md). Exercises 02–07 each extend exercise-01 on their own, so you can do them in any order after it. Reference solutions live in the paired [solutions repo](https://github.com/ai-engineering-curriculum/agentic-ai-engineer-solutions).

- [exercise-01: Build a minimal harness](exercises/exercise-01-build-a-minimal-harness.md) — the loop, tool registry, permission gate, hooks, and stop conditions from scratch. *(4h)*
- [exercise-02: A coding tool suite with isolated execution](exercises/exercise-02-coding-tool-suite.md) — paged reads, exact-match edits, search, a Docker runner, and git checkpoints. *(4h)*
- [exercise-03: Prompt assembly and a resilient model layer](exercises/exercise-03-prompt-assembly-and-model-layer.md) — instruction discovery, skills, cache breakpoints, retries, streaming events, and parallel reads. *(3h)*
- [exercise-04: Context management and session state](exercises/exercise-04-context-and-session-state.md) — clearing, compaction, and a guarded feature list and progress file for multi-session work. *(3h)*
- [exercise-05: Subagents, MCP tools, and declarative config](exercises/exercise-05-subagents-mcp-and-config.md) — a `task` tool, an MCP client adapter, and settings-driven permissions and hooks. *(4h)*
- [exercise-06: Observability, replay testing, and shipping](exercises/exercise-06-observability-replay-and-shipping.md) — JSONL run logs, OpenTelemetry spans, record and replay, and a headless CLI. *(3h)*
- [exercise-07: Add a trust layer with VeriSwarm](exercises/exercise-07-trust-layer-veriswarm.md) — Gate as a permission policy, result and input filter stages, PII tokenization and rehydration, injection scanning, and batched behavioral events. *(3h)*

## Prerequisites

- [mod-201: Agent Fundamentals](../mod-201-agent-fundamentals/README.md) — the reason-act loop and tool calling.
- [mod-208: Harness Engineering](../mod-208-harness-engineering/README.md) — the harness surfaces from the user's side, the hook contract (exercise-02), and the eval runner (exercise-03).
- [mod-205](../mod-205-evaluation-observability/README.md) and [mod-206](../mod-206-guardrails-implementation/README.md) — tracing and sandboxing concepts that chapters 4 and 8 build on.
- Solid Python: typing, dataclasses, `subprocess`, threads or `asyncio`. Docker for exercise-02's container runner.

See [resources.md](resources.md) for primary references.
