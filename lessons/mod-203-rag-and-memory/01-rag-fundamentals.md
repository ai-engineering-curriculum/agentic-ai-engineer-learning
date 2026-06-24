# Chapter 1 — Retrieval-Augmented Generation from the Embedding Up

Retrieval-augmented generation (RAG) is the simplest cure for a model that doesn't know your data: instead of fine-tuning facts in, you fetch the relevant facts at query time and put them in the prompt. The model reasons over text you control, so it can answer from a corpus it was never trained on — and you can update that corpus by changing a row, not retraining a model.

```text
  documents ──chunk──▶ chunks ──embed──▶ vectors ──┐
                                                    ▼
                                            ┌──────────────┐
   query ──embed──▶ query vector ──────────▶│ vector store │  top-k by similarity
                                            └──────┬───────┘
                                                   ▼
            retrieved chunks + query ──▶ [LLM] ──▶ grounded answer
```

## The five stages (and where each breaks)

**1. Chunk.** Split documents into passages small enough to be specific but large enough to be self-contained. Too small and a chunk loses the context that makes it meaningful; too large and the embedding blurs many topics into one vector. Start at ~200-500 tokens with ~10-15% overlap, and split on structure (headings, paragraphs) before you split on length.

**2. Embed.** Turn each chunk into a vector with an embedding model so semantically similar text lands near in vector space. The model that embeds your chunks **must** be the same one that embeds your queries — mixing embedding models means comparing vectors from different spaces, which is just noise.

**3. Store.** Put `(vector, chunk_text, metadata)` rows into a vector database that does approximate nearest-neighbor (ANN) search. The metadata (source, section, timestamp) is not optional — it's how you filter, cite, and debug.

**4. Retrieve.** Embed the query, ask the store for the top-k nearest chunks. Default k is small (4-8). Pure vector similarity has a blind spot: exact keywords, IDs, and rare terms. That's why production systems often add keyword (BM25) search and fuse the two — **hybrid search**.

**5. Ground.** Put the retrieved chunks in the prompt with an instruction to answer *only* from them and to cite. This is where hallucination is contained: a model told "answer from the context; if it's not there, say you don't know" fabricates far less than one asked the bare question.

## Cosine similarity, briefly

Most stores rank by **cosine similarity** — the cosine of the angle between two vectors, which measures direction regardless of magnitude. Identical direction scores 1.0, orthogonal scores 0.0. Many embedding models return normalized vectors, in which case cosine similarity and dot product are equivalent. You rarely compute this by hand — the vector DB does — but knowing the metric tells you why a "relevant-looking" chunk scored low: it was about the right *topic* but used different *language*.

```python
import numpy as np

def cosine(a: np.ndarray, b: np.ndarray) -> float:
    return float(a @ b / (np.linalg.norm(a) * np.linalg.norm(b)))

# query and chunks already embedded with the SAME model
scores = [(cosine(q_vec, c_vec), text) for c_vec, text in chunks]
top_k = sorted(scores, reverse=True)[:5]
```

## Picking a vector store

You do not need a specialized database to start. The honest decision tree:

- **Already on Postgres?** Use **pgvector** — your chunks live next to your relational data, you get SQL filtering for free, and there's no new service to operate.
- **Prototyping locally?** **Chroma** runs in-process with near-zero setup.
- **Scaling to millions of vectors with heavy filtering?** A purpose-built store like **Qdrant** gives you payload filtering, quantization, and tuned ANN indexes.

The retrieval *interface* is the same across all three — embed, upsert, query — so write your code against a thin wrapper and the store becomes swappable.

## Retrieval is just a tool

For an agent, RAG isn't a separate pipeline bolted onto the side — it's a **tool** the reason-act loop ([mod-201](../mod-201-agent-fundamentals/README.md)) can call: `search_docs(query) -> chunks`. The agent decides *whether* and *with what query* to retrieve. That framing is what makes Chapter 3's just-in-time retrieval possible: retrieval on demand instead of stuffing the prompt eagerly on every turn.

## Key takeaways

- RAG = chunk → embed → store → retrieve → ground; each stage has its own failure mode.
- Embed queries and chunks with the **same** model, or you're comparing noise.
- Vector similarity misses exact keywords — add keyword search (hybrid) when IDs and rare terms matter.
- Grounding instructions ("answer only from context; cite; say you don't know") are your main hallucination control.
- The store is swappable behind a thin interface; pick pgvector / Chroma / Qdrant by where your data and scale already are.
