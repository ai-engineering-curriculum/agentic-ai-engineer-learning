# Job Requirements — Agentic AI Engineer

**Role level:** 30 (build altitude)
**Track:** `agentic-ai-engineer-learning`
**Research window:** 2026-07-05 → 2026-10-03 (last 90 days)
**Today:** 2026-10-03
**Prior refresh:** 2026-09-03

This file maps verbatim requirements from current Agentic AI Engineer job postings to the existing curriculum. Raw normalized data lives in [`.aicg/job-requirements.json`](.aicg/job-requirements.json); the strictly-additive proposal lives in [`.aicg/curriculum-plan-delta.json`](.aicg/curriculum-plan-delta.json).

## Summary

- Postings sampled: **28** in-window (2026-07-05 → 2026-10-03). This is a full re-baseline, not a 30-day delta. Ashby-hosted postings (Benchling, Cohere, OpenRouter, Clera, DataSnipper, Alterion, Block Labs, G2i, Blooming Health, General Intelligence NYC, Eigen Labs) continued to render only skeleton markup to WebFetch; the sample recovered Benchling via a dynamitejobs.com mirror but five Ashby pages contributed title-only evidence. Dice.com job detail URLs expire within ~30 days and returned 410/404 for most prior-cycle anchors; Capital One's HTML now 404s on the Workday rewrite. Sample quality was preferred over padding to 30+.
- Equivalent titles counted: `Agentic AI Engineer`, `AI Agent Engineer`, `Agent Engineer`, `Applied AI Engineer`, `Generative AI Engineer`, `LLM Engineer`. Title mix: 14 Agentic AI Engineer (or close variants: *Senior AI & Agentic Engineer*, *AI Engineer | Agentic Systems*, *Forward Deployed Engineer, Agentic AI*), 4 Senior AI Engineer, 3 AI Engineer, 2 AI/ML Engineer, 1 Applied AI Engineer, 1 Agentic AI Security Engineer, 3 Software Engineer (AI SDK / Agent SDK / AI Engineering).
- **Proposed delta this cycle: 0 modules, 0 exercises, 0 projects.** Every requirement above the continuity-bias threshold (≥3 distinct postings AND ≥30% frequency) is already owned by an existing module in `mod-201..207`. This is the **third consecutive zero-additions refresh** (2026-08-03, 2026-09-03, 2026-10-03).
- Two notable shifts this cycle, both already covered:
  - **Human-in-the-loop oversight jumped 6% → 39%** (11 of 28 postings explicitly name HITL, approval checkpoints, HITL annotation, or audit-trail oversight — e.g., Deloitte Healthcare/Senior-Anthropic/Anthropic, Booz Allen x2, Anduril, Redapt, Benchling, Crexi, Joblogic, LTS). Sample composition contributed, but the directional move is also real: enterprise and defense postings now name HITL as a load-bearing requirement rather than a nice-to-have. Already covered by [`mod-206-guardrails-implementation/exercises/exercise-04-human-approval-checkpoints.md`](lessons/mod-206-guardrails-implementation/exercises/exercise-04-human-approval-checkpoints.md) and [`mod-207-productionizing-agents/exercises/exercise-04-hitl-with-persistence.md`](lessons/mod-207-productionizing-agents/exercises/exercise-04-hitl-with-persistence.md).
  - **Observability crossed 50% (33% → 54%)**. Named stacks consistent with prior cycle (Langfuse, LangSmith, Arize, OpenTelemetry GenAI, Datadog, Braintrust, Logfire). Already covered by [`mod-205-evaluation-observability/exercises/exercise-02-otel-tracing-wireup.md`](lessons/mod-205-evaluation-observability/exercises/exercise-02-otel-tracing-wireup.md).
