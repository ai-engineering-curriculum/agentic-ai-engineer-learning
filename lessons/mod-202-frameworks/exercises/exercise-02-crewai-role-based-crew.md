# exercise-02: CrewAI Role-Based Crew

**Estimated effort:** 3 hours

## Objective

Re-implement the same research-assistant task as a CrewAI crew: a researcher role and a quantitative-analyst role, composed with `Task.context`, run first as a `Process.sequential` crew and then as a `Process.hierarchical` crew. By the end you'll feel where role decomposition helps, where the hierarchical manager costs you, and why most CrewAI bugs are org-design bugs.

## Background

This exercise covers material from:

- [Chapter 3 — CrewAI: Role-Based Crews](../03-crewai-role-based-crews.md)
- [Chapter 5 — Comparing LangGraph, CrewAI, and AutoGen](../05-comparing-langgraph-crewai-autogen.md)

Reuse the `search` and `calc` tools from [exercise-01](exercise-01-langgraph-stateful-agent.md) so the three builds in this module solve an identical task. A stubbed `search` keeps the run deterministic.

## Prerequisites

- [exercise-01](exercise-01-langgraph-stateful-agent.md) completed, so you have the tools and the task fixed.
- `crewai` (and optionally `crewai-tools`) installed.
- A provider API key in an environment variable; a small spend cap.

## Tasks

### 1. Define two agents

- A `researcher` with a job-title `role`, a one-line `goal`, and a two-line `backstory`; tools `[search]`; `allow_delegation=False`.
- A `synthesizer` (quantitative analyst) with `[calc]`; `allow_delegation=False`.
- Keep backstories short and informative — no lorem-ipsum novellas.

### 2. Define two tasks composed by context

- `gather_populations` (assigned to `researcher`): emit a strict JSON object with the two countries, their populations, and a source. Put a literal example in `expected_output`.
- `compute_ratio` (assigned to `synthesizer`): `context=[gather_populations]`, compute `country_a.pop / country_b.pop`, return one sentence with the ratio to two decimals.

### 3. Run it sequentially

- Build a `Crew` with `process=Process.sequential` and `kickoff(inputs={"question": ...})`.
- Confirm the synthesizer only ever sees the structured JSON, not the researcher's tool transcript — so it cannot hallucinate populations.

### 4. Run it hierarchically and measure the delta

- Build a second crew with `process=Process.hierarchical` and a manager model.
- Run the same task. Record the difference in total LLM calls and token spend versus the sequential crew.

### 5. Provoke and fix an org-design failure

- Introduce a second researcher with an overlapping role and run the hierarchical crew. Observe the manager thrashing over task assignment.
- Fix it by giving each agent a distinct, non-overlapping slice and write down what changed.

## Starter guidance

```python
from crewai import Agent, Task, Crew, Process

# Reuse search() and calc() from exercise-01, wrapped as CrewAI tools.
from tools import search, calc  # crewai @tool-decorated callables


def build_agents(llm):
    researcher = Agent(
        role="Population researcher",
        goal="Find current population figures with a source.",
        backstory="You read demographic releases and cite the UN WPP when possible.",
        tools=[search],
        llm=llm,
        allow_delegation=False,
    )
    synthesizer = Agent(
        role="Quantitative analyst",
        goal="Compute requested ratios and summarize in one sentence.",
        backstory="You give numeric answers to two decimal places.",
        tools=[calc],
        llm=llm,
        allow_delegation=False,
    )
    return researcher, synthesizer


def build_tasks(researcher, synthesizer):
    raise NotImplementedError  # gather_populations + compute_ratio with context=[...]


def run(process: Process, llm) -> str:
    raise NotImplementedError  # assemble Crew(process=process), kickoff, return result


if __name__ == "__main__":
    ...  # run sequential, then hierarchical; print both and the call/token deltas
```

## Acceptance criteria

You can demonstrate that:

- The sequential crew completes the population-ratio task and the synthesizer's prompt contains only the structured JSON from the upstream task.
- `Task.context` is the composition mechanism — the downstream task reads the upstream output, not a shared global.
- The hierarchical crew produces the same answer but measurably more LLM calls and tokens, and you have the numbers recorded.
- An overlapping-role crew visibly thrashes, and your fix (distinct role slices) resolves it.
- `expected_output` uses a literal example, not a prose description, and the model honors the shape.

## Reflection

In `NOTES.md`:

1. How many extra LLM calls did the hierarchical manager add for this task? When would that premium be worth paying?
2. Where did you have to fight CrewAI to express control flow it doesn't model well? Would a LangGraph conditional edge have been cleaner?
3. Which of your two backstories actually changed the output, and which was decoration you could delete?

## Stretch goals

- Enable `Crew(memory=True)` and run two related questions in sequence; show the second run referencing an entity from the first.
- Add a third role (an editor) that critiques the synthesizer's sentence and have it run as a third sequential task.
- Wrap the sequential crew as a step inside a CrewAI `Flow` and route to a different crew based on the question type.
- Swap one agent's `llm` for a cheaper model and measure the quality/cost trade-off.
