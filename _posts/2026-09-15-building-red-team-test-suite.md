---
title: "Building a Red Team Test Suite"
date: 2026-09-15 08:00:00 +0530
categories: [AI, Security]
tags: [security, ai-security-series, red-teaming, python, ci-cd]
---

Yesterday's red-team exercise produces findings. This post turns those findings into a maintained, versioned test suite — the security equivalent of June's golden dataset, gating every deploy against previously-found attack techniques regressing back into the system.

## The Test Suite as a Living Artifact

```python
@dataclass
class RedTeamTestCase:
    id: str
    category: str
    attack_input: str
    context: dict | None
    success_criteria: dict
    severity: str
    added_from: str  # "internal_redteam" | "external_pentest" | "production_incident" | "public_disclosure"
    added_date: date
```

Every successful attack found through red-teaming, every incident, and every publicly disclosed technique relevant to your stack becomes a permanent test case — the same "never regress on a known failure" discipline from June's golden dataset, applied to security specifically.

## CI Integration

```python
def security_regression_gate(test_suite: list[RedTeamTestCase], system_under_test) -> dict:
    results = run_red_team_suite(system_under_test, [asdict(c) for c in test_suite])
    newly_vulnerable = [r for r in results if r["succeeded"]]
    return {
        "passed": len(newly_vulnerable) == 0,
        "vulnerable_cases": newly_vulnerable,
    }
```

```yaml
# GitHub Actions, alongside June's quality regression gate
- name: Security regression check
  run: python security/run_red_team_gate.py --suite security/red_team_cases.jsonl
```

Wiring this into the same CI pipeline as the quality regression gate from June means a prompt or tool change that reintroduces a previously-fixed vulnerability blocks the merge automatically — the security equivalent of catching a quality regression before it ships.

## Organizing Cases for Coverage Tracking

```python
def coverage_report(test_suite: list[RedTeamTestCase]) -> dict:
    by_category = Counter(c.category for c in test_suite)
    owasp_categories = {"prompt_injection", "excessive_agency", "sensitive_info_disclosure", "insecure_output_handling"}
    missing = owasp_categories - set(by_category.keys())
    return {"coverage_by_category": dict(by_category), "gaps": missing}
```

Cross-referencing test suite coverage against the OWASP Top 10 categories from this month's opening post surfaces gaps explicitly — a category with zero test cases isn't necessarily secure, it's untested, and that distinction matters for prioritizing where to invest red-teaming effort next.

## Severity-Based Gating

```python
def apply_severity_gate(results: list[dict], min_blocking_severity: str = "high") -> bool:
    severity_order = {"low": 0, "medium": 1, "high": 2, "critical": 3}
    blocking_failures = [r for r in results if r["succeeded"] and severity_order[r["severity"]] >= severity_order[min_blocking_severity]]
    return len(blocking_failures) == 0
```

Not every finding needs to block every deploy — a low-severity edge case might be tracked and scheduled for a fix without blocking unrelated work, while a critical finding (data exfiltration, unauthorized irreversible actions) should block immediately, mirroring the severity-based triage in any mature vulnerability management process.

## Sourcing New Test Cases Continuously

```python
sources_for_new_cases = {
    "red_team_exercises": "from yesterday's structured exercises",
    "bug_bounty_reports": "if you run one, every valid report becomes a test case",
    "public_disclosures": "published jailbreak/injection techniques relevant to your model provider",
    "production_incidents": "any real attack attempt caught in production monitoring",
}
```

## Sharing Test Suites Across Teams

For organizations running multiple AI applications (echoing August's multi-team infrastructure posts), a shared base test suite covering universal categories (prompt injection, jailbreaks) combined with application-specific cases avoids every team rediscovering the same known techniques independently — a natural artifact for a platform or security team to own and distribute.

## The Suite Only Has Value If It's Actually Run

The single most common failure mode for a red-team test suite is building it once, running it once, and letting it go stale as the CI gate silently stops running or the suite stops being updated — treat it with the same operational rigor as the quality evaluation gate from June, actively maintained, not a one-time deliverable.

---

*Part of the [AI Security series]({{ site.baseurl }}/tags/ai-security-series/) — next: [access control and authentication for AI applications]({{ site.baseurl }}/posts/access-control-authentication-ai-applications/), a foundational security layer this month has assumed but not yet covered directly.*
