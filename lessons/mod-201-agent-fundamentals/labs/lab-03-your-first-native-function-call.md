# Lab 3 — Your First Native Function Call

**Duration:** ~45 minutes
**Covers:** [Chapter 4 — Tool / Function Calling with Python Functions as Tools](../04-function-calling.md)

## Objective

Make one round-trip against your provider's native **tool use** / **function calling** API — schema in → `tool_calls` out → run the Python function → feed the result back → final answer — without any loop, agent framework, or schema-generator magic. The point is to see the wire format with your own eyes before Exercise 02 asks you to wrap it in a reusable `ToolCallingAgent`.

By the end of the lab you will:

- Hand-write one JSON Schema tool definition.
- See the structured `tool_calls` array come back from the model (not a regex-parsed string).
- Build the follow-up `role: "tool"` message keyed by `tool_call_id`.
- Observe how `tool_choice` lets you force or forbid a tool call.

## Prerequisites

- Python 3.11+ and a fresh virtualenv.
- **One** of the following, with a working API key exported in your shell:
  - OpenAI (`OPENAI_API_KEY`), using a tool-use-capable model such as `gpt-4o` or `gpt-4o-mini`.
  - Anthropic (`ANTHROPIC_API_KEY`), using a tool-use-capable Claude model such as `claude-sonnet-4-6`.
- Chapters 2 and 4 read.

> Pick one provider; the lab works end-to-end with either. The steps below show the OpenAI path first and the Anthropic path immediately after each code block.

## Setup

```bash
python -m venv .venv && source .venv/bin/activate
pip install openai anthropic   # install both; pick one path below
mkdir lab-03 && cd lab-03
```

## Step 1 — Define one tool as a plain Python function

The tool is deliberately trivial so that nothing — the model picking it, your dispatch logic, the final answer — can blame tool complexity.

Save as `tools.py`:

```python
from datetime import datetime, timezone, timedelta

def get_time(tz_offset_hours: int) -> dict:
    """Return the current UTC time shifted by `tz_offset_hours`."""
    now = datetime.now(timezone.utc) + timedelta(hours=tz_offset_hours)
    return {"iso": now.isoformat(timespec="seconds"), "tz_offset_hours": tz_offset_hours}
```

## Step 2 — Hand-write the JSON Schema

Normally you'd generate this from a `pydantic` model (that's Exercise 02). Writing it by hand once makes the shape stick.

Append to `tools.py`:

```python
GET_TIME_SCHEMA = {
    "type": "function",
    "function": {
        "name": "get_time",
        "description": "Return the current UTC time shifted by a timezone offset in hours.",
        "parameters": {
            "type": "object",
            "properties": {
                "tz_offset_hours": {
                    "type": "integer",
                    "description": "Hours to add to UTC. Example: 1 for Berlin in winter, -5 for New York in winter.",
                    "minimum": -12,
                    "maximum": 14,
                },
            },
            "required": ["tz_offset_hours"],
        },
    },
}
```

Notice:

- One clear job per tool (Chapter 4 guidance).
- Required parameter is marked.
- Tight bounds on the integer — looser schemas invite hallucinated arguments.
- The `description` field is part of the prompt the model sees; it's not documentation for you.

## Step 3 — First request: watch `tool_calls` come back

### Option A — OpenAI

Save as `one_call_openai.py`:

```python
import json
from openai import OpenAI
from tools import GET_TIME_SCHEMA, get_time

client = OpenAI()

messages = [
    {"role": "system", "content": "You are a concise assistant. Use tools when helpful."},
    {"role": "user",   "content": "What time is it right now in Berlin (UTC+1)?"},
]

resp = client.chat.completions.create(
    model="gpt-4o",
    messages=messages,
    tools=[GET_TIME_SCHEMA],
    tool_choice="auto",
)

msg = resp.choices[0].message
print("=== assistant message ===")
print(json.dumps(msg.model_dump(exclude_none=True), indent=2))
```

Run:

```bash
python one_call_openai.py
```

### Option B — Anthropic

Save as `one_call_anthropic.py`:

```python
import json
import anthropic
from tools import GET_TIME_SCHEMA, get_time

client = anthropic.Anthropic()

# Anthropic uses a slightly different schema shape: no outer "function" wrapper.
ANTHROPIC_TOOL = {
    "name":         GET_TIME_SCHEMA["function"]["name"],
    "description":  GET_TIME_SCHEMA["function"]["description"],
    "input_schema": GET_TIME_SCHEMA["function"]["parameters"],
}

resp = client.messages.create(
    model="claude-sonnet-4-6",
    max_tokens=512,
    system="You are a concise assistant. Use tools when helpful.",
    tools=[ANTHROPIC_TOOL],
    messages=[{"role": "user", "content": "What time is it right now in Berlin (UTC+1)?"}],
)

print("=== assistant message ===")
print(json.dumps([b.model_dump() for b in resp.content], indent=2))
print("stop_reason:", resp.stop_reason)
```

Run:

```bash
python one_call_anthropic.py
```

