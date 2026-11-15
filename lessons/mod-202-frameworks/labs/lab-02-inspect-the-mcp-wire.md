# Lab 2 — Inspect the MCP Wire: initialize, discover, call, break

**Duration:** ~45 minutes
**Covers:** [Chapter 7 — MCP: Standardized Tool Use](../07-mcp-standardized-tool-use.md)

## Objective

Chapter 7 lays out the MCP lifecycle — `initialize` handshake, `tools/list` discovery, `tools/call` execution — and warns about one specific footgun: in the stdio transport, stdout carries the protocol, so any stray `print(...)` on the server's stdout corrupts the wire.

This lab puts you one layer below the tutorial. You will:

- Stand up the minimal `FastMCP` server from Chapter 7.
- Drive it from a raw `ClientSession` and inspect what discovery actually returns (names, descriptions, JSON Schemas).
- Deliberately break the stdout/stderr contract and watch the client fail; then fix it.
- Treat a tool argument as hostile and return a structured error instead of raising.

[Exercise 04](../exercises/exercise-04-mcp-tool-server.md) is where you wire the same server into LangGraph / CrewAI / an SDK and prove portability. This lab stays one process pair deep — the point is the *protocol*, not the framework glue.

## Prerequisites

- Python 3.11+ and a fresh virtualenv.
- Chapter 7 read.
- No provider API key needed; every call stays local.

## Setup

```bash
python -m venv .venv && source .venv/bin/activate
pip install "mcp[cli]"
mkdir lab-02 && cd lab-02
```

The `mcp` package provides both `FastMCP` (server) and the `ClientSession` / `stdio_client` helpers (client).

## Step 1 — A minimal server

Save as `server.py`:

```python
import ast
import operator as op
import sys

from mcp.server.fastmcp import FastMCP

mcp = FastMCP("lab-search-calc")

_OPS = {ast.Add: op.add, ast.Sub: op.sub, ast.Mult: op.mul,
        ast.Div: op.truediv, ast.USub: op.neg}


def _safe_eval(node):
    if isinstance(node, ast.Expression):
        return _safe_eval(node.body)
    if isinstance(node, ast.Constant) and isinstance(node.value, (int, float)):
        return node.value
    if isinstance(node, ast.BinOp) and type(node.op) in _OPS:
        return _OPS[type(node.op)](_safe_eval(node.left), _safe_eval(node.right))
    if isinstance(node, ast.UnaryOp) and type(node.op) in _OPS:
        return _OPS[type(node.op)](_safe_eval(node.operand))
    raise ValueError("disallowed expression")


@mcp.tool()
def search(query: str) -> list[dict]:
    """Search a tiny fixed corpus for the query. Returns up to 5 results."""
    corpus = [
        {"title": "MCP specification",     "url": "https://modelcontextprotocol.io"},
        {"title": "ReAct paper",           "url": "https://arxiv.org/abs/2210.03629"},
        {"title": "LangGraph docs",        "url": "https://langchain-ai.github.io/langgraph/"},
        {"title": "CrewAI docs",           "url": "https://docs.crewai.com/"},
        {"title": "AutoGen docs",          "url": "https://microsoft.github.io/autogen/"},
    ]
    q = query.lower()
    return [r for r in corpus if q in r["title"].lower()][:5]


@mcp.tool()
def calc(expression: str) -> float:
    """Evaluate a basic arithmetic expression. Supports + - * / and parentheses."""
    return _safe_eval(ast.parse(expression, mode="eval"))


if __name__ == "__main__":
    # Every log line goes to stderr; stdout is reserved for the JSON-RPC protocol.
    print("lab-search-calc starting", file=sys.stderr)
    mcp.run(transport="stdio")
```

You can leave this file alone and run it by hand to confirm it imports:

```bash
python -c "import server; print(server.mcp.name)"
```

**Expected output:** `lab-search-calc`. If this blows up, fix the import before you touch the client.

## Step 2 — A raw client that drives the lifecycle

Save as `client.py`:

```python
import asyncio
import json

from mcp import ClientSession, StdioServerParameters
from mcp.client.stdio import stdio_client


def _dump(label, obj):
    """Render a protocol object as pretty JSON (best-effort)."""
    try:
        data = obj.model_dump()
    except AttributeError:
        data = obj
    print(f"--- {label} ---")
    print(json.dumps(data, indent=2, default=str))


async def main() -> None:
    params = StdioServerParameters(command="python", args=["server.py"])
    async with stdio_client(params) as (read, write):
        async with ClientSession(read, write) as session:
            init = await session.initialize()
            _dump("initialize result", init)

            listed = await session.list_tools()
            _dump("tools/list", listed)

            r1 = await session.call_tool("search", {"query": "MCP"})
            _dump("tools/call search(query='MCP')", r1)

            r2 = await session.call_tool("calc", {"expression": "(2 + 3) * 4"})
            _dump("tools/call calc(expression='(2 + 3) * 4')", r2)


if __name__ == "__main__":
    asyncio.run(main())
```

Run it:

```bash
python client.py
```

**Expected output (shape, not exact bytes):**

