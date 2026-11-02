# Chapter 9 — Adding a Trust Layer: VeriSwarm

Everything in this module so far runs inside your harness: one process, one run, one machine. Some controls need a wider view. A permission rule that security owns and changes without redeploying your agent. A trust score built from how an agent has behaved across thousands of runs. A kill switch that stops every copy of an agent at once. Personal data kept out of the model provider's logs. An audit trail an outside auditor can verify. These belong in a service the harness calls, a **trust layer**. This chapter shows how to integrate one, using VeriSwarm as the worked example.

> **Disclosure.** This curriculum is maintained by VeriSwarm.ai, the company that makes VeriSwarm. The integration pattern in this chapter works with any trust or policy service, and each section names open-source tools that cover the same ground. Choose on your requirements.

## What a harness can't do alone

| Need | Why it doesn't fit inside the harness |
|---|---|
| Central policy | Rules live with the agent's code; changing them means a redeploy. Security teams want to change them in one place, for every agent. |
| Behavioral trust | One run sees one run. Deciding that an agent which keeps hitting blocks deserves less autonomy needs history across runs. |
| Fleet-wide stop | A local flag stops one process. A kill switch has to reach every running copy. |
| PII protection | Tool output goes straight into the prompt, and from there to the model provider. Keeping personal data out takes detection and reversible tokenization. |
| Verifiable audit | A local log can be edited by whoever runs the box. An auditor wants a tamper-evident record held elsewhere. |

None of this replaces what you've built. The sandbox (Chapter 4) is still the boundary, and local permissions and hooks (Chapter 2) still apply. A trust layer adds decisions and evidence that come from outside the process.

## VeriSwarm's pieces, mapped to the harness

VeriSwarm has four parts. Each maps onto an extension point this module already built:

| VeriSwarm | What it does | Harness extension point | Plan |
|---|---|---|---|
| **Gate** | Trust scores (identity, risk, reliability, autonomy) and policy decisions: `allow`, `review`, or `deny` for an agent taking an action | The permission gate (Chapter 2) | Free (decisions are quota-limited) |
| **Gate events** | Ingests behavioral events that feed the scores | The event bus (Chapter 8) | Free |
| **Guard** | PII tokenization and rehydration, prompt-injection scanning, MCP tool-definition scanning, per-tool policies, kill switch | Tool-result filters, the MCP client (Chapter 7) | Max |
| **Passport** | Verified agent identity, signed manifests, delegated permissions | Agent and subagent identity (Chapter 7) | Pro |
| **Vault** | Hash-chained audit ledger with integrity checks and compliance exports | Audit, alongside your run logs (Chapter 8) | Max |

Plans and limits change; check the current ones before you design around them. Integration goes through the Python SDK's core client (`pip install veriswarm`, then `from veriswarm import VeriSwarmClient`) or the REST API directly.

## Gate as a permission source

Chapter 2's permission gate answers allow, deny, or ask. VeriSwarm's decision API answers `allow`, `review`, or `deny`. The mapping is direct:

| VeriSwarm | Harness gate |
|---|---|
| `allow` | allow |
| `review` | ask (route to your approver, or deny when headless) |
| `deny` | deny, with VeriSwarm's `reason_code` in the message the model sees |

Make VeriSwarm one more policy, not a replacement for yours:

```python
@dataclass(frozen=True)
class VeriSwarmPolicy:
    local: PermissionPolicy
    client: TrustClient
    agent_id: str
    governed: frozenset[str]        # tools with side effects
    fail_closed: frozenset[str]     # tools to deny when VeriSwarm is unreachable

    def decide(self, call: ToolCall) -> Verdict:
        local = self.local.decide(call)
        if local.decision == "deny" or call.name not in self.governed:
            return local                                   # local deny always wins
        try:
            d = self.client.check_decision(agent_id=self.agent_id,
                                           action_type="tool_call", resource_type=call.name)
        except Exception:
            if call.name in self.fail_closed:
                return Verdict("deny", "trust service unreachable; refusing a side-effecting tool")
            return local
        return Verdict({"allow": "allow", "review": "ask", "deny": "deny"}[d["decision"]],
                       f"VeriSwarm: {d.get('reason_code', '')}")
```

