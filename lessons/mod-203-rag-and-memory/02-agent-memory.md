# Chapter 2 — Memory Systems for Agents

RAG grounds an agent in a corpus. **Memory** grounds it in its own history — what happened in this task, earlier in this conversation, and in past sessions with this user. Without memory, every turn starts from zero: the agent re-asks for the user's name, forgets the constraint it agreed to two steps ago, and can't learn a preference. The fix is not one big store but **three tiers**, each with a different scope, lifetime, and storage mechanism.

```text
┌─────────────────────────────────────────────────────────┐
│ working memory   — this task        — context window     │
│ episodic memory  — this conversation— message history    │
│ long-term memory — across sessions  — vector / KV store  │
└─────────────────────────────────────────────────────────┘
   scope widens, lifetime lengthens, and storage moves
   out of the context window as you go down.
```

## Working memory: the scratchpad

Working memory is what the agent is holding *right now* to finish the current task: the sub-goals, intermediate results, the tool outputs it just got back. It lives in the **context window**. Its whole problem is that the window is finite — a long task overflows it. The tools are the ones from your context-engineering work: summarize completed steps, write intermediate results to a file or scratchpad and keep only a pointer, and prune tool outputs you no longer need. Working memory is volatile by design; when the task ends, it's gone.

## Episodic memory: the conversation so far

Episodic memory is the record of *what happened* across the turns of a single conversation: the messages exchanged, the decisions made, the facts established. Most simply, it's the message history you replay into each turn. But replaying the full transcript doesn't scale — a 200-turn conversation won't fit. So episodic memory becomes a managed thing: keep recent turns verbatim, summarize older ones into a running "conversation summary," and optionally embed each turn so you can *retrieve* the relevant past exchange instead of replaying all of it.

```python
class EpisodicMemory:
    def __init__(self, keep_recent: int = 10):
        self.turns: list[dict] = []
        self.summary: str = ""
        self.keep_recent = keep_recent

    def add(self, role: str, content: str) -> None:
        self.turns.append({"role": role, "content": content})
        if len(self.turns) > self.keep_recent:
            old = self.turns[: -self.keep_recent]
            self.summary = summarize(self.summary, old)   # roll into summary
            self.turns = self.turns[-self.keep_recent :]

    def context(self) -> list[dict]:
        head = [{"role": "system", "content": f"Earlier: {self.summary}"}]
        return head + self.turns if self.summary else self.turns
```

## Long-term memory: across sessions

Long-term memory survives the conversation ending. It's where you store durable facts: user preferences ("prefers metric units," "deploys on AWS"), learned outcomes ("the last migration of this kind broke on the FK constraint"), and stable profile data. It lives **outside the context window** — typically a vector store (for semantic recall: "anything I know relevant to *this*?") often paired with a key-value store (for exact lookups: "this user's timezone"). The agent doesn't carry it all; it **retrieves** the relevant slice at the start of a session or when a turn needs it — which is exactly the just-in-time retrieval of [Chapter 3](03-jit-retrieval-conflict.md).

Two operations define the tier:

- **Write / consolidation.** Not every turn deserves a long-term memory. Decide *what's worth remembering* — durable facts and preferences, not transient chatter — and write a clean, self-contained statement, not a raw transcript dump. Many systems run consolidation as a separate step at session end.
- **Recall.** Embed a cue (the current query, or the user id) and retrieve the top relevant memories, then inject them into the prompt.

## What goes where

| | Working | Episodic | Long-term |
|---|---|---|---|
| Scope | Current task | Current conversation | All sessions |
| Lifetime | Until task ends | Until conversation ends | Indefinite |
| Storage | Context window | Message history (+ summary) | Vector / KV store |
| Failure if missing | Loses its place mid-task | Repeats itself, forgets decisions | Never learns the user |

The common mistake is collapsing these into one bucket — dumping everything into the context window (overflows) or everything into a vector store (slow, and recall noise drowns the current task). Keep them separate and let each hand off to the next: working memory's finished results get consolidated into episodic; episodic's durable facts get consolidated into long-term.

## Key takeaways

- Agents need three memory tiers: **working** (in-task, context window), **episodic** (this conversation, message history + summary), **long-term** (across sessions, external store).
- Scope widens and lifetime lengthens down the tiers, and storage moves *out* of the context window.
- Long-term memory is **write-then-recall**: decide what's worth keeping, store a clean fact, and retrieve the relevant slice on demand.
- Don't collapse the tiers into one bucket — overflow and recall noise are the symptoms when you do.
