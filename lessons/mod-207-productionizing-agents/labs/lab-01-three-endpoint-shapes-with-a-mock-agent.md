# Lab 1 — Three Endpoint Shapes with a Mock Agent

**Duration:** ~45 minutes
**Covers:** [Chapter 1 — Deploying Agents Behind APIs](../01-agent-apis-and-containers.md)

## Objective

Chapter 1 argues the hard part of serving an agent isn't the FastAPI app — it's **matching the endpoint shape to the run length**. Before [Exercise 01](../exercises/exercise-01-agent-api-deployment.md) asks you to wire the three shapes to a real agent and ship a container, spend 45 minutes driving sync / streaming / background against a **mock agent** so every chunk and every job state is deterministic.

By the end of the lab you will:

- Serve a sync `POST /agent/run`, a streaming `POST /agent/stream`, and a background `POST /agent/jobs` + `GET /agent/jobs/{id}` against a scripted mock agent — no tokens, no network.
- Watch the streaming endpoint deliver chunks *as they're produced*, not as one buffered blob at the end.
- Watch the background endpoint return a `job_id` instantly while the job runs after the request completes, and poll it to `done`.
- Prove by killing the Uvicorn process mid-job that `BackgroundTasks` is **in-process** — the job is lost, exactly as Chapter 1 warns.

This lab is intentionally narrower than Exercise 01: no concurrency cap, no timeouts, no Docker, no real model. The mock agent is the lever that makes "which shape when" visible before you spend a token on it.

## Prerequisites

- Python 3.11+ and a fresh virtualenv.
- [Chapter 1](../01-agent-apis-and-containers.md) read.
- `fastapi`, `uvicorn`, and `httpx` installed (`httpx` ships a test client plus a SSE-friendly request client).

## Setup

```bash
python -m venv .venv && source .venv/bin/activate
pip install "fastapi>=0.110" "uvicorn[standard]>=0.29" "httpx>=0.27"
mkdir lab-01 && cd lab-01
```

## Step 1 — A scripted mock agent

A real agent produces chunks non-deterministically. A mock agent produces exactly the chunks you tell it to, with exactly the per-chunk delay you tell it to, so every endpoint shape is reproducible.

Save as `mock_agent.py`:

```python
import asyncio
from collections.abc import AsyncIterator


CHUNKS = ["Reading", " the", " question", ".", " ", "Thinking", ".", " ", "Answer", ": 42."]


async def run_agent(prompt: str) -> str:
    """Synchronous-feeling run: returns the whole answer after the full delay."""
    out = []
    async for chunk in run_agent_streaming(prompt):
        out.append(chunk)
    return "".join(out)


async def run_agent_streaming(prompt: str, per_chunk_delay: float = 0.15) -> AsyncIterator[str]:
    """Yields chunks with a per-chunk delay so streaming is visibly incremental."""
    for chunk in CHUNKS:
        await asyncio.sleep(per_chunk_delay)
        yield chunk
```

Two details worth noticing:

- `run_agent` is just `run_agent_streaming` drained to completion. In a real service the sync and stream endpoints are usually two views on the **same** underlying generator — the shape is a presentation choice, not a different agent.
- The `per_chunk_delay` is the whole reason you'll see streaming beat sync on *perceived* latency. Total wall-clock is the same for both; only the first-byte time differs.

## Step 2 — Three endpoints, one app

Save as `main.py`:

```python
import asyncio
import uuid

from fastapi import BackgroundTasks, FastAPI, HTTPException
from fastapi.responses import StreamingResponse
from pydantic import BaseModel

from mock_agent import run_agent, run_agent_streaming


app = FastAPI()

JOBS: dict[str, dict] = {}  # Chapter 1: in-process only, lost on restart.


class AgentRequest(BaseModel):
    prompt: str


@app.post("/agent/run")
async def run(req: AgentRequest) -> dict:
    answer = await run_agent(req.prompt)
    return {"answer": answer}


@app.post("/agent/stream")
async def stream(req: AgentRequest) -> StreamingResponse:
    async def events():
        async for chunk in run_agent_streaming(req.prompt):
            yield f"data: {chunk}\n\n"
    return StreamingResponse(events(), media_type="text/event-stream")


@app.post("/agent/jobs")
async def start_job(req: AgentRequest, bg: BackgroundTasks) -> dict:
    job_id = uuid.uuid4().hex[:8]
    JOBS[job_id] = {"status": "running", "answer": None}

    async def _work():
        answer = await run_agent(req.prompt)
        JOBS[job_id] = {"status": "done", "answer": answer}

    bg.add_task(_work)
    return {"job_id": job_id, "status": "running"}


@app.get("/agent/jobs/{job_id}")
async def job_status(job_id: str) -> dict:
    job = JOBS.get(job_id)
    if job is None:
        raise HTTPException(404, "unknown job_id")
    return {"job_id": job_id, **job}


@app.get("/healthz")
async def healthz() -> dict:
    return {"ok": True}
```

Three things to notice before you start the server:

- `JOBS` is a module-level dict. Chapter 1: *"BackgroundTasks runs in the same process... it dies if the process restarts."* This dict is that in-process store — Step 5 proves what "dies" means.
- Validation is at the boundary: `AgentRequest` rejects a request with the wrong shape before it reaches the agent. A hand-rolled dict access would let garbage flow into the prompt.
- No concurrency cap, no timeout, no cancellation. Exercise 01 adds those. Keeping them out of the lab keeps the three *shapes* in focus.

## Step 3 — Run it and hit the sync endpoint

Start Uvicorn:

```bash
uvicorn main:app --port 8000
```

In another terminal:

```bash
time curl -sS -X POST http://127.0.0.1:8000/agent/run \
     -H 'content-type: application/json' \
     -d '{"prompt": "what is the answer?"}'
```

**Expected output:**

```
{"answer":"Reading the question. Thinking. Answer: 42."}
real    0m1.5xxs
```

The client waited the full ~1.5 s (10 chunks × 0.15 s) and then got the whole answer. Simple, but on a real multi-minute agent run this would hit timeouts.

## Step 4 — Hit the streaming endpoint and watch chunks arrive

```bash
curl -N -sS -X POST http://127.0.0.1:8000/agent/stream \
     -H 'content-type: application/json' \
     -d '{"prompt": "what is the answer?"}'
```

The `-N` flag tells curl not to buffer. You should see the SSE frames arrive incrementally, roughly one every 150 ms:

```
data: Reading

data:  the

data:  question
...
data: : 42.
```

Total wall-clock is the same as the sync call. What changed is **perceived latency**: the first `data:` line arrives ~150 ms after you hit enter, not 1.5 s later. Chapter 1: *"Same total time, far better perceived latency, and the connection stays warm."*

> If you see one big blob at the very end instead of incremental frames, a proxy or client in front of your request is buffering. `curl -N` directly against `127.0.0.1` is the simplest way to rule that out.

## Step 5 — Hit the background endpoint and poll

```bash
JOB=$(curl -sS -X POST http://127.0.0.1:8000/agent/jobs \
      -H 'content-type: application/json' \
      -d '{"prompt": "what is the answer?"}' | python -c "import sys, json; print(json.load(sys.stdin)['job_id'])")
echo "job_id=$JOB"

curl -sS "http://127.0.0.1:8000/agent/jobs/$JOB"
# ...wait a moment, then:
sleep 2
curl -sS "http://127.0.0.1:8000/agent/jobs/$JOB"
```

**Expected output:**

```
job_id=a1b2c3d4
{"job_id":"a1b2c3d4","status":"running","answer":null}
{"job_id":"a1b2c3d4","status":"done","answer":"Reading the question. Thinking. Answer: 42."}
```

The `POST` returned instantly — the client didn't wait the full 1.5 s. The job ran *after* the request completed, inside the same Uvicorn process, via `BackgroundTasks`. Polling flipped `running` → `done`.

## Step 6 — Feel the `BackgroundTasks` footgun

