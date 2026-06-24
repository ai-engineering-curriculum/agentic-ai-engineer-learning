# Resources for mod-206-guardrails-implementation

Primary references for guardrails and agent safety. Verify against current docs — this space moves fast and threat guidance is revised often.

## Threats and standards

- **OWASP Top 10 for LLM Applications** ([genai.owasp.org/llm-top-10](https://genai.owasp.org/llm-top-10/)) — the canonical risk list. Start with **LLM01: Prompt Injection** and **LLM02: Sensitive Information Disclosure**. This is the threat model the whole module defends against.
- **OWASP GenAI Security Project** ([genai.owasp.org](https://genai.owasp.org/)) — the umbrella for the LLM Top 10 plus deeper guides on agentic threats and mitigations.

## Prompt injection

- **Anthropic — Mitigate jailbreaks and prompt injections** ([docs.anthropic.com/en/docs/test-and-evaluate/strengthen-guardrails/mitigate-jailbreaks](https://docs.anthropic.com/en/docs/test-and-evaluate/strengthen-guardrails/mitigate-jailbreaks)) — practical, code-oriented techniques for hardening against injection and jailbreaks.
- **Anthropic — Reduce prompt leak** ([docs.anthropic.com/en/docs/test-and-evaluate/strengthen-guardrails/reduce-prompt-leak](https://docs.anthropic.com/en/docs/test-and-evaluate/strengthen-guardrails/reduce-prompt-leak)) — keeping system prompts and secrets out of model output.
- **Anthropic — Building effective agents** ([anthropic.com/engineering/building-effective-agents](https://www.anthropic.com/engineering/building-effective-agents)) — the human-in-the-loop and tool-design guidance underpinning Chapters 3 and 4.

## Guardrail frameworks

- **NeMo Guardrails** ([github.com/NVIDIA/NeMo-Guardrails](https://github.com/NVIDIA/NeMo-Guardrails), docs at [docs.nvidia.com/nemo/guardrails](https://docs.nvidia.com/nemo/guardrails/latest/index.html)) — input/output rails and dialogue flows defined in config; used in exercise-01.
- **Guardrails AI** ([guardrailsai.com/docs](https://www.guardrailsai.com/docs), hub at [hub.guardrailsai.com](https://hub.guardrailsai.com/)) — composable validators (PII, toxicity, regex, structured-output) you assemble into a `Guard`; the validator-centric alternative in exercise-01.

## Moderation and sandboxing

- **OpenAI — Moderation guide** ([platform.openai.com/docs/guides/moderation](https://platform.openai.com/docs/guides/moderation)) — a reference moderation API and category taxonomy for the input/output rails in Chapter 1.
- **Python `subprocess`** ([docs.python.org/3/library/subprocess.html](https://docs.python.org/3/library/subprocess.html)) and **`resource`** ([docs.python.org/3/library/resource.html](https://docs.python.org/3/library/resource.html)) — process isolation, timeouts, and rlimits for the sandboxing exercise.

> You built each control by hand in this module. Frameworks package the moderation/validation catalog, but the security-critical tool-call rails, permission policies, and approval gates are short enough to own outright — and you should read every line.
