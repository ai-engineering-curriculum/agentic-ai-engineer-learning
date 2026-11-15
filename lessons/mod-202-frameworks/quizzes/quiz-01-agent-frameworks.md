# Quiz 1 — Agent Frameworks in Practice

Knowledge check for [mod-202](../README.md). Answers are at the bottom; try each question before scrolling. Covers all seven chapters.

## Questions

### 1. What a framework actually buys you

Your mod-201 ReAct loop runs one agent on one task in one process and the team is happy with it. Which of the following, on its own, is the **weakest** reason to port it into LangGraph / CrewAI / AutoGen now?

- A. You need the run to survive a container restart and resume where it stopped.
- B. A new feature requires human approval before the agent calls any side-effecting tool.
- C. You want to stream intermediate events to a UI that renders per-step progress.
- D. The 200-line loop is working but the engineer who wrote it finds the framework landing pages exciting.

### 2. The five parts

Chapter 1 argues that every framework in the module — once you strip the marketing — has the same five parts your hand-rolled loop already had. Which list below matches those five parts?

- A. Vector store, retriever, reranker, prompt template, output parser.
- B. Model abstraction, tool abstraction, loop, state object, termination rule.
- C. Planner, executor, critic, memory, scheduler.
- D. Router, supervisor, worker, observer, logger.

### 3. State slots and reducers

A LangGraph `AgentState` declares `messages: Annotated[list[BaseMessage], add_messages]` and `step: Annotated[int, add]`. Node `A` returns `{"step": 1}`; node `B` returns `{"step": 1}`; node `C` returns `{"messages": [new_msg]}`. After all three run on the same thread, the state is:

- A. `messages` has `new_msg` appended, `step == 2`.
- B. `messages` is replaced with just `[new_msg]`, `step == 1` (last writer wins).
- C. `messages` gets `new_msg` three times, `step == 3`.
- D. The second `{"step": 1}` return raises a reducer conflict error.

### 4. `recursion_limit` is a step budget

You port the mod-201 ReAct loop to LangGraph with `model` and `tools` nodes and a conditional edge back to `model`. In production you want to cap the agent at four tool calls before it must either answer or fail. Chapter 2 says `recursion_limit` counts **node transitions, not ReAct iterations**. Which `recursion_limit` value fits that cap?

- A. 4 — one per tool call.
- B. 8 — two transitions per ReAct iteration, so four iterations is eight transitions.
- C. 16 — Chapter 2's default is 16 and you should never change it.
- D. `None` — rely on `max_consecutive_auto_reply` on the tool node.

### 5. CrewAI backstories

A teammate ships a CrewAI crew where each agent carries a five-paragraph backstory describing their hobbies, their favorite demographer, and their preferred coffee order. Per Chapter 3, this is a problem because:

- A. CrewAI silently truncates any backstory longer than 500 characters, so most of the text never reaches the model.
- B. Long novellas cost tokens on every turn and dilute the routing signal the manager (and the agent itself) uses to pick tasks; the Chapter 3 advice is short, specific, and expertise-focused.
- C. Backstories must be in third person and the shipped ones use first person.
- D. `Agent.backstory` is not actually included in the system prompt and the whole field is purely decorative.

### 6. Task composition in CrewAI

You have two CrewAI tasks: `gather_populations` emits a JSON object with two countries and their populations; `compute_ratio` needs to divide one by the other. Per Chapter 3, the *idiomatic* way to make the ratio task read the research task's output is:

- A. Set a module-level global dict that both task functions read and write.
- B. Give `compute_ratio` a `context=[gather_populations]` list so CrewAI injects the upstream task's output into its prompt.
- C. Enable `allow_delegation=True` on the synthesizer so it can call the researcher as a sub-agent.
- D. Switch to `Process.hierarchical` so the manager LLM routes the JSON by hand.

### 7. Sequential vs. hierarchical processes

Chapter 3 defaults to `Process.sequential` and reaches for `Process.hierarchical` only when control flow has to be dynamic. The main cost argument against hierarchical is:

- A. The hierarchical process disables all tools on worker agents.
- B. Hierarchical processes cannot use the `@tool` decorator and require `BaseTool` subclasses.
- C. The manager is itself an LLM, so every routing decision is an extra model call with the full task list and agent roster in context — a measurable premium on every hop.
- D. `Process.hierarchical` is only available on paid CrewAI tiers.

### 8. AutoGen's two-sided tool registration

In AutoGen v0.2 you register a tool with `assistant.register_for_llm(...)` **and** `user_proxy.register_for_execution(...)`. Chapter 4 says this split exists because:

- A. The two decorators generate different JSON Schemas and the provider needs both.
- B. Tool *selection* and tool *execution* live on different agents in AutoGen's actor model — the assistant decides to call the tool, the proxy runs it — so each half must opt in independently.
- C. `register_for_llm` registers for OpenAI models and `register_for_execution` registers for Anthropic models.
- D. It's a legacy quirk with no runtime effect; either one alone is sufficient.

### 9. Termination in a group chat

Chapter 4 calls termination a first-class design step, not an afterthought. Which of the following is **not** a termination mechanism the chapter lists for AutoGen?

- A. A `max_round` / `MaxMessageTermination` hard cap on the number of messages.
- B. An `is_termination_msg` callable (v0.2) or `TextMentionTermination("TERMINATE")` (v0.4) that matches a sentinel in the last message.
- C. A `TokenUsageTermination` that fires when the running conversation exceeds a token budget.
- D. A `confidence_threshold` on the group chat that stops when the manager's log-probability on the next speaker exceeds a cutoff.

### 10. Where the complexity lives

Chapter 5's side-by-side gives the following summary: "LangGraph puts complexity in the graph; AutoGen puts it in the prompts." Which anti-pattern is the direct consequence of ignoring that advice when writing **AutoGen**?

- A. Making every node a persona with a backstory and a goal.
- B. Building a stateful machine inside each CrewAI agent with hand-rolled step counters.
- C. Encoding control flow in conversation rules — "if the planner has spoken twice, the executor must speak next" — instead of using a graph.
- D. Putting a 2 MB PDF into a LangGraph state slot.

### 11. SDKs vs. frameworks

Chapter 6 offers the heuristic **"SDKs own the loop, frameworks own the topology."** Which situation is the *clearest* fit for an SDK rather than a framework?

- A. A planner, three specialist workers, and a critic coordinating through a shared scratchpad with explicit hand-offs and checkpoints.
- B. One agent with a handful of tools, running a single ReAct loop and needing tracing and tidy handoffs between two related personas.
- C. A long-running workflow where a human approves each side-effecting tool call and the run must resume after a crash.
- D. A multi-agent debate where ten participants exchange arguments and a judge agent scores the transcript.

### 12. smolagents' `CodeAgent`

Chapter 6 singles out smolagents' `CodeAgent` as the canonical implementation of one particular idea. Which idea?

- A. A declarative YAML-based agent spec compiled to Python at startup.
- B. **Code-as-action** — the agent emits Python at each step, the SDK runs it in a sandboxed kernel, and calling a tool is just calling a Python function in that kernel.
- C. A visual node editor that lets non-engineers wire agents without code.
- D. On-device fine-tuning of a small model on the user's own trajectories.

### 13. The MCP stdout footgun

Your minimal `FastMCP` server works in isolation but the client disconnects with a parse error the moment you add a `print(f"got query: {query}")` inside one of your `@mcp.tool()` functions. Per Chapter 7, the root cause is:

- A. The client validates every tool against a cached schema and the new log line changed the tool's hash.
- B. In the stdio transport, stdout carries length-prefixed JSON-RPC frames; a stray `print(...)` to stdout corrupts the wire protocol — all logs must go to stderr.
- C. `FastMCP` buffers stdout and times out after 30 seconds, dropping the connection.
- D. Tool implementations are not allowed to call any I/O in the Python SDK.

### 14. When MCP is **not** the right answer

