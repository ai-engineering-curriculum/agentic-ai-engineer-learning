# Chapter 2 — OpenTelemetry GenAI Tracing

Evals tell you *whether* an agent is good. Tracing tells you *what it actually did* on a specific run — every model call, every tool invocation, every nested sub-agent — so when something fails in production you can open one run and see exactly where it went wrong. The discipline that makes this portable instead of vendor-locked is **OpenTelemetry (OTel)** and its **GenAI semantic conventions**: a standard vocabulary for naming spans and attributes so the same instrumentation exports to LangSmith, Langfuse, Arize Phoenix, or any OTLP backend.

## Spans are the unit of tracing

A **span** represents one operation with a start time, end time, status, and attributes. Spans nest: a parent span (the whole agent run) contains child spans (each model call, each tool call), and the parent-child links form a **trace** — a tree you can read top to bottom.

```text
span: agent.run                                  (root)
 ├─ span: chat gpt-4o          gen_ai.* attrs
 ├─ span: execute_tool search  gen_ai.tool.name=search
 ├─ span: chat gpt-4o
 └─ span: execute_tool calc
```

The mental model maps cleanly onto [Chapter 1's trajectory](01-trajectory-tool-call-eval.md): the trace *is* the trajectory, captured as nested spans instead of a flat list. One representation for both human debugging and automated eval.

## The GenAI semantic conventions

The OTel GenAI conventions define standard span names and attribute keys so backends know how to render them. The keys live under the `gen_ai.*` namespace. The ones you'll use most:

- `gen_ai.operation.name` — `chat`, `execute_tool`, `embeddings`.
- `gen_ai.system` / `gen_ai.provider.name` — the provider (`anthropic`, `openai`).
- `gen_ai.request.model` and `gen_ai.response.model` — requested vs. served model.
- `gen_ai.usage.input_tokens` / `gen_ai.usage.output_tokens` — token counts, for cost.
- `gen_ai.tool.name` — the tool a span invoked.

Span names follow a convention too: `chat {model}` for a model call, `execute_tool {tool_name}` for a tool call. Using the standard keys is what lets a platform compute cost, latency, and token charts for free — invent your own keys and you get raw spans nobody's dashboard understands.

## Instrumenting a span by hand

The auto-instrumentation libraries are convenient, but you should know what they do. A manual span:

```python
from opentelemetry import trace

tracer = trace.get_tracer("agent")

def call_model(messages, model="claude-sonnet-4"):
    with tracer.start_as_current_span(f"chat {model}") as span:
        span.set_attribute("gen_ai.operation.name", "chat")
        span.set_attribute("gen_ai.request.model", model)
        resp = client.messages.create(model=model, messages=messages)
        span.set_attribute("gen_ai.usage.input_tokens", resp.usage.input_tokens)
        span.set_attribute("gen_ai.usage.output_tokens", resp.usage.output_tokens)
        return resp
```

Wrap each tool call in an `execute_tool {name}` span the same way, and the nesting falls out automatically because `start_as_current_span` makes the new span a child of whatever is active.

## Exporting to a platform

A span is useless until it leaves your process. You configure an **exporter** — usually OTLP over HTTP or gRPC — pointed at a collector or directly at a platform's OTLP endpoint. The platforms in this module each accept OTel:

- **Langfuse** — native OTLP endpoint; set the endpoint and auth headers, and GenAI spans show up as traces with cost and latency computed from the standard attributes.
- **Arize Phoenix** — open-source, OTel-native; runs locally or hosted, built around OpenInference/OTel spans.
- **LangSmith** — first-class for LangChain/LangGraph and also ingests OTel traces.

Because you used the standard conventions, switching backends is a config change, not a re-instrumentation.

## Errors, status, and sampling

Record failures on the span: set the status to error and attach the exception so a failed run is visibly red, not silently missing. And in production, **sample** — tracing every run at full fidelity is expensive. Keep a representative sample plus a rule that always traces errored runs, so you never lose the traces you most need.

## Key takeaways

- A **span** is one timed operation; nested spans form a **trace** that is your trajectory in another shape.
- Use the **GenAI semantic conventions** (`gen_ai.*` keys, `chat {model}` / `execute_tool {name}` span names) so platforms render cost, latency, and tokens automatically.
- Configure an **OTLP exporter** to ship spans to Langfuse, Phoenix, or LangSmith; standard conventions make the backend swappable.
- Record **error status** on failing spans and **sample** in production, always keeping errored runs.
