---
title: "Comparing Agentic Frameworks on a Real Benchmark Task"
date: 2026-10-30 08:00:00 +0530
categories: [AI, Agentic Frameworks]
tags: [agents, deep-dive-series, evaluation, python]
mermaid: true
---

April's framework comparison post was qualitative — a decision table based on mental models and fit. This post runs an actual empirical benchmark across LangGraph, CrewAI, and AutoGen on the same task, applying June's evaluation rigor to the framework choice itself.

## The Benchmark Task

A representative, moderately complex task: given a company name, research its recent news, competitive position, and produce a one-page briefing with citations — chosen because it exercises retrieval, multi-step reasoning, and synthesis, a reasonable proxy for many real production agent use cases.

```python
benchmark_task = {
    "input": "Produce a briefing on {company}",
    "success_criteria": ["covers recent news (within 6 months)", "includes competitive context",
                         "every claim is cited", "under 500 words"],
    "test_companies": ["a sample of 20 companies spanning different industries and sizes"],
}
```

## Implementing the Same Task in Each Framework

```python
implementations = {
    "langgraph": build_langgraph_briefing_agent(),   # April's graph-based implementation
    "crewai": build_crewai_briefing_crew(),           # April's role-based implementation
    "autogen": build_autogen_briefing_group_chat(),   # April's conversational implementation
}
```

Each implementation uses the same underlying model, the same search tool, and the same evaluation criteria — isolating the framework's orchestration approach as the variable under test, not confounding it with different tool quality or model choice.

## Running the Benchmark

```python
def run_framework_benchmark(implementations: dict, test_companies: list[str]) -> dict:
    results = {}
    for name, agent in implementations.items():
        task_results = []
        for company in test_companies:
            start = time.monotonic()
            output = agent.run(benchmark_task["input"].format(company=company))
            latency = time.monotonic() - start
            score = judge_response(output, benchmark_task["success_criteria"])
            task_results.append({"company": company, "score": score, "latency": latency, "cost": estimate_cost(agent, output)})
        results[name] = aggregate_report(task_results)
    return results
```

This is exactly June's evaluation harness, applied with "which framework" as the variable instead of "which prompt" — the same statistical rigor from June's significance-testing post applies to interpreting the results, since framework differences on a 20-example benchmark can easily be noise rather than a real effect.

## Illustrative Results Structure (Run This Yourself — Don't Trust Numbers That Age)

```python
example_results_shape = {
    "langgraph":  {"pass_rate": "...", "avg_latency_s": "...", "avg_cost_usd": "...", "citation_accuracy": "..."},
    "crewai":     {"pass_rate": "...", "avg_latency_s": "...", "avg_cost_usd": "...", "citation_accuracy": "..."},
    "autogen":    {"pass_rate": "...", "avg_latency_s": "...", "avg_cost_usd": "...", "citation_accuracy": "..."},
}
```

Any specific numbers published in a blog post go stale within months as frameworks and underlying models evolve — the actual deliverable of this post is the benchmark methodology, not a snapshot result; run it yourself against current framework versions before making a real decision.

## What a Benchmark Like This Actually Reveals

Framework differences on tasks like this tend to show up less in raw task success rate (all three can be prompted to comparable quality with enough tuning) and more in engineering-time-to-first-working-version, latency consistency (some orchestration patterns add more overhead per step than others), and how gracefully each handles the task's failure modes (a search returning no results, an ambiguous company name) — dimensions worth measuring explicitly, not just pass/fail.

## Controlling for Prompt Engineering Effort

```python
def normalize_effort(implementations: dict, effort_budget_hours: float) -> dict:
    # Give each framework's implementation an equal, bounded amount of prompt-tuning iteration
    # before running the final benchmark — otherwise the "winner" just reflects who got more tuning time
    return {name: tune_within_budget(impl, effort_budget_hours) for name, impl in implementations.items()}
```

A common flaw in framework comparisons: unequal tuning effort across implementations produces a result that reflects the comparer's familiarity with each framework more than genuine framework differences — worth explicitly bounding and reporting the tuning effort applied to each.

## Using This Methodology for Your Own Decisions

The real value of this post isn't "framework X wins" — it's a repeatable methodology for making this decision on your own actual task, with your own tools, your own model choices, and your own success criteria, rather than trusting a generic comparison (including this one) that was never run against your specific use case.

---

*Part of the [Agentic Framework Deep Dives series]({{ site.baseurl }}/tags/deep-dive-series/) — closing tomorrow with [a complete custom agent framework built from scratch]({{ site.baseurl }}/posts/complete-custom-agent-framework-scratch/), synthesizing this month's deep dives into first principles.*
