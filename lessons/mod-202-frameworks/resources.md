# Resources for mod-202-frameworks

Primary references for the frameworks, SDKs, and protocols in this module. Agent tooling moves fast — verify class names and import paths against the current docs before you cite them in a design review.

## Patterns and background

- **Anthropic — Building effective agents** ([anthropic.com/engineering/building-effective-agents](https://www.anthropic.com/engineering/building-effective-agents)) — the pattern taxonomy every framework packages: prompt chaining, routing, orchestrator-workers, evaluator-optimizer.
- **ReAct: Synergizing Reasoning and Acting in Language Models** ([arxiv.org/abs/2210.03629](https://arxiv.org/abs/2210.03629)) — the reason-act loop every framework's tool loop implements.

## LangGraph

- **LangGraph documentation** ([langchain-ai.github.io/langgraph](https://langchain-ai.github.io/langgraph/)) — `StateGraph`, reducers, conditional edges, checkpointers, and `create_react_agent`.
- **LangGraph repository** ([github.com/langchain-ai/langgraph](https://github.com/langchain-ai/langgraph)) — source, the prebuilt `ToolNode`, and the checkpoint backend packages.
- **LangSmith tracing** ([docs.smith.langchain.com](https://docs.smith.langchain.com/)) — the best-integrated observability for LangGraph runs.

## CrewAI

- **CrewAI documentation** ([docs.crewai.com](https://docs.crewai.com/)) — agents, tasks, `Process`, crews, memory, and Flows.
- **CrewAI repository** ([github.com/crewAIInc/crewAI](https://github.com/crewAIInc/crewAI)) — source and release notes.
- **crewai-tools** ([github.com/crewAIInc/crewAI-tools](https://github.com/crewAIInc/crewAI-tools)) — prebuilt tool integrations and the MCP adapter.

## AutoGen

- **AutoGen documentation** ([microsoft.github.io/autogen](https://microsoft.github.io/autogen/)) — the current (v0.4) `autogen-core` / `autogen-agentchat` stack and the v0.2 migration notes.
- **AutoGen repository** ([github.com/microsoft/autogen](https://github.com/microsoft/autogen)) — source, examples, and the version split discussion.

## Competing SDKs

- **OpenAI Agents SDK** ([github.com/openai/openai-agents-python](https://github.com/openai/openai-agents-python), [openai.github.io/openai-agents-python](https://openai.github.io/openai-agents-python/)) — agents, handoffs, guardrails, sessions, and built-in tracing.
- **Google Agent Development Kit (ADK)** ([github.com/google/adk-python](https://github.com/google/adk-python), [google.github.io/adk-docs](https://google.github.io/adk-docs/)) — planners, built-in code execution, eval tooling, and Vertex AI Agent Engine deployment.
- **Anthropic Claude Agent SDK** ([github.com/anthropics/claude-agent-sdk-python](https://github.com/anthropics/claude-agent-sdk-python), [docs.claude.com](https://docs.claude.com/en/api/agent-sdk/overview)) — sub-agents, file-system context, hooks, and first-class MCP.
- **Hugging Face smolagents** ([github.com/huggingface/smolagents](https://github.com/huggingface/smolagents), [huggingface.co/docs/smolagents](https://huggingface.co/docs/smolagents/index)) — `ToolCallingAgent`, `CodeAgent`, and code-as-action with sandboxed execution.

## Model Context Protocol (MCP)

- **MCP specification and docs** ([modelcontextprotocol.io](https://modelcontextprotocol.io)) — architecture, primitives (tools, resources, prompts), lifecycle, transports, and authorization.
- **MCP Python SDK** ([github.com/modelcontextprotocol/python-sdk](https://github.com/modelcontextprotocol/python-sdk)) — `FastMCP`, the low-level `Server`, and the stdio/HTTP client helpers.
- **MCP TypeScript SDK** ([github.com/modelcontextprotocol/typescript-sdk](https://github.com/modelcontextprotocol/typescript-sdk)) — the reference implementation for language-agnostic servers.
- **langchain-mcp-adapters** ([github.com/langchain-ai/langchain-mcp-adapters](https://github.com/langchain-ai/langchain-mcp-adapters)) — converts MCP tools into LangChain/LangGraph tools.

## Cross-references

- [mod-204: Multi-Agent Implementation](../mod-204-multi-agent-implementation/README.md) — the hand-built versions of the patterns these frameworks package.
