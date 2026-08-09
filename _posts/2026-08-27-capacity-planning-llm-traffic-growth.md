---
title: "Capacity Planning for LLM Traffic Growth"
date: 2026-08-27 08:00:00 +0530
categories: [AI, Infrastructure]
tags: [infrastructure, ai-infra-series, capacity-planning, python]
---

Yesterday's monitoring tells you current state. Capacity planning uses that data to answer a forward-looking question — how much infrastructure will you need in three months, and when do you need to start provisioning it, given GPU lead times that autoscaling alone can't fully absorb.

## Why Reactive Autoscaling Isn't Sufficient Alone

August's autoscaling post established that GPU nodes are slow to provision. Autoscaling handles variance *within* a provisioned capacity range well, but it can't summon GPU capacity that doesn't exist in your cloud account's quota or your reserved instance pool — that requires proactive planning with real lead time.

## Traffic Growth Forecasting

```python
def forecast_traffic(historical_requests: list[dict], growth_model: str = "linear") -> dict:
    if growth_model == "linear":
        trend = fit_linear_trend(historical_requests)
    elif growth_model == "seasonal":
        trend = fit_seasonal_model(historical_requests)  # accounts for weekly/monthly patterns
    return {"forecast_3mo": trend.predict(months_ahead=3), "confidence_interval": trend.confidence_interval()}
```

Feature launches, marketing campaigns, and seasonal patterns all break a naive linear extrapolation — capacity planning needs input from product and business teams about planned launches, not just a statistical projection from historical traffic alone.

## Translating Traffic Forecast to Infrastructure Needs

```python
def forecast_to_capacity(forecast_requests_per_second: float, current_capacity_per_gpu: float, safety_margin: float = 1.3) -> int:
    required_gpus = (forecast_requests_per_second / current_capacity_per_gpu) * safety_margin
    return math.ceil(required_gpus)
```

The safety margin matters — planning for exactly the forecasted average load leaves no headroom for the peak-to-average ratio real traffic exhibits, or for the forecast simply being wrong, which it frequently is even with good modeling.

## Lead Time Planning

```python
capacity_lead_times = {
    "cloud_on_demand_gpu": "minutes to hours, subject to availability constraints during high-demand periods",
    "cloud_reserved_instances": "days to weeks to negotiate and provision at scale",
    "on_premise_hardware": "weeks to months from order to deployment",
}
```

For any growth trajectory approaching the limits of on-demand cloud GPU availability (a real constraint during periods of industry-wide GPU demand), capacity planning needs a lead time buffer matched to whichever provisioning path you're relying on — waiting until on-demand capacity is visibly constrained before starting a reserved-instance negotiation is planning too late.

## Load Testing to Validate Capacity Assumptions

```python
def load_test_capacity_assumptions(target_rps: float, current_deployment) -> dict:
    results = run_load_test(current_deployment, ramp_to_rps=target_rps, duration_minutes=30)
    return {
        "sustained_target_met": results["achieved_rps"] >= target_rps * 0.95,
        "p95_latency_at_target": results["p95_latency_ms"],
        "gpu_utilization_at_target": results["avg_gpu_utilization"],
    }
```

Don't trust capacity math alone — validate it with actual load testing against forecasted traffic levels before it arrives, the same "measure, don't assume" principle from the GPU sizing post applied specifically to future, not current, load.

## Cost Implications of Capacity Planning Decisions

```python
def capacity_plan_cost_comparison(scenarios: dict) -> dict:
    return {
        name: {"cost": scenario["gpu_count"] * scenario["hourly_rate"] * 730,
               "headroom": scenario["capacity"] - scenario["forecast_demand"]}
        for name, scenario in scenarios.items()
    }
```

Every capacity planning decision is ultimately a cost-risk tradeoff — more headroom costs more in idle capacity but reduces the risk of a capacity-driven outage during a demand spike; less headroom saves cost but raises that risk. Make this tradeoff explicit and reviewed, not an implicit default, connecting directly to the cost attribution and budget posts from earlier this month.

## Building a Capacity Planning Review Cadence

Treat capacity planning as a recurring process (monthly or quarterly, aligned to your growth rate) rather than a one-time exercise — review actual growth against forecast, adjust the model, and re-forecast forward, the same continuous-improvement discipline from June's evaluation series applied to infrastructure sizing.

---

*Part of the [AI Infrastructure series]({{ site.baseurl }}/tags/ai-infra-series/) — next: [zero-downtime model upgrades]({{ site.baseurl }}/posts/zero-downtime-model-upgrades/), executing a capacity or model change without disrupting service.*
