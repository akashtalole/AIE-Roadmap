---
title: "Cost-Aware Agent Design: Budgets and Circuit Breakers"
date: 2026-04-20 08:00:00 +0530
categories: [AI, Agentic Frameworks]
tags: [agents, agentic-frameworks-series, cost-optimization, guardrails]
---

March's `AgentBudget` stopped a single run from spending too much. Production agent systems need cost control at every layer above that — per-user, per-feature, and organization-wide — or a handful of expensive edge-case goals can quietly dominate your bill.

## Layered Budgets

```python
class BudgetHierarchy:
    def __init__(self, org_daily_usd: float, user_daily_usd: float, run_max_usd: float):
        self.org_daily = org_daily_usd
        self.user_daily = user_daily_usd
        self.run_max = run_max_usd

    def check(self, org_spend_today: float, user_spend_today: float, run_spend: float):
        if run_spend > self.run_max:
            raise BudgetExceeded("run")
        if user_spend_today > self.user_daily:
            raise BudgetExceeded("user_daily")
        if org_spend_today > self.org_daily:
            raise BudgetExceeded("org_daily")
```

The org-level circuit breaker is the one teams forget until an incident forces the lesson: a bug that causes every agent run to loop near its per-run cap can still bankrupt a day's budget through sheer volume, even with a sane per-run limit in place.

## Circuit Breakers: Stop Before It Gets Worse

A circuit breaker trips after a threshold of failures or cost spikes, and stays open (blocking new runs) until manually reset or a cooldown passes — the same pattern used for any unreliable downstream dependency, applied to agent spend:

```python
class CostCircuitBreaker:
    def __init__(self, spike_threshold_usd: float, window_minutes: int = 5):
        self.spike_threshold = spike_threshold_usd
        self.window = timedelta(minutes=window_minutes)
        self.tripped = False

    def record_spend(self, amount: float):
        recent_spend = get_spend_in_window(self.window)
        if recent_spend + amount > self.spike_threshold:
            self.tripped = True
            alert_on_call("Cost circuit breaker tripped")
            raise CircuitOpen()
```

## Cheaper by Default: Model Routing

Not every step in an agent's loop needs the most capable model. Route cheap, high-confidence steps (intent classification, simple extraction) to a smaller model, and reserve the expensive model for the reasoning steps that actually need it:

```python
def choose_model(step_type: str) -> str:
    return {
        "intent_classification": "claude-haiku-4-5",
        "tool_argument_extraction": "claude-haiku-4-5",
        "planning": "claude-sonnet-5",
        "final_synthesis": "claude-sonnet-5",
    }.get(step_type, "claude-sonnet-5")
```

This alone commonly cuts 30-50% off agent run costs, since most steps in a typical loop are simple routing or extraction, not deep reasoning.

## Caching Repeated Sub-Steps

If multiple runs, or multiple steps within one run, ask semantically similar questions, cache the answer rather than re-calling the model — the semantic caching pattern covered in depth during August's infrastructure series applies directly to agent tool calls, not just top-level API requests.

## Making Cost Visible Where Decisions Happen

The highest-leverage cost intervention isn't a technical control at all — it's surfacing cost-per-run in the same dashboard product and engineering teams already look at daily, so cost becomes a design input at the point where a new agent feature is scoped, not a surprise discovered after launch.

---

*Part of the [Agentic Frameworks series]({{ site.baseurl }}/tags/agentic-frameworks-series/) — next: [Agent-to-Agent (A2A) protocols]({{ site.baseurl }}/posts/agent-to-agent-a2a-protocols/) for agents built by different teams to talk to each other.*
