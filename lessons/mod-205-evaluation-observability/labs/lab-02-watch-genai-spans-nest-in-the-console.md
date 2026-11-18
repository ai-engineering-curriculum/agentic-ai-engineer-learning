# Lab 2 — Watch GenAI Spans Nest in the Console

**Duration:** ~40 minutes
**Covers:** [Chapter 2 — OpenTelemetry GenAI Tracing](../02-otel-genai-tracing.md)

## Objective

Chapter 2 claims three things: `start_as_current_span` makes nesting automatic, the `gen_ai.*` semantic conventions are what let a backend compute cost and latency for free, and setting error status on a failed span makes a bad run visibly red. Before Exercise 02 asks you to ship traces to a real platform (Langfuse, Phoenix, or LangSmith), prove all three locally against a **console exporter** — no account, no network, no credentials. The point is to see the span tree with your own eyes before you point it anywhere.

By the end of the lab you will:

- Configure an OTel `TracerProvider` with the built-in `ConsoleSpanExporter` so every span is dumped as JSON to stdout.
- Run a fake agent task that emits a root `agent.run` span, two `chat {model}` spans, and two `execute_tool {name}` spans — all nested — using only the standard `gen_ai.*` attribute keys from the chapter.
- Force one task to throw, set the span status to error, and read the exception back from the emitted span.

Exercise 02 then swaps `ConsoleSpanExporter` for an **OTLP exporter** pointed at a real platform. Because the span *production* and span *transport* are different pieces, this lab lets you verify the production half first — if the console output already shows nested spans with the right keys, the exercise is one config change away.

## Prerequisites

- Python 3.11+ and a fresh virtualenv.
- [Chapter 2](../02-otel-genai-tracing.md) read.

## Setup

```bash
python -m venv .venv && source .venv/bin/activate
pip install opentelemetry-sdk opentelemetry-api
mkdir lab-02 && cd lab-02
```

`opentelemetry-sdk` bundles `ConsoleSpanExporter`. No OTLP exporter or platform SDK is needed for this lab — those are Exercise 02's territory.

## Step 1 — Configure the tracer with a console exporter

This is the same five-line setup Chapter 2 shows, with `ConsoleSpanExporter` swapped in for `OTLPSpanExporter`. Everything downstream is identical.

Save as `tracing_setup.py`:

```python
from opentelemetry import trace
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor, ConsoleSpanExporter


def install_console_tracer() -> trace.Tracer:
    provider = TracerProvider()
    provider.add_span_processor(BatchSpanProcessor(ConsoleSpanExporter()))
    trace.set_tracer_provider(provider)
    return trace.get_tracer("agent")
```

Chapter 2's exporter choice is OTLP because that's what ships to a platform. The console exporter writes the same spans to stdout instead of a network endpoint — so you can read exactly what would have been sent. The `BatchSpanProcessor` matches the chapter: spans are buffered and flushed, not emitted synchronously on span close.

## Step 2 — A fake model call that sets the standard GenAI attributes

Chapter 2 names the attribute keys explicitly: `gen_ai.operation.name`, `gen_ai.request.model`, `gen_ai.response.model`, `gen_ai.usage.input_tokens`, `gen_ai.usage.output_tokens`. Write a function that takes no real model — it just returns a pre-canned response — so the span-emission path is isolated from any API behavior.

Save as `fake_calls.py`:

```python
import time
from dataclasses import dataclass
from opentelemetry import trace


@dataclass
class FakeResponse:
    """What the chapter's `resp = client.messages.create(...)` would hand back,
    minus the parts the span doesn't use."""
    model: str
    text: str
    input_tokens: int
    output_tokens: int


def call_model(tracer: trace.Tracer, messages: list[dict], model: str) -> FakeResponse:
    """A stand-in for a provider SDK call. Emits a `chat {model}` span with
    every standard gen_ai.* attribute from Chapter 2."""
    with tracer.start_as_current_span(f"chat {model}") as span:
        span.set_attribute("gen_ai.operation.name", "chat")
        span.set_attribute("gen_ai.provider.name", "anthropic")
        span.set_attribute("gen_ai.request.model", model)
        time.sleep(0.05)                           # fake latency so the span has width
        resp = FakeResponse(
            model=model,
            text=f"(fake reply from {model} to: {messages[-1]['content'][:40]}...)",
            input_tokens=120,
            output_tokens=37,
        )
        span.set_attribute("gen_ai.response.model", resp.model)
        span.set_attribute("gen_ai.usage.input_tokens", resp.input_tokens)
        span.set_attribute("gen_ai.usage.output_tokens", resp.output_tokens)
        return resp


def call_tool(tracer: trace.Tracer, name: str, args: dict, fn) -> str:
    """Wrap one tool call in an `execute_tool {name}` span with gen_ai.tool.name,
    following the chapter's convention."""
    with tracer.start_as_current_span(f"execute_tool {name}") as span:
        span.set_attribute("gen_ai.operation.name", "execute_tool")
        span.set_attribute("gen_ai.tool.name", name)
        time.sleep(0.02)
        return fn(**args)
```

