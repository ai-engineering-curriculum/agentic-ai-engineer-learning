# Chapter 1 — Harness Architecture

[mod-208](../mod-208-harness-engineering/README.md) configured a harness someone else built. This module builds one: not a toy loop, but a harness with the same parts as Claude Code or Codex CLI. Coding tools, a sandbox, prompt assembly, context management, subagents, MCP, config files, tracing, and a headless CLI. This chapter maps the parts and the design rules that hold them together. The rest of the module builds them one at a time.

## When to build your own

Most teams should configure an existing harness, not build one. Build when one of these is true:

- **The agent lives inside your product.** A support agent, a data-analysis agent, or a code-review bot in your CI has users, tools, and permissions that a general coding CLI doesn't know about.
- **It runs unattended at scale.** Thousands of runs a day in a pipeline need tight control over cost, retries, isolation, and logging.
- **Your domain needs different tools or safety rules.** A harness for infrastructure changes, clinical documents, or financial data needs tools, approval flows, and audit trails that you have to own.
- **You need full control over the loop**, for research, evaluation, or because the behavior is the product.

Even then, look at the libraries first. The **Claude Agent SDK** exposes the Claude Code harness as a library: loop, tools, permissions, hooks, subagents, compaction, and MCP. Other agent SDKs package similar pieces. A common path is to start on an SDK and replace parts as your needs diverge. Building one from scratch in this module is how you learn what those parts do, so you can make that call, configure an SDK well, and debug it when it misbehaves.

## The parts

```text
┌──────────────────────────── harness ────────────────────────────┐
│                                                                  │
│  config ──▶ prompt assembly ──▶ ┌──────────── loop ───────────┐  │
│  (settings,  (base + env +      │ model layer (retry, stream, │  │
│   agents,     instructions +    │   cache, provider adapter)  │  │
│   hooks)      skills index)     │            │                │  │
│                                 │   tool dispatch:            │  │
│                                 │   validate → hooks →        │  │
│                                 │   permission → execute →    │  │
│                                 │   post-hooks                │  │
│                                 │            │                │  │
│                                 │   context manager (count,   │  │
│                                 │   clear, compact)           │  │
│                                 │            │                │  │
│                                 │   stop conditions           │  │
│                                 └─────────────────────────────┘  │
│                                                                  │
│  tools: files · search · bash ──▶ command runner ──▶ sandbox     │
│         load_skill · task (subagents) · mcp__* (MCP client)      │
│                                                                  │
│  session state: checkpoints · progress/feature files · memory    │
│  observability: event bus ──▶ JSONL log · traces · UI · replay   │
│  front ends: interactive CLI · headless JSON · library API       │
└──────────────────────────────────────────────────────────────────┘
```

| Part | Job | Chapter | Exercise |
|---|---|---|---|
| Loop | Call the model, dispatch tools, decide when to stop | 2 | 01 |
| Tools | Read, edit, search, run commands, built for a model | 3 | 02 |
| Command runner and sandbox | Run commands somewhere they can't do damage | 4 | 02 |
| Checkpoints | Undo any turn's file changes | 4 | 02 |
| Context manager | Keep the window under budget without losing the thread | 5 | 04 |
| Session state | Carry progress across sessions | 5 | 04 |
| Prompt assembly | Build the system prompt from the environment | 6 | 03 |
| Model layer | Retries, streaming, caching, provider differences | 6 | 03 |
| Subagents, skills, MCP | Extend the harness without editing its code | 7 | 05 |
| Config | Permissions, hooks, agents, servers as data | 7 | 05 |
| Observability | Events, logs, traces, replay tests | 8 | 06 |
| Front ends | Interactive CLI, headless mode, library API | 8 | 06 |

## Design rules

These rules recur in every chapter. Each one comes from a way harnesses commonly fail.

**1. The model's input is the product.** Everything the harness does ends up as tokens in the model's context: the system prompt, tool descriptions, tool results, error messages. Treat each as an interface with a user. A tool result that's clear and short is a better harness than one that's complete and long.

**2. Every failure is an observation.** Bad tool input, a denied permission, a crashed command, a timeout: each goes back to the model as a result it can read and act on. The loop only stops on purpose.

**3. The harness enforces; the prompt requests.** If a rule must hold, such as no writes outside the workspace, no edits to the feature list beyond `passes`, or no network, enforce it in code or at the OS level. The system prompt can explain the rule so the model doesn't waste turns hitting it.

**4. Defense in depth.** Path checks in the tool, permission rules in the gate, hooks for project rules, and a sandbox underneath. Any one layer can fail; the sandbox is the one you rely on.

**5. Everything is an event.** Model calls, tool calls, permission decisions, retries, compactions. Emit them all on one event bus. The CLI, the log file, the tracer, and your tests all subscribe to it. If you can't see it, you can't debug it.

**6. Pure core, pluggable edges.** The loop shouldn't know which model, which sandbox, which UI, or which storage it's using. Each is an interface with a fake for tests. That's what makes a harness testable without spending tokens, and what lets it run in a CLI, a CI job, and a web service with the same core.

**7. Deterministic tests for a non-deterministic system.** A scripted fake model drives most tests. Recorded real runs, replayed, catch regressions in what the harness sends. Live evals (mod-208, exercise-03) measure behavior. You need all three.

## Data model

A small set of immutable types carries through the whole harness. Getting them right early saves rework:

```python
@dataclass(frozen=True)
class ToolCall:
    id: str
    name: str
    input: Mapping[str, Any]

@dataclass(frozen=True)
class ToolResult:
    call_id: str
    content: str
    is_error: bool = False

@dataclass(frozen=True)
class ModelTurn:
    text: str
    tool_calls: tuple[ToolCall, ...]
    stop_reason: str
    usage: Usage

@dataclass(frozen=True)
class Event:
    run_id: str
    seq: int
    type: str
    data: Mapping[str, Any]
```

Keep provider-specific shapes, such as Anthropic content blocks or OpenAI `tool_calls`, at the edge in an adapter. The core speaks only these types. When you add a second provider (exercise-03's stretch goal), only a new adapter changes.

Keep two histories. **`messages`** is what the model sees next; context management trims and rewrites it. **`transcript`** is everything that happened, append-only, for logs, replay, and debugging. Mixing them is a common source of bugs: compaction quietly erases the record of what happened.

## What you'll build

By the end of this module your harness will be able to:

- Run a coding task with tools that behave like production ones, inside a container with no network, with every turn undoable.
- Assemble its system prompt from `AGENTS.md` files, skills, and the environment, with a cache-friendly layout.
- Survive rate limits and overloads, stream output live, and run independent reads in parallel.
- Work on tasks longer than one context window, and resume from disk.
- Delegate to subagents, use any MCP server's tools, and take its permissions and hooks from a config file.
- Log, trace, and replay every run, and run headless with JSON output for scripts and CI.

## Key takeaways

- Build your own harness when the agent lives in your product, runs unattended at scale, or needs domain tools and rules. Otherwise configure one or start from an SDK.
- A harness is a loop plus tools, a sandbox, prompt assembly, a model layer, context management, extensibility, observability, and front ends.
- Seven rules: the model's input is the product; failures are observations; enforce in code; defense in depth; everything is an event; pure core with pluggable edges; deterministic tests.
- Keep a small set of immutable core types, keep provider shapes at the edge, and keep `messages` separate from `transcript`.
