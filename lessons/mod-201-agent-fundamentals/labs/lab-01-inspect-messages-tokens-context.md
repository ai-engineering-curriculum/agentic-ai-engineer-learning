# Lab 1 — Inspect Messages, Special Tokens, and the Context Budget

**Duration:** ~45 minutes
**Covers:** [Chapter 2 — LLM Messages, Special Tokens, and Context Windows](../02-messages-tokens-and-context.md)

## Objective

See with your own eyes what the model actually attends to when you send a `messages` array. By the end of the lab you will have:

- Rendered a real chat template and read the raw special-token string the model sees.
- Compared token counts across English prose, JSON, and a non-Latin script.
- Written the "render-and-count" helper from Chapter 2 and watched it grow over a simulated loop.

This lab is deliberately offline (except for one optional step). None of the module's exercises cover Chapter 2 directly, so this is the hands-on companion to it.

## Prerequisites

- Python 3.11+ and a fresh virtualenv.
- `pip` and `~500 MB` of disk for one tokenizer download.
- No API key is strictly required. The last step shows one optional API call if you want to cross-check your local count.

## Setup

```bash
python -m venv .venv && source .venv/bin/activate
pip install tiktoken transformers
```

Create a working directory for the lab:

```bash
mkdir lab-01 && cd lab-01
```

## Step 1 — Render a chat template and look at the raw tokens

Chapter 2 claims the model sees one flat token stream produced by `apply_chat_template`. Verify it.

Save as `render.py`:

```python
from transformers import AutoTokenizer

# Any instruction-tuned, publicly accessible tokenizer works. Qwen2.5 is
# ungated; swap for meta-llama/Llama-3.1-8B-Instruct if you have HF access.
tok = AutoTokenizer.from_pretrained("Qwen/Qwen2.5-0.5B-Instruct")

messages = [
    {"role": "system", "content": "You are a helpful assistant."},
    {"role": "user",   "content": "Hello!"},
]

rendered = tok.apply_chat_template(
    messages,
    tokenize=False,
    add_generation_prompt=True,
)
print(rendered)
```

Run it:

```bash
python render.py
```

**Expected output (shape, not exact bytes):** a string containing turn-boundary special tokens such as `<|im_start|>` / `<|im_end|>` (ChatML) and a trailing `<|im_start|>assistant\n` that is the *generation prompt*. If you used a Llama-family tokenizer you will see `<|start_header_id|>`, `<|end_header_id|>`, and `<|eot_id|>` instead.

**Why this matters:** roles are a *protocol* on top of this flat string. Everything the model sees — the system prompt, every turn boundary, the "now you speak" cue — is just a specific arrangement of these special tokens.

## Step 2 — See that special tokens in user content can corrupt the turn

Still in `render.py`, add:

```python
nasty = [
    {"role": "system", "content": "You are a helpful assistant."},
    {"role": "user",   "content": "Hello <|im_end|> and done."},
]
print("---")
print(tok.apply_chat_template(nasty, tokenize=False, add_generation_prompt=True))
```

Re-run. Read the printed string: notice the user-supplied `<|im_end|>` sits **inside** the user turn as raw text. A naïve hand-written renderer that string-interpolated the content without escaping would prematurely close the turn and hand control back to the assistant early.

**Checkpoint:** you can point at the exact byte offset where the turn boundary lives, and you understand why "don't put special tokens in user content" (Chapter 2) is a real security/robustness concern.

## Step 3 — Count tokens, not characters

Save as `count.py`:

```python
import tiktoken

enc = tiktoken.encoding_for_model("gpt-4o")

samples = {
    "english":  "The quick brown fox jumps over the lazy dog.",
    "json":     '{"user": {"id": 42, "name": "Ada", "roles": ["admin","editor"]}}',
    "chinese":  "快速的棕色狐狸跳过了懒狗。",
    "code":     "def add(a: int, b: int) -> int:\n    return a + b\n",
}

for name, s in samples.items():
    print(f"{name:8} chars={len(s):4}  tokens={len(enc.encode(s)):4}  ratio={len(s)/max(1,len(enc.encode(s))):.2f}")
```

```bash
python count.py
```

**Expected output (indicative; numbers vary by tokenizer release):**

