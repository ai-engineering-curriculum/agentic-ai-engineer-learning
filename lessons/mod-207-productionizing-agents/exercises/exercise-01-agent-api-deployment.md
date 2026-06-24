# exercise-01: Agent API Deployment

**Estimated effort:** 3 hours

## Objective

Wrap an existing agent in a FastAPI service that supports a synchronous endpoint, a streaming endpoint, and a background-job endpoint, then containerize it so it runs identically on your laptop and in a cluster. By the end you'll have a deployable agent service and a clear feel for which endpoint shape fits which run length.

## Background

This exercise covers material from:

- [Chapter 1 — Deploying Agents Behind APIs](../01-agent-apis-and-containers.md)

Reuse the agent loop you built in [mod-201](../../mod-201-agent-fundamentals/README.md). Any model provider works; the agent only needs to expose a "run this prompt" entry point and, for streaming, an async generator of chunks.

## Prerequisites

- A working agent loop with tool calling (from mod-201).
- `async` Python, FastAPI, and `uvicorn` installed.
- Docker installed and running.
- API key in an environment variable; small spend cap.

## Tasks

### 1. Synchronous endpoint

- Add `POST /agent/run` that takes a validated `AgentRequest` (Pydantic) and returns the agent's final answer.
- Enforce a server-side timeout on the whole run so a stuck agent can't hold the connection forever.

### 2. Streaming endpoint

- Add `POST /agent/stream` that returns a `StreamingResponse` of Server-Sent Events, emitting chunks (tokens or step events) as the agent produces them.
- On client disconnect, stop the agent run — do not keep burning tokens for a connection nobody is reading.

### 3. Background job

- Add `POST /agent/jobs` that returns a `job_id` immediately and runs the agent in the background.
- Add `GET /agent/jobs/{job_id}` that returns status (`running` / `done` / `failed`) and, when done, the result.
- Use an in-process job store (a dict is fine for this exercise) and note in `NOTES.md` why it would not survive a restart.

### 4. Operational basics

- Add `GET /healthz` that verifies the model client is configured.
- Cap concurrent in-flight runs with a semaphore so a burst can't exhaust your rate limit.
- Read the API key from the environment; never hardcode it.

### 5. Containerize

- Write a `Dockerfile` that pins the base image, caches the dependency layer, runs as a non-root user, and starts `uvicorn`.
- Build the image and run it locally, passing the API key via `-e`. Hit all three endpoints against the container.

## Starter guidance

```python
import asyncio
from fastapi import FastAPI, BackgroundTasks, HTTPException
from fastapi.responses import StreamingResponse
from pydantic import BaseModel

app = FastAPI()
RUN_LIMIT = asyncio.Semaphore(4)
JOBS: dict[str, dict] = {}

class AgentRequest(BaseModel):
    prompt: str

@app.post("/agent/run")
async def run(req: AgentRequest) -> dict:
    async with RUN_LIMIT:
        try:
            answer = await asyncio.wait_for(run_agent(req.prompt), timeout=120)
        except asyncio.TimeoutError:
            raise HTTPException(504, "agent run timed out")
    return {"answer": answer}

@app.post("/agent/stream")
async def stream(req: AgentRequest) -> StreamingResponse:
    raise NotImplementedError  # SSE generator over run_agent_streaming(req.prompt)

@app.post("/agent/jobs")
async def start_job(req: AgentRequest, bg: BackgroundTasks) -> dict:
    raise NotImplementedError  # create job id, run in background, return id

@app.get("/healthz")
async def healthz() -> dict:
    return {"ok": True}
```

You do **not** need durable jobs here — the in-process store is intentional, and you'll replace it in [exercise-02](exercise-02-durable-execution-temporal.md).

## Acceptance criteria

You can demonstrate that:

- `POST /agent/run` returns a validated answer and returns `504` when the run exceeds the timeout.
- `POST /agent/stream` streams chunks incrementally (not one buffered blob at the end) and stops work on client disconnect.
- `POST /agent/jobs` returns immediately with a `job_id`, and polling `GET /agent/jobs/{job_id}` reflects `running` then `done`.
- The concurrency cap holds: more simultaneous requests than the semaphore allows queue rather than all fire at once.
- The container builds, runs with the key passed via `-e`, and serves all three endpoints.

## Reflection

In `NOTES.md`:

1. For your agent's typical run length, which endpoint shape is the right default, and why?
2. Your background job store loses everything on restart. List the cases where that's unacceptable — and what you'd reach for instead (preview of exercise-02).
3. What did you put in the container image vs. inject at runtime, and why does the API key belong in the second group?

## Stretch goals

- Add request cancellation to the streaming endpoint via `asyncio.CancelledError` and confirm token spend stops on disconnect.
- Add structured request logging (run id, model, token usage, wall-clock) you can later ship to your observability stack ([mod-205](../../mod-205-evaluation-observability/README.md)).
- Run the container under `gunicorn` with multiple uvicorn workers and measure throughput under a small load test.
