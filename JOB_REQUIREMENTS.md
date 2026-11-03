# Job Requirements — Agentic AI Engineer

**Role level:** 30 (build altitude)
**Track:** `agentic-ai-engineer-learning`
**Research window:** 2026-08-05 → 2026-11-03 (last 90 days)
**Today:** 2026-11-03
**Prior refresh:** 2026-10-03

This file maps verbatim requirements from current Agentic AI Engineer job postings to the existing curriculum. Raw normalized data lives in [`.aicg/job-requirements.json`](.aicg/job-requirements.json); the strictly-additive proposal lives in [`.aicg/curriculum-plan-delta.json`](.aicg/curriculum-plan-delta.json).

## Summary

- Postings sampled: **28** in-window (2026-08-05 → 2026-11-03). Twenty-three fresh pulls this cycle plus five still-active prior-cycle anchors re-verified in window. Ashby-hosted boards (Benchling, Snorkel AI, Dataiku, Extend, PairTeam, Translucent, AssetWatch, Hasbro, Spotter, Carrot, Splitero, Baselayer sub-postings) continued to render only skeleton markup to WebFetch; several promising employers (Devoteam AWS AI Engineer, Leidos AWS Generative AI Solution Engineer) returned 403 on direct fetch and contributed title/snippet evidence only via search-result excerpts. Materialbank root URL hit a 302 redirect loop on the Greenhouse index. Dice.com job URLs continue to expire within ~30 days. Sample size (28) comfortably exceeds the 25-posting minimum.
- Equivalent titles counted: `Agentic AI Engineer`, `AI Agent Engineer`, `Agent Engineer`, `Applied AI Engineer`, `Generative AI Engineer`, `LLM Engineer`. Title mix: 11 Agentic AI Engineer and close variants (*Senior AI / Agentic Engineer*, *Senior Python / Agentic AI Engineer*, *Agentic Engineer*, *AI Engineer | Agentic Systems*, *AI Engineer | Agentic AI, AWS Bedrock AgentCore*, *Agentic Integration & MCP Engineer*), 2 Senior Agentic AI Software Engineer, 7 Applied AI Engineer (NewsBreak x1, Material Bank, Fanatics, Anthropic x3 — Beneficial Deployments / Public Sector / Startups Physical AI, Future-carryover), 4 AI Engineer / AI Platform Engineer (NewsBreak Agent Platform, Pushpay, TMSLLC, 6sense), 2 Software Engineer (Foresite Labs AI Agents; Anthropic Agent SDK Claude Code), 1 Senior AI Engineer, Agents & Applications (Firmus), 1 Senior AI Engineer Security Infrastructure (Air), 1 Staff Software Engineer GTM AI Engineering (Anthropic), 1 Senior Forward Deployed AI Engineer, Data Platform & Agents (Indicium), 1 Staff+ SE Claude Managed Agents (Anthropic), 1 Data Engineer – Applied ML (Similarweb — classification/retrieval + LangGraph pipelines).
- **Proposed delta this cycle: 0 modules, 0 exercises, 0 projects.** Every requirement above the continuity-bias threshold (≥3 distinct postings AND ≥30% frequency) is already owned by an existing module in `mod-201..209`. This is the **fourth consecutive zero-additions refresh** (2026-08-03, 2026-09-03, 2026-10-03, 2026-11-03).
- Three notable shifts this cycle, all already covered:
  - **"Context engineering" vocabulary crossed threshold: 21% → ~54%** (15 of 28 postings). AspenView ("Engineer the context: retrieval and RAG strategy, scoping, structured summaries, and pruning rules"), Anthropic GTM ("context engineering, agent development, MCP development"), Anthropic Managed Agents ("manages context (caching, compaction, memory)"), NewsBreak Agent Platform ("Own the context engineering layer — retrieval, ranking, compression, and memory"), CodeRoad new ("dynamic, on-demand context-loading mechanisms"), Deloitte Senior Anthropic, LTS Senior Agentic, Machinify, and Anthropic Startups Physical AI all use the term explicitly or structurally. **This is a vocabulary shift, not a content shift.** The mechanics — prompt design, retrieval shaping, compaction, memory curation, prompt caching, token budgets — are already taught across [`mod-201-agent-fundamentals`](lessons/mod-201-agent-fundamentals), [`mod-203-rag-and-memory`](lessons/mod-203-rag-and-memory), [`mod-207-productionizing-agents/exercises/exercise-03-caching-and-routing.md`](lessons/mod-207-productionizing-agents/exercises/exercise-03-caching-and-routing.md), and [`mod-209-building-custom-harnesses/exercises/exercise-04-context-and-session-state.md`](lessons/mod-209-building-custom-harnesses/exercises/exercise-04-context-and-session-state.md). Surface the vocabulary in mod-201 and mod-203 READMEs on next content pass.
  - **"Harness" vocabulary emerged explicitly in top-tier postings** (~18% — sub-threshold). Anthropic's Claude Managed Agents posting now uses the word structurally: *"The harness is the loop that calls Claude, routes tool calls, manages context (caching, compaction, memory), and recovers from errors"* and asks for candidates who have *"built or contributed to an agent harness, agent framework, or LLM orchestration layer"*. NewsBreak Agent Platform asks for ownership of *"the eval and observability harness that runs thousands of agent traces per day"*. Foresite Labs asks for *"the eval and observability harness — the tests, metrics, tracing, and guardrails that make agentic systems trustworthy"*. Baselayer (title-only via search) describes a *"shared toolset registry... eval harness, and instrumentation surface for token-and-tool tracing."* This is validation that the newly-shipped [`mod-208-harness-engineering`](lessons/mod-208-harness-engineering) and [`mod-209-building-custom-harnesses`](lessons/mod-209-building-custom-harnesses) are correctly positioned. No delta.
  - **Observability steady at 54–61%** (consistent with 54% in the 2026-10 cycle). Named stacks consistent with the prior cycle (Langfuse, LangSmith, Phoenix/Arize, OpenTelemetry GenAI, Datadog, Braintrust, Weave, TruLens, Logfire). Firmus's exhaustive list — *"OpenTelemetry, OpenInference, LangSmith, Langfuse, Arize Phoenix, Weave, TruLens, Ragas, DeepEval, promptfoo"* — is a canonical reference. Already covered by [`mod-205-evaluation-observability/exercises/exercise-02-otel-tracing-wireup.md`](lessons/mod-205-evaluation-observability/exercises/exercise-02-otel-tracing-wireup.md).
