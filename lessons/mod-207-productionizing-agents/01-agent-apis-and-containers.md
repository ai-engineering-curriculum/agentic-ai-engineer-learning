# Chapter 1 — Deploying Agents Behind APIs

An agent that lives in a script is a tool *you* run. To make it a service other systems call, you put it behind an HTTP API and ship it as a container. FastAPI is the default for Python: it is async-native (your agent already awaits model and tool calls), it validates request/response bodies with Pydantic, and it streams.

## Three endpoint shapes

Agent latency is the problem an API design has to solve. A single agent turn can take seconds; a multi-step run can take minutes. You have three shapes:

- **Synchronous request/response.** The client waits for the whole answer. Simple, but only viable for short runs — long ones hit client and proxy timeouts.
- **Streaming.** The server sends tokens (or step events) as they arrive over Server-Sent Events. Same total time, far better perceived latency, and the connection stays warm.
- **Background job.** For runs that take minutes, return a job id immediately and let the client poll or subscribe. The work outlives the request.

```python
from fastapi import FastAPI, BackgroundTasks
from fastapi.responses import StreamingResponse
from pydantic import BaseModel

app = FastAPI()

class AgentRequest(BaseModel):
    prompt: str

@app.post("/agent/stream")
async def agent_stream(req: AgentRequest) -> StreamingResponse:
    async def events():
        async for chunk in run_agent_streaming(req.prompt):
            yield f"data: {chunk}\n\n"
    return StreamingResponse(events(), media_type="text/event-stream")

@app.post("/agent/jobs")
async def start_job(req: AgentRequest, bg: BackgroundTasks) -> dict:
    job_id = create_job()
    bg.add_task(run_agent_job, job_id, req.prompt)
    return {"job_id": job_id, "status": "running"}
```

> `BackgroundTasks` runs *in the same process*. It's fine for a short-lived async task, but it dies if the process restarts and doesn't survive a deploy. For anything you can't afford to lose, you want a real job store or a durable workflow ([Chapter 2](02-durable-execution.md)).

## Operational concerns the endpoint owns

- **Validation at the boundary.** Pydantic models reject malformed input before it reaches the agent. Never pass an unvalidated request body into a prompt.
- **Timeouts and cancellation.** Bound every model and tool call. When the client disconnects, stop the work — a runaway agent loop on an abandoned request is pure cost.
- **Concurrency limits.** One agent run can fan out to many model calls. Cap in-flight runs (a semaphore or a queue) so a traffic spike doesn't exhaust your rate limit or memory.
- **Health and readiness.** A `/healthz` endpoint that checks the model client and any datastore lets your orchestrator know when to route traffic.
- **Secrets from the environment.** The API key comes from an environment variable, never the image. The container should be runnable with `-e ANTHROPIC_API_KEY=...`.

## Containerize for reproducibility

The point of the container is that the agent runs the same on your laptop and in the cluster. Pin the base image, install dependencies first so layers cache, run as a non-root user, and start with an ASGI server.

```dockerfile
FROM python:3.12-slim

WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .
RUN useradd --create-home appuser
USER appuser

EXPOSE 8000
CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

Run `uvicorn` (or `gunicorn` with uvicorn workers) as the entrypoint — never the dev server's reload mode in production. Keep the image slim: a smaller image pulls and starts faster, which matters when the orchestrator is scaling you up under load.

## Key takeaways

- Match the endpoint shape to run length: synchronous for short, streaming for perceived latency, background jobs for long runs.
- `BackgroundTasks` is in-process and does not survive a restart — use it only for work you can afford to lose.
- The endpoint owns validation, timeouts, cancellation, concurrency limits, and health checks; the container owns reproducibility and secret hygiene.
- Pin the base image, cache the dependency layer, run as non-root, and start with a production ASGI server.
