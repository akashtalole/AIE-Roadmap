---
title: "Statistical Significance in LLM Evaluation Results"
date: 2026-06-25 08:00:00 +0530
categories: [AI, Evaluation]
tags: [evaluation, evaluation-series, statistics, python]
---

Earlier this month's A/B testing post flagged statistical rigor as a topic worth its own treatment. With non-deterministic outputs and inherently noisy judge scores, it's easy to see a difference between two prompts that's actually just sampling noise — this post covers the specific discipline that prevents that mistake.

## Why "It Scored Higher" Isn't Enough

A candidate prompt scoring 82% versus a baseline's 78% on a 50-example golden set sounds like an improvement. Whether it's a *real* improvement or noise depends on the sample size and the variance in scores — with 50 examples, that 4-point gap can easily be within the range you'd see from re-running the same prompt twice.

## A Basic Significance Test for Comparing Two Variants

```python
from scipy import stats

def compare_variants_significance(scores_a: list[float], scores_b: list[float]) -> dict:
    t_stat, p_value = stats.ttest_ind(scores_a, scores_b)
    return {
        "mean_a": mean(scores_a), "mean_b": mean(scores_b),
        "p_value": p_value,
        "significant": p_value < 0.05,
    }
```

A p-value below 0.05 is a common (not universal) threshold for "this difference is unlikely to be pure chance" — but it's not a magic bar; treat it as one input to a decision, not a binary ship/don't-ship gate on its own, especially since it says nothing about whether the difference is *practically* meaningful.

## Effect Size: Statistically Significant Isn't Always Meaningful

```python
def cohens_d(scores_a: list[float], scores_b: list[float]) -> float:
    pooled_std = ((stdev(scores_a) ** 2 + stdev(scores_b) ** 2) / 2) ** 0.5
    return (mean(scores_b) - mean(scores_a)) / pooled_std
```

A large enough sample can make a trivially small, practically irrelevant difference statistically significant. Effect size tells you whether the difference is actually worth caring about — a p-value of 0.001 with a tiny effect size is a weaker case for shipping a change than a p-value of 0.04 with a large effect size.

## Sample Size Planning Before Running an Experiment

```python
def required_sample_size(baseline_rate: float, minimum_detectable_effect: float, power: float = 0.8) -> int:
    from statsmodels.stats.power import zt_ind_solve_power
    effect_size = minimum_detectable_effect / (baseline_rate * (1 - baseline_rate)) ** 0.5
    return int(zt_ind_solve_power(effect_size=effect_size, power=power, alpha=0.05))
```

Deciding sample size *before* running an evaluation — based on the smallest improvement you'd actually care about detecting — avoids both wasted effort (running far more examples than needed) and, more dangerously, underpowered tests that fail to detect a real improvement and get discarded as "no effect."

## The Peeking Problem

Checking results continuously as an experiment runs and stopping the moment they look favorable inflates the false-positive rate substantially — this is the same trap flagged in the A/B testing post, worth restating precisely here: decide your sample size or evaluation window in advance, and only look at results once that predetermined point is reached.

## Applying This to Judge-Based Scoring Specifically

Because LLM-as-judge scores carry their own noise on top of genuine output variance, run each variant multiple times per example where feasible (temperature > 0 generation, multiple judge calls) and treat the resulting distribution, not a single score, as your unit of comparison — the same multi-run averaging discipline from April's agent testing post, now applied at the statistical-comparison level.

---

*Part of the [Evaluation series]({{ site.baseurl }}/tags/evaluation-series/) — next: [building an internal eval harness from scratch]({{ site.baseurl }}/posts/internal-eval-harness-from-scratch/), assembling this month's techniques into one reusable tool.*
