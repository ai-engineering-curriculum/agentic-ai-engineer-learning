# Quiz 1 — RAG & Memory Implementation

Knowledge check for [mod-203](../README.md). Answers are at the bottom; try each question before scrolling. Covers all four chapters.

## Questions

### 1. The five stages of RAG

Chapter 1 frames RAG as five stages with a distinct failure mode at each. Which sequence matches the pipeline the chapter draws?

- A. Retrieve → embed → chunk → store → ground.
- B. Chunk → embed → store → retrieve → ground.
- C. Embed → chunk → ground → retrieve → store.
- D. Ground → retrieve → chunk → embed → store.

### 2. Two embedding models

A colleague embeds the corpus with `all-MiniLM-L6-v2` on Monday and, after a model refresh, embeds *queries* with `text-embedding-3-small` on Tuesday. Chapter 1 says the top-k results will be:

- A. Fine — any modern embedding model produces vectors in the same space.
- B. Fine as long as both models have the same output dimension.
- C. Noise — chunk vectors and query vectors come from different spaces, so cosine similarity between them is meaningless.
- D. Slightly worse but recoverable by re-normalizing both vector sets.

### 3. The blind spot of pure vector similarity

Chapter 1 recommends combining vector search with keyword (BM25) search — "hybrid search" — specifically to handle:

- A. Documents longer than the context window.
- B. Exact keywords, IDs, and rare terms that semantic similarity tends to miss.
- C. Multilingual corpora where a single embedding model under-performs.
- D. Compliance redaction of personally identifying information.

### 4. Where hallucination gets contained

In Chapter 1's five-stage pipeline, which stage is the chapter's **primary** hallucination control?

- A. Chunking — small enough chunks mean the model has less room to invent.
- B. Embedding — normalizing vectors prevents off-topic retrieval.
- C. Storing — payload filtering guarantees only authoritative chunks are indexed.
- D. Grounding — the prompt instructs the model to answer *only* from the retrieved context, cite sources, and say "I don't know" when the answer isn't there.

### 5. Which memory tier for which fact

A user tells the agent, at the start of a new session, "I prefer metric units." Per Chapter 2's three-tier model, where should this fact live so it still applies a week later in a fresh conversation?

- A. Working memory — it's about the current task, so it belongs on the scratchpad.
- B. Episodic memory — the message history already carries it forward.
- C. Long-term memory — durable preferences survive outside the context window, typically in a vector or KV store, and are recalled on demand.
- D. Nowhere — the user will restate it if they want it respected.

### 6. The failure mode of one bucket

Chapter 2 warns against collapsing all three tiers into one store. If you dump working, episodic, and long-term memory all into a single vector database, the chapter predicts the symptoms will be:

- A. Perfect recall and no data loss, at the cost of some extra storage.
- B. Overflow of the context window on long tasks, plus recall noise that drowns the current task.
- C. A security incident because long-term facts leak into other users' prompts.
- D. An index corruption that only shows up after millions of writes.

### 7. Episodic memory under growth

A conversation runs for 200 turns. The `EpisodicMemory` class from Chapter 2 is initialized with `keep_recent=10`. What is in `self.turns` and `self.summary` by turn 200?

- A. `self.turns` holds all 200 turns verbatim; `self.summary` is empty.
- B. `self.turns` holds the most recent 10 turns verbatim; `self.summary` is a running summary of the earlier 190.
- C. `self.turns` is empty; `self.summary` holds every turn ever seen as one giant string.
- D. The class raises an `OverflowError` after 128 turns.

### 8. JIT vs. eager — when eager still wins

Chapter 3 says just-in-time retrieval is the default for agent workloads, but names a workload where eager pre-loading is simpler and cheaper. Which workload is it?

- A. A corpus of millions of documents where each turn's needs are unpredictable.
- B. A workload where **almost every turn needs the same documents**, so paying the retrieval cost once up front beats a decide-then-search round trip per turn.
- C. A multi-user chat where each user has different long-term memories.
- D. Any workload with a hard 200 ms latency budget on the first token.

