# Job Requirements — Agentic AI Engineer

**Role level:** 30 (build altitude)
**Track:** `agentic-ai-engineer-learning`
**Research window:** 2026-06-05 → 2026-09-03 (last 90 days)
**Today:** 2026-09-03
**Prior refresh:** 2026-08-03

This file maps verbatim requirements from current Agentic AI Engineer job postings to the existing curriculum. Raw normalized data lives in [`.aicg/job-requirements.json`](.aicg/job-requirements.json); the strictly-additive proposal lives in [`.aicg/curriculum-plan-delta.json`](.aicg/curriculum-plan-delta.json).

## Summary

- Postings sampled: **18** in-window (2026-06-05 → 2026-09-03). This is a 30-day delta check against the prior 2026-08-03 refresh (which sampled 37 postings and reached the same zero-additions conclusion), not a full re-baseline. Ashby-hosted (Netomi, APAS, Planera, Clera, Glacis) and Lever pages continued to block WebFetch; sample quality was preferred over padding to 25+.
- Equivalent titles counted: `Agentic AI Engineer`, `AI Agent Engineer`, `Agent Engineer`, `Applied AI Engineer`, `Generative AI Engineer`, `LLM Engineer`. Title mix: 8 Agentic AI Engineer, 2 AI Agent Engineer, 3 Applied AI Engineer, 2 Generative AI Engineer, 2 AI Engineer, 1 LLM Engineer.
- **Proposed delta this cycle: 0 modules, 0 exercises, 0 projects.** Every requirement above the continuity-bias threshold (≥3 distinct postings AND ≥30% frequency) is already owned by an existing module in `mod-201..207`.
- The three sub-threshold signals worth watching from last cycle all shifted, but none crossed 30%:
  - **AWS Bedrock AgentCore + Strands Agents**: 3% → 22% (Cloud202, Cognitive Space, Cbase, Sharebite). Meaningful climb; will be absorbed into mod-202 exercise-05 comparison if it crosses threshold next cycle.
  - **Google ADK**: 8% → 17% (TTEC, Realtech, Taller). Still below bar; already in mod-202 objectives.
  - **Voice / realtime agents**: 5% → 6% (Carbon Technology only). Voice work is cordoning off into specialty `Senior Voice AI Engineer` titles rather than surfacing in general Agentic AI Engineer requirements — mod-208 voice sibling remains unwarranted.
- Credential management / OAuth for agents held flat at 6% in job-requirement language *despite* the 2026-07-28 MCP spec now mandating OAuth 2.1 + PKCE + RFC 8707 Resource Indicators for remote MCP servers. Requirement wording lags spec changes; this is the strongest candidate to cross threshold in the next cycle.

## Methodology

- Sources: greenhouse.io/job_boards (bulk, full page fetches), jobs.ashbyhq.com (snippet/mirror extraction where JS-only rendering blocked WebFetch), jobs.lever.co (403 to WebFetch; mirror/snippet extraction), builtin.com, dice.com, techstars.com, linkedin.com/jobs (titles + snippets only), talent.com.
- Per-posting capture: employer, title, URL, date_observed, date_posted (marked `estimated:YYYY-MM` when inferred), location, 4–8 verbatim/near-verbatim requirement quotes.
- Frequency = distinct in-window postings citing the theme ÷ 18 in-window postings.
- Threshold for a new curriculum item: ≥ 3 postings AND ≥ 30% frequency AND no existing module/exercise can be incrementally extended to cover it.
- Source caveats documented in `summary.source_quality_caveats` in `.aicg/job-requirements.json`.

## Requirement themes → curriculum ownership

The table below lists every requirement theme observed, its in-window frequency, the role that owns it per the level hierarchy, and the curriculum coverage path. **Bold frequencies** are ≥ 30% (load-bearing under continuity bias).