- Three tracked sub-threshold signals from last cycle shifted; none crossed 30%:
  - **AWS Bedrock AgentCore + Strands REVERSED**: 22% → 14% (Amtech, Redapt, Sharebite, Crexi). The rising trend flagged last cycle did not continue. mod-202/exercise-05 absorption remains unnecessary.
  - **Google ADK dropped to 4%** (Artefact North America only). Sample-composition effect — ADK is a stable minor stack, not a growth signal. Already in mod-202 objectives.
  - **MCP OAuth 2.1 wording still flat at 7%** despite the 2026-07-28 spec mandating OAuth 2.1 + PKCE + RFC 8707. Only LTS (identity/least-privilege language) and Benchling (RBAC/audit/payload-encryption language) use adjacent phrasing. Requirement wording continues to lag spec changes; same watchpoint carries into the next cycle.

## Methodology

- Sources: greenhouse.io/job_boards (bulk, full page fetches), jobs.ashbyhq.com (title-only extraction where JS-only rendering blocked WebFetch), apply.deloitte.com, careers.boozallen.com, vercel.com/careers, supermetrics.com/careers, dynamitejobs.com (mirror), builtin.com, dice.com (several URLs returned 410/404 and had to be dropped), techstars.com, linkedin.com/jobs (titles + snippets only), talent.com.
- Per-posting capture: employer, title, URL, date_observed, date_posted (marked `estimated:YYYY-MM` when inferred), location, 4–8 verbatim/near-verbatim requirement quotes.
- Frequency = distinct in-window postings citing the theme ÷ 28 in-window postings.
- Threshold for a new curriculum item: ≥ 3 postings AND ≥ 30% frequency AND no existing module/exercise can be incrementally extended to cover it.
- Source caveats documented in `summary.source_quality_caveats` in `.aicg/job-requirements.json`.

## Requirement themes → curriculum ownership

The table below lists every requirement theme observed, its in-window frequency, the role that owns it per the level hierarchy, and the curriculum coverage path. **Bold frequencies** are ≥ 30% (load-bearing under continuity bias).