### 9. The round-trip cost of JIT

Which of the following is the honest trade-off Chapter 3 attributes to just-in-time retrieval?

- A. JIT reduces answer quality because the model never sees the full corpus.
- B. JIT adds an extra model round-trip when retrieval **is** needed (decide → search → answer instead of search-then-answer), in exchange for not paying retrieval cost on turns that don't need it.
- C. JIT requires a separate embedding model from eager retrieval.
- D. JIT only works with cross-encoder re-rankers.

### 10. Resolving contradictory facts

Your store returns two memories about the same user: (a) `"on the free tier"`, written yesterday, `source="user_stated"`, `confidence=0.9`; (b) `"upgraded to pro"`, written an hour ago, `source="agent_inferred"`, `confidence=0.4`. The `resolve` function in Chapter 3 keys on `(SOURCE_RANK[source], timestamp, confidence)`. Which memory wins?

- A. (b), because it is the most recent.
- B. (b), because confidence ties are broken by recency.
- C. (a), because `user_stated` outranks `agent_inferred` at the top of the key and the sort never falls through to timestamp or confidence.
- D. Neither — the function raises because the facts disagree.

### 11. Append-only memory rot

Chapter 3 says long-term memory should resolve conflicts not only on **recall** but also on **write**. What goes wrong if you treat the long-term store as append-only and only resolve on recall?

- A. Nothing — recall-time resolution is sufficient.
- B. Stale facts accumulate forever; every recall has to re-decide the same conflict, and a search can still surface the superseded memory next to the winner.
- C. Writes become O(n²) in the corpus size.
- D. The vector index silently drops the oldest row on every insert.

### 12. The retrieve/read mismatch

Chapter 4 opens with the core motivation for advanced retrieval. Which statement captures it?

- A. Retrieval models need GPUs; reading models do not.
- B. You want to match on small, focused units (one sentence embeds cleanly) but feed the model larger, coherent units (a paragraph reads coherently) — basic fixed chunking can't do both.
- C. Vector search and keyword search return different document orderings.
- D. Larger chunks always embed better than smaller chunks.

### 13. Sentence-window vs. auto-merging

Which pair correctly matches each advanced retriever to *what it indexes* and *what it returns*?

- A. Sentence-window indexes paragraphs and returns sentences; auto-merging indexes sentences and returns leaves.
- B. Sentence-window indexes sentences and returns a window of surrounding sentences; auto-merging indexes leaf chunks and returns the parent when enough children match.
- C. Both index whole documents; sentence-window returns the first sentence, auto-merging returns the last.
- D. Sentence-window indexes leaves and merges up; auto-merging indexes sentences and expands a window.

### 14. Why pair retrieval with a re-ranker

Chapter 4 recommends retrieving generously with vector search (e.g., top-20) and then re-ranking with a cross-encoder down to top-5. The split exists because:

- A. The cross-encoder is fast and the bi-encoder is slow, so you only run the bi-encoder on finalists.
- B. Vector search provides fast, approximate **recall** while a cross-encoder scores (query, chunk) pairs jointly for slower, more exact **precision** — you need both.
- C. Re-rankers are required to compute cosine similarity, and vector DBs cannot.
- D. The cross-encoder generates new chunk text that the bi-encoder cannot.

### 15. Reading the RAG triad together

A pipeline scores **high** context relevance, **low** groundedness, and (as a consequence) low answer relevance. Per Chapter 4's table, where is the fault?

- A. Retrieval — the wrong chunks are coming back; fix chunking, embedding, or k.
- B. Generation — retrieval found the right context but the model is ignoring it and hallucinating; fix the prompt or the model.
- C. Prompt — the model is grounded but answering the wrong question.
- D. Evaluation — the judge is miscalibrated and the numbers are meaningless.

