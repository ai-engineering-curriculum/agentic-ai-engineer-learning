# exercise-07: Add a Trust Layer with VeriSwarm

**Estimated effort:** 3 hours

## Objective

Connect your harness to an external trust service, using VeriSwarm as the example. Route permission decisions for side-effecting tools through VeriSwarm's Gate, keep personal data out of the model's context with Guard's PII tokenization, scan untrusted tool output for prompt injection, and stream tool-call events so VeriSwarm can score the agent's behavior and keep a tamper-evident audit trail. Every integration point is a harness extension point you already built. By the end you'll know how to add a trust layer to any harness, and which design choices (fail-open vs. fail-closed, caching, what data leaves the machine) are yours to make.

## Background

This exercise covers material from:

- [Chapter 9 — Adding a Trust Layer: VeriSwarm](../09-trust-layer-veriswarm.md)

Build on your [exercise-01](exercise-01-build-a-minimal-harness.md) harness or its reference solution. If you did [exercise-06](exercise-06-observability-replay-and-shipping.md), reuse its event hook for task 5.

## Prerequisites

- A working exercise-01 harness.
- Python 3.11+, `pytest`, and the VeriSwarm Python SDK: `pip install "veriswarm>=0.5"`. Use the core `VeriSwarmClient` (`from veriswarm import VeriSwarmClient`). Every task is tested offline against a fake, so you don't need an account until the live check.
- For the live check (task 7): a VeriSwarm workspace and API key. Gate decisions and event ingestion are on the free plan; Guard (PII and injection scanning) and Vault (audit ledger) need the Max plan. Check the current plans before you start.

## Tasks

### 1. A trust client interface

- Define a `TrustClient` protocol with the five calls you'll use, matching the SDK's names and keyword arguments: `check_decision(*, agent_id, action_type, resource_type)`, `tokenize_pii(*, text, agent_id, session_id)`, `rehydrate_pii(*, text, session_id)`, `scan_injection(*, text)`, and `ingest_events_batch(events)`.
- Implement `FakeTrust` for tests: scripted decisions per tool, a deterministic tokenizer (emails become `[VS:EMAIL:…]` tokens it can rehydrate), a scanner that flags known injection phrases, and a recorder for ingested events. Add a switch that makes every call raise, to simulate an outage.
- The real client (`veriswarm.VeriSwarmClient`) already satisfies the protocol. Read the API key from the environment.

### 2. Gate as a permission policy

- Write `VeriSwarmPolicy`, a `PermissionPolicy` that wraps your existing local policy. Local `deny` always wins. For tools in a configured `governed` set (the ones with side effects: writes, commands, network), call `check_decision(agent_id=..., action_type="tool_call", resource_type=<tool name>)`.
- Map VeriSwarm's answer onto the gate: `allow` → allow, `review` → ask, `deny` → deny. Put VeriSwarm's `reason_code` in the reason the model sees.
- Cache decisions per tool for a configurable time-to-live. Decisions count against a daily plan quota, and a tool's permission rarely changes mid-run.
- **Fail mode per tool:** when VeriSwarm can't be reached, a tool in `fail_closed` is denied with a clear reason, and any other governed tool falls back to the local policy's answer. Make writes and commands fail closed by default.

### 3. A result-filter stage

- Your exercise-01 post-tool hooks can only append notes. Add a `result_filters` stage to the harness: a tuple of functions `(ToolCall, ToolResult) -> ToolResult` that run in order after the tool, before the result enters the model's context. A filter that raises fails closed: the result is replaced by an error result saying which filter failed.
- Add the matching `input_filters` stage: functions `(ToolCall) -> ToolCall` that run after the permission gate and before the tool executes.

### 4. Guard filters