- Four tracked sub-threshold signals this cycle:
  - **AWS Bedrock AgentCore + Strands stabilized at ~11% explicit** (TMSLLC, Crexi, AspenView). If "AWS Bedrock" broadly (any flavor) is counted, this reaches ~25% (adds Pushpay, Redapt-carryover, Amtech-carryover, Sharebite-carryover). The reversal flagged last cycle (22% → 14%) did not reverse further. Still sub-threshold.
  - **MCP OAuth 2.1 / credential-mgmt for agents and MCP servers** remains flat at ~4% explicit. Only Capgemini — whose role is literally titled "Agentic Integration & MCP Engineer" — names *"OAuth 2.0, OpenID Connect"* alongside *"identity, authorization, secrets management, auditability and guardrails for agent actions."* This is the **fourth consecutive cycle flagging MCP auth as the strongest next-cycle candidate** following the 2026-07-28 MCP spec change. The 2026 AWS MCP Server OAuth support announcement and the ServiceNow / WorkOS writeups continue to lead requirement wording.
  - **Managed agent runtime pattern** (Anthropic Managed Agents, AWS Bedrock AgentCore, LangGraph Platform family) continuing to converge but requirement language still fragmented. Anthropic's Claude Managed Agents posting is the clearest anchor: *"long-lived, stateful sessions that execute autonomously for minutes or hours/days, persist through disconnections, and resume cleanly."* Already covered by [`mod-207-productionizing-agents/exercises/exercise-02-durable-execution-temporal.md`](lessons/mod-207-productionizing-agents/exercises/exercise-02-durable-execution-temporal.md).
  - **Voice / realtime agents** inched up (ElevenLabs Applied AI, LiveKit Upwork anchor from July 26, external to the sample). Still cordoned into specialty roles. mod-208 voice-agent sibling remains unwarranted.

## Methodology

- Sources: greenhouse.io/job_boards (bulk, full page fetches), jobs.ashbyhq.com (title-only extraction where JS-only rendering blocked WebFetch), apply.deloitte.com, careers.boozallen.com, careers.capgemini.com, freehire.me, directlyapply.com, upwork.com (voice-agent reference only — not counted toward sample), gravityer.com, elevenlabs.io/careers (title-only), zapply.jobs, careers.anduril.com (redirect), job-boards.eu.greenhouse.io.
- Per-posting capture: employer, title, URL, date_observed, date_posted (marked `estimated:YYYY-MM` when inferred), location, 4–10 verbatim/near-verbatim requirement quotes.
- Frequency = distinct in-window postings citing the theme ÷ 28 in-window postings.
- Threshold for a new curriculum item: ≥ 3 postings AND ≥ 30% frequency AND no existing module/exercise can be incrementally extended to cover it.
- Source caveats documented in `summary.source_quality_caveats` in `.aicg/job-requirements.json`.

## Requirement themes → curriculum ownership

The table below lists every requirement theme observed, its in-window frequency, the role that owns it per the level hierarchy, and the curriculum coverage path. **Bold frequencies** are ≥ 30% (load-bearing under continuity bias).

