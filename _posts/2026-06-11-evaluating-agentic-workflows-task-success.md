---
title: "Evaluating Agentic Workflows: Task Success Rate and Efficiency"
date: 2026-06-11 08:00:00 +0530
categories: [AI, Evaluation]
tags: [evaluation, evaluation-series, agents, python]
---

March's agent-evaluation post established task success rate as the core metric. With this month's broader evaluation toolkit in hand — golden datasets, calibrated judges, human review workflows — this post builds that out into a complete agent evaluation practice.

## A Structured Agent Golden Set

```python
@dataclass
class AgentGoldenExample:
    goal: str
    expected_tool_sequence: list[str] | None  # optional — not every goal has one right path
    success_criteria: list[str]               # checked against final state, not exact path
    max_acceptable_steps: int
    max_acceptable_cost: float
```

Note `expected_tool_sequence` is optional and separate from `success_criteria` — grading on outcome, not path, is the same principle from March's post, made explicit in the data structure itself so it can't be accidentally graded too strictly.

## Multi-Dimensional Scoring

```python
def evaluate_agent_run(example: AgentGoldenExample, trace: dict) -> dict:
    return {
        "task_success": judge_success(example.goal, example.success_criteria, trace["final_answer"]),
        "efficiency": trace["steps"] <= example.max_acceptable_steps,
        "cost_ok": trace["cost"] <= example.max_acceptable_cost,
        "no_unsafe_actions": not any(is_unsafe(s) for s in trace["steps"]),
        "graceful_on_failure": trace["status"] != "crashed",
    }
```

Reporting these dimensions separately, rather than collapsing to one pass/fail score, is what lets you distinguish "the agent got the right answer but took twice as long as it should have" from "the agent failed outright" — very different problems needing different fixes.

## Evaluating Tool-Call Correctness Independent of Final Outcome

An agent can reach the right final answer through a lucky recovery from a wrong tool call — worth catching even when the end result looks fine, since it indicates a latent reliability problem:

```python
def evaluate_tool_call_precision(trace: dict) -> float:
    calls = [s for s in trace["steps"] if s["type"] == "tool_call"]
    correct = sum(1 for c in calls if not resulted_in_error(c) and not was_redundant(c, calls))
    return correct / len(calls) if calls else 1.0
```

## Comparative Evaluation Across Agent Versions

When testing a change to an agent's prompt, tools, or underlying model, run the full golden set through both versions and compare on every dimension, not just aggregate success rate:

```python
def compare_agent_versions(golden_set, version_a, version_b) -> dict:
    results_a = [evaluate_agent_run(ex, version_a.run(ex.goal, return_trace=True)) for ex in golden_set]
    results_b = [evaluate_agent_run(ex, version_b.run(ex.goal, return_trace=True)) for ex in golden_set]
    return {dim: (mean(r[dim] for r in results_a), mean(r[dim] for r in results_b))
            for dim in results_a[0].keys()}
```

## Human Review for Agent Traces Specifically

Apply this month's structured human review workflow to full agent traces, not just final answers — a reviewer reading the whole trace can catch reasoning quality issues (a correct answer reached through flawed logic that happened to work out) that no automated check on the final output alone would surface.

## Setting Deploy Gates for Agent Changes

Combine this into the same CI regression gate pattern from earlier this month, with agent-specific thresholds: task success rate above X%, efficiency within Y% of baseline, zero new unsafe-action flags — blocking any agent change that regresses these before it reaches production.

---

*Part of the [Evaluation series]({{ site.baseurl }}/tags/evaluation-series/) — next: [building custom evaluators with DeepEval]({{ site.baseurl }}/posts/custom-evaluators-deepeval/), a framework that packages much of this month's pattern into reusable tooling.*
