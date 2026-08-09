---
title: "Documentation Practices for AI Systems"
date: 2026-11-20 08:00:00 +0530
categories: [AI, Cloud AI]
tags: [cloud-business-series, documentation]
---

Yesterday's onboarding post identified the documentation a new engineer needs most. This post covers documentation practice for AI systems more broadly — what to document, how to keep it from going stale (the single biggest documentation failure mode), and who the audience actually is for each type.

## What's Genuinely Different About Documenting an AI System

Traditional software documentation describes deterministic behavior — "this endpoint does X." AI system documentation needs to additionally capture *why* a prompt is phrased a certain way, what evaluation criteria define "working," and what known limitations exist — none of which is fully derivable from reading the code or prompt text alone, unlike a traditional function whose behavior is fully specified by its implementation.

## A Documentation Taxonomy for AI Systems

```python
documentation_types = {
    "architecture_docs": "the reference architecture (August's closing post) — audience: new engineers, cross-team collaborators",
    "eval_methodology_docs": "what the golden set covers and why (June's series) — audience: engineers, product, governance committee",
    "prompt_rationale": "why a prompt is phrased this way, what it's meant to prevent — audience: anyone modifying it later",
    "runbooks": "incident response procedures (June, September's posts) — audience: on-call engineers",
    "model_and_config_changelog": "August/October's versioning — audience: anyone debugging a behavior regression",
    "known_limitations": "explicitly documented failure modes and their status — audience: product, support, users",
}
```

## Prompt Rationale: The Documentation Type Most Often Skipped

```python
# A prompt with rationale documented alongside it, not just the prompt text
"""
system_prompt = '''
You are a support agent. Never promise a specific refund timeline —
say "within our standard processing window" instead.
'''

# RATIONALE: Added 2026-09-15 after incident #4471 — agent promised a specific
# date that legal team couldn't guarantee, leading to a customer complaint.
# Removing this constraint reopens that specific risk. See postmortem: link.
"""
```

Without this rationale captured, a future engineer (or the same engineer months later) sees an oddly specific constraint with no context, and either leaves it in place without understanding it or removes it without realizing why it was added — directly connecting to June's postmortem discipline, where every fix should be documented at the point of the fix, not just in a separate incident tracker that gets disconnected from the code over time.

## Keeping Documentation Current: The Real Challenge

```python
def detect_stale_documentation(docs: list[dict], codebase_state: dict) -> list[str]:
    stale = []
    for doc in docs:
        if doc["last_updated"] < get_last_significant_change_date(doc["covers"], codebase_state):
            stale.append(doc["path"])
    return stale
```

Documentation staleness is worse than no documentation — a new engineer following stale docs (yesterday's onboarding post) can introduce a regression with false confidence. Tying documentation updates to the same PR review checklist as evaluation updates (a config or prompt change should require a documentation review, not just an eval pass) is what keeps this from silently rotting.

## Auto-Generated Documentation Where Possible

```python
def generate_current_state_docs() -> dict:
    return {
        "active_model_versions": get_registry_active_versions(),  # August's model registry
        "active_agent_configs": get_active_configs(),              # October 27's config versioning
        "current_golden_set_coverage": generate_coverage_report(), # June's coverage report
    }
```

For anything that changes frequently and is queryable from a system of record (the model registry, the config store), generating documentation automatically rather than hand-maintaining it eliminates an entire category of staleness — reserve hand-written documentation for the "why," which can't be auto-generated, and auto-generate the "what's currently true," which can.

## Documentation for Non-Engineering Stakeholders

```python
stakeholder_facing_docs = {
    "known_limitations_page": "for support teams handling user complaints about AI feature behavior",
    "governance_documentation": "September's compliance framework docs, for auditors and the governance committee",
    "product_capability_summary": "for sales/customer success teams explaining what the AI feature can and can't do",
}
```

Engineers often under-invest in this category, but a support team without an accurate "known limitations" reference will either overpromise to frustrated users or escalate issues that are actually expected, known behavior — a genuine cost of under-documenting for the non-engineering audiences who also need to understand system behavior.

## Documentation as Part of the Definition of Done

The most effective practice is simply making documentation updates part of the same definition-of-done as evaluation passing — a feature or config change isn't complete until both the eval gate passes and the relevant documentation (rationale, known limitations, runbook if applicable) is updated, treated with equal seriousness rather than documentation being an optional afterthought.

---

*Part of the [Cloud AI Platforms & Business of AI series]({{ site.baseurl }}/tags/cloud-business-series/) — next: [change management for rolling out AI features]({{ site.baseurl }}/posts/change-management-ai-features-users/), where documentation quality directly affects rollout success.*
