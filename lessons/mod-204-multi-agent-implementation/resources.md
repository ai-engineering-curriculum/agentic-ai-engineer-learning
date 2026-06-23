# Resources for mod-204-multi-agent-implementation

Primary references for multi-agent implementation. Verify against current docs — agent tooling moves fast.

## Patterns and principles

- **Anthropic — Building effective agents** ([anthropic.com/engineering/building-effective-agents](https://www.anthropic.com/engineering/building-effective-agents)) — the canonical taxonomy: orchestrator-workers, routing, evaluator-optimizer, prompt chaining. Start here.
- **Anthropic — How we built our multi-agent research system** ([anthropic.com/engineering/multi-agent-research-system](https://www.anthropic.com/engineering/multi-agent-research-system)) — a real orchestrator-worker system, including the context-economics and failure modes.

## Protocols

- **Model Context Protocol** ([modelcontextprotocol.io](https://modelcontextprotocol.io)) — the spec for connecting agents to shared tools and resources. See also the [MCP guide in the AI Agent Guidebook](https://github.com/ai-infra-curriculum/ai-agent-guidebook/tree/main/guides/mcp-servers).
- **Agent2Agent (A2A) Protocol** ([a2a-protocol.org](https://a2a-protocol.org)) — an open protocol for agent-to-agent messaging. Treat the *pattern* (typed messages, intents, task correlation) as the lesson, independent of any one implementation.

## Frameworks (for comparison after you've built by hand)

- **OpenAI Agents SDK** — handoffs and routing as first-class primitives.
- **LangGraph** — graph-structured multi-agent orchestration with explicit state.
- **CrewAI / AutoGen** — role-based multi-agent crews.
- **Claude Agent SDK** — sub-agent isolation and tool scoping built in.

> You implemented these patterns from scratch in this module. When you adopt a framework, you'll recognize exactly which of these it's packaging — and you'll debug it faster for having built it yourself. See [mod-202: Agent Frameworks in Practice](../mod-202-frameworks/README.md).
