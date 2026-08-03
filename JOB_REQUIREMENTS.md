# Job Requirements — Agentic AI Engineer

**Role level:** 30 (build altitude)
**Track:** `agentic-ai-engineer-learning`
**Research window:** 2026-05-05 → 2026-08-03 (last 90 days)
**Today:** 2026-08-03
**Prior refresh:** 2026-06-15

This file maps verbatim requirements from current Agentic AI Engineer job postings to the existing curriculum. Raw normalized data lives in [`.aicg/job-requirements.json`](.aicg/job-requirements.json); the strictly-additive proposal lives in [`.aicg/curriculum-plan-delta.json`](.aicg/curriculum-plan-delta.json).

## Summary

- Postings sampled: **37** (all in the 2026-05-05 → 2026-08-03 window).
- Equivalent titles counted: `Agentic AI Engineer`, `AI Agent Engineer`, `Agent Engineer`, `Applied AI Engineer`, `Generative AI Engineer`, `LLM Engineer`.
- **Proposed delta this cycle: 0 modules, 0 exercises, 0 projects.** Every requirement above the continuity-bias threshold (≥3 distinct postings AND ≥30% frequency) is already owned by an existing module in `mod-201..207`.
- Guardrails/security frequency climbed from 18% → 32% — crosses the threshold, but every specific sub-theme (I/O moderation, prompt injection, tool-permission enforcement, HITL approvals) is already covered by mod-206's four exercises. The emerging *credential-management* sub-theme is at 3 postings / 8% frequency — sub-threshold, tracked for next cycle.
- Sub-threshold new signals tracked: AWS-native agent stack (Strands/AgentCore) 1 posting, Google ADK 3 postings (already in mod-202 scope), Claude Agent SDK as runtime 3+ postings (already in mod-202 scope), Pydantic AI + Vercel AI SDK 1–2 postings, AI-assisted dev-tool fluency 5 postings (not a curriculum-worthy build skill).

## Methodology

- Sources: greenhouse.io/job_boards (bulk), jobs.ashbyhq.com (postings extracted via search snippets when JS-only rendering blocked WebFetch), jobs.lever.co (403 to WebFetch; snippet extraction), builtin.com.
- Per-posting capture: employer, title, URL, date_observed, date_posted (marked `estimated:YYYY-MM` when inferred), location, 4–8 verbatim/near-verbatim requirement quotes.
- Frequency = distinct in-window postings citing the theme ÷ 37 in-window postings.
- Threshold for a new curriculum item: ≥ 3 postings AND ≥ 30% frequency AND no existing module/exercise can be incrementally extended to cover it.
- Source caveats documented in `summary.source_quality_caveats` in `.aicg/job-requirements.json`.

## Requirement themes → curriculum ownership

The table below lists every requirement theme observed, its in-window frequency, the role that owns it per the level hierarchy, and the curriculum coverage path. **Bold frequencies** are ≥ 30% (load-bearing under continuity bias).

