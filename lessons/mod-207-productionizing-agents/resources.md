# Resources for mod-207-productionizing-agents

Primary references for productionizing agents. Verify against current docs — these tools move fast.

## Serving and containers

- **FastAPI** ([fastapi.tiangolo.com](https://fastapi.tiangolo.com)) — the framework. See [Background Tasks](https://fastapi.tiangolo.com/tutorial/background-tasks/) and [Custom Response / StreamingResponse](https://fastapi.tiangolo.com/advanced/custom-response/) for the job and streaming endpoint shapes.
- **Uvicorn** ([uvicorn.org](https://www.uvicorn.org)) — the ASGI server you run FastAPI on in production.
- **Docker — Python language guide** ([docs.docker.com/language/python](https://docs.docker.com/language/python/)) — building, layering, and running a Python service image.

## Durable execution

- **Temporal — Python SDK docs** ([docs.temporal.io/develop/python](https://docs.temporal.io/develop/python)) — workflows, activities, workers, and retry policies in Python.
- **Temporal — Core concepts** ([docs.temporal.io/concepts](https://docs.temporal.io/concepts)) — workflow determinism, replay, and activity execution semantics. Read this before writing your first workflow.
- **Temporal Python SDK (GitHub)** ([github.com/temporalio/sdk-python](https://github.com/temporalio/sdk-python)) — samples and the `temporalio` package.

## Caching and routing

- **Anthropic — Prompt caching** ([docs.anthropic.com/en/docs/build-with-claude/prompt-caching](https://docs.anthropic.com/en/docs/build-with-claude/prompt-caching)) — `cache_control`, the cacheable-prefix rules, expiry, and the `usage` cache fields.
- **Anthropic — Models overview** ([docs.anthropic.com/en/docs/about-claude/models/overview](https://docs.anthropic.com/en/docs/about-claude/models/overview)) — model tiers and ids for choosing routing targets.
- **Anthropic — Message Batches** ([docs.anthropic.com/en/docs/build-with-claude/batch-processing](https://docs.anthropic.com/en/docs/build-with-claude/batch-processing)) — 50% savings for non-time-sensitive bulk turns.

## Human-in-the-loop and persistence

- **LangGraph — Persistence** ([langchain-ai.github.io/langgraph/concepts/persistence](https://langchain-ai.github.io/langgraph/concepts/persistence/)) — checkpointers, threads, and state across steps.
- **LangGraph — Human-in-the-loop** ([langchain-ai.github.io/langgraph/concepts/human_in_the_loop](https://langchain-ai.github.io/langgraph/concepts/human_in_the_loop/)) — the `interrupt()` / `Command(resume=...)` pause-and-resume pattern.
- **LangGraph — Add human-in-the-loop (how-to)** ([langchain-ai.github.io/langgraph/how-tos/human_in_the_loop/add-human-in-the-loop](https://langchain-ai.github.io/langgraph/how-tos/human_in_the_loop/add-human-in-the-loop/)) — a worked approval-gate example.

## Background reading

- **Anthropic — Building effective agents** ([anthropic.com/engineering/building-effective-agents](https://www.anthropic.com/engineering/building-effective-agents)) — the patterns you're now hardening for production.

> You built these patterns by hand in earlier modules. This module is about making them survive load, failure, a budget, and a human reviewer. See [mod-205: Evaluation & Observability](../mod-205-evaluation-observability/README.md) to measure the wins.