| # | Theme | Freq | Owner role | Coverage |
|---|---|---|---|---|
| 1 | Agent frameworks: LangGraph, LangChain, CrewAI, AutoGen, OpenAI Agents SDK, Google ADK, Anthropic/Claude Agent SDK, LlamaIndex, Semantic Kernel, Pydantic AI, Vercel AI SDK, smolagents, Strands | **89%** | `agentic-ai-engineer` (this) | [`mod-202-frameworks`](lessons/mod-202-frameworks) |
| 2 | RAG, vector DBs, embeddings, agent memory | **64%** | `agentic-ai-engineer` | [`mod-203-rag-and-memory`](lessons/mod-203-rag-and-memory) |
| 3 | Multi-agent orchestration (orchestrator-worker, handoffs, sub-agents, agent-to-agent, state-graph) | **36%** | `agentic-ai-engineer` | [`mod-204-multi-agent-implementation`](lessons/mod-204-multi-agent-implementation) |
| 4 | Evaluation harnesses (LLM-as-judge, regression suites, golden traces, CI-integrated eval bar) | **71%** | `agentic-ai-engineer` | [`mod-205-evaluation-observability`](lessons/mod-205-evaluation-observability) |
| 5 | Observability (Langfuse, LangSmith, Phoenix, Arize, OTel GenAI, Datadog, Braintrust, Logfire) | **54%** | `agentic-ai-engineer` | [`mod-205-evaluation-observability/exercises/exercise-02-otel-tracing-wireup.md`](lessons/mod-205-evaluation-observability/exercises/exercise-02-otel-tracing-wireup.md) |
| 6 | Function / tool calling, structured outputs (Pydantic / JSON schema) | **39%** | `agentic-ai-engineer` | [`mod-201-agent-fundamentals/exercises/exercise-02-function-calling-tools.md`](lessons/mod-201-agent-fundamentals/exercises/exercise-02-function-calling-tools.md) |
| 7 | Model Context Protocol (MCP) — consumption AND authoring | **50%** | `agentic-ai-engineer` | [`mod-202-frameworks/exercises/exercise-04-mcp-tool-server.md`](lessons/mod-202-frameworks/exercises/exercise-04-mcp-tool-server.md) |
| 8 | API deployment (FastAPI/Flask, REST/GraphQL, async Python, Docker/K8s/IaC, CI/CD) | **96%** | `agentic-ai-engineer` | [`mod-207-productionizing-agents/exercises/exercise-01-agent-api-deployment.md`](lessons/mod-207-productionizing-agents/exercises/exercise-01-agent-api-deployment.md) |
| 9 | Guardrails, prompt injection defense, safety/policy controls, permissions, audit logs | **36%** | `agentic-ai-engineer` (basics) → `ai-infra-security-learning` (depth) | [`mod-206-guardrails-implementation`](lessons/mod-206-guardrails-implementation); deep agent attack surface → `ai-infra-security-learning` (level 35) |
| 10 | Durable execution / workflow engines (Temporal, Airflow, Prefect, managed agent runtimes) | 11% | `agentic-ai-engineer` | [`mod-207-productionizing-agents/exercises/exercise-02-durable-execution-temporal.md`](lessons/mod-207-productionizing-agents/exercises/exercise-02-durable-execution-temporal.md) |
| 11 | Cost / latency optimization (prompt caching, model routing, token budgets) | 21% | `agentic-ai-engineer` | [`mod-207-productionizing-agents/exercises/exercise-03-caching-and-routing.md`](lessons/mod-207-productionizing-agents/exercises/exercise-03-caching-and-routing.md) |
| 14 | Human-in-the-loop oversight / escalation / approval checkpoints | **39%** | `agentic-ai-engineer` | [`mod-206-guardrails-implementation/exercises/exercise-04-human-approval-checkpoints.md`](lessons/mod-206-guardrails-implementation/exercises/exercise-04-human-approval-checkpoints.md), [`mod-207-productionizing-agents/exercises/exercise-04-hitl-with-persistence.md`](lessons/mod-207-productionizing-agents/exercises/exercise-04-hitl-with-persistence.md) |
| 16 | Voice / realtime / telephony agents | 11% | sub-threshold | External link-outs (below); mod-208 sibling still unwarranted |
| 18 | AWS-native agent runtime (Bedrock AgentCore, Strands Agents) | 14% | sub-threshold (reversed from 22%) | [`mod-202-frameworks/exercises/exercise-05-framework-tradeoff-bakeoff.md`](lessons/mod-202-frameworks/exercises/exercise-05-framework-tradeoff-bakeoff.md) — comparison line item sufficient |
| 19 | Google ADK + AgentSpace as named production stack | 4% | `agentic-ai-engineer` | Already in mod-202 objectives & exercise-05 comparison scope |
| 21 | A2A (Agent-to-Agent) protocol — inter-agent communication protocol | 7% | `agentic-ai-engineer` | [`mod-204-multi-agent-implementation`](lessons/mod-204-multi-agent-implementation) — agent-to-agent protocols already in module scope |
| 22 | "Context engineering" as named discipline distinct from prompt engineering | 21% | `agentic-ai-engineer` | Vocabulary shift, not content shift — surface in mod-201/mod-203 READMEs on next content pass |
| 24 | Credential-mgmt / OAuth 2.1 / RBAC / scoped tokens for agents and MCP servers | 7% | `agentic-ai-engineer` (basics) → `ai-infra-security-learning` (depth) | Still sub-threshold; strongest next-cycle candidate following 2026-07-28 MCP spec |

## Posting evidence for load-bearing themes

The table below lists the postings that anchor each ≥30% theme in this window. All 28 sampled postings fall within 2026-07-05 → 2026-10-03.

### Theme 1 — Agent frameworks (89%)