Three details that come straight from the chapter:

- The span **name** follows the convention (`chat {model}`, `execute_tool {name}`) — backends use the name for grouping and labeling. An ad-hoc name like `"anthropic_call"` would break their dashboards even if the attributes were correct.
- `gen_ai.request.model` vs. `gen_ai.response.model` are **both** set because the served model can differ from the requested one (e.g., an auto-routed alias). The chapter calls this out as a cost-tracking reason to keep them distinct.
- `gen_ai.usage.input_tokens` / `gen_ai.usage.output_tokens` are taken from the response, not estimated. Chapter 2: "the platform shows token counts and latency derived from your attributes (not hand-computed)."

## Step 3 — Compose a root span with nested children

Chapter 2: wrap the whole task in an `agent.run` root span so every model and tool span is a child, and nesting falls out of `start_as_current_span` with no manual parent threading.

Save as `agent_run.py`:

```python
from opentelemetry import trace

from fake_calls import call_model, call_tool


def search_tool(q: str) -> str:
    return f"(fake search result for: {q})"


def calc_tool(expr: str) -> str:
    return f"(fake calc result for: {expr})"


def run_task(tracer: trace.Tracer, task: str) -> str:
    """The Chapter 2 nesting shape: one root span, child chat/tool spans."""
    with tracer.start_as_current_span("agent.run") as root:
        root.set_attribute("agent.task", task)

        # Round 1: reason, then call a tool.
        _ = call_model(tracer, [{"role": "user", "content": task}],
                       model="claude-sonnet-4")
        _ = call_tool(tracer, "search", {"q": task}, search_tool)

        # Round 2: reason again, then call a second tool.
        _ = call_model(tracer, [{"role": "user", "content": task}],
                       model="claude-sonnet-4")
        _ = call_tool(tracer, "calc", {"expr": "2+2"}, calc_tool)

        return "4"
```

No parent IDs threaded by hand. `start_as_current_span` reads the active span from context and links the new one to it automatically — that's the one Chapter 2 sentence the whole nesting story rests on.

## Step 4 — Run it and read the span JSON

Save as `main.py`:

```python
from tracing_setup import install_console_tracer
from agent_run import run_task


def main() -> None:
    tracer = install_console_tracer()
    answer = run_task(tracer, "what is 2 + 2?")
    print(f"\nanswer: {answer}\n")


if __name__ == "__main__":
    # BatchSpanProcessor flushes at interpreter shutdown, so the spans print
    # *after* main() returns. That's expected — don't be alarmed if the answer
    # appears before the span dumps.
    main()
```

Run:

```bash
python main.py
```

**Expected output (abridged — the exporter prints each span as a JSON blob; exact field order varies):**

```
answer: 4

{
    "name": "chat claude-sonnet-4",
    "context": {"trace_id": "0x...", "span_id": "0x...child1...", ...},
    "parent_id": "0x...root...",
    "attributes": {
        "gen_ai.operation.name": "chat",
        "gen_ai.provider.name": "anthropic",
        "gen_ai.request.model": "claude-sonnet-4",
        "gen_ai.response.model": "claude-sonnet-4",
        "gen_ai.usage.input_tokens": 120,
        "gen_ai.usage.output_tokens": 37
    },
    ...
}
{"name": "execute_tool search", "parent_id": "0x...root...", "attributes": {"gen_ai.operation.name": "execute_tool", "gen_ai.tool.name": "search"}, ...}
{"name": "chat claude-sonnet-4", "parent_id": "0x...root...", ...}
{"name": "execute_tool calc", "parent_id": "0x...root...", ...}
{"name": "agent.run", "parent_id": null, "attributes": {"agent.task": "what is 2 + 2?"}, ...}
```

Three things to check in your actual output:

1. **All four child spans share the same `trace_id`** — that's what makes them one trace rather than four orphans.
2. **Each child's `parent_id` matches the `agent.run` span's `span_id`** — nesting came for free from `start_as_current_span`; no manual linking in the code.
3. **The attribute keys are literally `gen_ai.operation.name`, `gen_ai.request.model`, `gen_ai.tool.name`, etc.** — Chapter 2: using the standard keys is what unlocks the platform's cost and latency charts. If you had used `model_name` or `operation`, this trace would render as uncategorized raw spans in Langfuse/Phoenix/LangSmith.

The root `agent.run` span prints **last** because `BatchSpanProcessor` flushes on shutdown and the root closes after all its children — this is the usual gotcha; read the output bottom-up if you're matching nesting by eye.