### 16. LLM-as-judge discipline

Chapter 4 names two disciplines you must apply when scoring with an LLM-as-judge (RAGAS or hand-rolled). Which pair matches the chapter?

- A. Use the strongest available model as the judge; vary the eval set every run so the pipeline can't overfit.
- B. Use only cheap models as judges; discard any score below 0.5 as noise.
- C. Calibrate the judge against a handful of **human** labels, and **freeze** the evaluation set so a score change reflects a pipeline change, not a different test.
- D. Use the same model as the generator so scores are self-consistent; rotate the eval set weekly.

## Answer key

1. **B** — Chunk → embed → store → retrieve → ground; each stage has its own failure mode ([Chapter 1](../01-rag-fundamentals.md)).
2. **C** — "The model that embeds your chunks **must** be the same one that embeds your queries — mixing embedding models means comparing vectors from different spaces, which is just noise" ([Chapter 1](../01-rag-fundamentals.md)).
3. **B** — Pure vector similarity misses exact keywords, IDs, and rare terms; hybrid search fuses vector and BM25 to cover that blind spot ([Chapter 1](../01-rag-fundamentals.md)).
4. **D** — Grounding is where hallucination is contained: instructions to answer only from the context, cite, and say "I don't know" are the primary control ([Chapter 1](../01-rag-fundamentals.md)).
5. **C** — Durable preferences live in long-term memory outside the context window and are recalled on demand in a later session ([Chapter 2](../02-agent-memory.md)).
6. **B** — Collapsing the tiers produces overflow (too much in context) and recall noise (vector store returns drown the current task) ([Chapter 2](../02-agent-memory.md)).
7. **B** — `add()` rolls everything past `keep_recent` into `self.summary` and keeps the last 10 turns verbatim ([Chapter 2](../02-agent-memory.md)).
8. **B** — "JIT wins when retrieval is *sometimes* needed and the corpus is large… For a workload where almost every turn needs the same documents, eager pre-loading is simpler and cheaper" ([Chapter 3](../03-jit-retrieval-conflict.md)).
9. **B** — JIT adds a decide → search → answer round-trip when retrieval *is* needed; that's the price paid for not stuffing the prompt every turn ([Chapter 3](../03-jit-retrieval-conflict.md)).
10. **C** — The `resolve` key is `(SOURCE_RANK[source], timestamp, confidence)` in that order; `user_stated` (rank 2) outranks `agent_inferred` (rank 1) at the top of the tuple, so (a) wins before timestamp or confidence is consulted ([Chapter 3](../03-jit-retrieval-conflict.md)).
11. **B** — "Append-only memory stores rot; consolidation that updates and invalidates is what keeps them coherent" — stale memories otherwise keep resurfacing ([Chapter 3](../03-jit-retrieval-conflict.md)).
12. **B** — "You want to *match* on small, focused units … but *feed the model* larger, coherent units" — the retrieve/read mismatch is what advanced retrieval resolves ([Chapter 4](../04-advanced-rag-eval.md)).
13. **B** — Sentence-window embeds sentences and returns a window around the hit; auto-merging embeds leaves but merges up to a parent once enough children are retrieved ([Chapter 4](../04-advanced-rag-eval.md)).
14. **B** — Bi-encoder vector search gives fast approximate recall; a cross-encoder scores (query, chunk) jointly for slow exact precision. The pair covers both ends ([Chapter 4](../04-advanced-rag-eval.md)).
15. **B** — High context relevance means retrieval delivered the right chunks; low groundedness means the model ignored them and hallucinated. That's the generation row of the Chapter 4 localization table ([Chapter 4](../04-advanced-rag-eval.md)).
16. **C** — The chapter's two cautions: calibrate the LLM judge against human labels, and hold the eval set fixed so a score delta means a pipeline delta ([Chapter 4](../04-advanced-rag-eval.md)).