| Employer | Title | URL | Date observed | Posted |
|---|---|---|---|---|
| CodeRoad | Lead Agentic AI Engineer | https://job-boards.greenhouse.io/coderoad/jobs/4110608009 | 2026-10-03 | est:2026-09 |
| Machinify | AI Engineer \| Agentic Systems (L4) | https://job-boards.greenhouse.io/machinifyinc/jobs/4146862009 | 2026-10-03 | est:2026-09 |
| PhysicsX | Senior AI / Agentic Engineer | https://job-boards.greenhouse.io/physicsx/jobs/4804769101 | 2026-10-03 | est:2026-08 |
| Particle41 | AI Engineer | https://job-boards.greenhouse.io/particle41llc/jobs/5284201008 | 2026-10-03 | est:2026-08 |
| Amtech | Senior AI Engineer — Agentic Platform | https://job-boards.greenhouse.io/amtechsoftware/jobs/4387878009 | 2026-10-03 | est:2026-08 |
| GitLab | Backend Engineer, AI Engineering: Duo Chat | https://job-boards.greenhouse.io/gitlab/jobs/8698314002 | 2026-10-03 | est:2026-09 |
| LTS | Agentic AI Security Engineer | https://job-boards.greenhouse.io/lts/jobs/4340457009 | 2026-10-03 | est:2026-08 |
| FourKites | Senior AI Engineer | https://job-boards.greenhouse.io/fourkites/jobs/7981512 | 2026-10-03 | est:2026-09 |
| Deloitte | Agentic AI Engineer — Anthropic/Claude | https://apply.deloitte.com/en_US/careers/JobDetail/Agentic-AI-Engineer-Anthropic-Claude/365634 | 2026-10-03 | est:2026-09 (recruiting ends 2026-09-30) |
| Deloitte | Agentic AI Engineer, Senior — Anthropic/Claude | https://apply.deloitte.com/en_US/careers/JobDetail/Agentic-AI-Engineer-Senior-Anthropic-Claude/363947 | 2026-10-03 | est:2026-09 (recruiting ends 2026-10-30) |
| Deloitte | Agentic AI Engineer — Healthcare AI | https://apply.deloitte.com/en_US/careers/JobDetail/Agentic-AI-Engineer-Healthcare-AI/355577 | 2026-10-03 | est:2026-08 |
| Booz Allen Hamilton | Agentic AI Engineer | https://careers.boozallen.com/jobs/JobDetail/Alexandria-Agentic-AI-Engineer-R0249306/130301 | 2026-10-03 | est:2026-09 |
| Booz Allen Hamilton | Agentic AI Forward-Deployed Engineer | https://careers.boozallen.com/jobs/JobDetail/McLean-Agentic-AI-Forward-Deployed-Engineer-R0250142/130862 | 2026-10-03 | est:2026-09 |
| Benchling | Agentic AI Engineer | https://jobs.ashbyhq.com/benchling/d5896e95-fed2-4cd4-b104-1ea4df92f7d7 | 2026-10-03 | 2026-07-11 (via mirror) |
| Future | Applied AI Engineer | https://job-boards.greenhouse.io/future/jobs/4683133005 | 2026-10-03 | est:2026-08 |
| Anthropic | Staff+ Software Engineer, Claude Managed Agents | https://freehire.me/jobs/staff-software-engineer-claude-managed-agents-anthropic-safpgg7y | 2026-10-03 | 2026-08-21 |
| BPD Healthcare | AI Engineer | https://job-boards.greenhouse.io/bpd/jobs/5386839008 | 2026-10-03 | est:2026-09 |
| Ombud | Senior Full Stack AI Engineer | https://job-boards.greenhouse.io/ombud/jobs/8599080002 | 2026-10-03 | est:2026-08 |
| Redapt | Forward Deployed Engineer, Agentic AI | https://job-boards.greenhouse.io/redapt/jobs/5396488008 | 2026-10-03 | est:2026-09 |
| Vercel | Software Engineer, AI SDK | https://vercel.com/careers/software-engineer-ai-sdk-5474915004 | 2026-10-03 | est:2026-08 |
| Sharebite | AI Engineer | https://job-boards.greenhouse.io/sharebite/jobs/5848230004 | 2026-10-03 | est:2026-08 |
| AspenView Technology Partners | Agentic AI Engineer — RAG Architecture / LLM Systems | https://job-boards.greenhouse.io/aspenviewtech/jobs/4355206009 | 2026-10-03 | est:2026-09 |
| Crexi | Senior AI Engineer | https://job-boards.greenhouse.io/crexi/jobs/4723615005 | 2026-10-03 | est:2026-09 |
| Joblogic | AI/ML Engineer | https://job-boards.eu.greenhouse.io/joblogic/jobs/4927594101 | 2026-10-03 | est:2026-08 |
| Artefact North America | Senior, AI & Agentic Engineer | https://job-boards.greenhouse.io/artefactnoram/jobs/8731948002 | 2026-10-03 | est:2026-09 |