| # | Theme | Freq | Owner role | Coverage |
|---|---|---|---|---|
| 1 | Agent frameworks: LangGraph, LangChain, CrewAI, AutoGen, OpenAI Agents SDK, Google ADK, Anthropic/Claude Agent SDK, LlamaIndex, Semantic Kernel, PydanticAI, Haystack, DSPy, Mastra, Vercel AI SDK, smolagents, Strands | **79%** | `agentic-ai-engineer` (this) | [`mod-202-frameworks`](lessons/mod-202-frameworks) |
| 2 | RAG, vector DBs, embeddings, agent memory | **71%** | `agentic-ai-engineer` | [`mod-203-rag-and-memory`](lessons/mod-203-rag-and-memory) |
| 3 | Multi-agent orchestration (orchestrator-worker, handoffs, sub-agents, agent-to-agent, state-graph, swarm) | **50%** | `agentic-ai-engineer` | [`mod-204-multi-agent-implementation`](lessons/mod-204-multi-agent-implementation) |
| 4 | Evaluation harnesses (LLM-as-judge, regression suites, golden traces, CI-integrated eval bar, replay, simulation) | **96%** | `agentic-ai-engineer` | [`mod-205-evaluation-observability`](lessons/mod-205-evaluation-observability) |
| 5 | Observability (Langfuse, LangSmith, Phoenix, Arize, OTel GenAI, OpenInference, Datadog, Braintrust, Weave, Logfire) | **61%** | `agentic-ai-engineer` | [`mod-205-evaluation-observability/exercises/exercise-02-otel-tracing-wireup.md`](lessons/mod-205-evaluation-observability/exercises/exercise-02-otel-tracing-wireup.md) |
| 6 | Function / tool calling, structured outputs (Pydantic / JSON schema) | **54%** | `agentic-ai-engineer` | [`mod-201-agent-fundamentals/exercises/exercise-02-function-calling-tools.md`](lessons/mod-201-agent-fundamentals/exercises/exercise-02-function-calling-tools.md) |
| 7 | Model Context Protocol (MCP) — consumption AND authoring | **61%** | `agentic-ai-engineer` | [`mod-202-frameworks/exercises/exercise-04-mcp-tool-server.md`](lessons/mod-202-frameworks/exercises/exercise-04-mcp-tool-server.md) |
| 8 | API deployment (FastAPI/Flask, REST/GraphQL/gRPC, async Python, Docker/K8s/IaC, CI/CD) | **93%** | `agentic-ai-engineer` | [`mod-207-productionizing-agents/exercises/exercise-01-agent-api-deployment.md`](lessons/mod-207-productionizing-agents/exercises/exercise-01-agent-api-deployment.md) |
| 9 | Guardrails, prompt injection defense, safety/policy controls, permissions, audit logs | **39%** | `agentic-ai-engineer` (basics) → `ai-infra-security-learning` (depth) | [`mod-206-guardrails-implementation`](lessons/mod-206-guardrails-implementation); deep agent attack surface → `ai-infra-security-learning` (level 35) |
| 10 | Durable execution / workflow engines (Temporal, Airflow, Prefect, managed agent runtimes) | 14% | `agentic-ai-engineer` | [`mod-207-productionizing-agents/exercises/exercise-02-durable-execution-temporal.md`](lessons/mod-207-productionizing-agents/exercises/exercise-02-durable-execution-temporal.md) |
| 11 | Cost / latency optimization (prompt caching, model routing, token budgets, per-tenant quotas) | **36%** | `agentic-ai-engineer` | [`mod-207-productionizing-agents/exercises/exercise-03-caching-and-routing.md`](lessons/mod-207-productionizing-agents/exercises/exercise-03-caching-and-routing.md) |
| 14 | Human-in-the-loop oversight / escalation / approval checkpoints | **50%** | `agentic-ai-engineer` | [`mod-206-guardrails-implementation/exercises/exercise-04-human-approval-checkpoints.md`](lessons/mod-206-guardrails-implementation/exercises/exercise-04-human-approval-checkpoints.md), [`mod-207-productionizing-agents/exercises/exercise-04-hitl-with-persistence.md`](lessons/mod-207-productionizing-agents/exercises/exercise-04-hitl-with-persistence.md) |
| 16 | Voice / realtime / telephony agents | 11% | sub-threshold | External link-outs (below); mod-208 voice-agent sibling still unwarranted |
| 18 | AWS-native agent runtime (Bedrock AgentCore, Strands Agents) | 11% (explicit) / 25% (AWS Bedrock any flavor) | sub-threshold | [`mod-202-frameworks/exercises/exercise-05-framework-tradeoff-bakeoff.md`](lessons/mod-202-frameworks/exercises/exercise-05-framework-tradeoff-bakeoff.md) — comparison line item sufficient |
| 19 | Google ADK + AgentSpace as named production stack | 4% | `agentic-ai-engineer` | Already in mod-202 objectives & exercise-05 comparison scope |
| 21 | A2A (Agent-to-Agent) protocol — inter-agent communication protocol | 7% | `agentic-ai-engineer` | [`mod-204-multi-agent-implementation`](lessons/mod-204-multi-agent-implementation) — agent-to-agent protocols already in module scope |
| 22 | "Context engineering" as named discipline distinct from prompt engineering | **54%** | `agentic-ai-engineer` | Vocabulary shift, not content shift — mechanics already taught across [`mod-201`](lessons/mod-201-agent-fundamentals), [`mod-203`](lessons/mod-203-rag-and-memory), [`mod-207/exercise-03`](lessons/mod-207-productionizing-agents/exercises/exercise-03-caching-and-routing.md), [`mod-209/exercise-04`](lessons/mod-209-building-custom-harnesses/exercises/exercise-04-context-and-session-state.md). Surface the term in mod-201/mod-203 READMEs on next content pass. |
| 23 | Agent harness engineering (harness as named concept — loop, tool dispatch, context mgmt, instrumentation) | 18% | `agentic-ai-engineer` | [`mod-208-harness-engineering`](lessons/mod-208-harness-engineering) (user side) + [`mod-209-building-custom-harnesses`](lessons/mod-209-building-custom-harnesses) (build-your-own). Correctly positioned — this is the first cycle where "harness" is used as a top-level requirement word by top-tier postings (Anthropic, NewsBreak, Foresite, Baselayer). |
| 24 | Credential-mgmt / OAuth 2.1 / RBAC / scoped tokens for agents and MCP servers | 4% (explicit OAuth) / 21% (broader credential-mgmt/identity/least-privilege) | `agentic-ai-engineer` (basics) → `ai-infra-security-learning` (depth) | Still sub-threshold on explicit OAuth; **fourth consecutive cycle flagging MCP OAuth 2.1 as the strongest next-cycle candidate** following the 2026-07-28 MCP spec |

## Posting evidence for load-bearing themes

The table below lists the postings that anchor each ≥30% theme in this window. All 28 sampled postings fall within 2026-08-05 → 2026-11-03.

### Theme 1 — Agent frameworks (79%)

