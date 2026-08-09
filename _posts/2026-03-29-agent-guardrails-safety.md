---
title: "Agent Guardrails: Stopping Runaway Loops and Unsafe Actions"
date: 2026-03-29 08:00:00 +0530
categories: [AI, Agents]
tags: [agents, agents-series, guardrails, safety]
---

An agent that can call tools can also call the wrong tool, call the right tool with the wrong arguments, or call a perfectly correct tool an unbounded number of times. Guardrails are the layer between "the model decided to do X" and "X actually happens."

## Step and Cost Budgets

The cheapest guardrail is a hard ceiling on how much an agent is allowed to do before it's forced to stop and return whatever it has.

```python
class AgentBudget:
    def __init__(self, max_steps=15, max_cost_usd=0.50):
        self.max_steps = max_steps
        self.max_cost_usd = max_cost_usd
        self.steps = 0
        self.cost = 0.0

    def check(self, step_cost: float):
        self.steps += 1
        self.cost += step_cost
        if self.steps > self.max_steps:
            raise BudgetExceeded(f"Exceeded {self.max_steps} steps")
        if self.cost > self.max_cost_usd:
            raise BudgetExceeded(f"Exceeded ${self.max_cost_usd}")
```

Set both. A step limit alone doesn't catch an agent that calls an expensive tool a handful of times; a cost limit alone doesn't catch an agent stuck in a cheap, tight loop.

## Loop Detection

Step budgets stop infinite loops eventually, but they let a *lot* of wasted work happen first. Detecting the loop directly is faster:

```python
def detect_loop(action_history: list[tuple], window: int = 4) -> bool:
    if len(action_history) < window * 2:
        return False
    recent = action_history[-window:]
    previous = action_history[-window * 2:-window]
    return recent == previous
```

If the last N actions exactly match the N before them, the agent is repeating itself — stop and either replan or hand off to a human.

## Action-Level Guardrails: Confirm Before Acting

Not every tool call should be allowed to fire silently. Classify tools by risk and gate the risky ones:

| Risk tier | Example tools | Guardrail |
|---|---|---|
| Read-only | search, get_status, calculate | Auto-execute |
| Reversible write | create_draft, add_to_cart | Auto-execute, log |
| Irreversible | send_email, delete_record, charge_card | Require explicit confirmation |

```python
IRREVERSIBLE_TOOLS = {"send_email", "delete_record", "charge_card"}

def execute_tool(call, require_confirmation=True):
    if call.name in IRREVERSIBLE_TOOLS and require_confirmation:
        if not get_human_confirmation(call):
            return "Action cancelled: not confirmed."
    return tools[call.name](**call.arguments)
```

## Input and Output Filtering

Guardrails apply on both sides of the loop: filter what goes *into* tool arguments (an agent shouldn't be able to construct a SQL query with unescaped user input) and what comes *out* in the final answer (PII leakage, unsafe instructions, off-brand tone). We'll cover dedicated guardrail frameworks — NeMo Guardrails, Llama Guard, Guardrails AI — in depth during the AI Security series in September; for now, the principle is: never let raw model output reach a side effect without validation in between.

## Fail Closed, Not Open

The single most important guardrail design principle: when a check fails — a budget check errors, a confirmation call times out, a validator throws — the default behavior must be to stop the agent, not to proceed as if the check passed. Agents that fail open turn every guardrail bug into a safety bypass.

---

*Part of the [AI Agents series]({{ site.baseurl }}/tags/agents-series/) — guardrails get evaluated properly in tomorrow's post on [evaluating AI agents]({{ site.baseurl }}/posts/evaluating-ai-agents-benchmarks/).*
