# Labs for mod-202-frameworks (Agent Frameworks in Practice)

Labs are short, guided, observational sessions — distinct from the [exercises](../exercises/), which are longer, open-ended builds. Each lab targets one chapter and complements (does not duplicate) the exercises.

- [Lab 1 — LangGraph: Stream, Inspect State, Resume](lab-01-langgraph-stream-state-resume.md) — ~45 min. Chapter 2 runtime in depth: `stream_mode="values"` vs. `"updates"`, `get_state`, `thread_id`-keyed checkpoint resume, and the `recursion_limit` bounded-failure. Uses the prebuilt `create_react_agent`; [exercise-01](../exercises/exercise-01-langgraph-stateful-agent.md) is where you drop down to a custom `StateGraph`.
- [Lab 2 — Inspect the MCP Wire: initialize, discover, call, break](lab-02-inspect-the-mcp-wire.md) — ~45 min. Chapter 7 protocol up close: drive a `FastMCP` server with a raw `ClientSession`, read what discovery returns, trigger the stdout/stderr footgun on purpose, and harden a tool against adversarial arguments. [Exercise-04](../exercises/exercise-04-mcp-tool-server.md) is where you then wire the same server into a framework host.