| # | Theme | Freq | Owner role | Coverage |
|---|---|---|---|---|
| 1 | Agent frameworks: LangGraph, CrewAI, AutoGen, OpenAI Agents SDK, Google ADK, Anthropic/Claude Agent SDK, LlamaIndex, Semantic Kernel, Pydantic AI, Vercel AI SDK, smolagents | **65%** | `agentic-ai-engineer` (this) | [`mod-202-frameworks`](lessons/mod-202-frameworks) |
| 2 | RAG, vector DBs, embedding strategies, agent memory | **49%** | `agentic-ai-engineer` | [`mod-203-rag-and-memory`](lessons/mod-203-rag-and-memory) |
| 3 | Multi-agent orchestration (orchestrator-worker, handoffs, sub-agents, agent-to-agent) | **32%** | `agentic-ai-engineer` | [`mod-204-multi-agent-implementation`](lessons/mod-204-multi-agent-implementation) |
| 4 | Evaluation harnesses (trajectory, LLM-as-judge, regression suites, CI-integrated evals) | **46%** | `agentic-ai-engineer` | [`mod-205-evaluation-observability`](lessons/mod-205-evaluation-observability) |
| 5 | Observability (Langfuse, LangSmith, Phoenix, OTel GenAI, Datadog, Coralogix) | 27% | `agentic-ai-engineer` | [`mod-205-evaluation-observability/exercises/exercise-02-otel-tracing-wireup.md`](lessons/mod-205-evaluation-observability/exercises/exercise-02-otel-tracing-wireup.md) |
| 6 | Function / tool calling, structured outputs (Pydantic / JSON schema) | **35%** | `agentic-ai-engineer` | [`mod-201-agent-fundamentals/exercises/exercise-02-function-calling-tools.md`](lessons/mod-201-agent-fundamentals/exercises/exercise-02-function-calling-tools.md) |
| 7 | Model Context Protocol (MCP) — consumption AND authoring | 27% | `agentic-ai-engineer` | [`mod-202-frameworks/exercises/exercise-04-mcp-tool-server.md`](lessons/mod-202-frameworks/exercises/exercise-04-mcp-tool-server.md) |
| 8 | API deployment (FastAPI/Flask, REST/GraphQL, async Python, Docker/K8s/IaC) | **46%** | `agentic-ai-engineer` | [`mod-207-productionizing-agents/exercises/exercise-01-agent-api-deployment.md`](lessons/mod-207-productionizing-agents/exercises/exercise-01-agent-api-deployment.md) |
| 9 | Guardrails, prompt injection, OWASP LLM Top 10, agent sandboxing, credential mgmt, enterprise compliance | **32%** | `agentic-ai-engineer` (basics) → `ai-infra-security-learning` (depth) | [`mod-206-guardrails-implementation`](lessons/mod-206-guardrails-implementation); deep agent attack surface → `ai-infra-security-learning` (level 35) |
| 10 | Durable execution / workflow engines (Temporal, Airflow, managed agent runtimes) | 11% | `agentic-ai-engineer` | [`mod-207-productionizing-agents/exercises/exercise-02-durable-execution-temporal.md`](lessons/mod-207-productionizing-agents/exercises/exercise-02-durable-execution-temporal.md) |
| 11 | Cost / latency optimization (prompt caching, model routing, token budgets, LLM gateways) | 14% | `agentic-ai-engineer` | [`mod-207-productionizing-agents/exercises/exercise-03-caching-and-routing.md`](lessons/mod-207-productionizing-agents/exercises/exercise-03-caching-and-routing.md) |
| 12 | Code / computer-use agents (Claude Code as build target, browser automation, code-execution sandboxes) | 19% | sub-threshold | mod-201 exercise-03 covers build-altitude basics; deeper computer-use is out-of-scope-link-out |
| 13 | Forward-deployed / customer-facing agent engineering | 5% | soft-skill | Surface in `project-201` capstone README; not a curriculum module |
| 14 | Human-in-the-loop oversight / escalation | 11% | `agentic-ai-engineer` | [`mod-206-guardrails-implementation/exercises/exercise-04-human-approval-checkpoints.md`](lessons/mod-206-guardrails-implementation/exercises/exercise-04-human-approval-checkpoints.md), [`mod-207-productionizing-agents/exercises/exercise-04-hitl-with-persistence.md`](lessons/mod-207-productionizing-agents/exercises/exercise-04-hitl-with-persistence.md) |
| 15 | Agent modeling / post-training (SFT, RLHF, LoRA, synthetic data) | 16% | `ai-infra-ml-platform-learning` (level 30) | Out of scope here — link to ML Platform track. |
| 16 | Voice / realtime / telephony agents | 5% | sub-threshold | Track; out-of-scope-link-out (see below) |
| 17 | AI-assisted dev-tool fluency (Claude Code, Cursor, Copilot as USER) | 14% | cross-cutting expectation | Not curriculum-worthy — surface as a prerequisite expectation |
| 18 | AWS-native agent runtime (Strands, AgentCore, Bedrock AgentCore) | 3% | sub-threshold | If it grows, mod-202 exercise-05 absorbs the comparison |
| 19 | Google ADK + AgentSpace as named production stack | 8% | `agentic-ai-engineer` | Already in mod-202 objectives & exercise-05 comparison scope |
| 20 | Low-code orchestration (n8n, Make, Zapier) alongside agent frameworks | 5% | sub-threshold | Out of scope for build-altitude Agentic AI Engineer |