Representative quote: *"Expert proficiency with LangGraph / LangChain, Anthropic/Claude APIs (tool calling, prompt caching, Claude Code), MCP, and observability tools"* — CodeRoad.

→ Covered by [`mod-202-frameworks`](lessons/mod-202-frameworks). Strands Agents and Google ADK are still named in mod-202 objectives and `exercise-05-framework-tradeoff-bakeoff` comparison scope — no delta needed.

### Theme 2 — RAG, vector DBs, memory (64%)

Anchored by Particle41, Amtech, LTS, BPD, Deloitte-Healthcare, Deloitte-Senior-Anthropic, Deloitte-Anthropic, Booz Allen x2, Ombud, Redapt, Rackner, Sharebite, AspenView, Crexi, Joblogic, Artefact, Benchling. Named tools consistent with prior cycle: pgvector, Pinecone, Weaviate, Qdrant, Milvus, FAISS, Chroma, Azure AI Search.

Representative quote: *"Deep knowledge of Vector Databases (Pinecone, Qdrant, Milvus, pgvector)"* — AspenView.

→ Covered by [`mod-203-rag-and-memory`](lessons/mod-203-rag-and-memory).

### Theme 3 — Multi-agent orchestration (36%)

Anchored by CodeRoad, PhysicsX, Deloitte x3 (multi-agent preferred), Booz Allen x2, Anduril, Redapt, Rackner, Crexi.

Representative quote: *"Experience building agentic systems, including multi-agent orchestration, tool calling, prompt engineering, integration into classical software systems"* — Anduril.

→ Covered by [`mod-204-multi-agent-implementation`](lessons/mod-204-multi-agent-implementation). A2A protocol surface (PhysicsX, Booz Allen Alexandria) fits `exercise-02-agent-handoffs`.

### Theme 4 — Evaluation harnesses (71%)

Anchored by CodeRoad (Promptfoo, Ragas, LLM-as-judge), Machinify, PhysicsX (systematic evals), GitLab, Anthropic (eval infrastructure), BPD (evaluation harnesses), Future (Langfuse + rubrics), Deloitte x3, Booz Allen FDE (golden datasets, regression), Anduril, Ombud, Redapt, Rackner, AspenView (Ragas, TruLens, faithfulness/answer-relevance), Crexi (offline evals, HITL annotation), Joblogic (datasets, rubrics), Artefact (promptfoo), Benchling.

Representative quote: *"Hands-on experience with automated LLM eval suites (Promptfoo, Ragas, or LLM-as-a-judge patterns)"* — CodeRoad.

→ Covered by [`mod-205-evaluation-observability`](lessons/mod-205-evaluation-observability).

### Theme 5 — Observability (54%) — climbed from 33%

Anchored by PhysicsX (OTel, LangSmith, Arize, Braintrust), LTS (monitoring for abnormal agent behavior), Deloitte-Healthcare (tracing for prompts, tool calls, retrieval), Deloitte-Senior-Anthropic (preferred), Booz Allen FDE (production observability), Future (Langfuse, OTel, Datadog), Ombud, Redapt, Supermetrics (dashboards/logs), AspenView (LLM observability), Crexi (trace-level observability), Joblogic (LangSmith), Artefact (LangSmith, Langfuse), Benchling (LangSmith, Arize).

