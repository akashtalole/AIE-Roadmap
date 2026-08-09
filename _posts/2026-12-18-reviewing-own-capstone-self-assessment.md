---
title: "Reviewing Your Own Capstone: A Self-Assessment Checklist"
date: 2026-12-18 08:00:00 +0530
categories: [AI, Career]
tags: [career, career-series, capstone]
---

Eight capstone briefs are now complete. This post consolidates their individual rubrics into one comprehensive self-assessment framework — a genuine, honest review pass before calling any capstone portfolio-ready.

## The Consolidated Checklist

```python
universal_capstone_checklist = {
    "evaluation": "does it have a real golden set and reported metrics, not just 'it works' (June's series, every capstone)",
    "guardrails": "are failure modes handled explicitly, not just the happy path (March's principles)",
    "security": "has it had at least a basic security self-review appropriate to its risk level (September's series)",
    "documentation": "does the write-up explain decisions and tradeoffs, not just describe features (November's practices)",
    "honest_limitations": "is there a genuine 'known limitations' section (December 1's portfolio principle)",
}
```

## A Scoring Rubric You Can Actually Use

```python
def score_capstone(project: dict) -> dict:
    dimensions = {
        "functionality": score_functionality(project),      # does it do what it claims
        "evaluation_rigor": score_evaluation(project),        # June's discipline applied
        "safety_and_guardrails": score_guardrails(project),   # March/September's discipline
        "documentation_quality": score_docs(project),          # November's practices
        "production_readiness_awareness": score_prod_thinking(project),  # cost, monitoring, deployment awareness
    }
    return {"dimensions": dimensions, "overall": mean(dimensions.values())}
```

Scoring across these five dimensions separately — rather than one holistic "is it good" judgment — mirrors June's own multi-dimensional evaluation principle, applied reflexively to your own work; a project that's functionally excellent but has zero evaluation rigor should score honestly lower than that gut feeling might suggest.

## Getting a Second Opinion

```python
peer_review_prompts = [
    "What's the first thing that would break if this got real production traffic?",
    "Where in the write-up did you get confused about a decision I made?",
    "What's the weakest evaluation claim in this project?",
]
```

Self-assessment has a blind spot — asking a peer (or even doing this exercise as a mock interview discussion, following December 2-6's interview prep content) to poke at these specific questions surfaces gaps a self-review alone tends to miss, since you're too close to your own reasoning to notice its weak points.

## Prioritizing What to Fix Before Calling It Done

```python
def prioritize_gaps(score_results: dict) -> list[str]:
    gaps = [(dim, score) for dim, score in score_results["dimensions"].items() if score < ACCEPTABLE_THRESHOLD]
    return sorted(gaps, key=lambda x: x[1])  # fix the weakest dimensions first
```

Not every gap needs to be perfect before a capstone is portfolio-ready — but the weakest dimension, especially if it's evaluation rigor or a genuine security gap (not just polish), deserves attention before featuring the project prominently, since those are exactly what a technically sophisticated reviewer will probe first.

## When "Good Enough" Is Actually Good Enough

```python
def is_portfolio_ready(score_results: dict, target_audience: str) -> bool:
    if target_audience == "senior_role_application":
        return score_results["overall"] >= HIGH_BAR
    return score_results["overall"] >= REASONABLE_BAR  # earlier-career or a supplementary project
```

Calibrate the bar to the purpose — a flagship project for a senior-role application deserves the highest bar across every dimension; a supplementary project demonstrating range doesn't need the same exhaustive polish on every axis, and perfectionism that prevents ever finishing a project is its own real cost.

## Turning This Into a Habit Beyond the Capstones

This self-assessment habit — evaluation rigor, guardrails, honest documentation, security review — is exactly the practicing-engineer habit this entire roadmap has tried to build since March. Applying it to your own capstones now is rehearsal for applying it automatically to every real production feature you build afterward.

---

*Part of the [Career series]({{ site.baseurl }}/tags/career-series/) — next: [deploying your capstone project for free]({{ site.baseurl }}/posts/deploying-capstone-project-free/).*
