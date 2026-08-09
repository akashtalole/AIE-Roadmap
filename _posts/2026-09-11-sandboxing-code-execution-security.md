---
title: "Sandboxing Code Execution for AI Agents"
date: 2026-09-11 08:00:00 +0530
categories: [AI, Security]
tags: [security, ai-security-series, sandboxing, agents, python]
---

April's coding-agent post covered sandboxing from a reliability and correctness angle. This post revisits the same territory through a dedicated security threat-model lens — what a genuinely adversarial actor could attempt against a code-execution tool, and what defenses hold up against that, not just against accidental bad output.

## The Threat Model Is Different From "Buggy Generated Code"

April's post worried about a model generating an infinite loop or a resource leak by mistake. The security threat model assumes an adversary deliberately crafting input (through prompt injection) specifically designed to make the agent generate malicious code — container escapes, resource exhaustion attacks, or attempts to reach internal network services from inside the sandbox.

## Defense in Depth for Code Execution, Threat-Model-Driven

```python
sandbox_hardening_checklist = {
    "no_network_access": "prevents exfiltration and internal service access even from malicious code",
    "read_only_root_filesystem": "prevents persistence attempts within the container",
    "no_privileged_capabilities": "drop all Linux capabilities not explicitly required",
    "seccomp_profile": "restrict available syscalls to the minimum the task needs",
    "resource_limits": "CPU/memory/PID limits prevent resource-exhaustion attacks, not just accidental leaks",
    "ephemeral_and_disposable": "container destroyed after every execution, no state persists between sessions",
}
```

Each of these specifically defends against an adversarial actor, not just accidental misbehavior — `no_privileged_capabilities` and `seccomp_profile` in particular are meaningless against accidental bugs but critical against deliberate container-escape attempts.

## Network Isolation Deserves Special Emphasis

```python
def run_sandboxed_no_network(code: str, timeout: int = 10) -> dict:
    return docker_client.containers.run(
        "python:3.12-slim",
        command=["python3", "-c", code],
        network_mode="none",  # not just restricted — completely disabled
        mem_limit="256m",
        pids_limit=50,  # prevent fork-bomb style resource exhaustion
        remove=True,
        timeout=timeout,
    )
```

Even a heavily restricted network allowlist is a meaningfully larger attack surface than no network access at all — for the large majority of legitimate code-execution use cases (data analysis, calculations), no network access is both sufficient and the safest default; only enable specific, narrow network access when a genuine use case requires it, and treat that as an elevated-risk configuration warranting extra review.

## Detecting Attempted Escapes and Abuse

```python
def monitor_sandbox_behavior(execution_log: dict) -> list[str]:
    suspicious_signals = []
    if execution_log.get("syscalls_blocked", 0) > 0:
        suspicious_signals.append("attempted syscalls outside seccomp profile — possible escape attempt")
    if execution_log.get("network_attempts", 0) > 0 and execution_log["network_mode"] == "none":
        suspicious_signals.append("attempted network access despite network_mode=none")
    return suspicious_signals
```

Logging and alerting on *attempted* violations, not just successful ones, is valuable threat intelligence — a pattern of blocked syscalls or attempted network access in code-execution requests is a strong signal that either an injection attack is being attempted against your system, or a user is deliberately probing for sandbox weaknesses.

## The Multi-Tenancy Risk

For a code-execution feature serving multiple users or organizations, container-level isolation alone may not be sufficient defense-in-depth against a determined attacker targeting other tenants' data — consider stronger isolation (microVMs like Firecracker or gVisor, mentioned in April's post) specifically when the threat model includes cross-tenant attacks, not just a single trusted user's own potentially-injected session.

## Regular Sandbox Security Review

Container escape techniques and sandbox weaknesses evolve — treat the sandboxing configuration itself as security-critical infrastructure requiring the same patching discipline and periodic review as any other security boundary in your system, not a "set once during initial build" configuration.

## This Connects Directly to Supply Chain Concerns

The base container image used for code execution is itself part of your software supply chain — tomorrow's post on supply chain security applies directly to keeping that image patched and free of known vulnerabilities, since a vulnerable base image undermines every other hardening step above.

---

*Part of the [AI Security series]({{ site.baseurl }}/tags/ai-security-series/) — next: [supply chain security for AI]({{ site.baseurl }}/posts/supply-chain-security-models-weights-datasets/), covering models, weights, and datasets specifically.*