Representative quote: *"Establish observability for agent runs (tracing, failure analysis, latency/cost monitoring, tool-call success rates)"* — Deloitte-Healthcare / Cognitive Space (vocabulary carryover).

→ Covered by [`mod-205-evaluation-observability/exercises/exercise-02-otel-tracing-wireup.md`](lessons/mod-205-evaluation-observability/exercises/exercise-02-otel-tracing-wireup.md). Langfuse + OTel GenAI are already the canonical stack in the exercise. Arize and Braintrust are first-class named options in this cycle's sample; no exercise rework needed — resources.md already lists both.

### Theme 6 — Function / tool calling, structured outputs (39%)

Anchored by CodeRoad, Machinify (Pydantic/JSON Schema explicit), GitLab, Amtech, Deloitte-Anthropic, Deloitte-Senior-Anthropic (structured outputs), Anduril, Ombud, Redapt, Future (structured output), Crexi (tool/function calling, structured outputs).

Representative quote: *"Experience designing structured outputs (Pydantic / JSON Schema) and tool interfaces"* — Machinify.

→ Covered by [`mod-201-agent-fundamentals/exercises/exercise-02-function-calling-tools.md`](lessons/mod-201-agent-fundamentals/exercises/exercise-02-function-calling-tools.md).

### Theme 7 — MCP (50%)

Anchored by CodeRoad, PhysicsX (MCP + A2A + ACP), Particle41, Amtech (preferred), Deloitte x3, Booz Allen x2, Anduril (building and maintaining MCP servers), Joblogic (nice-to-have), Artefact, Benchling. Postings continue to shift from *consuming* MCP tools to *authoring* MCP servers wrapped around enterprise systems.

Representative quote: *"Experience with Model Context Protocols, including building and maintaining MCP servers"* — Anduril.

→ Covered by [`mod-202-frameworks/exercises/exercise-04-mcp-tool-server.md`](lessons/mod-202-frameworks/exercises/exercise-04-mcp-tool-server.md), which includes an authoring lab. **Watchpoint persists:** the 2026-07-28 MCP spec mandates OAuth 2.1 + PKCE + RFC 8707 Resource Indicators for remote MCP servers; that language is still absent from most postings (only LTS and Benchling use adjacent identity/RBAC phrasing — 7% in this cycle). This is the third consecutive cycle flagging it as the strongest next-cycle candidate.

### Theme 8 — API deployment / backend / containerization (96%)

Sample-composition is still skewed toward postings that explicitly name cloud/DevOps stacks (true of every mid+ Agentic AI Engineer JD now). Anchored by nearly every posting in the sample.

Representative quote: *"Hands-on AWS experience (e.g., Lambda, ECS/EKS, S3, IAM, CloudWatch) deploying and operating production workloads"* — Amtech.

→ Covered by [`mod-207-productionizing-agents/exercises/exercise-01-agent-api-deployment.md`](lessons/mod-207-productionizing-agents/exercises/exercise-01-agent-api-deployment.md). IaC (Terraform, CDK) surfaces alongside deployment but stays owned by `ai-infra-platform-engineer-learning` per hierarchy.

### Theme 9 — Guardrails / safety / policy (36%) — flat vs 39% prior cycle

Anchored by LTS (prompt injection, jailbreaking, data poisoning, model abuse explicit), BPD (LLM guardrails), Deloitte-Anthropic/Senior/Healthcare, Anduril (sandboxed execution), Supermetrics (campaigns-with-guardrails framing), AspenView (NeMo Guardrails + Llama Guard explicit), Crexi (structured outputs, source-linking, explainability), Benchling (audit logging, RBAC, payload encryption).

Representative quote: *"Strong mastery of modern AI orchestration ecosystems (LangChain, LangGraph, LlamaIndex, AutoGen) ... NeMo Guardrails, Llama Guard"* — AspenView.

→ Covered by [`mod-206-guardrails-implementation`](lessons/mod-206-guardrails-implementation): four exercises cover I/O moderation, prompt-injection defenses (OWASP LLM01), tool-permission enforcement, human-approval checkpoints.