## Posting evidence for load-bearing themes

The table below lists the postings that anchor each ≥30% theme in this window. All 37 sampled postings fall within 2026-05-05 → 2026-08-03.

### Theme 1 — Agent frameworks (65%)

| Employer | Title | URL | Date observed | Posted |
|---|---|---|---|---|
| LTS | Senior Agentic AI Software Engineer | https://job-boards.greenhouse.io/lts/jobs/4340374009 | 2026-08-03 | est:2026-07 |
| LTS | Senior Applied AI Engineer | https://job-boards.greenhouse.io/lts/jobs/4340498009 | 2026-08-03 | est:2026-07 |
| Fairmarkit | Agentic AI Engineer | https://job-boards.greenhouse.io/fairmarkit/jobs/6111188004 | 2026-08-03 | est:2026-07 |
| Snorkel AI | Applied AI Engineer - Enterprise Solutions | https://job-boards.greenhouse.io/snorkelai/jobs/5709067004 | 2026-08-03 | est:2026-07 |
| Future | Applied AI Engineer | https://job-boards.greenhouse.io/future/jobs/4683133005 | 2026-08-03 | est:2026-07 |
| Opaque Systems | Forward Deployed Engineer (AI) | https://job-boards.greenhouse.io/opaquesystems/jobs/4235505009 | 2026-08-03 | est:2026-07 |
| Capco | AI Engineer \| Banking | https://job-boards.greenhouse.io/capco/jobs/8080908 | 2026-08-03 | est:2026-07 |
| BrightAI | Senior AI Engineer - LLM, RAG | https://job-boards.greenhouse.io/brightai/jobs/5616545004 | 2026-08-03 | est:2026-07 |
| Taxbit | Agentic AI Engineer | https://job-boards.greenhouse.io/taxbit/jobs/6111141004 | 2026-08-03 | est:2026-07 |
| Extend | Senior AI Software Engineer, Internal Enablement | https://job-boards.greenhouse.io/extend/jobs/5989772004 | 2026-08-03 | 2026-05-02 |
| Air | Lead AI Engineer | https://job-boards.greenhouse.io/air/jobs/4114638009 | 2026-08-03 | est:2026-07 |
| Temporal | Software Engineer II, AI Foundations | https://job-boards.greenhouse.io/temporaltechnologies/jobs/5134414007 | 2026-08-03 | 2026-05-11 |
| Vercel | Software Engineer, AI SDK | https://job-boards.greenhouse.io/vercel/jobs/5474915004 | 2026-08-03 | est:2026-07 |
| Accuris | Agentic AI Engineer | https://builtin.com/job/agentic-ai-engineer-remote-6-month-contract/8816631 | 2026-08-03 | est:2026-05 |
| PayNearMe | Staff SWE - Agent Architecture | https://job-boards.greenhouse.io/paynearmeinc/jobs/4294822009 | 2026-08-03 | est:2026-07 |
| Splitero | Applied AI Engineer | https://job-boards.greenhouse.io/splitero/jobs/5162723008 | 2026-08-03 | est:2026-07 |
| Scale AI | Senior Staff Frontier Agents Engineer | https://job-boards.greenhouse.io/scaleai/jobs/4694869005 | 2026-08-03 | est:2026-07 |
| BLEN | AI Engineer | https://jobs.lever.co/blencorp/4b2e3689-9720-4785-b0fe-d09bd5325f74 | 2026-08-03 | est:2026-06 |
| Novara | Senior Applied AI Engineer | https://jobs.lever.co/novara/4756bca8-fe07-411f-8904-ec8660090912 | 2026-08-03 | est:2026-06 |
| Healx | Agentic AI Engineer (Life Sciences) | https://jobs.lever.co/healx/c1dc1b43-066f-427f-a299-0a0b0dc4748f | 2026-08-03 | est:2026-06 |
| Jobgether | AI Product Engineer | https://jobs.lever.co/jobgether/c53ae155-a701-4cb7-95b4-1c68791290d0 | 2026-08-03 | est:2026-06 |
| Planera | Senior AI Agent Engineer | https://jobs.ashbyhq.com/planera/d68c8a09-a11d-409e-85ca-5d434caf3fc8 | 2026-08-03 | 2026-06-29 |
| Capstone Investment Advisors | AI Infrastructure Engineer | https://job-boards.greenhouse.io/capstoneinvestmentadvisors/jobs/8427813002 | 2026-08-03 | est:2026-07 |
| Pipe17 | Junior SWE, AI-Native | https://job-boards.greenhouse.io/pipe17/jobs/4717950005 | 2026-08-03 | est:2026-07 |

