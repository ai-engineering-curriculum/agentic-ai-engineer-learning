# Resources for mod-203-rag-and-memory

Primary references for RAG and agent memory. Verify against current docs — retrieval tooling and embedding models move fast.

## RAG and context engineering

- **Anthropic — Effective context engineering for AI agents** ([anthropic.com/engineering/effective-context-engineering-for-ai-agents](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents)) — just-in-time retrieval, what to keep in context vs. fetch on demand, and why eager loading rots. Read alongside Chapter 3.
- **Anthropic — Introducing contextual retrieval** ([anthropic.com/news/contextual-retrieval](https://www.anthropic.com/news/contextual-retrieval)) — prepending chunk-specific context before embedding, plus hybrid (embeddings + BM25) and re-ranking, with measured retrieval-error reductions.
- **Anthropic — Building effective agents** ([anthropic.com/engineering/building-effective-agents](https://www.anthropic.com/engineering/building-effective-agents)) — retrieval-as-a-tool framing; where RAG fits in an agent loop.

## Vector databases

- **pgvector** ([github.com/pgvector/pgvector](https://github.com/pgvector/pgvector)) — vector similarity search inside Postgres; SQL filtering and HNSW/IVFFlat indexes. Best when your data already lives in Postgres.
- **Chroma** ([docs.trychroma.com](https://docs.trychroma.com)) — in-process, near-zero-setup store for prototyping.
- **Qdrant** ([qdrant.tech/documentation](https://qdrant.tech/documentation)) — purpose-built vector DB with payload filtering, quantization, and tuned ANN; for scale and heavy filtering.

## Embeddings

- **OpenAI — Embeddings guide** ([platform.openai.com/docs/guides/embeddings](https://platform.openai.com/docs/guides/embeddings)) — embedding API, dimensions, and similarity basics.
- **Sentence-Transformers (SBERT)** ([sbert.net](https://www.sbert.net)) — local embedding and **cross-encoder re-ranker** models; the library behind much of exercises 01 and 03.
- **MTEB leaderboard** ([huggingface.co/spaces/mteb/leaderboard](https://huggingface.co/spaces/mteb/leaderboard)) — benchmark for choosing an embedding model by task, not vibes.

## Advanced retrieval

- **LlamaIndex — Sentence-window retrieval** ([docs.llamaindex.ai/en/stable/examples/node_postprocessor/MetadataReplacementDemo](https://docs.llamaindex.ai/en/stable/examples/node_postprocessor/MetadataReplacementDemo/)) — embed a sentence, return its surrounding window.
- **LlamaIndex — Auto-merging retriever** ([docs.llamaindex.ai/en/stable/examples/retrievers/auto_merging_retriever](https://docs.llamaindex.ai/en/stable/examples/retrievers/auto_merging_retriever/)) — hierarchical chunks; merge leaves up to the parent when enough match.

## Evaluation

- **RAGAS** ([docs.ragas.io](https://docs.ragas.io)) — reference-free and reference-based RAG metrics: context precision/recall, faithfulness (groundedness), answer relevance. Used in exercise-04.
- **TruLens — The RAG triad** ([trulens.org/getting_started/core_concepts/rag_triad](https://www.trulens.org/getting_started/core_concepts/rag_triad/)) — the canonical framing of context relevance, groundedness, and answer relevance as a diagnostic triangle.

## Memory (for comparison after you've built by hand)

- **LangGraph — Memory** ([langchain-ai.github.io/langgraph/concepts/memory](https://langchain-ai.github.io/langgraph/concepts/memory/)) — short-term (thread) vs. long-term (cross-thread) memory and consolidation, framework-side.
- **Mem0** ([docs.mem0.ai](https://docs.mem0.ai)) — a memory layer that packages write/recall and consolidation; recognize the tiers from Chapter 2 in its API.

> You implemented chunking, retrieval, memory tiers, and evaluation by hand in this module. When you adopt a framework or a managed memory layer, you'll recognize exactly which of these pieces it's packaging — and you'll debug it faster for having built it yourself. See [mod-202: Agent Frameworks in Practice](../mod-202-frameworks/README.md).
