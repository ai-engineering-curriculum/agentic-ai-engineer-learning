# Chapter 4 — Human-Approval Checkpoints for High-Risk Actions

Some actions you can't take back: wiring money, deleting a production table, sending an email to a customer, merging to `main`, posting publicly. For those, the right control isn't a smarter filter — it's a **human in the loop**. The agent does the reasoning and the preparation, then *pauses* and asks a person to approve before the irreversible step runs. This is the last line behind moderation ([Chapter 1](01-io-moderation.md)), injection defenses ([Chapter 2](02-prompt-injection.md)), and least privilege ([Chapter 3](03-tool-permissions.md)): even if all three are bypassed, a human still has to say yes.

```text
   model wants to call a tool
        │
        ▼ classify risk
   low ──▶ run automatically
   high ─▶ PAUSE ──▶ present action + args to human ──▶ approve / deny
                                                          │        │
                                                       execute   skip + tell model
```

## Classify risk, then gate

Not every call deserves an interrupt — gate everything and humans start rubber-stamping, which is no control at all. Classify by **reversibility** and **impact**:

- **Auto-run:** read-only and cheaply reversible (search, read a file, draft text that isn't sent).
- **Require approval:** irreversible or externally visible (send/pay/delete/publish/deploy), spend over a threshold, or any action on data outside the sandbox.

Make the gate a property of the *tool plus its arguments*, not a vibe. `delete_file` always gates; `transfer_funds` gates above `$0`; `send_email` gates when the recipient is outside your domain.

```python
HIGH_RISK_TOOLS = {"send_email", "transfer_funds", "delete_file", "deploy", "publish"}

def needs_approval(tool: str, args: dict) -> bool:
    if tool in HIGH_RISK_TOOLS:
        return True
    if tool == "spend" and args.get("amount", 0) > 100:   # threshold gate
        return True
    return False
```

## The checkpoint mechanics

1. **Pause and surface the full action.** Show the human the exact tool, the exact arguments, and the agent's stated reason. "Approve action?" with no details trains people to click yes. Show "send_email to `outside@vendor.com`, subject …, body …".
2. **Capture a decision, not a guess.** Record `approved`/`denied`, who decided, when, and any edited arguments. This is your audit trail — pair it with the verdict logging from [mod-205](../mod-205-evaluation-observability/README.md).
3. **Fail closed on timeout.** No response within the window → treat as **denied**, not approved. An approval that defaults to yes is a backdoor.
4. **Feed the outcome back to the agent.** On deny, return a tool result like "action denied by human" so the loop can replan instead of retrying the same call forever.

## Pitfalls

- **Approval fatigue.** Gate too much and approvals become reflexive. Tune the risk classifier so humans only see decisions that matter.
- **Under-specified prompts.** If the human can't see the arguments, they can't catch a hijacked call (e.g., injection rewrote the recipient). The checkpoint is only as good as what it shows.
- **Approving the wrong layer.** Approve the *concrete tool call*, not a high-level plan. The plan can say "email the customer" while the argument says "email attacker@evil.com."

## Key takeaways

- Human-approval checkpoints gate **irreversible, high-impact** actions — the backstop when other guardrails fail.
- Classify risk by reversibility and impact on the tool **and its arguments**; auto-run the rest.
- Pause and show the *full, concrete* action; capture an auditable approve/deny; fail closed on timeout.
- Avoid approval fatigue and always approve the concrete call, never the abstract plan.