| # | Theme | Freq | Owner role | Coverage |
|---|---|---|---|---|
| 1 | Agent frameworks: LangGraph, LangChain, CrewAI, AutoGen, OpenAI Agents SDK, Google ADK, Anthropic/Claude Agent SDK, LlamaIndex, Semantic Kernel, Pydantic AI, Vercel AI SDK, smolagents, Strands | **56%** | `agentic-ai-engineer` (this) | [`mod-202-frameworks`](lessons/mod-202-frameworks) |
| 2 | RAG, vector DBs, embeddings, agent memory | **56%** | `agentic-ai-engineer` | [`mod-203-rag-and-memory`](lessons/mod-203-rag-and-memory) |
| 3 | Multi-agent orchestration (orchestrator-worker, handoffs, sub-agents, agent-to-agent, state-graph) | **50%** | `agentic-ai-engineer` | [`mod-204-multi-agent-implementation`](lessons/mod-204-multi-agent-implementation) |
| 4 | Evaluation harnesses (LLM-as-judge, regression suites, golden traces, CI-integrated eval bar) | **56%** | `agentic-ai-engineer` | [`mod-205-evaluation-observability`](lessons/mod-205-evaluation-observability) |
| 5 | Observability (Langfuse, LangSmith, Phoenix, OTel GenAI, Datadog, Coralogix) | **33%** | `agentic-ai-engineer` | [`mod-205-evaluation-observability/exercises/exercise-02-otel-tracing-wireup.md`](lessons/mod-205-evaluation-observability/exercises/exercise-02-otel-tracing-wireup.md) |
| 6 | Function / tool calling, structured outputs (Pydantic / JSON schema) | **44%** | `agentic-ai-engineer` | [`mod-201-agent-fundamentals/exercises/exercise-02-function-calling-tools.md`](lessons/mod-201-agent-fundamentals/exercises/exercise-02-function-calling-tools.md) |
| 7 | Model Context Protocol (MCP) — consumption AND authoring | **44%** | `agentic-ai-engineer` | [`mod-202-frameworks/exercises/exercise-04-mcp-tool-server.md`](lessons/mod-202-frameworks/exercises/exercise-04-mcp-tool-server.md) |
| 8 | API deployment (FastAPI/Flask, REST/GraphQL, async Python, Docker/K8s/IaC, CI/CD) | **78%** | `agentic-ai-engineer` | [`mod-207-productionizing-agents/exercises/exercise-01-agent-api-deployment.md`](lessons/mod-207-productionizing-agents/exercises/exercise-01-agent-api-deployment.md) |
| 9 | Guardrails, prompt injection defense, safety/policy controls, permissions, audit logs | **39%** | `agentic-ai-engineer` (basics) → `ai-infra-security-learning` (depth) | [`mod-206-guardrails-implementation`](lessons/mod-206-guardrails-implementation); deep agent attack surface → `ai-infra-security-learning` (level 35) |
| 10 | Durable execution / workflow engines (Temporal, Airflow, managed agent runtimes) | 0% | `agentic-ai-engineer` | [`mod-207-productionizing-agents/exercises/exercise-02-durable-execution-temporal.md`](lessons/mod-207-productionizing-agents/exercises/exercise-02-durable-execution-temporal.md) |
| 11 | Cost / latency optimization (prompt caching, model routing, token budgets) | 17% | `agentic-ai-engineer` | [`mod-207-productionizing-agents/exercises/exercise-03-caching-and-routing.md`](lessons/mod-207-productionizing-agents/exercises/exercise-03-caching-and-routing.md) |
| 14 | Human-in-the-loop oversight / escalation | 6% | `agentic-ai-engineer` | [`mod-206-guardrails-implementation/exercises/exercise-04-human-approval-checkpoints.md`](lessons/mod-206-guardrails-implementation/exercises/exercise-04-human-approval-checkpoints.md), [`mod-207-productionizing-agents/exercises/exercise-04-hitl-with-persistence.md`](lessons/mod-207-productionizing-agents/exercises/exercise-04-hitl-with-persistence.md) |
| 16 | Voice / realtime / telephony agents | 6% | sub-threshold | External link-outs (below); mod-208 sibling still unwarranted |
| 18 | AWS-native agent runtime (Bedrock AgentCore, Strands Agents) | 22% | sub-threshold (rising) | [`mod-202-frameworks/exercises/exercise-05-framework-tradeoff-bakeoff.md`](lessons/mod-202-frameworks/exercises/exercise-05-framework-tradeoff-bakeoff.md) — absorbs if it crosses threshold next cycle |
| 19 | Google ADK + AgentSpace as named production stack | 17% | `agentic-ai-engineer` | Already in mod-202 objectives & exercise-05 comparison scope |
| 21 | A2A (Agent-to-Agent) protocol — inter-agent communication protocol | 11% | `agentic-ai-engineer` | [`mod-204-multi-agent-implementation`](lessons/mod-204-multi-agent-implementation) — agent-to-agent protocols already in module scope |
| 22 | "Context engineering" as named discipline distinct from prompt engineering | 17% | `agentic-ai-engineer` | Vocabulary shift, not content shift — surface in mod-201/mod-203 READMEs on next content pass |

