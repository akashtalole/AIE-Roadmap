---
title: "LLM-as-a-Judge: Using Models to Grade Models"
date: 2026-06-03 08:00:00 +0530
categories: [AI, Evaluation]
tags: [evaluation, evaluation-series, llm-as-judge, python]
---

Human evaluation doesn't scale to thousands of examples on every prompt change. LLM-as-a-judge — using a capable model to score another model's output against a rubric — is what makes evaluation cheap enough to run continuously, with real, well-understood limitations to design around.

## A Basic Judge Prompt

{% raw %}
```python
def judge_response(input_text: str, response: str, criteria: list[str]) -> dict:
    resp = llm.chat([{
        "role": "user",
        "content": f"""Evaluate this response against each criterion. For each, answer met/not met with a one-sentence reason.

Input: {input_text}
Response: {response}

Criteria:
{chr(10).join(f'- {c}' for c in criteria)}

Return JSON: {{"criteria_results": [{{"criterion": str, "met": bool, "reason": str}}], "overall_pass": bool}}"""
    }], temperature=0)
    return json.loads(resp.content)
```
{% endraw %}

Checking against explicit criteria, not asking for a vague 1-10 quality score, produces far more consistent and actionable judgments — a criteria checklist is something the judge model can reason about concretely, where a holistic score invites noisy, hard-to-interpret variance.

## Known Biases to Design Around

- **Position bias** — when comparing two responses, judges tend to favor whichever is presented first; randomize order and average across both orderings for comparative judgments
- **Length bias** — judges (like many human evaluators) tend to rate longer responses as higher quality independent of actual content quality; be explicit in the rubric that verbosity is not itself a virtue
- **Self-preference bias** — a judge model tends to rate outputs from the same model family more favorably; use a different model family as judge than the one being evaluated where possible

```python
def compare_responses(input_text: str, response_a: str, response_b: str) -> str:
    # Run twice with swapped order, only trust the result if consistent both ways
    result_1 = judge_pairwise(input_text, response_a, response_b)
    result_2 = judge_pairwise(input_text, response_b, response_a)
    if result_1 == flip(result_2):
        return result_1
    return "inconsistent — needs human review"
```

## Calibrating a Judge Against Human Judgment

Before trusting a judge prompt at scale, validate it: have humans score a sample of the same examples, and check agreement rate between judge and human scores.

```python
def calibrate_judge(examples: list[dict], human_scores: list[bool]) -> float:
    judge_scores = [judge_response(ex["input"], ex["response"], ex["criteria"])["overall_pass"] for ex in examples]
    agreement = sum(j == h for j, h in zip(judge_scores, human_scores)) / len(examples)
    return agreement
```

An agreement rate below roughly 80-85% against human judgment on a representative sample means the judge prompt needs refinement before it's trustworthy for automated gating — treat this calibration step as mandatory, not optional, before wiring a judge into CI.

## What LLM Judges Are Good and Bad At

Judges are reliable for structural and rubric-based checks — did it follow the format, does it avoid a specific forbidden pattern, does it cover required points. They're less reliable for deep factual verification (they can be fooled by confident-sounding but wrong claims the same way they can generate them) and genuinely subjective quality judgments where human taste is the real bar. Use judges as a scalable first pass, and route uncertain or high-stakes cases to human review.

---

*Part of the [Evaluation series]({{ site.baseurl }}/tags/evaluation-series/) — next: [reference-based vs reference-free metrics]({{ site.baseurl }}/posts/reference-based-vs-reference-free-metrics/), a distinction that shapes what a judge like this can even check.*
