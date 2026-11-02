# Chapter 8 — Observability, Testing, and Shipping

A harness that works on your machine during development is a prototype. A harness you can ship has three more things: you can see what every run did; you can change the harness and know you haven't broken it; and other people and programs can run it without reading its source. This chapter covers each.

## Events are the foundation

Chapter 1's rule "everything is an event" pays off here. If the loop emits a structured event for every step, then logging, tracing, a live UI, metrics, and replay are all just subscribers:

```text
           ┌──────────▶ terminal UI (streams text, shows tool calls)
loop ─▶ event bus ─────▶ JSONL run log (one file per run)
           ├──────────▶ tracer (OpenTelemetry spans)
           ├──────────▶ metrics (tokens, cost, latency, stop reasons)
           └──────────▶ test collector (assert on event sequences)
```

A useful event set:

| Event | Key fields |
|---|---|
| `run_started` | run id, parent run id (for subagents), model, workspace, config summary |
| `model_request` | turn, message count, estimated input tokens |
| `model_response` | stop reason, tool calls, usage (input, output, cache read, cache write), latency |
| `tool_call` | name, input, permission decision and which rule decided, hook outcomes |
| `tool_result` | is_error, output size, duration |
| `context_managed` | cleared results, compaction happened, tokens before and after |
| `retry` | attempt, delay, reason |
| `run_finished` | stop reason, turns, total tokens, cost |

Give every event a run id and a sequence number, and give subagent runs a parent id, so a nested run can be reconstructed as a tree.

## Run logs

Write each run's events to a JSONL file, one object per line, flushed as each event happens. JSONL is easy to append to, survives a crash with everything up to the crash intact, and is easy to process with `jq` or a few lines of Python.

**Redact before writing.** Run logs end up in bug reports, shared drives, and support tickets. Strip values for keys like `api_key`, `authorization`, and `token`, and mask strings that match common credential formats. Test redaction with a planted fake secret. This is easy to get wrong silently.

Decide on retention. Logs of a coding agent contain source code and possibly customer data. Keep them as long as you need them for debugging, and no longer.

## Tracing

Logs show one run in detail. Traces put runs into your existing observability stack, where you can see latency, errors, and cost across many runs, and connect an agent run to the request or job that started it.

Map the harness onto spans:

```text
run (agent.run)                                 12.4s
├── chat claude-…                                2.1s   in=8,210 out=312
├── tool read_file                               0.01s
├── tool grep                                    0.04s
├── chat claude-…                                1.8s   in=9,904 out=540
├── tool task → subagent run                     6.2s
│   ├── chat …
│   └── tool …
└── chat claude-…                                1.1s
```

The OpenTelemetry **GenAI semantic conventions** define standard attribute names for model calls, such as `gen_ai.operation.name`, `gen_ai.request.model`, `gen_ai.usage.input_tokens`, and `gen_ai.usage.output_tokens`, so tools like Langfuse, Phoenix, and vendor APMs can understand your spans. The conventions are still evolving; check the current spec. You used them in [mod-205](../mod-205-evaluation-observability/README.md).

Two defaults to get right: **no prompt or tool content on spans** unless explicitly enabled, since traces often go to shared systems with broader access than run logs; and **tracing is optional**: the harness must run without the tracing library installed.

## Testing a harness

A harness has three kinds of behavior, and each needs its own tests:

| What | How | Speed | Catches |
|---|---|---|---|
| Harness logic | Unit tests with a scripted fake model | Milliseconds | Bugs in dispatch, permissions, hooks, context management, stop conditions |
| What the harness sends | Replay tests from recorded runs | Seconds | Unintended changes to prompts, tool schemas, message shapes |
| Agent behavior | Live evals with repeated trials | Minutes, costs money | Whether the agent actually does tasks well |

### Scripted models

You've used these since exercise-01: a fake `ModelClient` that returns a fixed sequence of turns. They make the harness's own logic fully testable. Every permission path, every stop condition, every compaction threshold can be tested without a network call. Most of a harness's tests should be these.

### Record and replay

A scripted model tests the scenarios you thought of. **Replay** tests the ones that actually happened. Wrap the real model client in a recorder that saves each request and response to a cassette file. Later, a replay client serves the saved responses in order, and checks that each request the harness sends matches what was recorded: same tool names, same number of messages, same tool-result ids.

When you change the harness, replay tests tell you whether the change altered what the model sees. Sometimes that's the point, like an improved tool description. Then re-record and review the diff. Sometimes it's an accident, like a tool renamed in one place but not another, or a prompt block reordered so the cache breaks. Then the test caught a bug before it cost anything.

Replay tests don't tell you whether the agent will still succeed with the new requests. A real model might respond differently. That's what live evals are for (mod-208, exercise-03).

### Live evals

Your harness's headless mode makes it a drop-in for the eval runner from mod-208: same tasks, same scoring, comparing harness versions instead of harness configurations. Run them before releasing a new harness version, and on a schedule to catch drift when the provider updates a model.

## Shipping

### Front ends

The same core should serve several front ends:

- **Interactive CLI.** Streams text, shows each tool call compactly, asks for `ask` decisions, supports interrupting.
- **Headless mode.** `harness -p "<prompt>"` for scripts and CI, with machine-readable output:
  - `--output-format text`: just the final answer.
  - `--output-format json`: one object at the end: result, stop reason, turns, usage, cost, run id, is_error.
  - `--output-format stream-json`: one event per line, as they happen.
- **Library API.** `Harness(config).run(prompt)` for embedding in a service.

Claude Code's `claude -p` and Codex's `codex exec` follow this pattern. That's what made them scriptable for mod-208's eval runner.

### Headless defaults

In headless mode no human is there to answer `ask`. Resolve `ask` to **deny** by default, with an explicit flag to allow. An unattended agent should never get more power than an attended one by default.

### Exit codes

Scripts branch on exit codes, so make them mean something and document them: `0` finished; `1` error; `2` bad usage or config; `3` stopped on a limit (turns, budget, loop). "Ran out of budget" must be distinguishable from "done" without parsing output.

### Cost

Track cost per run from token usage and a price table in config, including cache reads and writes, which are priced differently. Report it in headless JSON output and in `run_finished`. Keep the price table in config, not code, and keep it current. Prices change, and a stale table quietly under-reports spend.

### Packaging

Ship as an installable package with a console-script entry point (`pip install`, `uv tool install`), a versioned default config, and a `--version` flag. Put the harness version in every run log and trace, so you can connect a behavior change to the release that caused it.

## Key takeaways

- Emit a structured event for every step; logs, traces, UI, metrics, and tests all subscribe.
- Write JSONL run logs, flushed per event, redacted before writing, with a retention policy.
- Trace runs, model calls, and tool calls with GenAI attributes; no content by default; tracing optional.
- Test at three levels: scripted models for harness logic, replay for what the harness sends, live evals for behavior.
- Ship an interactive CLI, a headless mode with JSON output and meaningful exit codes, and a library API, with `ask` denied by default when unattended.
- Track cost from a configurable price table, and stamp the harness version on every run.
