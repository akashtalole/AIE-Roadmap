---
title: "Cost Monitoring and Budget Alerts for LLM Applications"
date: 2026-06-19 08:00:00 +0530
categories: [AI, Evaluation]
tags: [evaluation, evaluation-series, cost-optimization, observability, python]
---

April's agent cost-control post covered budgets at run-time, stopping an individual agent from overspending. This post covers the observability side — visibility into where spend is actually going across an entire application, which is what makes those runtime budgets well-calibrated in the first place.

## Structured Cost Logging at the Request Level

```python
def log_llm_call_cost(model: str, input_tokens: int, output_tokens: int, feature: str, user_id: str):
    cost = calculate_cost(model, input_tokens, output_tokens)
    cost_log.record({
        "model": model, "input_tokens": input_tokens, "output_tokens": output_tokens,
        "cost_usd": cost, "feature": feature, "user_id": user_id, "timestamp": now(),
    })

def calculate_cost(model: str, input_tokens: int, output_tokens: int) -> float:
    rates = MODEL_PRICING[model]  # per-model input/output rates, kept up to date
    return (input_tokens / 1_000_000 * rates["input"]) + (output_tokens / 1_000_000 * rates["output"])
```

Tagging every logged call with `feature` and `user_id` is what makes cost attributable later — without it, you know total spend but not which feature or which small set of high-usage users is actually driving it.

## Aggregating Into Actionable Views

```python
def cost_breakdown_by_feature(logs: list[dict], window_days: int = 7) -> dict:
    recent = [l for l in logs if l["timestamp"] > now() - timedelta(days=window_days)]
    by_feature = defaultdict(float)
    for log in recent:
        by_feature[log["feature"]] += log["cost_usd"]
    return dict(sorted(by_feature.items(), key=lambda x: -x[1]))
```

This is the same layered-budget concept from April's agent cost post, generalized to the whole application — cost by feature, by user cohort, by model, each a different lens for finding where spend concentrates and whether it matches expectations.

## Alerting Before a Spike Becomes an Incident

```python
def check_cost_anomaly(current_hour_spend: float, historical_hourly_avg: float, std_dev: float) -> bool:
    z_score = (current_hour_spend - historical_hourly_avg) / std_dev
    if z_score > 3:
        alert_on_call(f"Cost anomaly: {current_hour_spend:.2f} vs avg {historical_hourly_avg:.2f} (z={z_score:.1f})")
        return True
    return False
```

A z-score-based anomaly check catches a genuine spend spike (a bug causing retry storms, an agent stuck looping despite guardrails, a traffic surge) faster than a simple fixed daily threshold would, since it accounts for normal variation across the day and week rather than triggering on predictable peak-hour traffic.

## Per-User Cost Caps as a Direct Guardrail

```python
def check_user_budget(user_id: str, current_spend_today: float, tier_limit: float) -> bool:
    if current_spend_today > tier_limit:
        downgrade_to_cheaper_model_for_user(user_id)  # graceful degrade, not a hard block
        return False
    return True
```

Degrading gracefully — routing to a cheaper model rather than hard-failing a request — when a per-user budget is exceeded keeps the feature usable while still bounding worst-case cost exposure, echoing the "fail closed but gracefully" principle from the guardrails posts.

## Forecasting, Not Just Reacting

Beyond real-time alerting, project cost trends forward using recent growth rate against your current pricing tier and volume discounts — the input a finance or product conversation actually needs isn't "what did we spend yesterday" but "what will we spend next quarter at current growth," which connects directly to November's cost-modeling and ROI posts.

---

*Part of the [Evaluation series]({{ site.baseurl }}/tags/evaluation-series/) — next: [latency budgets]({{ site.baseurl }}/posts/latency-budgets-p50-p95-p99/), the other production metric users feel directly.*