Design points:

- **Govern only side-effecting tools.** Reads and searches don't need a remote decision on every call. Decisions count against a daily quota, and every call adds latency to the agent's turn.
- **Cache decisions.** A tool's permission rarely changes mid-run. Cache per tool for a short time-to-live, such as a few minutes. A kill switch takes effect within one TTL, which is a reasonable trade for most harnesses.
- **Per-tool policy lives in VeriSwarm.** Guard supports per-tool permissions (`allow`, `deny`, `review`, or score-gated: allowed only above a trust score or policy tier). Security can tighten `run_command` for a whole fleet without touching your code.
- **Unknown agents get safe defaults.** An agent VeriSwarm hasn't seen gets `review` for normal actions and `deny` for high-risk ones, so a misconfigured agent ID fails safe.

### Fail open or fail closed?

Every remote control needs an answer to "what if the service is down?". There's no single right one:

| Fail closed (deny) | Fail open (allow) |
|---|---|
| A wrongly allowed action is expensive or irreversible: commands, writes, payments, outbound messages | A wrongly blocked action is the bigger risk, and something else limits the damage: reads inside a sandbox |

Decide per tool, write it down, and test it with a simulated outage. One thing to know: the SDK's framework adapters fail open on API errors. That's reasonable for many apps, but in a harness the choice should be yours, which is one reason this chapter integrates through the core client and your own policy.

Also remember what an attacker gains from an outage. If blocking the trust service's network turns every check into "allow", an attacker who controls the network controls your agent. Fail closed for anything that matters.

## Events for trust scores and audit

VeriSwarm's scores come from behavior, so the harness has to report it. Chapter 8's event bus makes this a subscriber:

```python
{
  "event_id": "evt_" + uuid4().hex,
  "agent_id": "agt_…",
  "source_type": "agent",
  "event_type": "tool.call.blocked",           # or tool.call.success / tool.call.error
  "occurred_at": datetime.now(UTC).isoformat(),
  "payload": {"tool": "run_command", "decision": "deny", "duration_ms": 12}
}
```

- **Send names, decisions, and timings, not content.** The scores need what happened, not the file the agent read. Keep tool inputs and outputs out of events.
- **Batch.** `ingest_events_batch` takes up to 50 events. Buffer and flush, with a final flush at run end. Event ingestion is free and doesn't count against the decision quota.
- **Never fail the run on telemetry.** Log ingestion errors and carry on.

With Vault enabled, the record goes into a hash-chained ledger. `verify_vault_chain()` checks integrity, and exports serve audits. That's the tamper-evident record your local JSONL log can't be.

## Guard: keeping data out of the model's context

Tool results go straight into the prompt, and the prompt goes to the model provider. If the agent reads a customer record, the customer's email is now in the provider's request logs. Guard's PII tokenization replaces personal data with tokens before it enters the context:

```text
before:  Contact jane.doe@example.com about invoice 4471.
after:   Contact [VS:EMAIL:a1b2c3d4e5f6g7h8] about invoice 4471.
```

The model reasons over the token: "email the customer" still works. The real value comes back only where it's needed, by **rehydrating** at the boundary where the agent writes to a real system:

- **Tokenize every tool result**, with one session for the whole run, so the same value always gets the same token and the model can tell two customers apart.
- **Rehydrate only at write boundaries:** the content of `write_file`, an outbound email, a database update, and the final answer shown to the user. Never rehydrate into the model's context.
- **Fail closed.** If tokenization fails, withhold the result rather than pass raw PII through.

