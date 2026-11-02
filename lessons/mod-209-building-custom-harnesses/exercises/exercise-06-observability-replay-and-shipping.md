# exercise-06: Observability, Replay Testing, and Shipping

**Estimated effort:** 3 hours

## Objective

Make your harness something you can operate and change safely. Record every run as a structured event log, emit OpenTelemetry spans, build a record-and-replay model client so you can turn any real run into a deterministic regression test, and package the harness as a CLI with a headless mode that other programs can drive. By the end you can answer "what did the agent do, and why?" for any run, and you can change the harness without fear.

## Background

This exercise covers material from:

- [Chapter 8 — Observability, Testing, and Shipping](../08-observability-testing-shipping.md)

Build on your [exercise-01](exercise-01-build-a-minimal-harness.md) harness or its reference solution. If you did exercise-03, reuse its event callback; otherwise add one here.

## Prerequisites

- A working exercise-01 harness.
- Python 3.11+, `pytest`.
- `opentelemetry-sdk` for task 2 (its tests can use the in-memory span exporter). Optionally a local trace viewer such as Jaeger or Phoenix.

## Tasks

### 1. Event log

- Write every run to `~/.local/state/<harness>/runs/<run_id>.jsonl` (configurable): one JSON object per line, each with `ts`, `run_id`, `seq`, and `type`.
- Event types: `run_started` (config summary, model, workspace), `model_request` (message count, estimated tokens), `model_response` (stop reason, tool calls, usage), `tool_call` (name, input, permission decision, hook outcome), `tool_result` (is_error, size, duration), and `run_finished` (stop reason, totals).
- **Redact secrets before writing.** Strip values for keys like `api_key`, `authorization`, and `token`, and mask strings that look like common key formats. Test the redaction.
- Write each event as it happens and flush, so a crash leaves a usable log.

### 2. OpenTelemetry spans

- One span per run, a child span per model call, and a child span per tool call.
- Put attributes on the model spans following the OpenTelemetry GenAI semantic conventions where they fit (`gen_ai.operation.name`, `gen_ai.request.model`, `gen_ai.usage.input_tokens`, `gen_ai.usage.output_tokens`). Put tool name, permission decision, and error status on tool spans.
- Do **not** put prompt or tool content on spans by default; add a config flag to include it.
- Tracing is off unless configured, and the harness runs without `opentelemetry` installed.

### 3. Record and replay

- `RecordingModel` wraps a real `ModelClient` and saves each request and response to a cassette file.
- `ReplayModel` serves responses from a cassette, in order. For each call, it checks that the request matches what was recorded: same tool schema names and same number of messages, and the last message has the same role and the same tool-result ids. On a mismatch it fails with a clear diff of what changed.
- Write two regression tests from cassettes: one recorded from a successful task, and one where the model hits a permission denial and recovers. (If you have no API key, hand-write the cassettes; the format is yours.)
- Show that a deliberate harness change, such as renaming a tool, makes the replay test fail with a readable message.

### 4. Headless CLI

- `harness "<prompt>"` runs interactively: it streams text, shows tool calls, and prompts for `ask` decisions.
- `harness -p "<prompt>" --output-format text|json|stream-json` runs headless:
  - `text`: the final answer only.
  - `json`: one object at the end with `result`, `stop_reason`, `turns`, `usage`, `cost_usd` (from a price table you keep in config), `run_id`, and `is_error`.
  - `stream-json`: one JSON event per line as they happen.
- In headless mode `ask` resolves to deny, unless `--allow-ask` is passed.
- Exit codes: `0` done; `1` error; `2` bad usage or config; `3` stopped on a limit (`max_turns`, budget, loop).
- Package it with a `pyproject.toml` console-script entry point so `pip install -e .` gives you a `harness` command.

## Starter guidance

```python
REDACT_KEYS = {"api_key", "apikey", "authorization", "token", "password", "secret"}
KEY_PATTERNS = [re.compile(r"sk-[A-Za-z0-9_-]{16,}"), re.compile(r"ghp_[A-Za-z0-9]{20,}")]

def redact(value: Any) -> Any:
    if isinstance(value, dict):
        return {k: "[REDACTED]" if k.lower() in REDACT_KEYS else redact(v) for k, v in value.items()}
    if isinstance(value, list):
        return [redact(v) for v in value]
    if isinstance(value, str):
        return reduce(lambda s, p: p.sub("[REDACTED]", s), KEY_PATTERNS, value)
    return value
```

## Acceptance criteria

You can demonstrate that, in `pytest`:

- A scripted run produces a JSONL log with the expected event types in order, increasing `seq`, and no secrets (test with a planted fake key).
- With the in-memory exporter, a run produces one run span, one span per model call, and one per tool call, with the GenAI attributes and no content by default.
- Both cassette regression tests pass, and renaming a tool makes one fail with a message that names the difference.
- `harness -p ... --output-format json` prints a single valid JSON object with all the listed fields, and the exit code matches the stop reason.
- In headless mode an `ask` decision is denied unless `--allow-ask` is passed.

## Reflection

In `NOTES.md`:

1. Which events did you need most while debugging this module's exercises? Which ones were noise?
2. Replay tests check that the harness sends the same requests. What kinds of bugs do they catch, and what kinds do they miss?
3. You now have a harness CLI with a headless JSON mode. How would you plug it into the eval runner from mod-208's exercise-03?

## Stretch goals

- A `harness replay <run_id>` command that renders a past run's log as a readable transcript.
- A `harness resume <run_id>` command that rebuilds history from the event log and continues.
- A cost report across all runs in the log directory: totals by day, model, and stop reason.