| Employer | Title | URL | Date observed | Posted |
|---|---|---|---|---|
| CodeRoad | Agentic AI Engineer | https://job-boards.greenhouse.io/coderoad/jobs/4414020009 | 2026-11-03 | est:2026-10 |
| CodeRoad | Lead Agentic AI Engineer | https://job-boards.greenhouse.io/coderoad/jobs/4110608009 | 2026-11-03 | est:2026-09 (still active) |
| Machinify | AI Engineer \| Agentic Systems (L4) | https://job-boards.greenhouse.io/machinifyinc/jobs/4146862009 | 2026-11-03 | est:2026-09 (still active) |
| AspenView Technology Partners | Agentic Engineer | https://job-boards.greenhouse.io/aspenviewtech/jobs/4433524009 | 2026-11-03 | est:2026-10 |
| Anthropic | Staff Software Engineer, GTM AI Engineering | https://job-boards.greenhouse.io/anthropic/jobs/5390966008 | 2026-11-03 | est:2026-10 |
| Anthropic | Software Engineer, Agent SDK (Claude Code) | https://job-boards.greenhouse.io/anthropic/jobs/5126691008 | 2026-11-03 | est:2026-09 |
| Anthropic | Staff+ Software Engineer, Claude Managed Agents | https://job-boards.greenhouse.io/anthropic/jobs/5395767008 | 2026-11-03 | est:2026-09 |
| Anthropic | Applied AI Engineer, Beneficial Deployments (Life Sciences) | https://job-boards.greenhouse.io/anthropic/jobs/5437172008 | 2026-11-03 | est:2026-10 |
| Anthropic | Applied AI Engineer, Public Sector | https://job-boards.greenhouse.io/anthropic/jobs/5445410008 | 2026-11-03 | est:2026-10 |
| Anthropic | Applied AI Engineer, Startups (Physical AI) | https://job-boards.greenhouse.io/anthropic/jobs/5454673008 | 2026-11-03 | est:2026-10 |
| Indicium AI | Senior Forward Deployed AI Engineer, Data Platform & Agents | https://job-boards.greenhouse.io/indiciumai/jobs/4944657101 | 2026-11-03 | est:2026-10 |
| Pushpay | Staff AI Platform Engineer | https://job-boards.greenhouse.io/pushpay/jobs/8014545 | 2026-11-03 | est:2026-09 |
| Firmus Technologies | Senior AI Engineer, Agents & Applications | https://job-boards.greenhouse.io/firmus/jobs/5417755008 | 2026-11-03 | est:2026-10 |
| Material Bank | Applied AI Engineer | https://job-boards.greenhouse.io/materialbank/jobs/7732615003 | 2026-11-03 | est:2026-10 |
| NewsBreak | AI Engineer, Agent Platform | https://job-boards.greenhouse.io/newsbreak/jobs/4683873006 | 2026-11-03 | est:2026-09 |
| 6sense | Sr. AI Engineer | https://job-boards.greenhouse.io/6sense/jobs/8304080 | 2026-11-03 | est:2026-09 |
| TMSLLC | AI Engineer \| Agentic AI, AWS Bedrock AgentCore & Langfuse | https://freehire.me/jobs/hiring-ai-engineer-generative-ai-engineer-agentic-ai-aws-bedrock-agentcore-claude-agent-sdk-langfuse-los-angeles-ca-hybrid-tmsllc-mbgrjfbr | 2026-11-03 | 2026-10-19 (re-posted 2026-10-30) |
| Capgemini | Agentic Integration & MCP Engineer | https://careers.capgemini.com/job/Wrocław-Agentic-Integration-&-MCP-Engineer/1439557833/ | 2026-11-03 | 2026-10-20 |
| Foresite Labs | Software Engineer, AI Agents | https://job-boards.greenhouse.io/foresitelabs/jobs/8533284002 | 2026-11-03 | est:2026-10 |
| Similarweb | Data Engineer – Applied ML | https://job-boards.greenhouse.io/similarweb/jobs/8239378 | 2026-11-03 | est:2026-10 |
| Fanatics | Applied AI Engineer | https://job-boards.greenhouse.io/fanaticsinc/jobs/4457595009 | 2026-11-03 | est:2026-10 |
| Capco | Senior Python / Agentic AI Engineer | https://job-boards.greenhouse.io/capco/jobs/7778007 | 2026-11-03 | est:2026-10 |
| LTS | Senior Agentic AI Software Engineer | https://job-boards.greenhouse.io/lts/jobs/4340374009 | 2026-11-03 | est:2026-10 |
| Air | Senior AI Engineer, Security Infrastructure | https://job-boards.greenhouse.io/air/jobs/4394418009 | 2026-11-03 | est:2026-10 |
| Deloitte | Agentic AI Engineer, Senior — Anthropic/Claude | https://apply.deloitte.com/en_US/careers/JobDetail/Agentic-AI-Engineer-Senior-Anthropic-Claude/363947 | 2026-11-03 | est:2026-09 (recruiting ends 2026-10-30) |
| Booz Allen Hamilton | Agentic AI Forward-Deployed Engineer | https://careers.boozallen.com/jobs/JobDetail/McLean-Agentic-AI-Forward-Deployed-Engineer-R0250142/130862 | 2026-11-03 | est:2026-09 |
| Crexi | Senior AI Engineer | https://job-boards.greenhouse.io/crexi/jobs/4723615005 | 2026-11-03 | est:2026-09 |
| Artefact North America | Senior, AI & Agentic Engineer | https://job-boards.greenhouse.io/artefactnoram/jobs/8731948002 | 2026-11-03 | est:2026-09 |

Representative quote (fresh this cycle): *"Build orchestration workflows using appropriate agent frameworks and libraries, such as LangGraph, LangChain, LlamaIndex, Microsoft AutoGen, Semantic Kernel, CrewAI, PydanticAI, Haystack, DSPy, or equivalent custom-built frameworks."* — Firmus Technologies.

→ Covered by [`mod-202-frameworks`](lessons/mod-202-frameworks). Mastra joins the long tail this cycle (Material Bank), DSPy and Haystack get first-class mentions (Firmus); none shift the comparison bakeoff shape.

### Theme 2 — RAG, vector DBs, memory (71%)

Anchored by AspenView new, Anthropic Life Sciences, Indicium (hybrid RAG over Databricks), Pushpay, Firmus (short-term/long-term/episodic/semantic memory enumeration), Material Bank, NewsBreak NewGrad, NewsBreak Agent Platform (retrieval+ranking+compression+memory), 6sense, TMSLLC, Capgemini, Foresite Labs, Similarweb, Fanatics, Capco, LTS Senior Agentic, Deloitte Senior Anthropic, Booz Allen FDE, Crexi. Named tools consistent with prior cycle: pgvector, Pinecone, Weaviate, Qdrant, Milvus, FAISS, Chroma, Azure AI Search; graph layer: Neo4j, NebulaGraph.

Representative quote: *"Build Retrieval-Augmented Generation (RAG) pipelines using a combination of vector retrieval, hybrid search, metadata filtering, reranking, structured-data queries, graph-based retrieval where valuable."* — Firmus Technologies.

→ Covered by [`mod-203-rag-and-memory`](lessons/mod-203-rag-and-memory).

### Theme 3 — Multi-agent orchestration (50%)

Anchored by CodeRoad new ("hierarchical multi-agent workflows"), CodeRoad Lead, AspenView new (multi-agent + sub-agent decomposition), Anthropic Managed Agents ("multi-agent coordination"), Indicium (multi-agent systems over enterprise data), Pushpay (supervisor loops, router patterns, state machines), Firmus (sub-agents), Material Bank (multi-agent workflows and orchestration), Foresite Labs (sub-agent orchestration), LTS Senior Agentic, Deloitte Senior Anthropic, Booz Allen FDE, Booz Allen Alexandria (A2A explicit), Crexi.

Representative quote: *"Build orchestration workflows using appropriate agent frameworks... Define reference architectures for single-agent, multi-agent, workflow-based, event-driven, and human-in-the-loop agentic systems."* — Firmus Technologies.

→ Covered by [`mod-204-multi-agent-implementation`](lessons/mod-204-multi-agent-implementation). A2A protocol surface (PhysicsX-carryover, Booz Allen Alexandria) still fits `exercise-02-agent-handoffs`.

### Theme 4 — Evaluation harnesses (96%)