### Theme 14 — Human-in-the-loop oversight (39%) — climbed from 6%

Anchored by LTS (HITL workflows explicit), Deloitte-Anthropic (HITL controls), Deloitte-Healthcare (HITL checkpoints), Deloitte-Senior-Anthropic (HITL controls), Booz Allen Alexandria (ReAct + HITL evaluation), Booz Allen FDE (golden datasets + HITL feedback), Anduril (HITL checkpoints), Redapt (HITL design explicit), Joblogic (HITL review), Benchling (HITL controls), Crexi (HITL annotation).

Representative quote: *"Strong understanding of agentic patterns: tool use, RAG, memory management, multi-agent orchestration, and human-in-the-loop design"* — Redapt.

→ Already covered by two existing exercises: [`mod-206-guardrails-implementation/exercises/exercise-04-human-approval-checkpoints.md`](lessons/mod-206-guardrails-implementation/exercises/exercise-04-human-approval-checkpoints.md) (approval checkpoint patterns) and [`mod-207-productionizing-agents/exercises/exercise-04-hitl-with-persistence.md`](lessons/mod-207-productionizing-agents/exercises/exercise-04-hitl-with-persistence.md) (durable HITL with resume semantics). Both pieces of the HITL requirement — the approval UX and the pause/resume plumbing — are already taught. **Sample-composition contribution noted:** this cycle's sample is heavier on Deloitte, Booz Allen, and Benchling-style enterprise/regulated postings than prior cycles, which inflates HITL. The underlying signal is still real; the coverage is just already in place.

## Themes just below threshold this cycle

### Theme 22 — "Context engineering" (21%) — vocabulary spreading

CodeRoad, Machinify ("engineer context"), Deloitte-Anthropic / Deloitte-Senior-Anthropic (prompt engineering & context engineering), Anduril (context management + prompt caching), Ombud (context caching vs. retrieval trade-off).

Vocabulary shift, not content shift. Falls inside the prompt-design + retrieval-shaping + memory-curation loop already taught in mod-201 (prompt design) and mod-203 (retrieval + memory). Surface the term in mod-201 README on next content pass.

### Theme 11 — Cost / latency optimization (21%)

CodeRoad (prompt caching), PhysicsX (token-based cost tracking), Future (prompt caching, token budgets, retry logic), Anduril (prompt caching), Ombud (context-caching approaches), Supermetrics (cost/latency management).

Covered by [`mod-207-productionizing-agents/exercises/exercise-03-caching-and-routing.md`](lessons/mod-207-productionizing-agents/exercises/exercise-03-caching-and-routing.md). Climbing slowly (14% → 17% → 21%) but still below threshold.

### Theme 18 — AWS-native agent stack (14%) — reversed from 22%

Amtech (AWS Bedrock), Redapt (AWS Bedrock), Sharebite (AWS Bedrock), Crexi (AWS Bedrock AgentCore explicit).

The rising trend flagged last cycle did not continue — likely a mix of sample composition and that AgentCore adoption, while real, hasn't yet become a universal requirement language in the broader hiring market. Deferred to next-cycle recheck. mod-202/exercise-05 still absorbs if the trend resumes.

### Theme 10 — Durable execution (11%)

PhysicsX (Temporal explicit), Anthropic (durable, stateful, long-running sessions), Benchling (Temporal, Prefect, Airflow explicit).

Up slightly from 0% last cycle (small-sample effect last time). Still covered by [`mod-207-productionizing-agents/exercises/exercise-02-durable-execution-temporal.md`](lessons/mod-207-productionizing-agents/exercises/exercise-02-durable-execution-temporal.md). The Anthropic Managed Agents posting is the most interesting signal — managed agent runtimes (the AWS Bedrock AgentCore / Anthropic Managed Agents / LangGraph Platform family) is converging as a named production pattern, but still at low frequency in requirements language.

