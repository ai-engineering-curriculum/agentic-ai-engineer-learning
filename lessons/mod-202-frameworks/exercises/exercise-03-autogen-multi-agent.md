# exercise-03: AutoGen Multi-Agent

**Estimated effort:** 3 hours

## Objective

Re-implement the same research-assistant task as an AutoGen conversation: first a two-agent chat (`UserProxyAgent` + `AssistantAgent`), then a multi-participant group chat with a manager. By the end you'll know how speaker selection and termination conditions decide whether a conversational team converges or loops forever, and why the transcript is your only debugger.

## Background

This exercise covers material from:

- [Chapter 4 — AutoGen: Conversational Multi-Agent Patterns](../04-autogen-conversational-patterns.md)
- [Chapter 5 — Comparing LangGraph, CrewAI, and AutoGen](../05-comparing-langgraph-crewai-autogen.md)

Reuse the `search` and `calc` tools from [exercise-01](exercise-01-langgraph-stateful-agent.md). Pin a single AutoGen package family for the whole exercise — do not mix `autogen` (v0.2) and `autogen_agentchat` (v0.4) imports in one process.

## Prerequisites

- [exercise-01](exercise-01-langgraph-stateful-agent.md) completed.
- One AutoGen package family installed and pinned to an exact version.
- A provider API key in an environment variable; a small spend cap.

## Tasks

### 1. Two-agent chat

- Create an `AssistantAgent` whose system prompt instructs it to use `search`/`calc` and to end with `TERMINATE` on its own line.
- Create a `UserProxyAgent` with `human_input_mode="NEVER"`, a `max_consecutive_auto_reply` cap, and an `is_termination_msg` that matches `TERMINATE` strictly (own line, exact).
- Register each tool on **both** sides: `register_for_llm` on the assistant, `register_for_execution` on the proxy.
- Run `initiate_chat` on the population-ratio task.

### 2. Persist the transcript

- Capture every message (agent name, role, content, timestamp) to a log you can replay. The transcript is the program; you must be able to read it after the fact.

### 3. Group chat with a manager

- Add `planner`, `researcher`, and `synthesizer` assistant agents.
- Build a `GroupChat` (or v0.4 team) with a `max_round` cap and a `GroupChatManager` (or `SelectorGroupChat`).
- Run the same task and log the manager's speaker selections per turn.

### 4. Provoke and fix two failure modes

- **Mutual politeness / no-progress loop.** Loosen the termination condition and show the team running to `max_round` without converging; then tighten it.
- **Tool registration mismatch.** Register a tool for LLM but not for execution; observe the wedge; fix it.

### 5. Add a human gate

- Switch the proxy to `human_input_mode="TERMINATE"` so a human can override or extend before the conversation ends, and demonstrate one manual extension.

## Starter guidance

```python
# v0.2 idiom shown; the v0.4 equivalent uses a team + TerminationCondition.
from autogen import AssistantAgent, UserProxyAgent

from tools import search, calc  # reused from exercise-01

LLM_CONFIG = {
    "config_list": [{"model": "gpt-4o", "api_key": "${OPENAI_API_KEY}"}],
    "temperature": 0,
}


def is_terminate(msg) -> bool:
    raise NotImplementedError  # strict own-line, exact "TERMINATE" match


def build_two_agent_chat():
    assistant = AssistantAgent(
        name="researcher",
        system_message=("Use search/calc as needed. End with TERMINATE on its own line."),
        llm_config=LLM_CONFIG,
    )
    user_proxy = UserProxyAgent(
        name="user",
        human_input_mode="NEVER",
        max_consecutive_auto_reply=10,
        is_termination_msg=is_terminate,
        code_execution_config=False,
    )
    for fn, desc in [(search, "Web search."), (calc, "Evaluate a math expression.")]:
        assistant.register_for_llm(description=desc)(fn)
        user_proxy.register_for_execution()(fn)
    return user_proxy, assistant


def run_group_chat(task: str) -> str:
    raise NotImplementedError  # planner + researcher + synthesizer + manager, capped rounds


if __name__ == "__main__":
    proxy, assistant = build_two_agent_chat()
    proxy.initiate_chat(
        assistant,
        message=("Compare the population of the two most populous countries in 2024 "
                 "and report the ratio."),
    )
```

## Acceptance criteria

You can demonstrate that:

- The two-agent chat completes the task and terminates on the strict `TERMINATE` sentinel, not on `max_consecutive_auto_reply`.
- Every tool is registered on both the LLM side and the execution side; the task wedges when you remove one half and recovers when you restore it.
- The full transcript is persisted and replayable, with per-message agent names and timestamps.
- The group chat completes the task, and you have logged the manager's speaker selection on each turn.
- A loosened termination condition runs to `max_round`; tightening it makes the team converge.

## Reflection

In `NOTES.md`:

1. What did the manager's speaker-selection log reveal about where turns were wasted?
2. Compare debugging this team to debugging your LangGraph version. Where was "read the transcript" worse or better than "inspect the state"?
3. Why is matching `TERMINATE` strictly (own line, exact) load-bearing? Give an input that silently terminates a loose matcher mid-task.

## Stretch goals

- Port the two-agent chat to the other AutoGen package family and note every API that changed.
- Replace the LLM manager with a rule-based speaker-selection function and measure the token savings.
- Add a `CodeExecutorAgent` (Docker-backed) that runs the arithmetic in a sandbox instead of a `calc` tool, and document the sandbox configuration.
- Add a `TokenUsageTermination` (v0.4) or token budget check and show it firing before `max_round`.