Essentially universal in the sample. Anchored by every posting that discusses production agent operation. The clearest fresh anchors are TMSLLC (*"Design and implement continuous evaluation frameworks... Develop automated evaluation pipelines using Langfuse, LLM-as-a-Judge, and custom evaluation metrics... Integrate automated evaluations into CI/CD pipelines as quality gates"*), Anthropic Managed Agents (*"Build the eval infrastructure that measures harness quality against research baselines and real customer workloads"*), Anthropic GTM (*"Stand up the eval suite for an agent: seed scenarios, scoring rubrics, and regression runs on every change"*), and Firmus (full enumeration — *"automated tests, curated test sets, simulation, replay, benchmark tasks, regression suites, model-based evaluators, human review, and operational-outcome measures"*).

→ Covered by [`mod-205-evaluation-observability`](lessons/mod-205-evaluation-observability). The replay/simulation phrasing in Firmus overlaps with [`mod-209-building-custom-harnesses/exercises/exercise-06-observability-replay-and-shipping.md`](lessons/mod-209-building-custom-harnesses/exercises/exercise-06-observability-replay-and-shipping.md); no delta.

### Theme 5 — Observability (61%)

Steady from 54% last cycle; the named stack has not shifted. Anchored by Firmus (full enumeration), TMSLLC (*"Deploy, configure, and maintain Langfuse for comprehensive LLM and agent observability... Integrate Langfuse with AI agents to capture traces, prompts, responses, tool calls, and execution metrics"*), Capgemini (OpenTelemetry/Langfuse/Arize Phoenix), 6sense (*"Expand tracing and usage analytics, and report on task success rate, tool-call precision, groundedness, latency, and cost per successful outcome"*), AspenView new, Anthropic GTM, NewsBreak Agent Platform, CodeRoad new, Pushpay, Foresite Labs, Fanatics, LTS, Capco, Deloitte Senior Anthropic, Booz Allen FDE, Crexi, Artefact.

Representative quote: *"Hands-on experience with agent evaluation and observability tools or patterns, such as OpenTelemetry, OpenInference, LangSmith, Langfuse, Arize Phoenix, Weave, TruLens, Ragas, DeepEval, promptfoo."* — Firmus Technologies.

→ Covered by [`mod-205-evaluation-observability/exercises/exercise-02-otel-tracing-wireup.md`](lessons/mod-205-evaluation-observability/exercises/exercise-02-otel-tracing-wireup.md).

### Theme 6 — Function / tool calling, structured outputs (54%)

Anchored by CodeRoad new, AspenView new, Anthropic GTM, Anthropic Managed Agents, Indicium (*"function calling, and structured JSON schemas"*), 6sense (*"APIs and tool interfaces for LLM callers, including schemas, descriptions, error handling, and tool-selection behavior"*), Firmus, Foresite Labs, Fanatics (*"MCP servers, agent SDKs, orchestration tools, browser and computer use, and evals"*), Material Bank, NewsBreak NewGrad, Machinify, LTS Senior Agentic, Deloitte Senior Anthropic, Booz Allen FDE. Climbed from 39% — a real move driven by the increasing structural specificity of enterprise JDs.

Representative quote: *"Experience designing structured outputs (Pydantic / JSON Schema) and tool interfaces."* — Machinify.

→ Covered by [`mod-201-agent-fundamentals/exercises/exercise-02-function-calling-tools.md`](lessons/mod-201-agent-fundamentals/exercises/exercise-02-function-calling-tools.md).

### Theme 7 — MCP (61%)

Up from 50%. The pattern has fully shifted from *"uses MCP tools"* to *"builds, hosts, and secures MCP servers"*. Capgemini's role is literally titled "Agentic Integration & MCP Engineer." Anchored by CodeRoad new, AspenView new, Anthropic GTM (*"Ship MCP servers, agent skills, and web applications that connect to systems like our CRM"*), Anthropic Life Sciences (*"Develop reusable ecosystem infrastructure, like MCP servers for domain-specific data sources"*), Anthropic Public Sector, Indicium (*"Architect and deploy custom data interfaces and tool servers using the Model Context Protocol (MCP), function calling, and structured JSON schemas"*), Pushpay (*"custom MCP implementations"*), NewsBreak NewGrad, 6sense (*"Redesign and scale the MCP layer for production workloads, including concurrency, per-tenant quotas, latency budgets, caching, and graceful degradation"*), TMSLLC (good-to-have), Capgemini (role), Fanatics, CodeRoad Lead, Deloitte Senior Anthropic, Booz Allen FDE, Artefact. 6sense's phrasing — *"Redesign and scale the MCP layer for production workloads"* — is the clearest signal that MCP has become infrastructure, not a feature.

Representative quote: *"design and develop MCP servers, agent tools and secure connectors to enterprise systems... Strong understanding of REST, OpenAPI, OAuth 2.0, OpenID Connect, event-driven architecture, messaging and secure system integration."* — Capgemini.

→ Covered by [`mod-202-frameworks/exercises/exercise-04-mcp-tool-server.md`](lessons/mod-202-frameworks/exercises/exercise-04-mcp-tool-server.md), which includes an authoring lab. **Watchpoint carried into the next cycle:** the 2026-07-28 MCP spec mandates OAuth 2.1 + PKCE + RFC 8707 Resource Indicators for remote MCP servers; that explicit language still only appears in one in-window posting (Capgemini). This is the **fourth consecutive cycle flagging it as the strongest next-cycle candidate** — if MCP-server authoring with enterprise auth crosses 30% in 2027-01, propose a mod-206 exercise-05 or mod-202 exercise-04 extension focused on auth-wrapped MCP servers.

### Theme 8 — API deployment / backend / containerization (93%)

Essentially universal; near every posting names a cloud (AWS/GCP/Azure), a container runtime (Docker/Kubernetes/ECS), IaC (Terraform/CDK/Pulumi), and a CI/CD stack. Anchored by nearly every in-window posting.

Representative quote: *"Deep proficiency in C# or Python paired with strong AWS cloud-native expertise (Lambda, API Gateway, Bedrock, IAM/KMS)."* — Pushpay.

→ Covered by [`mod-207-productionizing-agents/exercises/exercise-01-agent-api-deployment.md`](lessons/mod-207-productionizing-agents/exercises/exercise-01-agent-api-deployment.md). IaC (Terraform, CDK, Pulumi) surfaces alongside deployment but stays owned by `ai-infra-platform-engineer-learning` per hierarchy.

