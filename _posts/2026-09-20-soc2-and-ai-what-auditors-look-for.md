---
title: "SOC 2 and AI: What Auditors Actually Look For"
date: 2026-09-20 08:00:00 +0530
categories: [AI, Security]
tags: [security, ai-security-series, soc2, compliance]
---

GDPR and HIPAA are domain- and region-specific. SOC 2 is a broader trust-and-security attestation many B2B AI products need regardless of industry — this post covers what a SOC 2 audit actually examines in an AI-powered product, and how this month's practices map onto its trust service criteria.

## The Trust Service Criteria, Mapped to This Month's Content

```python
soc2_criteria_mapping = {
    "security": "access control (Sep 16), secrets management (Sep 17), sandboxing (Sep 11) — the bulk of this month",
    "availability": "disaster recovery (Aug 14), fallback strategies (Aug 16), autoscaling (Aug 11)",
    "processing_integrity": "evaluation and quality gates (June's series) — the model does what it's supposed to, reliably",
    "confidentiality": "PII redaction (Sep 6), least-privilege access (Sep 10), secrets management (Sep 17)",
    "privacy": "GDPR/HIPAA-aligned data handling (yesterday's and Sep 18's posts)",
}
```

Most of what an auditor examines under SOC 2 isn't AI-specific at all — it's the same infrastructure and process discipline any SaaS product needs. What's new is applying it rigorously to the AI-specific surfaces this month has covered, which auditors increasingly understand well enough to probe directly.

## What Auditors Specifically Ask About AI Features

```python
common_auditor_questions = [
    "How do you prevent the model from being manipulated to bypass access controls?",  # prompt injection defenses
    "What happens if the model generates output containing another customer's data?",   # output filtering, PII
    "How do you ensure a fine-tuned model doesn't retain sensitive training data inappropriately?",
    "What's your process for evaluating and approving a new model version before production?",  # the eval gate + registry
    "How do you monitor for and respond to a security incident involving the AI system specifically?",
]
```

These map directly onto specific posts this month — an auditor asking "how do you prevent unauthorized actions" is asking about the least-privilege tool design and guardrails from earlier in September; "how do you evaluate a new model" is asking about June's evaluation gate and August's model registry.

## Documented Policies, Not Just Technical Controls

```python
required_documented_policies = {
    "ai_acceptable_use_policy": "what the AI system is and isn't authorized to do",
    "model_change_management": "the process from a model registry entry (Aug 29) to production deployment",
    "incident_response_plan": "specific to AI incidents (Sep 28's post), not just generic IT incident response",
    "vendor_risk_assessment": "for every third-party model provider and MCP server (Sep 27's post)",
}
```

SOC 2 audits examine documented process as much as technical implementation — having the right technical controls from this month without documented policies describing them, who owns them, and how they're reviewed, is a common gap that surprises teams expecting a purely technical audit.

## Evidence Collection for an Audit

```python
def compile_soc2_evidence_package(period_start: date, period_end: date) -> dict:
    return {
        "access_control_logs": get_audit_logs(period_start, period_end),
        "model_deployment_history": get_registry_deployment_history(period_start, period_end),
        "security_test_results": get_red_team_suite_history(period_start, period_end),  # from Sep 15
        "incident_reports": get_incidents_in_period(period_start, period_end),
        "eval_gate_pass_history": get_ci_eval_gate_history(period_start, period_end),
    }
```

This is where the audit logging, model registry, and red-team suite infrastructure built across this month and August pays off directly — a mature practice can compile this evidence package largely automatically; an ad hoc one requires scrambling to reconstruct history manually, often incompletely, during audit preparation.

## Type I vs Type II Audits

A Type I audit assesses whether controls are designed appropriately at a point in time; a Type II audit assesses whether they operated effectively over a period (commonly 6-12 months) — Type II is what most enterprise customers actually require, and it means the practices from this month need to be genuinely operating continuously, not just documented on paper, well before an audit window even begins.

## Starting Early

SOC 2 readiness for an AI product is meaningfully easier when the practices from this whole month are built in from the start, rather than retrofitted under audit-deadline pressure — the continuous evaluation, logging, and governance discipline this roadmap has advocated throughout is, not coincidentally, close to what a mature SOC 2 posture requires.

---

*Part of the [AI Security series]({{ site.baseurl }}/tags/ai-security-series/) — next: [the EU AI Act]({{ site.baseurl }}/posts/eu-ai-act-what-engineers-need-to-know/), a newer regulatory framework specifically targeting AI systems.*