Representative quote: *"plugin system that makes it easy to add Temporal's durable execution to popular open source agent frameworks such as Pydantic AI, AI SDK by Vercel, Google ADK, OpenAI Agents SDK"* — Temporal Technologies.

→ Covered by [`mod-202-frameworks`](lessons/mod-202-frameworks): builds the same agent across LangGraph, CrewAI, and AutoGen; benchmarks OpenAI Agents SDK, Google ADK, Anthropic Agent SDK, and smolagents in `exercise-05-framework-tradeoff-bakeoff`. Pydantic AI + Vercel AI SDK can be woven into the comparison on the next content pass — no delta needed.

### Theme 2 — RAG, vector DBs, memory (49%)

Anchored by LTS×2, Snorkel, Rackner, Mercury, WITHIN, Future, Cadence, Opaque, BrightAI, EvolutionIQ, BLEN, Scale AI, SecurityScorecard, Splitero, Xaira, Healx, Tessera. Named stacks: pgvector, Pinecone, Weaviate, Qdrant, Milvus, FAISS, Chroma, Azure AI Search.

Representative quote: *"Deep hands-on experience with RAG pipeline design: chunking strategies, embedding models, vector databases, retrieval quality evaluation, re-ranking."* — Opaque Systems.

→ Covered by [`mod-203-rag-and-memory`](lessons/mod-203-rag-and-memory).

### Theme 3 — Multi-agent orchestration (32%)

Anchored by Fairmarkit, Opaque, Capco, Air, Taxbit, Cadence, Xaira, SecurityScorecard, Tessera, Jobgether, Novara, Healx.

Representative quote: *"designing and implementing complex, multi-agent state machines and stateful graphs using LangGraph and LangChain"* — Novara.

→ Covered by [`mod-204-multi-agent-implementation`](lessons/mod-204-multi-agent-implementation). Agent-to-agent protocols (Jobgether) fit exercise-02-agent-handoffs. Higher-altitude architecture stays with `agentic-systems-architect-learning` (level 48).

### Theme 4 — Evaluation harnesses (46%)

Anchored by Honeycomb, Snorkel, Mercury, Rackner, Future, Opaque, Taxbit, Air, Gradial, Splitero, RxSense, Dialpad, Planera, Jobgether, Tessera, EvolutionIQ, PayNearMe.

Representative quote: *"evals into the CI/CD pipeline so no agent or LLM-powered service ships without passing a defined eval bar"* — RxSense.

→ Covered by [`mod-205-evaluation-observability`](lessons/mod-205-evaluation-observability). SWE-bench / TAU-bench (Gradial) sub-threshold; can be mentioned in exercise-04-agent-regression-suite on next content pass.