This needs a new pipeline stage. Chapter 2's post-tool hooks can only append notes, so add **result filters** (transform a tool result before it enters context) and **input filters** (transform a call before it runs). Exercise-07 adds both.

Be clear about the trade. Tokenization keeps personal data away from the *model provider*, but the data goes to the *trust service* to be tokenized. That's a better position when the trust service is built and contracted for it, with data-processing terms, retention controls, and the right compliance scope. But it's a decision to make deliberately, not a free win.

### Prompt-injection scanning

Chapter 7 treated tool output and MCP descriptions as untrusted input. Guard's `scan_injection(text=...)` returns `flagged`, details, and a `summary`. Run it on output from tools that bring in content the agent didn't write, such as web pages, issue text, MCP results, and files from outside the repo. When it's flagged, withhold the content and tell the model why. Scanners are probabilistic: they lower the risk, and the sandbox and permission gate still have to hold when one misses.

Two related Guard features fit the harness too:

- **MCP tool-definition scanning.** `scan_mcp_tools(tools)` checks a server's `tools/list` response for risky definitions before you register them. It maps findings to the OWASP MCP and agentic-security catalogs.
- **The secret tripwire.** With `secrets_detection` enabled on the client, `guard_outbound_text(text)` tokenizes secrets when the API is reachable and falls back to local, non-recoverable redaction when it isn't.

## Identity: Passport

Chapter 7 gave subagents their own tools and limits. A trust layer can give them their own identity too. Register each agent role (the main agent, the explorer, the reviewer) as a separate VeriSwarm agent, so decisions, scores, and audit entries are per role. A reviewer that never writes files shouldn't share a trust record with the agent that does. Passport adds verified identity, signed manifests that declare what an agent is for, and delegations that record who an agent acts for.

## Costs to plan for

- **Latency.** Each remote decision adds a network round trip to a turn. Govern only side-effecting tools, cache decisions, and set short client timeouts. The adapters default to 5 seconds and the client to 15.
- **Quota.** Decisions are metered; events are not. Design so routine reads never hit the decision API.
- **A dependency.** Your agent now relies on another service. Decide fail modes up front, and make sure a trust-layer outage degrades the agent rather than breaking it.
- **Data flow.** Write down exactly what leaves the machine: event metadata, text sent for tokenization or scanning, MCP definitions. Review it the way you'd review any vendor that touches production data.

## The pattern, with or without VeriSwarm

The integration points don't depend on the vendor:

| Capability | Harness extension point | Open-source options |
|---|---|---|
| Central policy decisions | Permission policy | Open Policy Agent (OPA) |
| PII detection and anonymization | Result and input filters | Microsoft Presidio |
| Prompt-injection detection | Result filter, MCP registration | Meta Prompt Guard, Llama Guard |
| Behavioral telemetry and audit | Event bus subscriber | OpenTelemetry with your own scoring and an append-only store |

A managed trust layer bundles these with cross-agent scoring, a fleet-wide kill switch, verified identity, and compliance exports. Self-hosting gives you full control of the data and the logic. Either way, the harness side is the same: a policy, two filter stages, and an event subscriber.

## Key takeaways

- Some controls need a wider view than one run: central policy, behavioral trust, a fleet-wide stop, PII protection, and verifiable audit. A trust layer provides them as a service.
- VeriSwarm's Gate maps onto the permission gate (`allow`/`review`/`deny` → allow/ask/deny), events onto the event bus, Guard onto new result and input filter stages, and Passport onto agent identity.
- Keep local policy authoritative for denials, govern only side-effecting tools, cache decisions, and decide fail-open vs. fail-closed per tool. Fail closed for anything that matters.
- Tokenize tool results so PII never enters the model's context, rehydrate only at write boundaries, and remember the data still goes to the trust service.
- Send events with names, decisions, and timings, never content; batch them, and never fail a run on telemetry.
- The same integration points work with open-source building blocks. The harness design doesn't depend on the vendor.
