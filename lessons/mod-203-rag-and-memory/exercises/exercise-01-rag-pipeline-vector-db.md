# exercise-01: Build a RAG Pipeline on a Vector DB

**Estimated effort:** 3 hours

## Objective

Build a retrieval-augmented generation pipeline end to end: chunk a small document corpus, embed the chunks, store them in a real vector database, retrieve the top-k for a query, and produce a grounded, cited answer. By the end you'll have a working `answer(question)` function and a concrete feel for how chunk size and k change what comes back.

## Background

This exercise covers material from:

- [Chapter 1 — Retrieval-Augmented Generation from the Embedding Up](../01-rag-fundamentals.md)

Use any embedding model (an API embeddings endpoint or a local sentence-transformer) and any of the vector stores from the chapter — **pgvector**, **Chroma**, or **Qdrant**. Pick the one closest to what you already run. Treat retrieval as a function now; you'll turn it into an agent tool in exercise-02.

## Prerequisites

- A vector store you can run locally (pgvector via Docker, Chroma in-process, or Qdrant via Docker).
- An embedding model and an LLM, each reachable via an API key in an environment variable, with a small spend cap.
- A corpus of ~10-30 documents (use your own notes, a project's docs, or a public dataset).

## Tasks

### 1. Chunk the corpus

- Split each document into passages of ~300 tokens with ~15% overlap.
- Split on structure (headings/paragraphs) before falling back to length.
- Attach metadata to every chunk: `source`, `section`, and a `timestamp`. You'll need `timestamp` and `source` again in exercise-02's conflict resolution.

### 2. Embed and store

- Embed every chunk and upsert `(vector, chunk_text, metadata)` into your vector store.
- Wrap the store behind a thin interface — `upsert(chunks)` and `query(text, k)` — so the store is swappable.
- **Use the same embedding model for chunks and queries.** Assert the embedding dimension matches the index.

### 3. Retrieve

- Implement `retrieve(question, k)` that embeds the question and returns the top-k chunks with their similarity scores and metadata.
- Print the scores. Confirm the ranking is sane on a question you know the answer to.

### 4. Ground the generation

- Build a prompt that injects the retrieved chunks and instructs the model to answer **only** from them, to **cite** the `source` of each claim, and to say "I don't know" when the answer isn't in the context.
- Ask a question whose answer is in the corpus and one whose answer is **not**. The second must produce "I don't know," not a fabrication.

### 5. Tune and observe

- Re-run with k = 2 and k = 8, and with chunk size 150 vs. 500. Note how the answer and the cited sources change.

## Starter guidance

```python
from dataclasses import dataclass

@dataclass
class Chunk:
    text: str
    source: str
    section: str
    timestamp: str

def chunk_document(doc: str, source: str) -> list[Chunk]:
    raise NotImplementedError  # structure-aware split, ~300 tokens, 15% overlap

class VectorStore:
    def upsert(self, chunks: list[Chunk]) -> None: ...
    def query(self, text: str, k: int) -> list[tuple[float, Chunk]]: ...

def retrieve(store: VectorStore, question: str, k: int = 5) -> list[tuple[float, Chunk]]:
    raise NotImplementedError

def answer(store: VectorStore, question: str, k: int = 5) -> str:
    hits = retrieve(store, question, k)
    context = "\n\n".join(f"[{c.source}] {c.text}" for _, c in hits)
    # prompt: answer ONLY from context, cite source, say "I don't know" if absent
    raise NotImplementedError
```

You do **not** need hybrid search, re-ranking, or memory here — those are exercises 02 and 03.

## Acceptance criteria

You can demonstrate that:

- Chunks are stored in a real vector DB with `source`, `section`, and `timestamp` metadata.
- The **same** embedding model embeds chunks and queries (and the code asserts dimension match).
- `retrieve` returns ranked chunks with scores; the top hits are visibly on-topic for a known question.
- An in-corpus question yields a **cited** answer; an out-of-corpus question yields "I don't know," not a fabrication.
- You can show how k and chunk size change the retrieved set and the answer.

## Reflection

In `NOTES.md`:

1. Show one question where the top chunk was about the right topic but scored lower than an off-topic chunk. What does that tell you about vector similarity's blind spot?
2. Which chunk size gave the best answers for your corpus, and why do you think small *and* large both hurt?
3. What would break if you embedded queries with a different model than your chunks?

## Stretch goals

- Add **keyword (BM25) search** and fuse it with vector results (reciprocal rank fusion) — a basic hybrid retriever. Find a query where hybrid beats pure vector (hint: an exact ID or rare term).
- Add a metadata **filter** (e.g., only chunks from a given `source` or after a `timestamp`) to the query path.
- Swap the vector store behind your interface (Chroma → Qdrant, or → pgvector) and confirm nothing above the interface changes.
