# Labs for mod-205-evaluation-observability (Evaluation & Observability Instrumentation)

Guided, step-by-step practice for [mod-205](../README.md). Labs are shorter and more scaffolded than the module's [exercises](../exercises/) — they warm you up for each chapter's pattern against mocked data, so you can see the eval/trace/gate behavior before Exercises 01–04 ask you to wire the same shape against a real agent and a real platform.

| # | Lab | Duration | Covers |
|---|---|---|---|
| 01 | [Score Mocked Trajectories Under Each Match Mode](lab-01-score-trajectories-under-each-match-mode.md) | ~40 min | [Chapter 1](../01-trajectory-tool-call-eval.md) |
| 02 | [Watch GenAI Spans Nest in the Console](lab-02-watch-genai-spans-nest-in-the-console.md) | ~40 min | [Chapter 2](../02-otel-genai-tracing.md) |
| 03 | [Walk a Regression Gate Red and Back to Green](lab-03-walk-a-regression-gate-red-and-back-to-green.md) | ~45 min | [Chapter 4](../04-regression-eval-suites.md) |

Suggested order: run each lab **before** the exercise that covers the same chapter. Lab 01 is a precursor to [Exercise 01](../exercises/exercise-01-trajectory-eval-implementation.md); Lab 02 to [Exercise 02](../exercises/exercise-02-otel-tracing-wireup.md); Lab 03 to [Exercise 04](../exercises/exercise-04-agent-regression-suite.md), and it builds on Lab 01's trajectory scorer. Chapter 3's judge is covered directly by [Exercise 03](../exercises/exercise-03-llm-judge-scoring.md) — hand-labeling and bias probes don't lend themselves to a short mock-based lab, so there's no companion lab here.

Every lab is offline — no API keys, no platform accounts, no network. The mocked inputs are what let you *see* each chapter's control flow in isolation before the real-agent, real-platform work in the exercises.