### Theme 9 — Guardrails / safety / policy (39%)

Anchored by Air (full enumeration of agent attack surface: *"prompt injection, indirect prompt injection, tool abuse, privilege escalation, data exfiltration, context or memory poisoning, and unintended agent behavior"*), AspenView new (*"prompt injection, untrusted tool output, secret leakage, and unsafe autonomous actions"*), Firmus (*"direct and indirect prompt injection, insecure output handling, excessive agency, unsafe tool use, data leakage"*), CodeRoad new (OWASP LLM10 explicit), LTS Senior Agentic (prior-cycle Agentic AI Security Engineer carryover), Pushpay (*"guardrails to ensure autonomous systems degrade safely"*), Indicium (*"multi-tenant isolation, row/column-level permissions, and zero-data-leakage constraints"*), Capgemini, Deloitte Senior Anthropic, Deloitte Healthcare (carryover), Anduril, BPD, Benchling (carryover).

Representative quote: *"Design adversarial evaluations targeting threats such as prompt injection, indirect prompt injection, tool abuse, privilege escalation, data exfiltration, context or memory poisoning, and unintended agent behavior."* — Air.

→ Covered by [`mod-206-guardrails-implementation`](lessons/mod-206-guardrails-implementation): four exercises cover I/O moderation, prompt-injection defenses (OWASP LLM01), tool-permission enforcement, human-approval checkpoints. Air's deeper threat-model work (adversarial testing / red-teaming / sandboxed execution) is depth-of-depth territory and remains owned by the planned `ai-infra-security-learning` track at level 35.

### Theme 11 — Cost / latency optimization (36%)

Climbed from 21% — crossed the 30% threshold for the first time. Anchored by 6sense (*"concurrency, per-tenant quotas, latency budgets, caching, and graceful degradation... cost per successful outcome"*), CodeRoad new (*"token costs"* explicit), CodeRoad Lead (*"prompt caching"*), Future (prompt caching + token budgets + retry logic — carryover), Pushpay (*"cost governance... token efficiency... SLOs for agent success rates, latency, and token efficiency"*), Firmus (*"token and cost tracking"*), Anduril (prompt caching — carryover), Anthropic Managed Agents (*"outcome-driven execution where developers specify success criteria and a budget"*), Supermetrics (carryover). The theme's elevation reflects real operational pressure — production agent deployments are now spending enough on tokens that cost per successful outcome and per-tenant latency SLOs are first-class requirements.

Representative quote: *"Establish SLOs for agent success rates, latency, and token efficiency. Take point on observability, prompt versioning, and incident response when agents misbehave or hallucinate."* — Pushpay.

→ Covered by [`mod-207-productionizing-agents/exercises/exercise-03-caching-and-routing.md`](lessons/mod-207-productionizing-agents/exercises/exercise-03-caching-and-routing.md). The exercise already includes a cost-per-successful-outcome metric and per-tenant routing policy; no delta.

### Theme 14 — Human-in-the-loop oversight (50%)

Climbed from 39% last cycle. 14 of 28 postings explicitly require HITL design, HITL controls, HITL annotation, approval gates, escalation paths, or human review before an agent action commits. Anchored by AspenView new (*"human-in-the-loop gates: what an agent may complete unattended, what needs review"*), Anthropic GTM (*"human oversight for each motion: approval gates, handoffs, and escalation paths that keep sellers in control"*), Firmus (*"Define reference architectures for single-agent, multi-agent, workflow-based, event-driven, and human-in-the-loop agentic systems... Design human approval and policy enforcement workflows that clearly present an agent's evidence, recommendation, expected impact, proposed action, confidence, risk classification, authorization scope, and rollback option"*), 6sense (*"human-in-the-loop steps"*), Material Bank (*"human-in-the-loop evaluation strategies"*), LTS Senior Agentic (*"human-in-the-loop collaboration"*), Deloitte Senior Anthropic, Deloitte Anthropic, Deloitte Healthcare, Booz Allen Alexandria, Booz Allen FDE, Anduril, Crexi (*"human-in-the-loop annotation"*), Redapt.

Representative quote: *"Design human approval and policy enforcement workflows that clearly present an agent's evidence, recommendation, expected impact, proposed action, confidence, risk classification, authorization scope, and rollback option."* — Firmus Technologies.

→ Already covered by two existing exercises: [`mod-206-guardrails-implementation/exercises/exercise-04-human-approval-checkpoints.md`](lessons/mod-206-guardrails-implementation/exercises/exercise-04-human-approval-checkpoints.md) (approval checkpoint patterns) and [`mod-207-productionizing-agents/exercises/exercise-04-hitl-with-persistence.md`](lessons/mod-207-productionizing-agents/exercises/exercise-04-hitl-with-persistence.md) (durable HITL with resume semantics). Both pieces of the HITL requirement — the approval UX and the pause/resume plumbing — are already taught. The Firmus "evidence / recommendation / expected impact / confidence / risk classification / authorization scope / rollback" enumeration is a richer surface than exercise-04 explicitly names today; consider adding this list as a non-normative sidebar in the exercise README on next content pass.

### Theme 22 — "Context engineering" (54%) — crossed threshold

Climbed from 21% to ~54%. Anchored by AspenView new (*"Engineer the context: retrieval and RAG strategy, scoping, structured summaries, and pruning rules"*), Anthropic GTM (*"context engineering, agent development, MCP development, tool use, and evaluation frameworks"*), Anthropic Managed Agents (*"manages context (caching, compaction, memory)"*), Anthropic Life Sciences (*"prompting, context engineering, agent architectures, evaluation frameworks"*), Anthropic Startups Physical (*"production experience building LLM-powered applications, including prompting, context engineering, agent architectures"*), NewsBreak Agent Platform (*"Own the context engineering layer — retrieval, ranking, compression, and memory"*), CodeRoad new (*"dynamic, on-demand context-loading mechanisms to streamline operational security, performance, and token costs"*), Machinify (*"Working knowledge of how modern coding agents are built and how they engineer context"*), Deloitte Senior Anthropic (prompt engineering & context engineering, 6+ months required), Deloitte Anthropic (same), Anduril (context management + prompt caching), Ombud (context-caching vs. retrieval trade-offs), LTS Senior Agentic, Foresite Labs (*"context strategy"*).

