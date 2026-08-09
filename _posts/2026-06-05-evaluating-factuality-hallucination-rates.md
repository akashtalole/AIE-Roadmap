---
title: "Evaluating Factuality and Hallucination Rates"
date: 2026-06-05 08:00:00 +0530
categories: [AI, Evaluation]
tags: [evaluation, evaluation-series, hallucination, factuality, python]
---

Hallucination — a model stating something false with the same fluent confidence as something true — is the single failure mode users trust an AI product least once they've caught it happening. Measuring it rigorously is worth its own dedicated post beyond the general faithfulness metric from March's RAG series.

## Two Distinct Kinds of Hallucination

- **Context-unfaithful hallucination** — the response contradicts or invents beyond the provided context, even though the correct information was right there (this is what RAGAS faithfulness measures)
- **Parametric hallucination** — the response states a confident but false fact from the model's training data, with no retrieved context to check against at all — a harder problem, since there's no source of truth in the request itself to verify against

## Measuring Context-Unfaithful Hallucination

```python
def measure_faithfulness(response: str, context: str) -> dict:
    claims = extract_atomic_claims(response)
    results = [{"claim": c, "supported": is_supported_by(c, context)} for c in claims]
    unsupported = [r for r in results if not r["supported"]]
    return {"faithfulness_score": 1 - len(unsupported) / len(claims), "unsupported_claims": unsupported}
```

Breaking a response into atomic, individually-checkable claims — rather than judging faithfulness holistically — is what makes this measurement precise enough to diagnose, not just detect: you get the specific sentence that hallucinated, not just a low aggregate score.

## Measuring Parametric Hallucination

Without retrieved context to check against, this needs either a trusted external knowledge source or careful construction of test cases with known ground truth:

```python
def test_parametric_hallucination(model, fact_check_set: list[dict]) -> float:
    results = []
    for item in fact_check_set:
        response = model.generate(item["question"])
        results.append(matches_known_fact(response, item["known_correct_fact"]))
    return sum(results) / len(results)
```

Build this fact-check set around domains where you can verify ground truth confidently — specific, checkable facts, not areas of genuine ambiguity or evolving consensus where "correct" itself is contested.

## Confidence Calibration: A Related, Underrated Metric

A model that says "I'm not certain, but..." before a wrong answer is a meaningfully different failure than one stating the same wrong answer with full confidence. Measure whether expressed confidence correlates with actual correctness:

```python
def calibration_score(predictions: list[dict]) -> float:
    # predictions: [{"stated_confidence": 0.9, "was_correct": True}, ...]
    buckets = defaultdict(list)
    for p in predictions:
        buckets[round(p["stated_confidence"], 1)].append(p["was_correct"])
    calibration_error = sum(
        abs(conf - mean(correct_list)) for conf, correct_list in buckets.items()
    ) / len(buckets)
    return calibration_error  # lower is better calibrated
```

## Reducing Hallucination Rate, Not Just Measuring It

Measurement tells you where the problem is; the fixes come from techniques already covered across this roadmap — better retrieval (March's RAG series), explicit instructions to cite sources and express uncertainty (prompt engineering series), and for stubborn domain-specific hallucination patterns, targeted fine-tuning data that demonstrates appropriate hedging (May's series).

## Setting a Production Threshold

Define an acceptable hallucination rate before launch, specific to your use case's stakes — a creative writing assistant can tolerate a much higher rate than a medical information tool — and gate deploys on it the same way any other regression metric gets gated, covered in this month's CI/CD post.

---

*Part of the [Evaluation series]({{ site.baseurl }}/tags/evaluation-series/) — next: [evaluating tone and brand voice consistency]({{ site.baseurl }}/posts/evaluating-tone-brand-voice-consistency/), a softer but equally important quality dimension.*
