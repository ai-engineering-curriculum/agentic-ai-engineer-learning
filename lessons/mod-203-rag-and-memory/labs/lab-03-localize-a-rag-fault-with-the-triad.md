# Lab 3 — Localize a RAG Fault with the Triad

**Duration:** ~45 minutes
**Covers:** [Chapter 4 — Advanced Retrieval and Evaluation](../04-advanced-rag-eval.md)

## Objective

Chapter 4 says the diagnostic power of the RAG triad is in reading the three metrics **together**: each one points at a different stage, and the joint reading **localizes** the fault to retrieval, generation, or prompting. This lab makes that abstract by driving three deliberately broken "pipelines" through the same triad scorer and matching each one to a row of the Chapter 4 localization table.

You will:

- Stand up three toy pipelines, each broken in a known way (retrieval-broken, generation-broken, prompt-broken).
- Score each with the same three-metric harness.
- Match the score pattern to the Chapter 4 table and state the fault *without* looking at the pipeline internals.

This lab uses **stub judges** (keyword overlap heuristics) instead of an LLM-as-judge. The point is to feel the three-metric *shape*, not to tune a judge. [Exercise 04](../exercises/exercise-04-rag-triad-evaluation.md) is where you replace the stubs with a real LLM judge / RAGAS over your own corpus.

## Prerequisites

- Python 3.11+ and a fresh virtualenv.
- Chapter 4 read.

## Setup

```bash
python -m venv .venv && source .venv/bin/activate
mkdir lab-03 && cd lab-03
```

No external packages required.

## Step 1 — A tiny frozen corpus and eval set

Save as `corpus.py`:

```python
# The "knowledge base". Each entry is one chunk with its source.
CORPUS = [
    {"id": "c1", "source": "handbook.md",
     "text": "The company's PTO policy grants 20 days of paid vacation per year."},
    {"id": "c2", "source": "handbook.md",
     "text": "Employees must submit reimbursement requests within 60 days of the expense."},
    {"id": "c3", "source": "security.md",
     "text": "All production deploys require a second reviewer's approval."},
    {"id": "c4", "source": "security.md",
     "text": "Access to the prod database is gated by MFA and expires after 12 hours."},
    {"id": "c5", "source": "onboarding.md",
     "text": "New hires are assigned a buddy for the first two weeks."},
    {"id": "c6", "source": "onboarding.md",
     "text": "Laptops are shipped to the hire's home address three days before the start date."},
]

# Frozen eval set. Each row has a question, the chunk IDs that are actually
# relevant, and the ground-truth answer the generator *should* produce.
EVAL_SET = [
    {"q":  "How many PTO days per year?",
     "gold_chunks": ["c1"],
     "gold_answer": "20 days per year"},
    {"q":  "What is the reimbursement deadline?",
     "gold_chunks": ["c2"],
     "gold_answer": "within 60 days of the expense"},
    {"q":  "Who must approve a prod deploy?",
     "gold_chunks": ["c3"],
     "gold_answer": "a second reviewer"},
    {"q":  "How long does prod DB access last?",
     "gold_chunks": ["c4"],
     "gold_answer": "12 hours"},
    {"q":  "What do new hires get assigned?",
     "gold_chunks": ["c5"],
     "gold_answer": "a buddy for two weeks"},
]
```

The ground-truth chunks mean you don't need an LLM judge for context relevance — you can compute it exactly.

## Step 2 — Three stubbed judges

Save as `judges.py`:

```python
def judge_context_relevance(gold_chunk_ids: list[str], retrieved_ids: list[str]) -> float:
    """Fraction of retrieved chunks that are in the gold set.
    Chapter 4: 'are retrieved chunks on-topic?' — isolates retrieval."""
    if not retrieved_ids:
        return 0.0
    hits = sum(1 for cid in retrieved_ids if cid in gold_chunk_ids)
    return hits / len(retrieved_ids)


def judge_groundedness(answer: str, contexts: list[str]) -> float:
    """Fraction of answer tokens (>=4 chars) that appear somewhere in
    the retrieved context. A rough but directional 'is the answer in the
    chunks' proxy. Chapter 4: 'is every claim supported by context?'"""
    tokens = [t.strip(".,:;").lower()
              for t in answer.split() if len(t.strip(".,:;")) >= 4]
    if not tokens:
        return 1.0
    joined = " ".join(contexts).lower()
    supported = sum(1 for t in tokens if t in joined)
    return supported / len(tokens)


def judge_answer_relevance(question: str, answer: str) -> float:
    """Fraction of question content words (>=4 chars) that appear in
    the answer OR fraction of answer words present in the question — whichever
    is higher. Chapter 4: 'does the answer address the question?'"""
    def toks(s): return {t.strip(".,:;?").lower()
                         for t in s.split() if len(t.strip(".,:;?")) >= 4}
    q, a = toks(question), toks(answer)
    if not q or not a:
        return 0.0
    overlap = len(q & a) / max(len(q), 1)
    reverse = len(q & a) / max(len(a), 1)
    return max(overlap, reverse)
```