### Theme 6 — Function / tool calling, structured outputs (35%)

Anchored by LTS×2, WITHIN, Air, Accuris, BLEN, Cadence, Opaque, Taxbit, Xaira, Tessera, Healx, Snorkel.

Representative quote: *"Tool/function calling and Structured outputs (JSON schema)"* — WITHIN.

→ Covered by [`mod-201-agent-fundamentals/exercises/exercise-02-function-calling-tools.md`](lessons/mod-201-agent-fundamentals/exercises/exercise-02-function-calling-tools.md).

### Theme 8 — API deployment / backend / containerization (46%)

Anchored by LTS×2, WITHIN, Future, BLEN, Fairmarkit, Scale AI, SecurityScorecard, Air, Splitero, Extend, Capstone, Novara, Rackner, EvolutionIQ, Vercel, Pipe17.

Representative quote: *"Comfort with async Python, HTTP APIs, and streaming protocols (SSE, webhooks)."* — Future.

→ Covered by [`mod-207-productionizing-agents/exercises/exercise-01-agent-api-deployment.md`](lessons/mod-207-productionizing-agents/exercises/exercise-01-agent-api-deployment.md). IaC (Terraform, CDK) surfacing more but stays owned by `ai-infra-platform-engineer-learning` per hierarchy.

### Theme 9 — Guardrails / security / compliance (32%) — climbed from 18% but still fully covered

Anchored by Extend, Opaque, Capstone, Rackner, Mercury, Taxbit, Xaira, Jobgether, RxSense, Accuris, SecurityScorecard, Healx.

Representative quote: *"Experience with LLM application security: OWASP LLM Top 10, prompt injection defense, agent sandboxing."* — Extend.

→ Covered by [`mod-206-guardrails-implementation`](lessons/mod-206-guardrails-implementation): four exercises cover I/O moderation, prompt-injection defenses (OWASP LLM01), tool-permission enforcement, human-approval checkpoints. Deep depth stays with `ai-infra-security-learning` (level 35).

**Continuity-bias check:** the theme frequency climbs from 18% → 32%, crossing the threshold. But the four existing mod-206 exercises collectively cover every named sub-theme observed. The strongest *emerging* sub-theme is **credential management for agents** (OAuth 2.0/OIDC, API key vending, secret rotation, RBAC) — cited by only 3 postings (Extend, Capstone, Accuris) = 8% frequency. Sub-threshold. If credential-mgmt + enterprise-auth crosses 30% next cycle, propose a mod-206 exercise-05 addition then.

## Themes just below threshold this cycle

### Theme 7 — MCP (27%)

Anchored by Mercury, Extend, Pipe17, Capstone, Accuris, BLEN, Novara, Healx, Jobgether, Planera.

Representative quote: *"Build and maintain Model Context Protocol (MCP) services and supporting infrastructure."* — Capstone Investment Advisors.

Dropped from baseline 32% → 27% (denominator effect only — absolute count went 7 → 10). Postings are shifting from *consuming* MCP tools to *authoring* MCP servers, often wrapping legacy REST APIs behind enterprise auth.

→ Already covered by [`mod-202-frameworks/exercises/exercise-04-mcp-tool-server.md`](lessons/mod-202-frameworks/exercises/exercise-04-mcp-tool-server.md), which includes an authoring lab. If MCP-authoring-with-enterprise-auth crosses 30% next cycle, propose a second MCP exercise focused on auth-wrapped MCP servers. Deep "MCP attack surface" is owned by `ai-infra-security-learning`.

### Theme 5 — Observability (27%)

Consistent with baseline. Langfuse and OTel GenAI both explicitly named. Covered by exercise-02-otel-tracing-wireup.

## Sub-threshold new signals — tracked for next cycle

### AI-assisted dev-tool fluency (14%) — new signal

Employers now write "daily fluency with Claude Code / Cursor / Copilot" into requirements as a work-style expectation, not a technical build skill. Cited by Pipe17, Splitero, SecurityScorecard, Air, Extend.

