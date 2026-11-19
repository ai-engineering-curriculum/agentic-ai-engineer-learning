# Labs for mod-206-guardrails-implementation (Guardrails & Safety Implementation)

Guided, step-by-step practice for [mod-206](../README.md). Labs are shorter and more scaffolded than the module's [exercises](../exercises/) — they warm you up for each chapter's pattern against mocked moderation, a scripted adversarial model, and a scripted approver, so you can see each control fire before the exercises ask you to wire the same shape against a real agent and a real backend.

| # | Lab | Duration | Covers |
|---|---|---|---|
| 01 | [Build the GuardrailPolicy with Mocked Moderation](lab-01-build-the-guardrail-policy-with-mocked-moderation.md) | ~45 min | [Chapter 1](../01-io-moderation.md) |
| 02 | [Defeat an Indirect Injection with a Scripted Adversarial Model](lab-02-defeat-an-indirect-injection-with-a-scripted-adversarial-model.md) | ~50 min | [Chapter 2](../02-prompt-injection.md), with containment from [Chapter 3](../03-tool-permissions.md) |
| 03 | [Gate High-Risk Tool Calls with a Scripted Approver](lab-03-gate-high-risk-tool-calls-with-a-scripted-approver.md) | ~45 min | [Chapter 4](../04-human-approval.md) |

Suggested order: run each lab **before** the exercise that covers the same chapter. Lab 01 is a precursor to [Exercise 01](../exercises/exercise-01-io-moderation-guardrails.md); Lab 02 to [Exercise 02](../exercises/exercise-02-prompt-injection-defenses.md); Lab 03 to [Exercise 04](../exercises/exercise-04-human-approval-checkpoints.md). Chapter 3's pure-mechanics work — path confinement, URL allowlisting, sandboxing an `exec` tool — is covered head-on by [Exercise 03](../exercises/exercise-03-tool-permission-enforcement.md) and previewed inside Lab 01's tool-call rail and Lab 02's containment step, so there's no standalone Lab 3-for-Chapter-3 here.

Every lab is offline — no API keys, no model calls, no network. Moderation is a keyword stub (Lab 01), the model is a scripted adversarial Python function (Lab 02), and the approver is a dictionary of pre-set decisions (Lab 03). The mocked backends are what let you *see* each chapter's control flow in isolation before the real-agent, real-backend work in the exercises.