**Vocabulary shift, not content shift.** The underlying mechanics are already taught:
- Prompt design, system prompt structure → [`mod-201-agent-fundamentals`](lessons/mod-201-agent-fundamentals).
- Retrieval shaping, structured summaries, memory curation → [`mod-203-rag-and-memory`](lessons/mod-203-rag-and-memory).
- Prompt caching, token budgets, routing → [`mod-207-productionizing-agents/exercises/exercise-03-caching-and-routing.md`](lessons/mod-207-productionizing-agents/exercises/exercise-03-caching-and-routing.md).
- Session-state compaction, tool-result clearing, on-disk state → [`mod-209-building-custom-harnesses/exercises/exercise-04-context-and-session-state.md`](lessons/mod-209-building-custom-harnesses/exercises/exercise-04-context-and-session-state.md) + chapter [`05-context-compaction-state.md`](lessons/mod-209-building-custom-harnesses/05-context-compaction-state.md).
- Instruction-file / skills discovery → [`mod-208-harness-engineering/02-guides-context-and-skills.md`](lessons/mod-208-harness-engineering/02-guides-context-and-skills.md) + [`mod-209-building-custom-harnesses/06-prompt-assembly-and-model-layer.md`](lessons/mod-209-building-custom-harnesses/06-prompt-assembly-and-model-layer.md).

No new module or exercise required. On the next content pass, surface the term "context engineering" as a named discipline in the mod-201 and mod-203 README overviews so learners searching for it hit the right chapters.

## Themes just below threshold this cycle

### Theme 23 — Agent harness engineering as named concept (18%)

Anthropic Managed Agents (*"The harness is the loop that calls Claude, routes tool calls, manages context (caching, compaction, memory), and recovers from errors... Built or contributed to an agent harness, agent framework, or LLM orchestration layer — tool execution, context management, memory, or multi-agent coordination"*), Anthropic Agent SDK (*"the same tools, agent loop, and context management that power Claude Code"*), NewsBreak Agent Platform (*"Design the eval and observability harness that runs thousands of agent traces per day"* + *"Build the agent runtime that orchestrates context assembly, tool invocation, model routing, and workflow tracking"*), Foresite Labs (*"Build the eval and observability harness — the tests, metrics, tracing, and guardrails that make agentic systems trustworthy"*), Baselayer (title-level — *"shared toolset registry... eval harness, and instrumentation surface for token-and-tool tracing"*).

Not yet above the 30% threshold, but the vocabulary is now explicit at the top tier of the market. **The newly-shipped [`mod-208-harness-engineering`](lessons/mod-208-harness-engineering) (user-side) and [`mod-209-building-custom-harnesses`](lessons/mod-209-building-custom-harnesses) (build-your-own) are correctly positioned to absorb this when it crosses threshold.** No delta.

### Theme 18 — AWS Bedrock AgentCore + Strands (11% explicit / 25% broader AWS Bedrock)

Explicit AgentCore mentions: TMSLLC (*"Deploy and manage AI agents using Amazon Bedrock AgentCore Runtime"*), Crexi (*"AWS Bedrock AgentCore"*), AspenView new (*"agent runtimes on Amazon Bedrock / AWS AgentCore"*). Broader "AWS Bedrock" (any flavor): Pushpay, Redapt-carryover, Amtech-carryover, Sharebite-carryover. External references (not counted in-sample): Devoteam (AWS AI Engineer, Strands Agents explicit), Leidos (Senior AWS Generative AI Solution Engineer, AgentCore + Strands + Guardrails explicit — both 403 on direct fetch), Cloud202 (AgentCore + Strands SDK — out-of-window at 2026-06-02).

The reversal flagged last cycle (22% → 14%) did not continue; AgentCore holds at ~11% explicit, ~25% including broader AWS Bedrock. The AWS 2026-03-31 GA of AgentCore Evaluations and 2026-03-03 GA of AgentCore Policy are now the baseline AWS JDs reference. Stable sub-threshold signal; mod-202/exercise-05 absorbs the comparison line item sufficiently.

External resources for learners interested now:
- Amazon Bedrock AgentCore — https://docs.aws.amazon.com/bedrock-agentcore
- Strands Agents SDK — https://strandsagents.com
- AWS Blog: Building trust in AI agents (AgentCore Evaluations) — https://aws.amazon.com/blogs/machine-learning/building-trust-in-ai-agents-a-production-ready-evaluation-framework-with-amazon-bedrock-agentcore/
- AWS Blog: AgentOps on AgentCore — https://aws.amazon.com/blogs/machine-learning/agentops-operationalize-agentic-ai-at-scale-with-amazon-bedrock-agentcore/

### Theme 10 — Durable execution (14%)

PhysicsX-carryover (Temporal explicit — if still in-window at est:2026-08), Anthropic Managed Agents (*"long-lived, stateful sessions that execute autonomously for minutes or hours/days, persist through disconnections, and resume cleanly... durable session and event storage, sandbox orchestration, streaming, scheduling, and multi-tenant isolation"*), 6sense (*"typed state, conditional routing, subgraphs, checkpointing, durable execution, and human-in-the-loop steps"*), Benchling-carryover (Temporal, Prefect, Airflow explicit).

Marginally up vs 11% last cycle. 6sense's language — treating durable execution as a first-class LangGraph feature (checkpointing + resume) — is new. Covered by [`mod-207-productionizing-agents/exercises/exercise-02-durable-execution-temporal.md`](lessons/mod-207-productionizing-agents/exercises/exercise-02-durable-execution-temporal.md). The managed-agent-runtime pattern (AgentCore / Anthropic Managed Agents / LangGraph Platform family) continues to converge as a named production pattern.

### Theme 21 — A2A (Agent-to-Agent) protocol (7%)

PhysicsX-carryover (*"strong, reasoned opinions on emerging open standards (such as MCP, A2A, and ACP)"*) and Booz Allen Alexandria-carryover (*"MCP for tool integration and A2A for agent-to-agent collaboration"*). Flat from last cycle. mod-204 already covers agent-to-agent patterns generically. Watch for a named protocol version (A2A 1.0 shipped 2026-Q1) to enter requirements next cycle.

