---
title: "Model Risk Management Frameworks"
date: 2026-09-22 08:00:00 +0530
categories: [AI, Security]
tags: [security, ai-security-series, risk-management]
---

GDPR, HIPAA, SOC 2, and the EU AI Act all point back to the same underlying need: a systematic, documented process for identifying, assessing, and managing the risk each deployed model carries. This post covers building that process directly, drawing on established model risk management practice from financial services and adapting it for the broader range of systems this roadmap has covered.

## The Core Risk Management Cycle

```python
risk_management_lifecycle = {
    "identification": "what could go wrong with this model in this specific use case?",
    "assessment": "how likely, and how severe, is each identified risk?",
    "mitigation": "what controls reduce likelihood or severity?",
    "monitoring": "how do we know if a mitigated risk is recurring or a new one is emerging?",
    "governance": "who reviews and approves this assessment, and how often?",
}
```

## A Model Risk Assessment Template

```python
@dataclass
class ModelRiskAssessment:
    model_id: str
    use_case: str
    risk_tier: Literal["low", "medium", "high"]  # informed by the EU AI Act framework from yesterday
    identified_risks: list[dict]  # {risk_type, likelihood, severity, mitigation}
    evaluation_results: dict       # from June's series
    approval_status: Literal["pending", "approved", "rejected", "conditionally_approved"]
    approved_by: str | None
    review_date: date
    next_review_due: date
```

Tying this directly to the model registry from August's infrastructure series — every model entering production should have an associated risk assessment, not as a parallel bureaucratic process, but as a required field before a model can transition from "staged" to "production" status.

## Risk Categories Specific to This Roadmap's Content

```python
risk_category_checklist = {
    "hallucination_and_factuality": "measured via June's factuality evaluation",
    "security": "measured via September's red-team suite",
    "bias_and_fairness": "measured via tomorrow's bias testing post",
    "explainability": "assessed via the day-after-tomorrow's explainability post",
    "data_privacy": "assessed against this month's GDPR/HIPAA posts",
    "operational_resilience": "assessed via August's disaster recovery and fallback planning",
}
```

Each category maps to a concrete, already-covered technical practice in this roadmap — model risk management isn't a new set of activities layered on top, it's the governance structure that makes sure all of them are actually being done, tracked, and reviewed consistently across every model, rather than inconsistently applied per team or per project.

## Tiered Review Rigor

```python
def required_review_process(risk_tier: str) -> dict:
    return {
        "low": {"reviewer": "team_lead", "cadence_months": 12},
        "medium": {"reviewer": "platform_team", "cadence_months": 6},
        "high": {"reviewer": "governance_committee", "cadence_months": 3},  # committee covered in Sep 29's post
    }[risk_tier]
```

Not every model warrants the same review intensity — an internal tool with low stakes needs lighter process than a customer-facing system making consequential decisions; calibrating review rigor to actual risk avoids both under-scrutinizing high-stakes systems and creating unsustainable process burden on low-stakes ones.

## Ongoing Monitoring, Not Just Point-in-Time Approval

```python
def check_risk_assessment_currency(assessment: ModelRiskAssessment) -> bool:
    if assessment.next_review_due < now():
        return False  # stale assessment — model risk status is not currently validated
    if get_latest_eval_results(assessment.model_id)["quality"] < assessment.evaluation_results["quality"] - 0.1:
        return False  # meaningful regression since last formal review
    return True
```

An approval granted once and never revisited doesn't reflect a model's actual current risk — connecting directly to June's drift detection and continuous evaluation posts, a risk assessment needs the same "stays current" discipline as any other quality signal in this roadmap.

## Documentation as a First-Class Deliverable

The written risk assessment itself — not just the underlying technical controls — is what auditors, regulators, and internal governance committees actually review. Treat documentation quality as seriously as the technical mitigations it describes; a strong technical posture poorly documented fails an audit just as thoroughly as a genuinely weak one.

---

*Part of the [AI Security series]({{ site.baseurl }}/tags/ai-security-series/) — next: [bias and fairness testing]({{ site.baseurl }}/posts/bias-fairness-testing-llm-outputs/), one of the risk categories this framework tracks in depth.*