### Theme 21 — A2A (Agent-to-Agent) protocol (7%)

PhysicsX ("strong, reasoned opinions on emerging open standards (such as MCP, A2A, and ACP)"); Booz Allen Alexandria ("MCP for tool integration and A2A for agent-to-agent collaboration" explicit).

Flat from last cycle. mod-204 already covers agent-to-agent protocols generically. Watch for a named protocol version (A2A 1.0) to enter requirements next cycle.

### Theme 24 — Credential-mgmt / OAuth 2.1 for agents and MCP servers (7%)

LTS ("identity and least privilege for agents"), Benchling ("multi-tenant isolation, secrets management, audit logging, payload encryption, role-based access controls").

**Still flat at 7% despite the 2026-07-28 MCP spec change** mandating OAuth 2.1 + PKCE + RFC 8707 Resource Indicators for remote MCP servers. Requirement wording continues to lag spec changes. This is the strongest candidate to cross threshold in the next cycle; if MCP-authoring-with-enterprise-auth crosses 30% then, propose a mod-206 exercise-05 addition focused on auth-wrapped MCP servers.

## Sub-threshold new signals — tracked for next cycle

### Voice / realtime / telephony agents — 11% (up from 6%)

Particle41 (Deepgram, ElevenLabs, LiveKit named), FourKites (voice agent for carrier communication), Joblogic (multi-channel: voice, SMS, WhatsApp).

Voice remains cordoned into either (a) specialty *Voice AI Engineer* titles that aren't counted here (Blooming Health, Carbon Technology) or (b) a secondary channel inside otherwise-standard agentic roles. mod-208 voice-agent sibling still unwarranted.

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

Fine-tuning / SFT / RLHF / LoRA mentions (Booz Allen Alexandria lists Hugging Face / PEFT / LoRA; EvolutionIQ, Realtech from prior cycle) stay owned by `ai-infra-ml-platform-learning` (level 30). No change.

### AI-assisted developer-tool fluency — climbed from 6% to ~29%

Postings naming daily use of Claude Code / Codex / Cursor as a required or strongly-preferred skill: CodeRoad, Machinify ("Fluency with Claude Code / Codex as a power user"), Booz Allen FDE (Claude Code, Codex, Cursor explicit), Ombud (Claude Code), Crexi (Claude Code, Codex), Artefact (Claude Code, Gemini CLI, Codex, Cursor), Netomi (prior cycle), Benchling (Claude Code implied).

**Important classification call:** this is a work-style requirement (how you build), not a curriculum-worthy build skill. It belongs in prerequisites / orientation guidance, not in a mod-20X exercise. No delta is proposed. If the shape of the requirement shifts from "use these tools" to "author harness/skills/subagents for these tools" (which would be curriculum-worthy), re-evaluate at the next refresh.

## Conclusion

<!-- needs-research: monitor MCP-authoring-with-OAuth-2.1 requirement wording in the next cycle following the 2026-07-28 MCP spec change — this is now the third consecutive cycle flagging it as the strongest next-cycle candidate; monitor whether AWS Bedrock AgentCore + Strands Agents resumes its climb (reversed from 22% to 14% this cycle); monitor whether "authoring Claude Code / Codex harnesses/subagents" shifts from a work-style skill to a buildable curriculum skill. -->

The Agentic AI Engineer curriculum (mod-201..207 plus `project-201-production-multi-agent-system` and `project-202-benchmark-agent`) covers every job-market requirement that clears the continuity-bias thresholds. **No delta is proposed this cycle** — this is the third consecutive zero-additions refresh (2026-08-03, 2026-09-03, 2026-10-03). The two large directional moves this cycle (HITL 6% → 39%, Observability 33% → 54%) are both already fully covered by existing exercises. The AWS Bedrock AgentCore watchpoint from last cycle reversed; the MCP OAuth-2.1 watchpoint continues to lag spec changes. Re-run on the next quarterly cycle (2027-01) to catch the delayed language propagation and the AgentCore trajectory recheck.