- **PII filter (result):** tokenize each tool result with `tokenize_pii`, reusing one `session_id` for the whole run so the same value always gets the same token. The model sees `[VS:EMAIL:…]`, never the address.
- **Rehydrate at the boundary (input):** before `write_file` runs, rehydrate its `content` with `rehydrate_pii`, so files get real values instead of tokens. Also rehydrate the final answer before it's returned to the user. Nothing else is rehydrated.
- **Injection filter (result):** run `scan_injection` on output from tools that read untrusted content (`read_file` on files the agent didn't write, `run_command`, any MCP tool). If the response says `flagged`, replace the content with a short notice that includes the scanner's `summary` and states that the original output was withheld.
- Decide and document what each filter does when VeriSwarm is unreachable. The defaults: PII fails closed (withhold the result rather than leak it), and the injection scan fails closed for MCP tools and open for local reads.

### 5. Events for scoring and audit

- Subscribe to the harness's tool-call events (exercise-06's `on_event`, or a minimal callback you add). For each call, build a VeriSwarm event: a unique `event_id`, your `agent_id`, `source_type="agent"`, an `event_type` such as `tool.call.success`, `tool.call.error`, or `tool.call.blocked`, an ISO-8601 UTC `occurred_at`, and a `payload` with the tool name, permission decision, and duration.
- **No content in events:** no tool inputs, no outputs, no file contents. Send names, decisions, and timings only.
- Buffer events and send them with `ingest_events_batch` in batches of at most 50, plus a final flush at run end. Ingestion failures are logged and never fail the run.

### 6. Wire it together

- A factory, `build_trusted_harness(base_harness, client, agent_id, config)`, that returns a new harness with the policy, filters, and event sink installed, without changing the base harness.
- A config dataclass holding the governed tools, fail-closed tools, untrusted-output tools, and cache TTL.

### 7. Live check (optional, needs an account)

- Register an agent in your VeriSwarm workspace and set a per-tool permission that denies `run_command`. Run a task that needs a command and show the denial, with VeriSwarm's reason, reaching the model.
- With Guard enabled, read a fixture file containing an email address and show the token, not the address, in the transcript, and the real address in a file the agent writes.

## Starter guidance

```python
from typing import Any, Protocol

class TrustClient(Protocol):
    def check_decision(self, *, agent_id: str, action_type: str,
                       resource_type: str | None = None) -> dict[str, Any]: ...
    def tokenize_pii(self, *, text: str, agent_id: str | None = None,
                     session_id: str | None = None) -> dict[str, Any]: ...
    def rehydrate_pii(self, *, text: str, session_id: str) -> dict[str, Any]: ...
    def scan_injection(self, *, text: str) -> dict[str, Any]: ...
    def ingest_events_batch(self, events: list[dict[str, Any]]) -> dict[str, Any]: ...

# Response fields you'll use (see the API reference):
#   check_decision   -> {"decision": "allow" | "review" | "deny", "reason_code": str, "policy_tier": str}
#   tokenize_pii     -> {"tokenized_text": str, "session_id": str, "tokens_created": int}
#   rehydrate_pii    -> {"rehydrated_text": str, "tokens_resolved": int}
#   scan_injection   -> {"flagged": bool, "summary": str, ...}
```

## Acceptance criteria

You can demonstrate that, in `pytest` with `FakeTrust`:

- A local `deny` wins even when VeriSwarm says `allow`; `allow`, `review`, and `deny` map to allow, ask, and deny; the reason code reaches the model.
- A cached decision isn't re-fetched within the time-to-live and is re-fetched after it.
- During a simulated outage, a fail-closed tool is denied, another governed tool falls back to the local policy, and ungoverned tools never call VeriSwarm.
- Emails in a tool result reach the model as tokens, and the same email gets the same token across calls in a run.
- A `write_file` call with tokenized content writes the real values to disk, and the final answer is rehydrated.
- A flagged injection in untrusted output is withheld with the scanner's summary; trusted output isn't scanned.
- A filter that raises produces an error result naming the filter; a PII outage withholds the result.
- Events carry no tool input or output, are sent in batches of at most 50, flush at run end, and an ingestion failure doesn't fail the run.
- `build_trusted_harness` leaves the base harness unchanged.

## Reflection

In `NOTES.md`:

1. Where did you choose fail-open, where fail-closed, and why? What would an attacker gain from forcing an outage?
2. PII tokenization keeps personal data away from the model provider, but the data still goes to VeriSwarm. When is that the right trade, and what would you check in a vendor's data handling before turning it on?
3. Which of these controls could your harness enforce alone, and which need a service that sees more than one run, one agent, or one machine?

## Stretch goals

- Scan an MCP server's tool definitions with `scan_mcp_tools` before registering its tools (exercise-05), and refuse servers with critical findings.
- Turn on the client's secret tripwire (`secrets_detection`) and pass every tool result through `guard_outbound_text`, which redacts secrets locally if the API is down.
- Give each subagent from exercise-05 its own VeriSwarm agent identity, so trust decisions and scores are per role instead of shared.
- Pull the agent's current scores (`get_agent_scores`) into the run log at start and end, and see how a run with blocked calls moves the risk score.
