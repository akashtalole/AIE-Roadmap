---
title: "OpenAI Agents SDK: Handoffs and Guardrails"
date: 2026-04-07 08:00:00 +0530
categories: [AI, Agentic Frameworks]
tags: [openai, agentic-frameworks-series, python, multi-agent, guardrails]
---

OpenAI's Agents SDK (the successor to the earlier Swarm experiment) picks a narrower abstraction than LangGraph or AutoGen: agents, tools, and **handoffs** — a first-class primitive for one agent to transfer an entire conversation to another.

## Agents and Handoffs

```python
from agents import Agent, Runner, handoff

billing_agent = Agent(
    name="Billing Agent",
    instructions="Handle billing questions: refunds, invoices, payment methods.",
)

technical_agent = Agent(
    name="Technical Agent",
    instructions="Handle technical issues: bugs, outages, integration problems.",
)

triage_agent = Agent(
    name="Triage Agent",
    instructions="Determine if the user needs billing or technical help, then hand off.",
    handoffs=[billing_agent, technical_agent],
)

result = Runner.run_sync(triage_agent, "My last invoice charged me twice.")
print(result.final_output)
```

A handoff isn't a tool call that returns control to the original agent — it's a full transfer. Once `triage_agent` hands off to `billing_agent`, the billing agent owns the rest of the conversation, with the full message history carried over.

## Guardrails as a Built-In Concept

The SDK treats guardrails as first-class, run in parallel with the agent rather than bolted on afterward:

```python
from agents import Agent, GuardrailFunctionOutput, input_guardrail

@input_guardrail
async def block_pii_requests(ctx, agent, input_text: str) -> GuardrailFunctionOutput:
    contains_ssn = bool(re.search(r"\d{3}-\d{2}-\d{4}", input_text))
    return GuardrailFunctionOutput(
        output_info={"contains_ssn": contains_ssn},
        tripwire_triggered=contains_ssn,
    )

agent = Agent(name="Support Agent", instructions="...", input_guardrails=[block_pii_requests])
```

When `tripwire_triggered` is `True`, the SDK halts execution before the agent processes the input at all — the guardrail runs concurrently with the agent call itself, so it doesn't add latency in the common case where it passes.

## Structured Handoff Data

A handoff can carry structured context along with it, not just the raw conversation:

```python
class BillingContext(BaseModel):
    account_id: str
    issue_summary: str

billing_handoff = handoff(
    agent=billing_agent,
    input_type=BillingContext,
    on_handoff=lambda ctx, input_data: log_handoff(input_data),
)
```

This is the mechanism that keeps a handoff from losing information the triage agent already extracted — instead of the billing agent re-deriving the account ID from scratch, it receives it directly.

## Tracing Built In

Every `Runner.run` call is automatically traced — agent decisions, tool calls, and handoffs all show up in a structured trace viewer without any extra instrumentation code, which matters a lot once you're debugging why a handoff went to the wrong agent.

## Handoffs vs Delegation

CrewAI's delegation and AutoGen's group chat both keep the delegating agent "in the loop" conceptually — it stays part of the conversation. A handoff is a cleaner break: ownership fully transfers. Reach for the handoff model when your agents map naturally onto separate specialists a user shouldn't need to route between manually — the classic customer-support triage pattern.

---

*Part of the [Agentic Frameworks series]({{ site.baseurl }}/tags/agentic-frameworks-series/) — next: [the Claude Agent SDK]({{ site.baseurl }}/posts/claude-agent-sdk-production-agents/) for building production Claude-based agents.*