**Expected output (shape, not exact bytes):**

- **OpenAI:** `message.content` is `null` and `message.tool_calls` is a list with one entry — a `function` object whose `name` is `"get_time"` and whose `arguments` is a JSON **string** (not an object) that parses to `{"tz_offset_hours": 1}`.
- **Anthropic:** `resp.content` contains a `tool_use` block with `name="get_time"` and `input={"tz_offset_hours": 1}` (already a dict). `stop_reason` is `"tool_use"`.

**Why this matters:** no regex, no `Action:` line parsing. The provider validated the model's call against your JSON Schema and handed you a typed structure. That is the whole Chapter 4 win over Chapter 3's ReAct string-parsing.

## Step 4 — Dispatch the call and build the follow-up message

### Option A — OpenAI

Append to `one_call_openai.py`:

```python
assert msg.tool_calls, "expected a tool call; got a plain reply"
call = msg.tool_calls[0]
args = json.loads(call.function.arguments)
result = get_time(**args)

followup_messages = messages + [
    msg.model_dump(exclude_none=True),           # the assistant turn, verbatim
    {
        "role":         "tool",
        "tool_call_id": call.id,                 # pairs this result to that call
        "content":      json.dumps(result),
    },
]

final = client.chat.completions.create(
    model="gpt-4o",
    messages=followup_messages,
    tools=[GET_TIME_SCHEMA],
    tool_choice="auto",
)
print("=== final assistant message ===")
print(final.choices[0].message.content)
```

### Option B — Anthropic

Append to `one_call_anthropic.py`:

```python
tool_use = next(b for b in resp.content if b.type == "tool_use")
result = get_time(**tool_use.input)

final = client.messages.create(
    model="claude-sonnet-4-6",
    max_tokens=512,
    system="You are a concise assistant. Use tools when helpful.",
    tools=[ANTHROPIC_TOOL],
    messages=[
        {"role": "user",      "content": "What time is it right now in Berlin (UTC+1)?"},
        {"role": "assistant", "content": [b.model_dump() for b in resp.content]},
        {"role": "user",      "content": [{
            "type":        "tool_result",
            "tool_use_id": tool_use.id,
            "content":     json.dumps(result),
        }]},
    ],
)
print("=== final assistant message ===")
print("".join(b.text for b in final.content if b.type == "text"))
```

**Expected output:** a one-sentence natural-language answer like *"It's roughly 14:32 local time in Berlin right now."* The timestamp should be plausible for UTC+1 at the moment you run the lab.

**The key detail:** both providers require you to pair the tool result with the specific call that produced it — `tool_call_id` on OpenAI, `tool_use_id` on Anthropic. Chapter 4 calls this out because when the model emits two parallel calls in one turn, you *must* key the results so the provider can line them up.

## Step 5 — Force the model's hand with `tool_choice`

### Option A — OpenAI

Run the same prompt, but change `tool_choice`:

```python
# Instead of tool_choice="auto":
tool_choice={"type": "function", "function": {"name": "get_time"}}
```

Re-run. Observe: even if you ask something off-topic ("What's a palindrome?"), the model is now forced to produce a `get_time` call — including a reasonable guess at `tz_offset_hours` if it has to. That is exactly how Chapter 4 says people implement structured output on top of the function-calling API: force-call a `submit_answer`-shaped tool.

Try the three modes in Chapter 4 and note the behavior:

- `tool_choice="auto"` — model decides.
- `tool_choice="none"` — the tool is listed but off; you get a plain reply.
- `tool_choice="required"` — model *must* call some tool, but picks which.

### Option B — Anthropic

Anthropic's equivalents are:

- `tool_choice={"type": "auto"}` (default).
- `tool_choice={"type": "any"}` — any tool (OpenAI's `required`).
- `tool_choice={"type": "tool", "name": "get_time"}` — force this specific tool.

Try each and watch `stop_reason` and the `content` blocks shift.

## Step 6 — Break the schema on purpose

Change the schema's `maximum` to `0` so Berlin's `+1` offset is out of range, and rerun Step 3. Depending on the provider you will see one of:

- The model invents a value inside the allowed range (e.g. `tz_offset_hours=0`) and gets the answer slightly wrong.
- The model refuses the tool and answers in plain text.

Either outcome is useful: it shows how **schema tightness drives model behavior**. Loosen the schema back when done.

**Checkpoint:** you can state, in your own words:

1. What the wire shape of a `tool_calls` / `tool_use` response looks like for your chosen provider.
2. Why `tool_call_id` (or `tool_use_id`) is non-negotiable when results come back.
3. When `tool_choice` is the right lever instead of a system-prompt nudge.

## Wrap-up

This lab is the "smallest native tool-use round-trip that teaches anything." Exercise 02 then asks you to (a) generate the schema from a type-hinted Python function, (b) wrap this round-trip in a loop, and (c) handle parallel tool calls. The piece you just did by hand — build the schema, read the structured response, key the result with the right ID — is the core each of those pieces is built around.
