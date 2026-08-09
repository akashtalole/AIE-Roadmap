---
title: "Latency Budgets: P50, P95, and P99 for LLM Endpoints"
date: 2026-06-20 08:00:00 +0530
categories: [AI, Evaluation]
tags: [evaluation, evaluation-series, latency, observability, python]
---

Average latency hides the experience of your worst-served users. A feature with a 400ms average and a 8-second P99 feels broken to one in a hundred users every single time, no matter how good the average looks on a dashboard.

## Why Percentiles, Not Averages

```python
def latency_summary(latencies_ms: list[float]) -> dict:
    sorted_lat = sorted(latencies_ms)
    n = len(sorted_lat)
    return {
        "p50": sorted_lat[int(n * 0.50)],
        "p95": sorted_lat[int(n * 0.95)],
        "p99": sorted_lat[int(n * 0.99)],
        "mean": mean(sorted_lat),  # included for reference, not the number to optimize
    }
```

P50 (median) tells you the typical experience. P95 and P99 tell you what your unluckiest users experience — and for a system serving meaningful volume, "one in a hundred requests" happens constantly, not rarely, making P99 a real user-facing metric, not an edge-case statistic.

## Setting Budgets Per Stage, Not Just End-to-End

```python
latency_budget = {
    "retrieval": 200,       # ms
    "first_token": 800,     # time to first streamed token
    "full_generation": 4000,
    "total_request": 4500,
}

def check_budget_violations(trace: dict, budget: dict) -> list[str]:
    return [stage for stage, limit in budget.items() if trace.get(f"{stage}_ms", 0) > limit]
```

A per-stage budget, built directly on the span structure from earlier this week's trace-structuring post, tells you *which stage* blew the budget when a request is slow — essential for diagnosis, since "the request was slow" alone doesn't tell you whether retrieval, generation, or a downstream tool call was the bottleneck.

## Time to First Token Matters More Than Total Latency for Perceived Speed

For any streamed response (the pattern from April's streaming-agent-output post), users perceive responsiveness based on how quickly the first token appears, not total completion time. Optimizing for a fast first token — even if total generation time is unchanged — measurably improves perceived performance:

```python
def measure_streaming_latency(stream) -> dict:
    start = time.monotonic()
    first_token_time = None
    for chunk in stream:
        if first_token_time is None:
            first_token_time = time.monotonic() - start
    total_time = time.monotonic() - start
    return {"time_to_first_token": first_token_time, "total_time": total_time}
```

## Setting SLOs and Alerting on Budget Burn

```python
def slo_status(p99_history: list[float], slo_target: float, error_budget_pct: float = 5) -> dict:
    violations = sum(1 for p in p99_history if p > slo_target)
    burn_rate = violations / len(p99_history) * 100
    return {"within_slo": burn_rate <= error_budget_pct, "burn_rate": burn_rate}
```

Treating latency the same way infrastructure teams treat any SLO — an error budget that can be "spent," with alerting on burn rate rather than every single violation — avoids alert fatigue from noisy individual slow requests while still catching a genuine sustained degradation.

## Where Latency Regressions Usually Come From

Beyond raw model inference time, common latency regressions trace back to retrieval index growth without corresponding scaling, an added synchronous tool call in an agent loop, or a new guardrail check added without async execution — the span-level breakdown above is what turns "latency got worse last week" into a specific, fixable cause.

---

*Part of the [Evaluation series]({{ site.baseurl }}/tags/evaluation-series/) — next: bringing cost, latency, and quality together in [dashboards for LLM application health]({{ site.baseurl }}/posts/dashboards-llm-application-health/).*