This is the move Chapter 1 is explicit about: *"`BackgroundTasks` runs in the same process. It's fine for a short-lived async task, but it dies if the process restarts."* Prove it.

Edit `mock_agent.py` and slow the agent down by raising `per_chunk_delay`:

```python
async def run_agent_streaming(prompt: str, per_chunk_delay: float = 2.0) -> AsyncIterator[str]:
```

Restart Uvicorn. Then start a job and immediately kill the server before it finishes:

```bash
JOB=$(curl -sS -X POST http://127.0.0.1:8000/agent/jobs \
      -H 'content-type: application/json' \
      -d '{"prompt": "a long one"}' | python -c "import sys, json; print(json.load(sys.stdin)['job_id'])")
echo "job_id=$JOB"
```

In the Uvicorn terminal, press `Ctrl+C` while the job is still `running`. Then restart Uvicorn (`uvicorn main:app --port 8000`) and poll:

```bash
curl -sS "http://127.0.0.1:8000/agent/jobs/$JOB"
```

**Expected output:**

```
{"detail":"unknown job_id"}
```

The job is **gone**. The `JOBS` dict lived in the dead process; the new process has no record of it. The user who kicked it off has no answer and no way to recover it. Chapter 1: *"For anything you can't afford to lose, you want a real job store or a durable workflow."* That real durable workflow is what [Chapter 2](../02-durable-execution.md) and [Exercise 02](../exercises/exercise-02-durable-execution-temporal.md) build.

Revert `per_chunk_delay` to `0.15` when you're done.

## Step 7 — Validation at the boundary

Chapter 1: *"Pydantic models reject malformed input before it reaches the agent. Never pass an unvalidated request body into a prompt."* Feel it:

```bash
curl -sS -X POST http://127.0.0.1:8000/agent/run \
     -H 'content-type: application/json' \
     -d '{"question": "oops, wrong key"}'
```

**Expected output:**

```
{"detail":[{"type":"missing","loc":["body","prompt"],"msg":"Field required", ...}]}
```

Uvicorn returned `422 Unprocessable Entity` without ever calling the agent. In an un-validated handler that typo would silently become `prompt = ""` and you'd spend tokens on an empty run.

## Step 8 — Checkpoint

Without peeking, you can state:

1. What total wall-clock time and first-byte time look like for sync vs. streaming on the same underlying agent generator — and which one changes when you add `per_chunk_delay`.
2. The exact sequence of calls (POST + 1..N polls) that makes a client talk to the background endpoint, and the one line of state that the restart in Step 6 wiped out.
3. Why `BackgroundTasks` **cannot** be the final answer for "a run you can't afford to lose" — in one sentence that would fit in `NOTES.md` for Exercise 01.

## Wrap-up

Carry the three endpoint handlers from `main.py` into [Exercise 01](../exercises/exercise-01-agent-api-deployment.md) and swap `mock_agent.py` for your real agent loop from [mod-201](../../mod-201-agent-fundamentals/README.md). What's left for the exercise:

- A server-side timeout on `/agent/run` and client-disconnect cancellation on `/agent/stream`.
- A concurrency semaphore so a traffic spike can't exhaust your rate limit.
- A `/healthz` that verifies the model client, not just returns `{"ok": True}`.
- A `Dockerfile` that pins the base image, caches the dependency layer, runs as non-root, and reads the API key from `-e`.

The three endpoint **shapes** themselves — and the feeling of which one to reach for — you already have.

### Stretch

- Add client-disconnect cancellation to `/agent/stream`: wrap the generator so `asyncio.CancelledError` stops producing chunks, and prove by `Ctrl+C`-ing a slow stream that no further chunks render server-side.
- Replace the in-process `JOBS` dict with a SQLite file so `/agent/jobs` survives a restart — but *don't* think this makes it durable: an activity interrupted mid-run still didn't complete. That's the distinction [Chapter 2](../02-durable-execution.md) opens with.
- Run Uvicorn under Gunicorn with 2+ workers and watch what happens to `/agent/jobs/{id}` polling when the job was created by the *other* worker — another argument for a shared store that isn't module-level memory.