These stubs are **not** good enough for production — Chapter 4 explicitly says you'd calibrate an LLM judge against human labels. They are good enough to produce the *shape* of the triad for three deliberately-broken pipelines, and that's what the lab needs.

## Step 3 — Three deliberately-broken pipelines

Save as `pipelines.py`:

```python
import re
from corpus import CORPUS


def _keyword_score(query: str, chunk_text: str) -> int:
    q_words = {w.lower() for w in re.findall(r"\w+", query) if len(w) >= 4}
    c_words = {w.lower() for w in re.findall(r"\w+", chunk_text)}
    return len(q_words & c_words)


def retrieve_good(query: str, k: int = 2) -> list[dict]:
    scored = [(_keyword_score(query, c["text"]), c) for c in CORPUS]
    scored.sort(key=lambda x: x[0], reverse=True)
    return [c for _, c in scored[:k]]


def retrieve_broken(query: str, k: int = 2) -> list[dict]:
    """A retriever that ignores the query — returns the first k chunks always.
    Simulates 'chunking, embedding, or k is wrong'."""
    return CORPUS[:k]


# ---- pipeline A: retrieval-broken, generator and prompt are fine ----
def pipeline_a(row):
    chunks = retrieve_broken(row["q"], k=2)
    context = " | ".join(c["text"] for c in chunks)
    # Faithful generator: it copies whatever is in the context.
    answer = f"Based on context: {context}"
    return chunks, answer


# ---- pipeline B: retrieval is good; generator hallucinates ----
def pipeline_b(row):
    chunks = retrieve_good(row["q"], k=2)
    # "Hallucinating" generator: ignores context and emits a plausible fabrication.
    fabrications = {
        "How many PTO days per year?":          "You get 15 PTO days per year.",
        "What is the reimbursement deadline?":  "Reimbursements are due within 30 days.",
        "Who must approve a prod deploy?":      "The CTO approves all prod deploys.",
        "How long does prod DB access last?":   "Prod DB access lasts 24 hours.",
        "What do new hires get assigned?":      "New hires get a mentor for one month.",
    }
    answer = fabrications.get(row["q"], "I made something up.")
    return chunks, answer


# ---- pipeline C: retrieval and grounding fine; answers off-topic ----
def pipeline_c(row):
    chunks = retrieve_good(row["q"], k=2)
    # "Wrong-question" generator: grounded, but answers a different question.
    #   The answer parrots context text (so groundedness stays high) without
    #   actually addressing the question.
    answer = chunks[1]["text"] if len(chunks) > 1 else chunks[0]["text"]
    return chunks, answer
```

Each pipeline is broken in exactly one of Chapter 4's three stages.

## Step 4 — Run the triad across all three pipelines

Save as `run_triad.py`:

```python
from statistics import mean
from corpus import EVAL_SET
from pipelines import pipeline_a, pipeline_b, pipeline_c
from judges import (judge_context_relevance,
                    judge_groundedness,
                    judge_answer_relevance)

PIPELINES = {
    "A (retrieval-broken)":   pipeline_a,
    "B (hallucinating gen)":  pipeline_b,
    "C (wrong-question gen)": pipeline_c,
}


def score_pipeline(fn) -> dict[str, float]:
    ctx, grd, ans = [], [], []
    for row in EVAL_SET:
        chunks, answer = fn(row)
        retrieved_ids = [c["id"] for c in chunks]
        contexts = [c["text"] for c in chunks]
        ctx.append(judge_context_relevance(row["gold_chunks"], retrieved_ids))
        grd.append(judge_groundedness(answer, contexts))
        ans.append(judge_answer_relevance(row["q"], answer))
    return {"context_rel": mean(ctx),
            "groundedness": mean(grd),
            "answer_rel":   mean(ans)}


print(f"{'pipeline':<28} {'context':>8} {'ground':>8} {'answer':>8}")
print("-" * 56)
for name, fn in PIPELINES.items():
    s = score_pipeline(fn)
    print(f"{name:<28} "
          f"{s['context_rel']:>8.2f} {s['groundedness']:>8.2f} {s['answer_rel']:>8.2f}")
```

