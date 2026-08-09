---
title: "Building Custom Evaluators with DeepEval"
date: 2026-06-12 08:00:00 +0530
categories: [AI, Evaluation]
tags: [evaluation, evaluation-series, deepeval, python]
---

Every evaluation pattern this month — LLM-as-judge, reference-based scoring, golden set gating — has been hand-rolled so far. DeepEval packages these patterns into a testable, pytest-compatible framework, which matters for wiring evaluation into the same CI infrastructure your team already trusts for regular code.

## Basic Metric Usage

```python
from deepeval import assert_test
from deepeval.metrics import FaithfulnessMetric, AnswerRelevancyMetric
from deepeval.test_case import LLMTestCase

def test_rag_response_quality():
    test_case = LLMTestCase(
        input="What's the refund window?",
        actual_output=generate_response("What's the refund window?"),
        retrieval_context=retrieved_docs,
    )
    faithfulness = FaithfulnessMetric(threshold=0.8)
    relevancy = AnswerRelevancyMetric(threshold=0.7)
    assert_test(test_case, [faithfulness, relevancy])
```

This runs as an ordinary pytest test — `pytest tests/test_rag_quality.py` — meaning it plugs directly into whatever CI pipeline already runs your test suite, no separate eval infrastructure needed.

## Writing a Custom Metric

For quality dimensions specific to your product — the tone rubric from earlier this month, for instance — DeepEval's `GEval` lets you define a criteria-based LLM judge without hand-rolling the judge-calling and parsing logic:

```python
from deepeval.metrics import GEval
from deepeval.test_case import LLMTestCaseParams

brand_voice_metric = GEval(
    name="Brand Voice",
    criteria="The response should be concise, avoid corporate jargon, and never use "
             "generic AI-assistant phrases like 'as an AI' or 'I'd be happy to'.",
    evaluation_params=[LLMTestCaseParams.INPUT, LLMTestCaseParams.ACTUAL_OUTPUT],
    threshold=0.7,
)
```

## Building a Full Test Suite from the Golden Dataset

```python
import pytest

@pytest.mark.parametrize("example", load_golden_dataset("golden_set.jsonl"))
def test_golden_set_case(example):
    test_case = LLMTestCase(
        input=example["input"],
        actual_output=generate_response(example["input"]),
        expected_output=example.get("reference"),
        retrieval_context=example.get("context"),
    )
    assert_test(test_case, [FaithfulnessMetric(0.8), brand_voice_metric])
```

Parametrizing over the entire golden dataset turns every golden example into its own pytest test case, with individual pass/fail reporting per example — directly feeding the CI regression gate pattern from earlier this month, with far less custom plumbing.

## Dataset Management Built In

DeepEval also provides dataset versioning and a hosted platform (Confident AI) for tracking evaluation results over time — an alternative to rolling your own tracking with the Weights & Biases pattern from May, worth evaluating on its own merits if your team wants an eval-specific tool rather than a general ML experiment tracker.

## When a Framework Beats Hand-Rolled Evaluation Code

Every technique DeepEval provides could be hand-written, as most of this month's earlier posts did directly. The framework earns its adoption cost once your evaluation suite grows past a handful of ad hoc scripts — standardized metric definitions, pytest integration, and dataset versioning save real maintenance effort at that scale, at the cost of a dependency and someone else's abstractions to work within.

---

*Part of the [Evaluation series]({{ site.baseurl }}/tags/evaluation-series/) — next: [LangSmith]({{ site.baseurl }}/posts/langsmith-tracing-evaluation/), shifting from evaluation frameworks to full observability platforms.*