### Theme 24 — Credential-mgmt / OAuth 2.1 for agents and MCP servers (4% explicit / 21% broader)

Explicit OAuth / OAuth 2.1 / OpenID Connect only in Capgemini (*"Strong understanding of REST, OpenAPI, OAuth 2.0, OpenID Connect, event-driven architecture, messaging and secure system integration"*). Broader credential-mgmt / identity / RBAC / least-privilege / audit-logging / multi-tenant isolation: Capgemini, Indicium (ACLs + multi-tenant isolation + row/column-level permissions + zero-data-leakage constraints), Air (threat model + privileged capabilities), Pushpay (IAM/KMS), Firmus (authorization scope + rollback), LTS-carryover (identity + least privilege), Benchling-carryover (RBAC + audit + payload encryption).

**Still flat at 4% on explicit OAuth 2.1 despite the 2026-07-28 MCP spec change** mandating OAuth 2.1 + PKCE + RFC 8707 Resource Indicators for remote MCP servers. The AWS 2026-07 announcement of AWS MCP Server OAuth support and the ServiceNow MCP OAuth 2.1 docs are leading requirement wording. **This is the fourth consecutive cycle flagging MCP OAuth 2.1 as the strongest next-cycle candidate.** If this crosses 30% next cycle (2027-01), propose a mod-206 exercise-05 or mod-202/exercise-04 extension focused on OAuth-wrapped MCP servers.

## Sub-threshold new signals — tracked for next cycle

### Voice / realtime / telephony agents — 11% (flat)

Particle41-carryover (Deepgram, ElevenLabs, LiveKit named), FourKites-carryover (voice agent for carrier communication), Joblogic-carryover (multi-channel: voice, SMS, WhatsApp). External references (not counted in-sample): LiveKit Upwork anchor 2026-07-26, ElevenLabs Applied AI Engineer, ElevenLabs Full-Stack Engineer (Creative Agents).

Voice remains cordoned into either (a) specialty *Voice AI Engineer* titles that aren't counted here (Blooming Health, Carbon Technology, ElevenLabs Creative Agents) or (b) a secondary channel inside otherwise-standard agentic roles. mod-208 voice-agent sibling still unwarranted.

External resources for learners interested now:
- LiveKit Agents — https://docs.livekit.io/agents
- Vapi — https://docs.vapi.ai
- Pipecat — https://docs.pipecat.ai
- Deepgram — https://developers.deepgram.com
- ElevenLabs — https://elevenlabs.io/docs
- Cartesia — https://docs.cartesia.ai

### Code / computer-use agents — not resampled separately this cycle

Build-altitude basics covered by mod-201/exercise-03; computer-use depth remains out-of-scope link-outs:
- Anthropic Computer Use — https://docs.anthropic.com/en/docs/build-with-claude/computer-use
- e2b sandbox — https://e2b.dev/docs
- Modal sandboxes — https://modal.com/docs/guide/sandbox
- BrowserBase — https://docs.browserbase.com

### Agent modeling / post-training — remains owned elsewhere

Fine-tuning / SFT / RLHF / LoRA mentions (Booz Allen Alexandria-carryover: Hugging Face / PEFT / LoRA) stay owned by `ai-infra-ml-platform-learning` (level 30). No change.

### AI-assisted developer-tool fluency — flat at ~30%

Postings naming daily use of Claude Code / Codex / Cursor / Gemini CLI as a required or strongly-preferred skill: CodeRoad new, CodeRoad Lead, Machinify (*"Fluency with Claude Code / Codex as a power user"*), Pushpay (*"you will drive org-wide adoption of tools like Claude Code and custom MCP implementations"*), Booz Allen FDE (Claude Code, Codex, Cursor explicit), Ombud-carryover (Claude Code), Crexi (Claude Code, Codex), Artefact-carryover (Claude Code, Gemini CLI, Codex, Cursor), NewsBreak Agent Platform (*"active, hands-on familiarity with modern AI development tooling (Cursor, Claude Code, Codex, v0)"*), Anthropic Managed Agents (*"Use Claude or other AI tools as a core part of how you build software"*).

**Important classification call (unchanged):** this is a work-style requirement (how you build), not a curriculum-worthy build skill. It belongs in prerequisites / orientation guidance, not in a mod-20X exercise. mod-208 and mod-209 cover authoring harnesses/skills/subagents for these tools — which *is* the curriculum-worthy build skill — and that coverage is in the right place. No delta.

## Conclusion

<!-- needs-research: monitor MCP-authoring-with-OAuth-2.1 requirement wording in the next cycle — this is now the fourth consecutive cycle flagging it as the strongest next-cycle candidate (only Capgemini explicit in-window); monitor whether AWS Bedrock AgentCore stabilizes at ~11% or re-accelerates following the 2026-H2 GA cadence; watch for the "harness" vocabulary (currently 18%) to cross 30% in top-tier postings as Anthropic Managed Agents / NewsBreak-style wording spreads; verify the mod-201 and mod-203 README surface a "context engineering" callout on next content pass. -->

The Agentic AI Engineer curriculum (mod-201..209 plus `project-201-production-multi-agent-system` and `project-202-benchmark-agent`) covers every job-market requirement that clears the continuity-bias thresholds. **No delta is proposed this cycle** — this is the **fourth consecutive zero-additions refresh** (2026-08-03, 2026-09-03, 2026-10-03, 2026-11-03).

Three directional moves this cycle, all already fully covered: (1) "context engineering" vocabulary crossed 30% but the mechanics are already taught across mod-201/203/207/209 — surface the term in READMEs on next content pass; (2) "harness" vocabulary emerged as explicit requirement wording in Anthropic Managed Agents, NewsBreak Agent Platform, Foresite Labs, and Baselayer postings, validating the newly-shipped mod-208 and mod-209 positioning; (3) tool/function calling + structured outputs climbed to 54% and cost/latency optimization crossed 30% for the first time — both inside mod-201 and mod-207 scope. The MCP OAuth 2.1 watchpoint continues to lag spec changes (Capgemini remains the only in-window explicit anchor at 4%). Re-run on the next quarterly cycle (2027-01) to catch delayed MCP-OAuth language propagation and the AgentCore trajectory recheck.