- An `initialize result` block that includes a `protocolVersion`, server `serverInfo` (name `"lab-search-calc"`), and a `capabilities` object with at least `tools`.
- A `tools/list` block showing two entries — `search` and `calc` — each with a `description` matching the docstring and an `inputSchema` (JSON Schema) listing the single string parameter.
- A `tools/call search` block whose `content` carries the five-entry corpus filtered to titles that contain `"MCP"` (one or two hits depending on the corpus).
- A `tools/call calc` block whose `content` carries `20` (as a string or structured value, depending on SDK version).

**Why this matters:** the client did not hard-code either tool's schema. It called `list_tools` and got them from the server at runtime. That is the whole point of MCP's discovery lifecycle — Chapter 7 steps 1 and 2 (initialize + discovery) in the flesh.

**Checkpoint:** point at the exact line in the output where the model would have been told, by the host, that `calc` takes an `expression: string`. That is the schema the LLM will route against.

## Step 3 — Break the stdout contract on purpose

Edit `server.py`. Inside the `calc` body, add a stray `print` — note it goes to **stdout** (the default):

```python
@mcp.tool()
def calc(expression: str) -> float:
    """Evaluate a basic arithmetic expression. Supports + - * / and parentheses."""
    print(f"[DEBUG] calc got {expression!r}")   # <-- stdout; wrong for stdio transport
    return _safe_eval(ast.parse(expression, mode="eval"))
```

Rerun:

```bash
python client.py
```

**Expected result:** the client's `call_tool("calc", ...)` fails. Depending on SDK version you will see either a JSON parse error, a connection-closed error, or a hang followed by a `ClosedResourceError`. The `initialize` and `list_tools` results above *still* succeed, because the `calc` tool is what triggered the stray `print`.

Pause here and re-read Chapter 7's stdio paragraph:

> *in stdio transport, stdout carries length-prefixed JSON-RPC frames. A stray `print(...)` to stdout corrupts the wire protocol and the client disconnects with a parse error.*

That is exactly what you just caused. Fix it by routing the debug line to stderr instead:

```python
print(f"[DEBUG] calc got {expression!r}", file=sys.stderr)
```

Rerun. `client.py` is green again and your `[DEBUG]` lines appear on the terminal (because `stdio_client` forwards the subprocess's stderr by default).

**Checkpoint:** you can explain, without looking, why the exact same server code works fine as an in-process Python import but kills the client when spawned over stdio — and you know which stream to log to.

## Step 4 — Treat tool arguments as hostile

The current `calc` raises `ValueError("disallowed expression")` on anything it does not understand. Chapter 7 (and Chapter 4 of mod-201 before it) says tools should return **errors as tool results**, not raise raw exceptions that the client has to interpret.

Replace `calc` in `server.py` with the structured-error version:

```python
@mcp.tool()
def calc(expression: str) -> dict:
    """Evaluate a basic arithmetic expression. Supports + - * / and parentheses.

    Returns {"ok": True, "value": float} on success or
    {"ok": False, "error": "..."} on a disallowed expression.
    """
    try:
        return {"ok": True, "value": _safe_eval(ast.parse(expression, mode="eval"))}
    except Exception as exc:
        return {"ok": False, "error": str(exc)}
```

Then add this adversarial probe to `client.py` after the existing `calc` call:

```python
            r3 = await session.call_tool("calc", {"expression": "__import__('os').system('echo pwned')"})
            _dump("tools/call calc(expression=<malicious>)", r3)

            r4 = await session.call_tool("calc", {"expression": "1 + 2"})
            _dump("tools/call calc(expression='1 + 2') after rejection", r4)
```

Rerun:

```bash
python client.py
```

**Expected output:**

- `r3`'s `content` is a structured `{"ok": false, "error": "..."}` object. `_safe_eval` refuses because `Call` / `Attribute` / `Name` nodes are not in its whitelist, so the AST walker never touches `os.system`.
- `r4` succeeds with `{"ok": true, "value": 3}`. Crucially, the server kept running — one bad argument did not take the process down.

**Why this matters:** Chapter 7's security advice — *treat every argument coming in over the wire as adversarial* — is only useful if your server hardens the boundary. Raising lets the error leak as a protocol-level fault; returning a structured error lets the agent read it as an observation and re-plan.

## Step 5 — One server, two clients (optional stretch)

Leave `server.py` running the same way. Write a **second** client, `client_minimal.py`, that only calls `calc("2*21")` and exits. Run both back-to-back:

```bash
python client.py
python client_minimal.py
```

Each client spawned the server as a subprocess, ran the lifecycle, and exited — the server file did not need a single edit. That is the Chapter 7 payoff Exercise 04 will later cash in across *framework* hosts: swap the host, keep the server.

## Wrap-up

You have now, by hand, exercised every piece of the MCP lifecycle Chapter 7 names:

- `initialize` is a handshake that negotiates protocol version and capabilities.
- `tools/list` returns a server-advertised catalog of names, descriptions, and JSON Schemas — the host does not hard-code them.
- `tools/call` is the execution RPC; its content is structured, including structured errors.
- stdio transport uses stdout for the protocol; every log line *must* go to stderr. This is the number one footgun and now you have felt it.
- Hardening the tool against adversarial arguments at the server boundary is what makes the process-level isolation Chapter 7 touts actually load-bearing.

[Exercise 04](../exercises/exercise-04-mcp-tool-server.md) is where the same server stops talking to a raw client and starts talking to a LangGraph (or CrewAI, or Claude Agent SDK) host. Everything you learned about discovery, schemas, and structured errors still applies — the host swap is mechanical.