Chapter 7 lists cases where sticking with inline function-calling (mod-201 chapter 04) beats introducing an MCP server. Which option below is **not** one of those cases?

- A. One agent with a handful of Python tools, running in a single process for a single user.
- B. A latency-critical tool whose round trip must be sub-millisecond.
- C. Tools whose argument and return shapes are changing daily during a prototype.
- D. Tools you want to reuse from LangGraph, CrewAI, and Claude Desktop without rewriting them three times.

## Answer key

1. **D** — Chapters 1 and 5 both frame framework adoption as earning its weight against a concrete pain: durable state, checkpointing, streaming, multi-agent orchestration, production lifecycle. "The landing page is exciting" is the one bad reason ([Chapter 1](../01-the-framework-landscape.md)).
2. **B** — Model abstraction, tool abstraction, loop, state object, termination rule — the five parts Chapter 1 says every framework re-labels but does not reinvent ([Chapter 1](../01-the-framework-landscape.md)).
3. **A** — Each slot's reducer decides how partial updates merge: `add_messages` appends messages, `operator.add` sums integers. Two `{"step": 1}` updates sum to 2; the `new_msg` appends ([Chapter 2](../02-langgraph-stateful-graphs.md)).
4. **B** — LangGraph counts node transitions, and one ReAct iteration is two transitions (`model -> tools` and `tools -> model`). Four iterations is eight transitions ([Chapter 2](../02-langgraph-stateful-graphs.md)).
5. **B** — Chapter 3's explicit rule: short, expertise-focused, and output-preference-focused beats a novella. The novella costs tokens on every turn and dilutes the routing signal ([Chapter 3](../03-crewai-role-based-crews.md)).
6. **B** — `Task.context=[...]` is Chapter 3's "main composition primitive in CrewAI": upstream task outputs become structured context for downstream tasks, without delegation or globals ([Chapter 3](../03-crewai-role-based-crews.md)).
7. **C** — The manager is itself an LLM; every routing decision is an extra call with the full task list and agent roster in context. That is the Chapter 3 cost argument ([Chapter 3](../03-crewai-role-based-crews.md)).
8. **B** — Selection lives on the assistant, execution lives on the proxy. The chapter calls this the moment AutoGen's actor model becomes visible, and warns that forgetting one half wedges the loop ([Chapter 4](../04-autogen-conversational-patterns.md)).
9. **D** — There is no `confidence_threshold` termination in the Chapter 4 list. `max_round`, sentinel match, and token-usage termination are real; log-prob thresholds are invented ([Chapter 4](../04-autogen-conversational-patterns.md)).
10. **C** — Chapter 5's "AutoGen-as-workflow-engine" anti-pattern is exactly this: sequencing turns by hand with conversation rules instead of using a graph. A is the LangGraph-as-CrewAI anti-pattern; B is CrewAI-as-LangGraph; D is a different operational pitfall ([Chapter 5](../05-comparing-langgraph-crewai-autogen.md)).
11. **B** — One loop plus a couple of related personas with handoffs is the sweet spot Chapter 6 names for an SDK. A, C, and D are all topology-first problems where a framework earns its weight ([Chapter 6](../06-evaluating-competing-sdks.md)).
12. **B** — Chapter 6 presents `CodeAgent` as the canonical implementation of the code-as-action thesis: control flow lives in the agent's generated Python rather than in a tool-call-dispatch loop ([Chapter 6](../06-evaluating-competing-sdks.md)).
13. **B** — The stdio transport uses stdout for the JSON-RPC frames; routing logs to stderr is the Chapter 7 discipline, and `print(...)` to stdout is the canonical MCP footgun ([Chapter 7](../07-mcp-standardized-tool-use.md)).
14. **D** — Cross-framework tool reuse is one of the cases Chapter 7 explicitly says MCP *is* for. Single-process single-user agents, latency-critical paths, and tools with unstable shapes are the "skip MCP" cases ([Chapter 7](../07-mcp-standardized-tool-use.md)).