Run:

```bash
python run_triad.py
```

**Expected output (shape — exact decimals will depend on the stub judges but the *pattern* is what matters):**

```
pipeline                      context   ground   answer
--------------------------------------------------------
A (retrieval-broken)             0.00     high      low
B (hallucinating gen)            high      low      low
C (wrong-question gen)           high     high      low
```

Concretely you should see pipeline A's `context_rel` at or near **0.00** (its retriever never returns the gold chunk), pipeline B's `groundedness` **low** (its fabricated numbers don't appear in the real chunks), and pipeline C's `context_rel` and `groundedness` **both high** but `answer_rel` **low** (the answer parrots a different chunk from the same source).

## Step 5 — Map each row to the Chapter 4 table — without looking at the code

Read only the score table above (not `pipelines.py`) and fill in:

| Pipeline | Likely culprit from Chapter 4's table |
|---|---|
| A | ? |
| B | ? |
| C | ? |

Then open [Chapter 4](../04-advanced-rag-eval.md) and compare against the table under "Reading the RAG triad together."

**Expected mapping:**

- **A** → *low context relevance* → **retrieval**: chunking, embedding, or k. (Here it's a stub retriever that ignores the query, but the triad *alone* tells you to look at retrieval before you look at anything else.)
- **B** → *high context relevance, low groundedness* → **generation**: the model is ignoring the context and hallucinating.
- **C** → *high context relevance, high groundedness, low answer relevance* → **prompt**: the generator is grounded but answering the wrong thing.

**Why this matters:** the triad doesn't just say "the pipeline is bad" — it tells you **which stage to go fix**. Any single metric in isolation would miss two of the three faults.

## Step 6 — Break the eval set on purpose

Chapter 4: *"you must hold the evaluation set **fixed** so a score change reflects a pipeline change, not a different test set."* Prove it in the lab.

Open `corpus.py` and change one row's `gold_chunks` from `["c1"]` to `["c2"]`. Rerun `run_triad.py`. Pipeline A's `context_rel` and `answer_rel` for that row shift even though the *pipeline* hasn't changed. Revert the edit.

**Checkpoint:** you can state, from a three-number score, which Chapter 4 row the pipeline falls into, and you understand why freezing the eval set is a precondition for comparing runs at all.

## Step 7 — Honest limits of the stub judges

Run this check to see where the stubs break down. Save as `stub_limits.py`:

```python
from judges import judge_groundedness, judge_answer_relevance

# The answer is a correct paraphrase, but shares no 4+ char tokens with the context.
paraphrase_answer = "Twenty workdays off annually."
context = ["The company's PTO policy grants 20 days of paid vacation per year."]
print("paraphrase groundedness:", judge_groundedness(paraphrase_answer, context))

# The answer is literally in the context but off-topic for the question.
offtopic_answer = "Laptops are shipped three days before the start date."
question = "How many PTO days per year?"
print("off-topic answer_rel:", judge_answer_relevance(question, offtopic_answer))
```

Run:

```bash
python stub_limits.py
```

**Expected output:** the paraphrase groundedness is low (the stub counts token overlap, so a correct paraphrase scores near zero). The off-topic answer's relevance is also low — the stub did something right.

**Why this matters:** the stubs are directional but not accurate. An LLM judge (RAGAS, or a hand-written judge prompt) handles paraphrase; the Chapter 4 discipline — **calibrate the judge against human labels** — exists precisely because even an LLM judge needs its own ground truth. The stubs here were enough to localize a fault *because* the faults were large. Real pipelines fail subtly, and that's what [exercise-04](../exercises/exercise-04-rag-triad-evaluation.md) walks you through fixing.

## Wrap-up

Three pipelines, three score patterns, three rows of the Chapter 4 localization table — now visible as actual numbers instead of a diagnostic rule you have to take on faith. Carry this harness into [exercise-04](../exercises/exercise-04-rag-triad-evaluation.md):

- Replace the stub judges with an LLM judge (RAGAS or a hand-written prompt), calibrated against ~5 human labels.
- Replace the toy corpus / eval set with your own frozen set.
- Run it across the naive, sentence-window, and auto-merging retrievers from exercise-03 — and let the triad, not your eyeballs, pick the winner.