## Step 5 — Force an error and set the span status

Chapter 2: "Record failures on the span: set the status to error and attach the exception so a failed run is visibly red, not silently missing."

Save as `main_error.py`:

```python
from opentelemetry import trace
from opentelemetry.trace import Status, StatusCode

from tracing_setup import install_console_tracer
from fake_calls import call_tool


def flaky_tool(key: str) -> str:
    raise RuntimeError(f"lookup failed for key={key!r}")


def run_failing_task(tracer: trace.Tracer) -> None:
    with tracer.start_as_current_span("agent.run") as root:
        try:
            call_tool(tracer, "lookup", {"key": "missing"}, flaky_tool)
        except Exception as e:
            # Chapter 2: set the status and record the exception.
            root.set_status(Status(StatusCode.ERROR, str(e)))
            root.record_exception(e)
            # Deliberately not re-raising — we want the interpreter to shut
            # down cleanly so the batch processor flushes the error span.


if __name__ == "__main__":
    tracer = install_console_tracer()
    run_failing_task(tracer)
```

Run:

```bash
python main_error.py
```

**Expected output (abridged):**

```
{
    "name": "execute_tool lookup",
    "parent_id": "0x...root...",
    "attributes": {"gen_ai.operation.name": "execute_tool", "gen_ai.tool.name": "lookup"},
    ...
}
{
    "name": "agent.run",
    "parent_id": null,
    "status": {"status_code": "ERROR", "description": "lookup failed for key='missing'"},
    "events": [
        {
            "name": "exception",
            "attributes": {
                "exception.type": "RuntimeError",
                "exception.message": "lookup failed for key='missing'",
                "exception.stacktrace": "Traceback (most recent call last):\n..."
            }
        }
    ],
    ...
}
```

Two things to confirm:

- The `agent.run` span's `status` is `ERROR` with the description set from the exception message. **This is what makes the trace render red in a platform UI.** A span that silently completes with `UNSET` status looks indistinguishable from a successful one.
- The exception is attached as an `events` entry with type, message, and stacktrace. The platform's run-detail view will surface these so the on-call engineer sees what broke without digging through logs.

Notice the tool span itself (`execute_tool lookup`) has no error status set. The exception bubbled *out* of `call_tool` before any post-error attribute was added, and the tool span is closed by the `with` block's `__exit__`. If you want **both** the tool span and the root to render red, you'd set `Status(ERROR)` on the tool span before re-raising — a stretch exercise below. For most backends, having the root red is already enough to route attention.

## Step 6 — Break the convention on purpose and see what rots

Edit `fake_calls.py` and replace `"gen_ai.operation.name"` with `"operation_name"` (a custom key). Rerun `main.py`.

Re-read the JSON output: the span still has an `operation_name` attribute, but it is **not** a `gen_ai.*` key. If you shipped this to Langfuse, Phoenix, or LangSmith, the platform would treat the span as uncategorized — no `chat`/`execute_tool` grouping, no cost derivation from token counts, no latency breakdown by operation. The trace would still be *readable* in raw JSON form, but every dashboard built around the GenAI conventions would be blind to it.

Revert the edit. The point is Chapter 2's literal claim: *"invent your own keys and you get raw spans nobody's dashboard understands."* This lab lets you prove that without actually shipping bad spans to a shared backend.

**Checkpoint:** without peeking, you can state:

1. The two Chapter 2 moves that make nesting automatic (hint: one is `start_as_current_span`, one is the fact that `BatchSpanProcessor` preserves the parent-child link through context propagation).
2. The three `gen_ai.*` attributes a platform needs to compute cost for a chat span, and which one names the *served* model vs. the *requested* one.
3. Why setting `Status(ERROR)` on the root span is not the same as raising the exception — and what the operator sees differently in each case.

## Wrap-up

Carry the span layout from this lab — `agent.run` root, `chat {model}` and `execute_tool {name}` children, the full `gen_ai.*` attribute set, and the error-status pattern — directly into [Exercise 02](../exercises/exercise-02-otel-tracing-wireup.md). The only line that changes between this lab and the exercise is in `tracing_setup.py`: swap `ConsoleSpanExporter` for an OTLP exporter and set the endpoint/headers from environment variables. Everything else — the nesting, the attributes, the error-status handling — is already the shape the exercise asks for.

### Stretch

- Also set `Status(ERROR)` on the failing tool span before re-raising, so both the tool span and the root render red.
- Add a second root `agent.run` for a successful task immediately after the failing one, and confirm the error does not "stick" to the next trace — they share no `trace_id`.
- Swap `ConsoleSpanExporter` for `InMemorySpanExporter` (also in `opentelemetry-sdk`) and write a tiny assertion that verifies the number of emitted spans and their names, as a template for unit-testing your instrumentation.
