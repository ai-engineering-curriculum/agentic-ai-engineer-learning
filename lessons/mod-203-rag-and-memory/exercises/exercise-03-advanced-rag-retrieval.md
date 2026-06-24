# exercise-03: Advanced RAG Retrieval

**Estimated effort:** 3 hours

## Objective

Upgrade the basic pipeline from exercise-01 with advanced retrieval that decouples the unit you search from the unit you read. You'll implement **sentence-window** retrieval and **auto-merging** (hierarchical) retrieval, add a **re-ranker** for precision, and show on concrete queries where each beats naive fixed-size chunking.

## Background

This exercise covers material from:

- [Chapter 4 — Advanced Retrieval and Evaluation](../04-advanced-rag-eval.md)
- [Chapter 1 — Retrieval-Augmented Generation from the Embedding Up](../01-rag-fundamentals.md)

Build on the corpus, embedding wrapper, and vector store from [exercise-01](exercise-01-rag-pipeline-vector-db.md). You may implement these by hand or use a framework's retrievers — but you must be able to explain what the retriever returns and why.

## Prerequisites

- The exercise-01 RAG pipeline (chunking, embedding, vector store, grounded generation).
- A re-ranker: a cross-encoder model (local sentence-transformers cross-encoder) or a hosted re-rank endpoint.
- The same corpus as exercise-01 so you can compare retrievers fairly.

## Tasks

### 1. Sentence-window retrieval

- Index the corpus at the **sentence** level: embed each sentence, but store with it a window of the W sentences before and after.
- On retrieval, match on the sentence embedding, then expand each hit to its window before building the prompt context.
- Make W a parameter; try W = 1 and W = 3.

### 2. Auto-merging retrieval

- Build a two-level chunk tree: parent chunks (e.g., a section) split into child leaf chunks.
- Embed and retrieve **leaves**, but track each leaf's parent.
- When the number of retrieved children of one parent crosses a threshold, **merge up**: replace those leaves with the single parent chunk in the context.

### 3. Re-ranking

- For one retriever, retrieve generously (top-20) with vector search, then re-rank with a cross-encoder and keep the top-5.
- Compare the top-5 *before* and *after* re-ranking on a query where vector search alone ranks a marginal chunk too high.

### 4. Head-to-head comparison

- Pick 5-8 questions against your corpus.
- For each, run naive fixed-chunk retrieval, sentence-window, and auto-merging, and record the retrieved context.
- Note a query where sentence-window's sharp matching wins, and one where auto-merging's adaptive granularity wins.

## Starter guidance

```python
from dataclasses import dataclass

@dataclass
class SentenceUnit:
    sentence: str
    window: str          # the W sentences around it, stored alongside
    source: str

def sentence_window_index(docs: list[str], w: int = 2) -> list[SentenceUnit]:
    raise NotImplementedError  # embed `sentence`, return `window` on retrieval

@dataclass
class TreeChunk:
    text: str
    parent_id: str | None
    chunk_id: str

def auto_merge(hits: list[TreeChunk], threshold: int) -> list[TreeChunk]:
    raise NotImplementedError  # merge children up to parent when enough match

def rerank(query: str, candidates: list[str], top_k: int = 5) -> list[str]:
    raise NotImplementedError  # cross-encoder scores (query, candidate) pairs
```

You do **not** need the full RAG-triad harness here — that's exercise-04, which will *measure* whether these retrievers actually help.

## Acceptance criteria

You can demonstrate that:

- Sentence-window retrieval matches on a sentence but feeds the model the surrounding window.
- Auto-merging returns leaf chunks for narrow queries and a merged parent when enough children of one parent are retrieved.
- Re-ranking reorders a top-20 candidate set and the new top-5 is visibly more on-point than the pre-rerank top-5 on at least one query.
- A side-by-side table shows context retrieved by naive vs. sentence-window vs. auto-merging for your question set.

## Reflection

In `NOTES.md`:

1. Show a query where naive fixed chunks returned a fragment that lost its context, and how sentence-window fixed it.
2. For auto-merging, show a query that returned a single leaf and one that merged up to the parent. What made the difference?
3. Did re-ranking ever *hurt* — promote a worse chunk? When does the cross-encoder disagree with vector similarity, and which was right?

## Stretch goals

- Tune the auto-merge threshold and the window size W, and describe the precision/coherence trade-off you observe.
- Add **parent-document retrieval** as a third variant (retrieve small, return the full parent doc) and compare it to auto-merging.
- Carry these retrievers into exercise-04 and let the RAG triad decide which one actually wins — don't trust your eyeballed comparison.
