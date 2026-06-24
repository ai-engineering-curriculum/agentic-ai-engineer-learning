# exercise-04: MCP Tool Server

**Estimated effort:** 3 hours

## Objective

Factor the `search` and `calc` tools out of your agent code and into a standalone MCP server, then call that one server from two different hosts — a raw MCP client and at least one of your framework builds from exercises 01–03. By the end you'll have proven the central MCP claim: write the tool once, run it everywhere, change the agent without changing the tool.

## Background

This exercise covers material from:

- [Chapter 7 — MCP: Standardized Tool Use](../07-mcp-standardized-tool-use.md)
- [Chapter 6 — Evaluating Competing SDKs](../06-evaluating-competing-sdks.md)

You'll reuse the same `search` and `calc` logic from [exercise-01](exercise-01-langgraph-stateful-agent.md), but this time behind a JSON-RPC boundary instead of an in-process import.

## Prerequisites

- The MCP Python SDK installed (`mcp`, providing `FastMCP` and the stdio client helpers).
- At least one completed framework build from [exercise-01](exercise-01-langgraph-stateful-agent.md), [exercise-02](exercise-02-crewai-role-based-crew.md), or [exercise-03](exercise-03-autogen-multi-agent.md), plus its MCP adapter package.
- A provider API key for the framework host; a small spend cap.

## Tasks

### 1. Build the server

- Create `search_calc_server.py` using `FastMCP`.
- Expose `search` and `calc` as `@mcp.tool()` functions with precise docstrings (the docstring becomes the tool description the model sees).
- Use a safe `ast`-based evaluator for `calc` — never `eval`.
- Route **all** logging to `stderr`; stdout carries the protocol over stdio.
- Run with `mcp.run(transport="stdio")`.

### 2. Call it from a raw MCP client

- Write `client.py` that spawns the server over stdio, runs the `initialize` handshake, calls `list_tools`, and invokes both tools.
- Confirm discovery works: the client learns the tool names and schemas without hard-coding them.

### 3. Call the same server from a framework host

- Wire the server into one framework build using its MCP adapter (for example `langchain-mcp-adapters` for LangGraph, the OpenAI Agents SDK `MCPServerStdio`, or the CrewAI MCP integration).
- Run your population-ratio task end to end with the tools coming from the server, not from in-process functions.

### 4. Prove portability

- Connect a **second** host to the **unchanged** server binary. Show that swapping the host does not require touching `search_calc_server.py`.

### 5. Add an adversarial-input guard

- Treat every tool argument as hostile: validate the `calc` expression against the whitelist and return a structured error (not an exception) for anything outside it.
- Show a malicious input (e.g. an attempt to call a disallowed function) being rejected cleanly without crashing the server.

## Starter guidance

```python
# search_calc_server.py
import ast
import operator as op
import sys

from mcp.server.fastmcp import FastMCP

mcp = FastMCP("search-calc")

_OPS = {ast.Add: op.add, ast.Sub: op.sub, ast.Mult: op.mul,
        ast.Div: op.truediv, ast.USub: op.neg}


@mcp.tool()
def search(query: str) -> list[dict]:
    """Search the local index for the query. Returns up to 5 results."""
    raise NotImplementedError  # deterministic fixed corpus for offline runs


@mcp.tool()
def calc(expression: str) -> float:
    """Evaluate a basic arithmetic expression. Supports + - * / and parentheses."""
    def _eval(node):
        raise NotImplementedError  # recurse over whitelisted ast nodes only
    return _eval(ast.parse(expression, mode="eval").body)


if __name__ == "__main__":
    print("search-calc server starting", file=sys.stderr)
    mcp.run(transport="stdio")
```

```python
# client.py
import asyncio

from mcp import ClientSession, StdioServerParameters
from mcp.client.stdio import stdio_client


async def main() -> None:
    params = StdioServerParameters(command="python", args=["search_calc_server.py"])
    async with stdio_client(params) as (read, write):
        async with ClientSession(read, write) as session:
            await session.initialize()
            tools = await session.list_tools()
            print("tools:", [t.name for t in tools.tools])
            raise NotImplementedError  # call_tool("search", ...) and call_tool("calc", ...)


if __name__ == "__main__":
    asyncio.run(main())
```

## Acceptance criteria

You can demonstrate that:

- The raw client discovers the tools via `list_tools` and invokes both successfully over stdio.
- A framework host runs the population-ratio task with tools served from the MCP server, not imported in-process.
- A second host runs against the same server binary with zero changes to the server file.
- A disallowed `calc` expression returns a structured error and the server keeps running.
- No stray `print` to stdout corrupts the protocol; all logs go to stderr.

## Reflection

In `NOTES.md`:

1. What did you have to change in each host to consume the MCP server versus an in-process tool? What stayed identical?
2. Measure one tool call inline versus over MCP. Where would that latency overhead make MCP the wrong choice?
3. Which security control mattered most here — input validation, the process boundary, or restricting the server's filesystem/network — and why?

## Stretch goals

- Add an MCP **resource** (a read-only URI, e.g. a cached country-population table) and have the host pull it in as context.
- Add an MCP **prompt** template that documents the right way to call `calc`.
- Containerize the server with a read-only filesystem and no network, and run the agent against the containerized server.
- Rewrite `search` as a non-Python binary (any language) behind the same MCP interface and confirm the Python host is unaffected.
