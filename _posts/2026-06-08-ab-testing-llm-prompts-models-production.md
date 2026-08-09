---
title: "A/B Testing LLM Prompts and Models in Production"
date: 2026-06-08 08:00:00 +0530
categories: [AI, Evaluation]
tags: [evaluation, evaluation-series, ab-testing, python]
---

A golden set catches known regressions before deploy. It can't tell you whether a genuinely new prompt variant performs better on the full diversity of real user traffic — for that, you need a live experiment.

## Basic Setup: Traffic Splitting

```python
def route_variant(user_id: str, experiment: dict) -> str:
    bucket = hash(f"{experiment['name']}:{user_id}") % 100
    cumulative = 0
    for variant, pct in experiment["allocation"].items():
        cumulative += pct
        if bucket < cumulative:
            return variant
    return experiment["default_variant"]

experiment = {"name": "concise-prompt-v2", "allocation": {"control": 90, "treatment": 10}, "default_variant": "control"}
```

Hashing on user ID, not randomizing per-request, is what keeps a given user consistently in one variant across a session — inconsistent variant assignment mid-conversation produces a confusing, disjointed experience and muddies the experiment's results.

## What to Measure

Beyond the obvious quality metrics from earlier this month, production A/B tests should track business-relevant outcomes the golden set can't capture — task completion rate, user-initiated regeneration rate (a strong implicit negative signal), session length, and escalation rate for support-style features:

```python
def log_experiment_event(user_id: str, variant: str, event_type: str, metadata: dict):
    analytics.track({
        "user_id": user_id, "experiment": "concise-prompt-v2", "variant": variant,
        "event": event_type, "metadata": metadata, "timestamp": now(),
    })
```

## Guardrail Metrics: What Must Not Get Worse

Every experiment needs explicit guardrail metrics that can independently kill it even if the primary metric improves — cost per request, latency, and hallucination rate are common guardrails that a prompt optimized purely for the primary metric might silently regress:

```python
def check_guardrails(variant_metrics: dict, guardrails: dict) -> list[str]:
    violations = []
    for metric, threshold in guardrails.items():
        if variant_metrics[metric] > threshold:
            violations.append(f"{metric}: {variant_metrics[metric]} exceeds guardrail {threshold}")
    return violations
```

## Ramp-Up, Not All-at-Once

Start any new variant at a small allocation (5-10%), confirm guardrails hold and no obvious issues surface, then ramp up gradually — the same canary-deployment discipline from April's production rollout post, applied specifically to prompt and model experiments rather than code changes.

## Statistical Rigor Isn't Optional

A common mistake is calling a winner based on a small sample or checking results repeatedly and stopping the moment a metric looks favorable ("peeking"), which inflates false-positive rates significantly. Tomorrow's statistical-significance post covers the specific discipline needed here — for now, the rule of thumb is: decide your minimum sample size and evaluation window before starting the experiment, and don't stop early based on interim results looking good.

## Model A/B Tests Are the Same Pattern, Higher Stakes

Everything here applies directly to comparing two model providers or versions, not just prompt variants — with the added guardrail of cost, since switching models can shift your per-request economics substantially even when quality metrics look comparable.

---

*Part of the [Evaluation series]({{ site.baseurl }}/tags/evaluation-series/) — next: [human evaluation workflows that scale]({{ site.baseurl }}/posts/human-evaluation-workflows-that-scale/), for the quality dimensions neither a golden set nor an A/B test alone fully captures.*