## Posting evidence for load-bearing themes

The table below lists the postings that anchor each ≥30% theme in this window. All 18 sampled postings fall within 2026-06-05 → 2026-09-03.

### Theme 1 — Agent frameworks (56%)

| Employer | Title | URL | Date observed | Posted |
|---|---|---|---|---|
| Future | Applied AI Engineer | https://job-boards.greenhouse.io/future/jobs/4683133005 | 2026-09-03 | est:2026-08 |
| Cloud202 | AI Engineer | https://www.gravityer.com/jobs/ai-engineer-cloud202 | 2026-09-03 | 2026-06-02 |
| Cognitive Space | AI Engineer — Agentic Workflows | https://jobs.techstars.com/companies/cognitive-space/jobs/67412130-ai-engineer-agentic-workflows | 2026-09-03 | est:2026-07 |
| Netomi | Staff Agentic AI Engineer | https://jobs.lever.co/netomi/3fe31ab4-1e79-4b1c-8493-dfec68e69ba7 | 2026-09-03 | est:2026-08-06 |
| Sharebite | AI Engineer | https://job-boards.greenhouse.io/sharebite/jobs/5848230004 | 2026-09-03 | est:2026-08 |
| TTEC | Agentic AI Engineer (Google ADK) | https://ph.talent.com/view?id=629808943968619465 | 2026-09-03 | est:2026-08 |
| Realtech Services | Gen AI Engineer — Google ADK & Agentic AI | https://www.dice.com/job-detail/ef9b17af-409d-402d-b552-4d2976bfa1f2 | 2026-09-03 | est:2026-07 |
| Taller | Agentic AI Engineer | https://builtin.com/job/102735-agentic-ai-engineer/10537194 | 2026-09-03 | est:2026-07 |
| Vercel | Software Engineer, AI SDK | https://vercel.com/careers/software-engineer-ai-sdk-5474915004 | 2026-09-03 | est:2026-08-27 |
| Planera | Senior AI Agent Engineer | https://jobs.ashbyhq.com/planera/d68c8a09-a11d-409e-85ca-5d434caf3fc8 | 2026-09-03 | 2026-06-29 |

Representative quote: *"Evaluate Deep Agent, Claude Agent, OpenAI Agent, LangGraph, and other emerging agent orchestration patterns"* — Netomi.

→ Covered by [`mod-202-frameworks`](lessons/mod-202-frameworks). Strands Agents and Google ADK can be woven into `exercise-05-framework-tradeoff-bakeoff` on the next content pass — no delta needed.

### Theme 2 — RAG, vector DBs, memory (56%)

Anchored by Future, Cloud202, Cognitive Space, Netomi, Cbase, Sharebite, EvolutionIQ, Realtech, DeepSeas, APAS. Named tools consistent with prior cycle: pgvector, Pinecone, Weaviate, Qdrant, FAISS, Chroma.

Representative quote: *"Hands-on experience implementing RAG architectures and vector search solutions"* — Realtech Services.

→ Covered by [`mod-203-rag-and-memory`](lessons/mod-203-rag-and-memory).

### Theme 3 — Multi-agent orchestration (50%)

Anchored by iCapital, Cloud202, Cognitive Space, Netomi, Cbase, TTEC, Realtech, Taller, Accenture.

Representative quote: *"Deep understanding of multi-agent orchestration patterns, state graph architectures, and deterministic routing"* — TTEC.

→ Covered by [`mod-204-multi-agent-implementation`](lessons/mod-204-multi-agent-implementation). A2A protocol (Cloud202) fits `exercise-02-agent-handoffs`. Higher-altitude architecture stays with `agentic-systems-architect-learning` (level 48).

### Theme 4 — Evaluation harnesses (56%)

Anchored by Future, Cloud202, Cognitive Space, Netomi, Cbase, Sharebite, EvolutionIQ, Realtech, TTEC, Planera.

Representative quote: *"Experience developing AI evaluation systems, including LLM-as-judge, guardrail evaluation, regression testing, and production quality measurement"* — Netomi.

→ Covered by [`mod-205-evaluation-observability`](lessons/mod-205-evaluation-observability). AgentCore Evaluations (Cloud202) is the AWS-native flavor of the same pattern — no separate coverage needed.

