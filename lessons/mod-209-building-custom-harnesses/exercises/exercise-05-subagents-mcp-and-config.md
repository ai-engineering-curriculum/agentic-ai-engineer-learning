# exercise-05: Subagents, MCP Tools, and Declarative Config

**Estimated effort:** 4 hours

## Objective

Make your harness extensible. Add a subagent tool that runs a nested harness with a fresh context and a restricted tool set and returns only a summary. Connect tools from any MCP server into your registry. Load agents, permissions, and hooks from a config file instead of Python code. By the end, other people can extend your harness without editing its source, which is what separates a harness from a script.

## Background

This exercise covers material from:

- [Chapter 7 — Subagents, Skills, and MCP](../07-subagents-skills-mcp.md)

Build on your [exercise-01](exercise-01-build-a-minimal-harness.md) harness or its reference solution.

## Prerequisites

- A working exercise-01 harness.
- Python 3.11+, `pytest`, and the official MCP Python SDK (`pip install mcp`). The starter code targets SDK 2.x; check the SDK's README for the version you install, since names changed between 1.x and 2.x.

## Tasks

### 1. Agent definitions

- An agent definition has `name`, `description`, `system_prompt`, `tools` (an allowlist of tool names), `max_turns`, and optionally `model`.
- Load definitions from Markdown files with frontmatter in `<workspace>/.harness/agents/*.md`: frontmatter for the fields, body for the system prompt. Validate them and report errors with the file name.

### 2. The `task` tool (subagents)

- Input: `agent` (a definition name) and `prompt`.
- Build a child harness with the agent's system prompt, a **new, empty history**, and a registry filtered to the agent's allowlist. A child must never get the `task` tool itself unless the depth limit allows it; enforce a maximum depth (default 1).
- The child inherits the parent's permission policy and hooks. A subagent can't do anything the parent couldn't.
- The child's token usage counts against the parent's budget. Stop the child when the remaining budget runs out.
- Return only the child's final text to the parent, plus a one-line footer (`[subagent explorer: 7 turns, 18,402 tokens, stop=done]`). If the child stopped for any reason other than `done`, say so clearly.
- Allow several `task` calls in one turn to run concurrently.

### 3. MCP client adapter

- Connect to an MCP server over stdio using the MCP Python SDK, list its tools, and register each as a harness tool named `mcp__<server>__<tool>`, with the server's description and input schema.
- Calling the tool forwards to the server and converts the result (text content blocks) into a tool result. MCP errors (`isError` in the protocol, `is_error` in the 2.x SDK) become error results.
- Manage the server's lifetime: start on harness startup, shut down cleanly on exit, and report a server that fails to start without crashing the harness.
- MCP tools go through the same permission gate and hooks as built-in tools. Default policy for MCP tools is `ask`.
- Write a tiny MCP server for the tests (two tools, using the SDK's high-level server class: `MCPServer` in 2.x, `FastMCP` in 1.x) and run the adapter against it.

### 4. Declarative config

- Load `.harness/settings.json` with this shape, and validate it with clear errors:

```json
{
  "permissions": {
    "allow": ["read_file", "glob", "grep", "bash(pytest*)", "bash(git status)"],
    "deny": ["bash(git push*)", "write_file(.env*)"],
    "default": "ask"
  },
  "hooks": {
    "pre_tool": [{ "match": "bash|write_file|edit_file", "command": "python3 .harness/hooks/guard.py" }],
    "post_tool": [{ "match": "write_file|edit_file", "command": "python3 .harness/hooks/lint.py" }]
  },
  "mcp_servers": {
    "notes": { "command": "python3", "args": ["tests/fixtures/notes_server.py"] }
  }
}
```

- Permission rules: `tool` matches any call to that tool; `tool(pattern)` matches when the tool's main argument (command or path) matches the glob. Deny beats allow; then allow; then `default`.
- Command hooks use the same contract as exercise-02 in mod-208: JSON event on stdin, exit `0` allow, exit `2` block with stderr as the reason. Wrap them as pre- and post-tool hooks in your harness.
- Merge a user-level settings file under the project one: project rules first, user rules after.

## Starter guidance

```python
from mcp import ClientSession, StdioServerParameters
from mcp.client.stdio import stdio_client

async def list_server_tools(command: str, args: list[str]) -> list[dict]:
    params = StdioServerParameters(command=command, args=args)
    async with stdio_client(params) as (read, write):
        async with ClientSession(read, write) as session:
            await session.initialize()
            result = await session.list_tools()
            return [{"name": t.name, "description": t.description, "schema": t.input_schema} for t in result.tools]
```

SDK 2.x uses snake_case fields (`input_schema`, `is_error`); 1.x used camelCase (`inputSchema`, `isError`).

Your harness loop is synchronous. The MCP SDK is async. Decide how to bridge them: a background event loop thread that owns the sessions is one clean option. Explain your choice in `NOTES.md`.

## Acceptance criteria

You can demonstrate that, in `pytest`:

- Agent definitions load from Markdown, and a malformed one is reported with its file name.
- A subagent starts with an empty history, sees only its allowlisted tools, returns only its final text plus the footer, and counts against the parent's budget.
- The depth limit stops a subagent from spawning another.
- A deny rule from the parent's policy also blocks the same call inside a subagent.
- Tools from the test MCP server appear as `mcp__notes__<tool>`, can be called, pass through the permission gate, and turn MCP errors into error results.
- A server that fails to start is reported and the harness still runs with its other tools.
- Config validation rejects unknown keys and malformed rules with messages naming the problem; deny-beats-allow holds.
- A command hook from config blocks a call with exit `2` and the model sees its stderr.

## Reflection

In `NOTES.md`:

1. What does the parent lose by getting only the subagent's summary? When is that a good trade?
2. Why should subagents inherit the parent's permissions rather than having their own?
3. An MCP server's tool descriptions go into your model's context. What could a malicious server do with that, and how does your harness limit it?

## Stretch goals

- Background subagents: `task` returns a handle immediately, and a `task_result` tool collects the output later.
- MCP over HTTP (streamable HTTP transport) in addition to stdio.
- Hot-reload `settings.json` between turns, and log which rules changed.
