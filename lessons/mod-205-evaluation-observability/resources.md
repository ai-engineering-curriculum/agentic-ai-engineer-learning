# Resources for mod-205-evaluation-observability

Primary references for agent evaluation and observability. Verify against current docs — eval tooling and the OTel GenAI conventions are evolving fast.

## Evaluation principles

- **Anthropic — Building effective agents** ([anthropic.com/engineering/building-effective-agents](https://www.anthropic.com/engineering/building-effective-agents)) — the evaluator-optimizer pattern and why you measure agents, not just models.
- **Anthropic — A statistical approach to model evals** ([anthropic.com/research/statistical-approach-to-model-evals](https://www.anthropic.com/research/statistical-approach-to-model-evals)) — reporting eval results with confidence, handling noise.
- **Anthropic — Defining and evaluating agents (docs)** ([docs.anthropic.com/en/docs/agents-and-tools/agent-evaluation](https://docs.anthropic.com/en/docs/agents-and-tools/agent-evaluation)) — outcome vs. trajectory evaluation guidance.

## OpenTelemetry GenAI

- **OTel — GenAI semantic conventions** ([opentelemetry.io/docs/specs/semconv/gen-ai/](https://opentelemetry.io/docs/specs/semconv/gen-ai/)) — the `gen_ai.*` attribute keys and span naming. The authoritative reference for Chapter 2.
- **OTel — GenAI agent spans** ([opentelemetry.io/docs/specs/semconv/gen-ai/gen-ai-agent-spans/](https://opentelemetry.io/docs/specs/semconv/gen-ai/gen-ai-agent-spans/)) — conventions specific to agent and tool-execution spans.
- **OTel — Python SDK** ([opentelemetry.io/docs/languages/python/](https://opentelemetry.io/docs/languages/python/)) — tracer setup, span processors, and OTLP exporters.

## Observability platforms

- **Langfuse** ([langfuse.com/docs](https://langfuse.com/docs)) — open-source LLM observability; see the [OpenTelemetry integration](https://langfuse.com/docs/opentelemetry/get-started) for the OTLP endpoint.
- **Arize Phoenix** ([docs.arize.com/phoenix](https://docs.arize.com/phoenix)) — open-source, OTel-native tracing and eval; runs locally or hosted.
- **LangSmith** ([docs.smith.langchain.com](https://docs.smith.langchain.com)) — tracing and eval, first-class for LangChain/LangGraph and OTel-compatible.

## LLM-as-judge

- **Judging LLM-as-a-Judge with MT-Bench and Chatbot Arena** ([arxiv.org/abs/2306.05685](https://arxiv.org/abs/2306.05685)) — the foundational paper; documents position, verbosity, and self-enhancement biases.
- **Langfuse — LLM-as-a-judge evaluation** ([langfuse.com/docs/evaluation/evaluation-methods/llm-as-a-judge](https://langfuse.com/docs/evaluation/evaluation-methods/llm-as-a-judge)) — practical rubric and scoring setup.

> You instrumented trajectory evals, OTel spans, a calibrated judge, and a CI-gated regression suite by hand in this module. When you adopt a platform's built-in evaluators and dashboards, you'll know exactly what they're computing — and you'll trust (and debug) them faster for having built each piece yourself.