*Not curriculum-worthy for the agent-build track.* Surface as a repo-wide prerequisites expectation if it crosses 40%+ next cycle.

### AWS-native agent runtime (3%) — new signal

Only Taxbit fully names the Strands Agents / AgentCore / Bedrock AgentCore stack this window. If ≥3 more postings appear, `mod-202/exercise-05-framework-tradeoff-bakeoff` can absorb by adding Strands/AgentCore to the comparison — no new module needed.

### Google ADK + AgentSpace as named stack (8%) — new signal

Cited by Capco, Healx, and Temporal (as plugin target). ADK is already listed in `mod-202` objectives and `exercise-05` comparison scope — coverage stands.

### Voice / realtime / telephony agents — 5% (flat)

Anchored by PayNearMe (LangGraph + ElevenLabs + Twilio, voice-first agent architecture) and Dialpad (NVIDIA NeMo, ESPnet, Coqui, ElevenLabs, Rime, Cartesia — TTS-quality tier). Named vendor stack has crystallized this window even though posting count is flat.

External resources for learners interested now:
- LiveKit Agents — https://docs.livekit.io/agents
- Vapi — https://docs.vapi.ai
- Pipecat — https://docs.pipecat.ai
- Deepgram — https://developers.deepgram.com
- ElevenLabs — https://elevenlabs.io/docs
- Cartesia — https://docs.cartesia.ai

If posting count reaches ≥ 3 in a future cycle *and* frequency crosses 15%, propose a `mod-208-voice-agents` sibling.

### Code / computer-use agents — 19% (flat)

Anchored by Pipe17, Splitero, SecurityScorecard, Air, Extend (Claude Code as build target), plus Gradial (SFT/RL for tool-using coding agents) and Temporal (Codex / Claude Code integration). `mod-201/exercise-03-coding-agent-read-write-execute` already covers build-altitude basics. Computer-use depth stays out-of-scope link-outs:
- Anthropic Computer Use — https://docs.anthropic.com/en/docs/build-with-claude/computer-use
- e2b sandbox — https://e2b.dev/docs
- Modal sandboxes — https://modal.com/docs/guide/sandbox
- BrowserBase — https://docs.browserbase.com
- Daytona sandboxes — https://www.daytona.io/docs
- SWE-bench — https://www.swebench.com
- TAU-bench — https://github.com/sierra-research/tau-bench

### Agent modeling / post-training — 16% (owned elsewhere)

Cadence, Gradial, BrightAI, Xaira, EvolutionIQ, Capco cite fine-tuning / SFT / RLHF / LoRA. Owned by `ai-infra-ml-platform-learning` (level 30).

### Forward-deployed / customer-facing — 5% (down from baseline)

Opaque Systems FDE (AI) is the only clear FDE-titled posting this window. Still a process pattern, not a curriculum module. Surface in `project-201-production-multi-agent-system` capstone README.

### Low-code orchestration (n8n / Make / Zapier) — 5% (new signal)

Splitero and Jobgether mention n8n/Make specifically for Applied-AI-Engineer-flavored roles. Not curriculum-worthy for the build-altitude agent-engineering track.

## Conclusion

<!-- needs-research: monitor guardrails/credential-management + MCP-authoring frequencies in the next cycle; if credential-mgmt crosses 30% or MCP-authoring hits 40%+, propose mod-206 exercise-05 and/or a second MCP exercise. Also monitor voice-agent and AWS-native-agent-stack posting counts — either could become module-worthy. -->

The Agentic AI Engineer curriculum (mod-201..207 plus `project-201-production-multi-agent-system` and `project-202-benchmark-agent`) covers every job-market requirement that clears the continuity-bias thresholds. **No delta is proposed this cycle.** Re-run on the next quarterly cycle (2026-11) to catch shifts in guardrails/credential-mgmt, MCP-authoring, and voice/computer-use agent demand.
