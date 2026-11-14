# Quiz 1 — Agent Fundamentals: The Reason-Act Loop

Knowledge check for [mod-201](../README.md). Answers are at the bottom; try each question before scrolling. Covers all six chapters.

## Questions

### 1. Agent vs. workflow

Which of the following systems is an **agent** under the working definition used in this module (rather than a workflow)?

- A. A pipeline that sends a user message through a fixed sequence of three LLM calls — classify, retrieve, answer — with branching decided by hand-wired Python.
- B. A loop that gives an LLM a set of tools, lets it pick which to call and when to stop, and feeds each tool result back as the next observation.
- C. A scheduled cron job that calls an LLM once a day to summarize yesterday's logs.
- D. A microservice that exposes an HTTP endpoint wrapping a single LLM completion.

### 2. Why default to a workflow

Chapter 1 recommends defaulting to a workflow and reaching for an agent only when the task is genuinely open-ended. Which pair of properties best captures *why*?

- A. Workflows are more accurate than agents on every task; agents only win on creativity.
- B. Workflows have predictable cost/latency and simple failure modes (step errors), while agents have variable cost, tail-heavy latency, and can loop, drift, or never decide to stop.
- C. Workflows can call tools and agents cannot, so workflows are strictly more powerful.
- D. Workflows run on cheaper models and agents require frontier models by definition.

### 3. What the model actually sees

You send a `messages` array of `system`/`user`/`assistant` dicts to a chat endpoint. What is the model itself attending to at the lowest level?

- A. A JSON object with role-keyed fields, parsed natively by the model.
- B. One flat sequence of tokens produced by applying a model-specific **chat template** with special tokens like `<|eot_id|>` or `<|im_end|>` that mark turn boundaries.
- C. Only the final user message; earlier turns are summarized by the provider before the call.
- D. A graph of messages where roles are edges and content is nodes.

### 4. Context windows are not free

A teammate argues: "Our model has a 1M-token window, so just stuff the whole repo plus the full conversation into every turn." Which objection most directly reflects Chapter 2's reasoning?

- A. Provider SDKs reject requests above 100k tokens regardless of the advertised window.
- B. Long prompts raise cost and latency, and attention quality degrades in the middle of long contexts (the lost-in-the-middle effect) — bigger isn't automatically better.
- C. Tokens in the middle of the window are silently dropped by the model.
- D. The 1M figure refers to output tokens, not input.

### 5. The `stop=["Observation:"]` trick

In the Chapter 3 ReAct loop, why does the runtime pass `stop=["Observation:"]` to the LLM call?

- A. To make the model generate faster by capping output length.
- B. To force the model to always emit an `Observation:` line itself.
- C. So the model stops generating right before it would hallucinate a fake observation, letting the runtime inject the real tool result instead.
- D. To hide the observation from the user interface.

### 6. Errors in a ReAct loop

A tool raises a Python exception during a ReAct step. Chapter 3 says the loop should:

- A. Propagate the exception and abort the whole run.
- B. Retry the exact same Action and Action Input until it succeeds.
- C. Catch the exception and feed its message back to the model as the next `Observation:`, so the model can re-plan or try a different tool.
- D. Silently drop the step and continue with no observation.

### 7. The role of `tool_call_id`

When using native function calling, why does the follow-up `role: "tool"` message have to carry a `tool_call_id`?

- A. For analytics only; the model ignores the field.
- B. To key the tool result to the specific call that requested it, so the provider can pair them correctly when the assistant made several `tool_calls` in one turn.
- C. To authenticate the caller against the provider API.
- D. Because the OpenAI SDK requires monotonically increasing integer IDs.

### 8. Designing tools the model can actually use

Which of these is **most consistent** with Chapter 4's guidance on tool design?

- A. Ship one generic `file_op(op, path, data)` tool to keep the catalog small.
- B. Keep each tool scoped to one intent (`read_file`, `write_file`), write clear descriptions, tighten the schema (enums/required/patterns), and return structured JSON — including errors as tool results, not exceptions.
- C. Expose 40+ tools so the model always has the perfect one for the job.
- D. Omit parameter descriptions to keep the system prompt short.

### 9. Three primitive tools for a coding agent

