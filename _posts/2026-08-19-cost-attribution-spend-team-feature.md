---
title: "Cost Attribution: Tracking Spend by Team and Feature"
date: 2026-08-19 08:00:00 +0530
categories: [AI, Infrastructure]
tags: [infrastructure, ai-infra-series, cost-optimization, python]
---

Yesterday's budget management assumed accurate attribution already exists. This post covers actually building that — turning a lump-sum monthly provider bill into a precise breakdown by team, feature, and even individual user, which is the foundation every budget and forecasting decision depends on.

## Tagging Every Request at the Source

```python
def make_attributed_request(prompt: str, team: str, feature: str, user_id: str | None = None) -> dict:
    request_metadata = {"team": team, "feature": feature, "user_id": user_id, "request_id": str(uuid4())}
    result = gateway.generate(prompt, metadata=request_metadata)
    log_attributed_cost(request_metadata, result["usage"], result["cost"])
    return result
```

This has to happen at the gateway from two posts ago — retrofitting attribution after the fact from raw provider logs (which typically only show API-key-level usage, not feature or team) is far harder than requiring every request to carry attribution metadata from the start.

## Aggregation Views

```python
def cost_by_dimension(logs: list[dict], dimension: str, window_days: int = 30) -> dict:
    recent = [l for l in logs if l["timestamp"] > now() - timedelta(days=window_days)]
    breakdown = defaultdict(float)
    for log in recent:
        breakdown[log[dimension]] += log["cost"]
    return dict(sorted(breakdown.items(), key=lambda x: -x[1]))

by_team = cost_by_dimension(logs, "team")
by_feature = cost_by_dimension(logs, "feature")
by_model = cost_by_dimension(logs, "model")
```

Slicing the same underlying data by different dimensions — team, feature, model, even time-of-day — answers different questions: "who's spending the most" (team), "what's the most expensive thing we do" (feature), "should we be routing more traffic to a cheaper model" (model breakdown feeding back into the router from earlier this month).

## Handling Shared/Ambiguous Attribution

```python
def attribute_shared_cost(request: dict) -> dict:
    if request.get("feature") == "shared_infra":
        return distribute_proportionally(request, based_on="team_usage_share")
    return {"team": request["team"], "feature": request["feature"]}
```

Not every cost cleanly maps to one team or feature — a shared retrieval index update, a periodic evaluation run against the golden set — needs an explicit, documented allocation rule (proportional to usage, or a flat platform overhead line) rather than being silently dropped from attribution or arbitrarily assigned.

## Cost Attribution at the Per-Feature Granularity

```python
def feature_cost_efficiency(feature: str, logs: list[dict], business_metric: dict) -> dict:
    total_cost = sum(l["cost"] for l in logs if l["feature"] == feature)
    metric_value = business_metric[feature]  # e.g. tickets resolved, documents processed
    return {"cost_per_unit": total_cost / metric_value if metric_value else None}
```

Connecting cost attribution to a business outcome metric — cost per resolved ticket, cost per document processed — turns a raw spend number into something a product decision can actually use, directly feeding into November's ROI-measurement post.

## Anomaly Detection at the Team/Feature Level

Extend June's cost-anomaly detection (which flagged organization-wide spikes) down to the team and feature granularity — a single feature's cost doubling week-over-week is a much more actionable signal than "total spend went up," since it points directly at where to investigate.

```python
def detect_feature_cost_anomaly(feature: str, current_week_cost: float, historical_weekly: list[float]) -> bool:
    baseline = mean(historical_weekly)
    return current_week_cost > baseline * 1.5
```

## Making Attribution Data Actionable, Not Just Reported

A monthly cost report nobody reads doesn't change behavior. Feed attribution data into the same dashboards from June's observability series, visible to the teams actually making the spending decisions, and pair it with periodic review meetings where the data actually drives a decision (optimize a feature, adjust routing, request more budget) — data without a decision-making process attached to it is just an interesting report.

---

*Part of the [AI Infrastructure series]({{ site.baseurl }}/tags/ai-infra-series/) — next: [speculative decoding]({{ site.baseurl }}/posts/speculative-decoding-reducing-latency/), a technical latency optimization independent of the cost/organizational focus of the last two posts.*