### Theme 5 — Observability (33%)

Just crossed the threshold this cycle (was 27% prior). Anchored by Future (Langfuse + OTel + Datadog), Cognitive Space (tracing, tool-call success rates), Netomi, Cbase, Cloud202 (AgentCore Evaluations), Sharebite.

Representative quote: *"Establish observability for agent runs (tracing, failure analysis, latency/cost monitoring, tool-call success rates)"* — Cognitive Space.

→ Covered by [`mod-205-evaluation-observability/exercises/exercise-02-otel-tracing-wireup.md`](lessons/mod-205-evaluation-observability/exercises/exercise-02-otel-tracing-wireup.md). Langfuse + OTel GenAI are already the canonical stack in the exercise.

### Theme 6 — Function / tool calling, structured outputs (44%)

Anchored by Future, Cognitive Space, Netomi, TTEC, Realtech, Accenture, Cbase, Cloud202.

Representative quote: *"Implement and maintain a robust tool interface layer (tool schemas/contracts, structured I/O, validation, retries)"* — Cognitive Space.

→ Covered by [`mod-201-agent-fundamentals/exercises/exercise-02-function-calling-tools.md`](lessons/mod-201-agent-fundamentals/exercises/exercise-02-function-calling-tools.md).

### Theme 7 — MCP (44%)

Anchored by Cloud202, Netomi, Cbase, TTEC, Taller, Planera, Cognitive Space, Sharebite. Postings continue to shift from *consuming* MCP tools to *authoring* MCP servers wrapped around enterprise systems.

Representative quote: *"Implement and extend Model Context Protocol (MCP) servers and clients to integrate enterprise tools"* — Cbase.

→ Covered by [`mod-202-frameworks/exercises/exercise-04-mcp-tool-server.md`](lessons/mod-202-frameworks/exercises/exercise-04-mcp-tool-server.md), which includes an authoring lab. **Watchpoint:** the 2026-07-28 MCP spec now mandates OAuth 2.1 + PKCE + RFC 8707 Resource Indicators for remote MCP servers; that language has not yet surfaced in job requirements but is the highest-probability next-cycle signal. If MCP-authoring-with-enterprise-auth crosses 30% next cycle, propose a second MCP exercise focused on auth-wrapped MCP servers.

### Theme 8 — API deployment / backend / containerization (78%)

Sample-composition effect: this cycle's sample skews toward postings that explicitly named cloud/DevOps stacks. Anchored by Future, Cloud202, Cognitive Space, Netomi, Cbase, Sharebite, EvolutionIQ, TTEC, Realtech, CrowdStrike, Accenture, Vercel, Planera, iCapital.

Representative quote: *"Deploy and operate AI agents securely at scale with serverless infrastructure"* — Cloud202.

→ Covered by [`mod-207-productionizing-agents/exercises/exercise-01-agent-api-deployment.md`](lessons/mod-207-productionizing-agents/exercises/exercise-01-agent-api-deployment.md). IaC (Terraform, CDK) surfaces alongside deployment but stays owned by `ai-infra-platform-engineer-learning` per hierarchy.

### Theme 9 — Guardrails / safety / policy (39%) — climbed from 32%, still fully covered

Anchored by Cognitive Space (permissions, policy checks, audit logs), Netomi, Cbase, TTEC, Taller, Realtech (governance/compliance), Accenture.

Representative quote: *"Implement safety and governance controls (permissions, policy checks, audit logs)"* — Cognitive Space.

→ Covered by [`mod-206-guardrails-implementation`](lessons/mod-206-guardrails-implementation): four exercises cover I/O moderation, prompt-injection defenses (OWASP LLM01), tool-permission enforcement, human-approval checkpoints.

**Continuity-bias check:** the theme frequency climbed from 32% → 39%. The named sub-themes (permissions, policy-as-code, audit logs, prompt injection defense) remain covered by mod-206's four existing exercises. The credential-management sub-theme (OAuth 2.0/OIDC, API key vending, RBAC) actually held flat at 6% this cycle despite the 2026-07-28 MCP spec change mandating OAuth 2.1 + PKCE + RFC 8707 for remote MCP servers. Requirement wording lags spec changes — expect this to surface in the next cycle. If credential-mgmt + enterprise-auth crosses 30% next cycle, propose a mod-206 exercise-05 addition then.

## Themes just below threshold this cycle

