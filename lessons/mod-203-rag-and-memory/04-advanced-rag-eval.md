# Chapter 4 — Advanced Retrieval and Evaluation

Basic RAG ([Chapter 1](01-rag-fundamentals.md)) forces one chunk size to do two jobs it can't do at once: chunks small enough to *embed* precisely are too small to *answer* from, and chunks large enough to answer from embed imprecisely. **Advanced retrieval** breaks that tie by decoupling the unit you search from the unit you read. Two patterns dominate, and once you have them you need a way to prove they help — which is what the RAG triad gives you.

## The retrieve/read mismatch

You want to *match* on small, focused units (a single sentence embeds cleanly) but *feed the model* larger, coherent units (a paragraph reads coherently). The two advanced patterns are different answers to "embed small, return large."

### Sentence-window retrieval

Embed and index **individual sentences**, but store with each one a "window" of the sentences around it. You retrieve on the precise sentence match, then expand to the window before handing context to the model.

```text
index:   [s1][s2][s3][s4][s5][s6][s7]   ← embed each sentence
match:                 [s4]             ← s4 is the nearest neighbor
return:           [s3][s4][s5]          ← s4 plus its window
```

The embedding is sharp (one sentence, one topic) but the model reads a coherent passage. The window size is the knob: too small and you're back to fragments, too large and you reintroduce the blur.

### Auto-merging (hierarchical) retrieval

Chunk the document into a **tree**: large parent chunks split into smaller children. You embed and retrieve the *leaf* chunks, but track which parent each belongs to. When enough children of the same parent get retrieved, you **merge them up** — replace the scattered leaves with the single coherent parent chunk.

```text
            [parent: section]
           /        |        \
      [leaf a]  [leaf b]  [leaf c]   ← embed & retrieve leaves
   if a + b + c all retrieved ──▶ return the parent instead
```

This automatically returns a tight chunk when only one part is relevant and a broad chunk when the whole section is — the granularity adapts to the query instead of being fixed at index time.

## Re-ranking: cheap recall, precise top-k

Both patterns pair well with a **re-ranker**. Retrieve generously (top-20) with fast vector search to get *recall*, then run a slower, more accurate **cross-encoder** re-ranker that scores each (query, chunk) pair jointly and keep the top-5 for *precision*. The bi-encoder vector search and the cross-encoder re-ranker have complementary strengths: one is fast and approximate, the other is slow and exact. Use the first for recall, the second for the final ordering.

## Evaluating RAG: the triad

"It feels better" is not a result. The **RAG triad** decomposes RAG quality into three measurable relationships between the query, the retrieved context, and the answer — popularized by TruLens and operationalized by frameworks like RAGAS.

```text
              ┌──────────── answer relevance ────────────┐
              │  (does the answer address the query?)     │
              ▼                                           │
          [query] ──── context relevance ────▶ [context]  │
              │   (are retrieved chunks on-topic?)        │
              │                                           ▼
              └────── groundedness ──────────────────▶ [answer]
                  (is the answer supported by the context?)
```

- **Context relevance** — are the retrieved chunks actually relevant to the query? This isolates **retrieval** quality. Low here means your chunking, embedding, or k is wrong — the generator never had a chance.
- **Groundedness (faithfulness)** — is every claim in the answer supported by the retrieved context? This catches **hallucination**: the model inventing facts not in the chunks. Low here with high context relevance means a generation/prompting problem.
- **Answer relevance** — does the answer actually address the user's question? An answer can be perfectly grounded and still not answer what was asked.

The diagnostic power is in reading them *together*. Each metric points at a different stage, so a low score localizes the fault instead of just saying "bad."

| Context rel. | Groundedness | Answer rel. | Likely culprit |
|---|---|---|---|
| Low | — | — | Retrieval: chunking, embedding, or k |
| High | Low | — | Generation: model ignoring the context / hallucinating |
| High | High | Low | Prompt: model grounded but answering the wrong thing |

## How the scores get computed

In practice these are scored with an **LLM-as-judge**: a model is prompted to rate, for example, "what fraction of sentences in this answer are supported by this context?" RAGAS automates the triad (and more) over a dataset of `(question, contexts, answer, ground_truth)` rows. Two cautions: LLM judges are themselves fallible — calibrate against a handful of human labels — and you must hold the evaluation set *fixed* so a score change reflects a pipeline change, not a different test set. With a frozen eval set and the triad, "I changed the chunk size from 256 to 384 and context relevance went up 9 points" becomes a sentence you can actually say.

## Key takeaways

- Advanced retrieval **decouples the unit you search from the unit you read**: sentence-window embeds sentences but returns their neighborhood; auto-merging embeds leaves but returns the parent when enough children match.
- **Re-rank** to get both recall and precision: fast vector search for top-20, a cross-encoder re-ranker for the final top-5.
- The **RAG triad** — context relevance, groundedness, answer relevance — makes quality measurable and, read together, **localizes the fault** to retrieval, generation, or prompting.
- Score with an LLM-judge (RAGAS), calibrate it against human labels, and freeze the eval set so a score delta means a pipeline delta.
