# Chapter 3 — Inter-Agent Communication: A2A and MCP

When agents coordinate, the hard part isn't the agents — it's the **interface between them**. Loose, free-text coupling (one agent dumps prose at another) is where multi-agent systems rot. This chapter is about giving agents explicit contracts: typed messages for talking to each other (A2A), and a shared protocol for using each other's tools (MCP).

## Why the interface is the hard part

A single agent's mistakes stay contained. In a multi-agent system, agent A's sloppy output becomes agent B's corrupted input, and the error compounds down the chain. Two disciplines prevent this:

1. **Type the messages.** An agent should hand off a structured object with known fields, not a paragraph the next agent has to re-parse.
2. **Validate at the boundary.** Treat another agent's output as untrusted input — validate it against a schema before acting on it, exactly as you'd validate a user's input.

## Agent-to-Agent (A2A) messaging

A2A is the pattern (and an emerging open protocol) for agents exchanging **typed messages** instead of raw text. The mechanics matter more than the brand: define a message contract and make every agent speak it.

```python
from pydantic import BaseModel
from typing import Literal

class AgentMessage(BaseModel):
    sender: str
    recipient: str
    intent: Literal["request", "result", "error"]
    task_id: str
    payload: dict          # schema depends on intent
    confidence: float | None = None
```

Design guidance:

- **Correlate with a `task_id`** so a fan-out's results can be matched back to their assignments.
- **Make `error` a first-class intent.** A worker that fails should return a structured error, not a hallucinated success. Downstream agents branch on `intent`, not on parsing prose.
- **Carry `confidence` when it's cheap.** It lets an orchestrator decide what to trust or double-check — but only use it if your agents calibrate it honestly; otherwise drop it.
- **Keep payloads small.** Pass references (an ID, a file path, a URL), not megabytes of inlined content, between agents — see the context economics in [Chapter 4](04-subagent-isolation.md).

## Sharing tools over MCP

The **Model Context Protocol (MCP)** standardizes how an agent connects to tools and data sources. In a multi-agent system it solves a specific problem: **giving several agents access to the same capability without re-implementing it in each.**

Instead of every agent shipping its own database client, you run one MCP server (database access, web search, a ticketing API) and point each agent's runtime at it:

```jsonc
// each agent runtime loads the shared servers it needs
{
  "mcpServers": {
    "warehouse": { "type": "http", "url": "https://mcp.internal/warehouse" },
    "search":    { "command": "npx", "args": ["-y", "search-mcp-server"] }
  }
}
```

Why this matters for multi-agent design:

- **One source of truth for a capability.** Fix a bug in the warehouse tool once; every agent gets it.
- **Per-agent tool scoping.** Give the `writer` agent read-only search and the `executor` agent the write tools. Same servers, different allowlists — least privilege across the fleet.
- **Context cost is real.** Each server's tool definitions consume the connecting agent's context window. Connect an agent only to the servers it needs; an agent wired to ten servers it never calls is paying for ten unused manuals every turn.

## A2A vs. MCP — they're orthogonal

- **MCP** = how an agent talks to **tools/resources** (vertical: agent → capability).
- **A2A** = how an agent talks to **other agents** (horizontal: agent ↔ agent).

A typical system uses both: agents coordinate over A2A-style typed messages, and each reaches shared capabilities over MCP.

## Key takeaways

- The interface, not the agent, is the failure point — **type your inter-agent messages and validate them at the boundary.**
- A2A: typed messages with `intent`, `task_id`, and a first-class `error` case; pass references, not payloads.
- MCP: share one implementation of a capability across agents, scope tools per agent (least privilege), and connect only to servers an agent actually uses.