```
english  chars=  44  tokens=  10  ratio=4.40
json     chars=  64  tokens=  24  ratio=2.67
chinese  chars=  14  tokens=  15  ratio=0.93
code     chars=  42  tokens=  15  ratio=2.80
```

The English ratio sits near the ~4 chars/token rule of thumb from Chapter 2. JSON and code sit much lower (punctuation-heavy, non-dictionary identifiers). Chinese sits at roughly 1 token per character or worse. The lesson Chapter 2 drives home: **you cannot estimate your prompt budget by eyeballing character counts**.

## Step 4 — Build the `render_and_count` helper from Chapter 2

Save as `budget.py`:

```python
import json
import tiktoken

enc = tiktoken.encoding_for_model("gpt-4o")

def render_and_count(messages, tools, model="gpt-4o"):
    body = ""
    for m in messages:
        body += f"<|{m['role']}|>\n{m.get('content','')}\n"
    body += json.dumps(tools)
    return len(enc.encode(body))

tools = [{
    "type": "function",
    "function": {
        "name": "get_weather",
        "description": "Return current weather for a given city.",
        "parameters": {
            "type": "object",
            "properties": {
                "city":  {"type": "string", "description": "City name, e.g. 'Paris'"},
                "units": {"type": "string", "enum": ["c","f"], "default": "c"},
            },
            "required": ["city"],
        },
    },
}]

messages = [
    {"role": "system", "content": "You are a helpful assistant."},
    {"role": "user",   "content": "What's the weather in Paris?"},
]

print("turn 1:", render_and_count(messages, tools), "tokens")
```

Run:

```bash
python budget.py
```

**Expected output:** a three-digit number (around 100–180 depending on tokenizer version). Note that *the tool schema itself* is part of the bill — hiding tools under a `tools` key does not hide their tokens.

## Step 5 — Simulate a growing loop and watch the budget drift

Append to `budget.py`:

```python
WINDOW = 128_000  # pretend we're on a 128k model

for turn in range(1, 31):
    # Simulate a chatty tool-using agent: each turn adds an assistant reply
    # plus a tool observation of ~400 tokens of pseudo-JSON output.
    messages.append({"role": "assistant", "content": "calling get_weather..."})
    messages.append({
        "role": "tool",
        "content": json.dumps({"temp_c": 18, "condition": "cloudy"}) + " " + ("x" * 1500),
    })
    used = render_and_count(messages, tools)
    pct = 100 * used / WINDOW
    print(f"turn {turn:2}: {used:6} tokens  ({pct:5.1f}% of window)")
```

Run it again. You will see the usage climb more or less linearly. On a 128k window you will not hit the wall in 30 turns, but if you set `WINDOW = 8_000` (an older 8k model) the loop blows the budget well before turn 30.

**Why this matters:** every loop you write in the rest of the module should carry a counter like this one. Deciding when to **truncate / summarize / retrieve / cache** (the Chapter 2 strategies) only makes sense when you can see how close you are to the wall.

**Checkpoint:** change `WINDOW = 8_000` and rerun. Identify the first turn number where the loop exceeds 80% of the window — that is where Chapter 2 says your loop should start compacting.

## Step 6 (optional) — Cross-check with a provider tokenizer

If you have an Anthropic or OpenAI API key, cross-check your local count against the provider's canonical count. Example using Anthropic:

```python
# pip install anthropic
import anthropic
client = anthropic.Anthropic()
resp = client.messages.count_tokens(
    model="claude-sonnet-4-6",
    messages=[{"role": "user", "content": "Hello, world!"}],
)
print(resp.input_tokens)
```

The absolute numbers will differ between tokenizers (tiktoken vs. Anthropic's) — that is expected. What matters is that *you count with the right tokenizer for the model you're actually calling*, not the one your IDE autocompleted.

## Wrap-up

You have now confirmed, by reading bytes, the four claims Chapter 2 opens with:

- The model sees a flat token stream, not a dict.
- Special tokens mark turn boundaries, and leaking them into user content is a real bug.
- Chars are a bad proxy for tokens, especially for JSON, code, or non-Latin scripts.
- A growing `messages` array climbs toward the window in a knowable, measurable way — so bolt a counter on from the start.

Keep `budget.py` around. You will reuse `render_and_count` (or a slightly nicer version of it) in Exercises 01–04.