Chapter 5 argues a minimal coding agent fits in `read_file`, `write_file`, and `run_shell`. Which of the following is **not** a reason given for resisting a fourth tool early on?

- A. `grep` and `find` are already reachable through `run_shell`.
- B. `apply_patch` is `read_file` + `write_file` once a read-before-write guard is in place.
- C. A larger tool catalog hurts tool-selection accuracy and adds surface area.
- D. Models refuse to call tools whose names contain underscores.

### 10. The read-before-write guard

Chapter 5 recommends refusing a `write_file` to an existing file the model has not read this run. The primary bug class this guard closes is:

- A. The model writing to the wrong filesystem partition.
- B. The model hallucinating a file's contents and then clobbering the real file with its invented version.
- C. UTF-8 encoding mismatches between the agent and the OS.
- D. Race conditions between parallel tool calls.

### 11. When to prefer Plan-and-Execute over reactive ReAct

For which task shape does Chapter 6 recommend a Plan-and-Execute style agent as the default?

- A. A five-step coding refactor where every step depends on what the compiler just said.
- B. A short, fluid question-answering task with two or three tool calls.
- C. "Compare GDP and life expectancy for Germany and Japan" — several independent sub-questions you would happily fan out in parallel.
- D. A benchmark task you can retry many times and want the agent to learn from failed attempts.

### 12. Reflexion vs. Plan-and-Execute

How does Reflexion differ from Plan-and-Execute as described in Chapter 6?

- A. Reflexion plans over *runs* — after a failed trajectory the agent writes a reflection that is fed into the next attempt — whereas Plan-and-Execute plans over *steps* within a single run.
- B. Reflexion is a stricter variant of Plan-and-Execute that forbids replanning.
- C. Reflexion requires a DAG scheduler; Plan-and-Execute does not.
- D. They are the same technique under two different names.

## Answer key

1. **B** — An agent is a system where the LLM dynamically directs its own process and tool usage inside a loop; A is a workflow with hand-wired control flow, and C/D are single-shot LLM calls ([Chapter 1](../01-what-is-an-agent.md)).
2. **B** — Workflows are cheaper, faster, and easier to debug; agents trade predictability for the ability to handle open-ended tasks, and can loop, drift, or fail to stop ([Chapter 1](../01-what-is-an-agent.md)).
3. **B** — Roles are a protocol layered on top of a flat token stream; the provider applies a chat template with model-specific special tokens to produce what the model actually attends to ([Chapter 2](../02-messages-tokens-and-context.md)).
4. **B** — Cost and latency scale with input length, and attention quality degrades in the middle of long contexts (lost-in-the-middle). Larger windows don't remove these costs ([Chapter 2](../02-messages-tokens-and-context.md)).
5. **C** — The stop sequence halts generation right before the model would write its own `Observation:` line, so the runtime can inject the real tool result ([Chapter 3](../03-react-loop.md)).
6. **C** — Errors are observations: the loop surfaces the error message back to the model so it can re-plan, instead of propagating the exception ([Chapter 3](../03-react-loop.md)).
7. **B** — The ID pairs each `role: "tool"` message with the specific `tool_call` that produced it, which matters whenever the assistant emits more than one call in a turn ([Chapter 4](../04-function-calling.md)).
8. **B** — One clear job per tool, descriptive names and param docs, tight schemas, structured returns, and errors-as-observations — the full Chapter 4 checklist for tools the model can actually route to ([Chapter 4](../04-function-calling.md)).
9. **D** — The chapter never claims models refuse underscored tool names. The other three (shell covers grep/find, apply_patch collapses to read+write, bigger catalogs hurt selection) are the actual reasons given ([Chapter 5](../05-coding-agent-tools.md)).
10. **B** — Requiring a read before any overwrite prevents the "model hallucinated the file's contents and clobbered the real file" failure mode ([Chapter 5](../05-coding-agent-tools.md)).
11. **C** — Independent sub-questions you would fan out in parallel are the sweet spot for Plan-and-Execute. A is iterative (reactive wins), B is short/fluid (reactive), D fits Reflexion ([Chapter 6](../06-planning-vs-reactive.md)).
12. **A** — Reflexion operates across *runs* via a lessons-learned reflection, not across *steps* of a single run; Plan-and-Execute is the per-run planner/executor split ([Chapter 6](../06-planning-vs-reactive.md)).
