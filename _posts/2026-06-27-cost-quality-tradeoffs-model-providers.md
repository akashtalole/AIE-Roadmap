---
title: "Evaluating Cost-Quality Tradeoffs Across Model Providers"
date: 2026-06-27 08:00:00 +0530
categories: [AI, Evaluation]
tags: [evaluation, evaluation-series, cost-optimization, python]
---

The internal eval harness from yesterday isn't just for catching regressions — it's the right tool for the recurring question every AI engineer eventually faces: is a cheaper model actually good enough for this task, or does quality genuinely require the expensive one?

## Running the Same Golden Set Across Providers

```python
def compare_providers(cases: list[EvalCase], providers: dict[str, callable]) -> dict:
    results = {}
    for name, generate_fn in providers.items():
        provider_results = run_eval_suite(cases, generate_fn, scorers)
        report = aggregate_report(provider_results, cases)
        results[name] = {
            "pass_rate": report["overall_pass_rate"],
            "avg_latency_ms": report["avg_latency_ms"],
            "cost_per_1k_requests": report["total_cost_usd"] / len(cases) * 1000,
        }
    return results

providers = {
    "claude-sonnet-5": lambda inp, context: generate_with_model("claude-sonnet-5", inp, context),
    "claude-haiku-4-5": lambda inp, context: generate_with_model("claude-haiku-4-5", inp, context),
    "gpt-4.1-mini": lambda inp, context: generate_with_model("gpt-4.1-mini", inp, context),
}
```

## Visualizing the Tradeoff Curve

```python
def quality_per_dollar(results: dict) -> dict:
    return {name: r["pass_rate"] / r["cost_per_1k_requests"] for name, r in results.items()}
```

Plotting pass rate against cost per 1,000 requests across providers makes the actual tradeoff visible — often a smaller/cheaper model clears your quality bar comfortably on the *specific task*, even though it scores lower on general benchmarks, because your task's evaluation criteria are narrower than a general capability benchmark tests for.

## Task-Specific Evaluation Beats General Benchmarks for This Decision

Public leaderboards rank models on broad capability. Your production task is almost always narrower than that — a classification task, a specific style of summarization, a bounded tool-use pattern — and a model that ranks lower generally can still be the right, cheaper choice for your specific narrow slice. This is precisely why the golden dataset from earlier this month, built from your own real task distribution, is the right basis for this decision, not a public benchmark score.

## Segmenting the Comparison by Difficulty

```python
def compare_by_difficulty(results_by_provider: dict, cases: list[EvalCase]) -> dict:
    by_difficulty = defaultdict(lambda: defaultdict(list))
    for provider, case_results in results_by_provider.items():
        for r in case_results:
            case = next(c for c in cases if c.id == r.case_id)
            by_difficulty[case.category][provider].append(r.passed)
    return {cat: {p: mean(scores) for p, scores in providers.items()} for cat, providers in by_difficulty.items()}
```

A common finding: the cheap model matches the expensive one on easy cases and falls off sharply on hard ones. This supports the model-routing pattern from April's cost-aware agent design post directly — route easy cases (by category or a confidence heuristic) to the cheap model, reserve the expensive one specifically for the hard segment where the quality gap is real.

## Re-Running This Periodically, Not Once

Model pricing and capability both shift regularly — a comparison run six months ago may no longer reflect current reality, especially as providers release new model tiers. Treat this comparison as a quarterly (or trigger-based, on major provider releases) exercise, not a one-time decision baked in permanently.

## The Decision Isn't Purely Quantitative

Beyond the pass-rate-per-dollar number, weigh provider reliability (uptime, rate limits), data handling terms, and switching cost — a marginally cheaper model isn't worth adopting if it requires meaningfully more prompt engineering rework or introduces provider concentration risk your team hasn't otherwise accepted.

---

*Part of the [Evaluation series]({{ site.baseurl }}/tags/evaluation-series/) — next: [continuous evaluation]({{ site.baseurl }}/posts/continuous-evaluation-every-deploy/), running this harness automatically rather than only when someone remembers to.*
