# exercise-02: OpenTelemetry Tracing Wire-Up

**Estimated effort:** 3 hours

## Objective

Instrument your agent with OpenTelemetry spans that follow the GenAI semantic conventions, then export the traces to a real platform (Langfuse, Arize Phoenix, or LangSmith) and read your agent's behavior as a nested trace. By the end you'll have provider-portable instrumentation that gives you cost, latency, and token charts for free.

## Background

This exercise covers material from:

- [Chapter 2 — OpenTelemetry GenAI Tracing](../02-otel-genai-tracing.md)

Use the agent loop from [exercise-01](exercise-01-trajectory-eval-implementation.md) — you're adding spans around the same model and tool calls you already instrumented. Pick **one** platform; all three accept OTLP. Arize Phoenix runs locally with no signup, which makes it the lowest-friction choice for this exercise.

## Prerequisites

- The instrumented agent loop from exercise-01.
- `opentelemetry-sdk` and an OTLP exporter installed.
- An account/endpoint for one platform (or Phoenix running locally), with credentials in environment variables — **never** hardcoded.

## Tasks

### 1. Set up the tracer and exporter

- Configure an OTel `TracerProvider` with an OTLP exporter pointed at your chosen platform's endpoint.
- Read the endpoint and auth from environment variables. Confirm a trivial test span shows up in the platform UI before going further.

### 2. Span the model calls

- Wrap each model call in a span named `chat {model}`.
- Set the standard attributes: `gen_ai.operation.name=chat`, `gen_ai.request.model`, `gen_ai.response.model`, and `gen_ai.usage.input_tokens` / `gen_ai.usage.output_tokens` from the response.

### 3. Span the tool calls

- Wrap each tool call in a span named `execute_tool {tool_name}` with `gen_ai.operation.name=execute_tool` and `gen_ai.tool.name`.
- Confirm the spans nest correctly under the root run span (use `start_as_current_span` so nesting is automatic).

### 4. Root span and errors

- Open a root `agent.run` span around the whole task so every model/tool span is a child.
- On any exception, set the span status to error and record the exception, so failed runs render red in the UI.

### 5. Read a trace

- Run 3–5 tasks, including one that errors. Open the traces in the platform and confirm you can see the full nested trajectory, per-call token counts, and total latency.

## Starter guidance

```python
from opentelemetry import trace
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor
from opentelemetry.exporter.otlp.proto.http.trace_exporter import OTLPSpanExporter

provider = TracerProvider()
provider.add_span_processor(BatchSpanProcessor(OTLPSpanExporter()))  # endpoint from env
trace.set_tracer_provider(provider)
tracer = trace.get_tracer("agent")

def call_model(messages, model):
    with tracer.start_as_current_span(f"chat {model}") as span:
        span.set_attribute("gen_ai.operation.name", "chat")
        span.set_attribute("gen_ai.request.model", model)
        resp = client.messages.create(model=model, messages=messages)
        span.set_attribute("gen_ai.usage.input_tokens", resp.usage.input_tokens)
        span.set_attribute("gen_ai.usage.output_tokens", resp.usage.output_tokens)
        return resp

def call_tool(name, args, fn):
    with tracer.start_as_current_span(f"execute_tool {name}") as span:
        span.set_attribute("gen_ai.operation.name", "execute_tool")
        span.set_attribute("gen_ai.tool.name", name)
        return fn(**args)
```

You may use a platform's auto-instrumentation as a comparison, but do the manual spans first so you understand what it generates.

## Acceptance criteria

You can demonstrate that:

- Traces appear in your chosen platform with model and tool spans **nested** under a root run span.
- Spans use the standard `gen_ai.*` attribute keys and the `chat {model}` / `execute_tool {name}` naming convention.
- The platform shows **token counts and latency** derived from your attributes (not hand-computed).
- A run that throws shows a **red/errored** span with the exception recorded.
- No credentials are hardcoded — all read from the environment.

## Reflection

In `NOTES.md`:

1. Which `gen_ai.*` attributes did the platform need to compute cost? What broke when one was missing?
2. How does the trace map onto the `Trajectory` from exercise-01? What does each representation make easier?
3. What would you sample in production, and what would you force-trace regardless?

## Stretch goals

- Add a sampler that traces 10% of runs but always traces errored runs.
- Export to a **second** platform by changing only the exporter config — prove the instrumentation is portable.
- Attach the exercise-01 trajectory match result as a span attribute so eval outcome and trace live together.