### Theme 18 — AWS-native agent stack (22%) — rising fast

Cloud202 (full Bedrock AgentCore + Strands Agents stack), Cognitive Space ("LangChain/Strands/Bedrock"), Cbase ("AWS Agent Core or similar"), Sharebite ("LangChain, LangGraph, or AWS Bedrock").

Climbed 3% → 22% this cycle. If it hits 30% next cycle, mod-202/exercise-05-framework-tradeoff-bakeoff absorbs Strands/AgentCore into the comparison — no new module needed. AgentCore Evaluations is a rebranding of standard eval workflows and does not require separate curriculum coverage.

### Theme 19 — Google ADK + AgentSpace (17%)

TTEC ("official open-source Google Agent Development Kit"), Realtech, Taller ("Google Agentic Orchestration").

Climbed 8% → 17%. Still below threshold. ADK is already listed in mod-202 objectives and exercise-05 comparison scope.

### Theme 21 — A2A (Agent-to-Agent) protocol (11%) — new signal

Cloud202 explicit: *"Build inter-agent communication systems using A2A protocol for peer-to-peer agent collaboration."* Netomi and Cbase use supporting language.

New named requirement this cycle. mod-204 already covers agent-to-agent protocols generically. Watch for A2A 1.0 spec to enter requirements next cycle.

### Theme 22 — "Context engineering" as named discipline (17%) — new vocabulary

iCapital ("context engineering"), DeepSeas ("Strong context engineering skills. Able to curate, compress, and structure large datasets"), Anthropic Applied AI (cross-reference).

Vocabulary shift, not content shift. Falls inside the prompt-design + retrieval-shaping + memory-curation loop already taught in mod-201 (prompt design) and mod-203 (retrieval + memory). Surface the term in mod-201 README on next content pass.

## Sub-threshold new signals — tracked for next cycle

### Voice / realtime / telephony agents — 6% (flat)

Carbon Technology's *"Senior Voice AI Engineer (Founding Team)"* is the only anchor in-window. Voice work continues to cordon off into specialty *Voice AI Engineer* titles rather than surfacing in general Agentic AI Engineer requirements — the previously-set "cross 15% justifies mod-208" trigger did not fire.

External resources for learners interested now:
- LiveKit Agents — https://docs.livekit.io/agents
- Vapi — https://docs.vapi.ai
- Pipecat — https://docs.pipecat.ai
- Deepgram — https://developers.deepgram.com
- ElevenLabs — https://elevenlabs.io/docs
- Cartesia — https://docs.cartesia.ai

### Code / computer-use agents — did not resample separately this cycle

Coverage stands at mod-201/exercise-03 for build-altitude basics; computer-use depth remains out-of-scope link-outs:
- Anthropic Computer Use — https://docs.anthropic.com/en/docs/build-with-claude/computer-use
- e2b sandbox — https://e2b.dev/docs
- Modal sandboxes — https://modal.com/docs/guide/sandbox
- BrowserBase — https://docs.browserbase.com

### Agent modeling / post-training — remains owned elsewhere

Fine-tuning / SFT / RLHF / LoRA mentions stay owned by `ai-infra-ml-platform-learning` (level 30). No change.

### AI-assisted developer-tool fluency — 6% (down from 14%)

Only Netomi (Staff level) explicitly named daily use of Codex/Claude Code/Cursor as a requirement this cycle. This remains a work-style expectation, not a curriculum-worthy build skill.

## Conclusion

<!-- needs-research: monitor MCP-authoring-with-OAuth-2.1 requirement wording in the next cycle following the 2026-07-28 MCP spec change; monitor AWS Bedrock AgentCore + Strands Agents (currently 22%, rising fast) — if either crosses 30%, propose a mod-206 exercise-05 (auth-wrapped MCP) or a mod-202 exercise-05 addendum (Strands/AgentCore in the comparison). -->

The Agentic AI Engineer curriculum (mod-201..207 plus `project-201-production-multi-agent-system` and `project-202-benchmark-agent`) covers every job-market requirement that clears the continuity-bias thresholds. **No delta is proposed this cycle** — this is the second consecutive zero-additions refresh (2026-08-03 and 2026-09-03). Re-run on the next quarterly cycle (2026-12) to catch the delayed language propagation from the 2026-07-28 MCP OAuth-2.1 spec change and to check whether AWS Bedrock AgentCore + Strands Agents has crossed the 30% frequency threshold.
