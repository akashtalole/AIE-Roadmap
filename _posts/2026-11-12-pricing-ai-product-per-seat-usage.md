---
title: "Pricing Your AI Product: Per-Seat vs Usage-Based"
date: 2026-11-12 08:00:00 +0530
categories: [AI, Cloud AI]
tags: [cloud-business-series, pricing, business]
---

Yesterday's cost model gives the cost side. This post covers the pricing decision it feeds into — AI features have a cost structure genuinely different from traditional SaaS, and the common per-seat pricing model doesn't always fit well as a result.

## Why Per-Seat Pricing Breaks Down for AI Features

Traditional SaaS has near-zero marginal cost per additional user action — a per-seat price works because usage intensity doesn't dramatically change your costs. AI features have real, variable marginal cost per request (yesterday's cost model) — a power user running an agent hundreds of times a day costs meaningfully more to serve than a light user, a dynamic flat per-seat pricing doesn't capture.

## The Pricing Models Available

```python
pricing_models = {
    "per_seat_flat": "simple, predictable for customers, risks losing money on heavy users",
    "usage_based": "aligns revenue with cost directly, but unpredictable for customers, harder to sell",
    "seat_plus_usage_allowance": "flat base + included usage + overage — common hybrid",
    "tiered_by_capability": "different price tiers unlock different model quality/features, not raw usage volume",
    "outcome_based": "priced per resolved ticket / successful task, not per request — highest alignment, hardest to implement",
}
```

## Modeling Margin Under Each Pricing Structure

```python
def model_margin_by_pricing(customer_usage_distribution: list[float], pricing_model: dict) -> dict:
    margins = []
    for usage in customer_usage_distribution:
        cost = usage * COST_PER_UNIT  # from yesterday's cost model
        revenue = calculate_revenue(usage, pricing_model)
        margins.append(revenue - cost)
    return {"avg_margin": mean(margins), "worst_case_margin": min(margins), "margin_variance": stdev(margins)}
```

Running this against your actual (or projected) customer usage distribution — not just an average customer — is essential, since per-seat pricing's risk is concentrated specifically in the tail of heavy users, which an average-case calculation would miss entirely.

## Usage Allowances as the Common Middle Ground

```python
pricing_tier = {
    "base_price": 49,
    "included_requests_per_month": 1000,
    "overage_rate_per_request": 0.05,
}

def calculate_customer_bill(usage: int, tier: dict) -> float:
    overage = max(0, usage - tier["included_requests_per_month"])
    return tier["base_price"] + overage * tier["overage_rate_per_request"]
```

This hybrid gives customers pricing predictability for typical usage (the appeal of flat per-seat pricing) while protecting margin against genuinely heavy usage (the appeal of usage-based pricing) — the most common pattern in practice for exactly this reason, balancing sales simplicity against margin protection.

## Setting the Overage Rate to Actually Protect Margin

```python
def set_overage_rate(cost_per_unit: float, target_margin_pct: float) -> float:
    return cost_per_unit / (1 - target_margin_pct)
```

The overage rate needs to be set from the actual unit cost model (yesterday's post), not an arbitrary round number — an overage rate set without reference to true cost risks the exact margin erosion per-seat pricing was meant to avoid, just pushed into the "included usage" tier instead of eliminated.

## Outcome-Based Pricing: Higher Alignment, Higher Complexity

```python
def outcome_based_pricing(resolved_tickets: int, price_per_resolution: float) -> float:
    return resolved_tickets * price_per_resolution
```

Pricing per successful outcome (a resolved support ticket, not per API call) aligns price directly with delivered value and sidesteps customer anxiety about unpredictable per-request billing — but requires a reliable, disputable-resistant definition of "successful outcome," which needs the evaluation infrastructure from June's series to operationalize fairly and defensibly.

## Communicating Pricing Changes to Existing Customers

Any shift from a pricing model that doesn't reflect true costs (a flat per-seat price losing money on heavy users) to one that does needs careful change management — grandfathering existing customers, clear advance communication, and transparent reasoning tend to preserve trust better than an abrupt repricing, connecting to November 21's change-management post later this month.

## The Decision Isn't Purely About Margin Protection

Pricing also signals product positioning — usage-based pricing can feel more "fair" to customers and better support a land-and-expand sales motion, while flat pricing simplifies budgeting conversations with enterprise buyers. The right choice weighs the cost-model reality from yesterday against these genuine go-to-market considerations, not cost protection alone.

---

*Part of the [Cloud AI Platforms & Business of AI series]({{ site.baseurl }}/tags/cloud-business-series/) — next: [build vs buy]({{ site.baseurl }}/posts/build-vs-buy-vendor-in-house/), a decision this cost/pricing framework directly informs.*
