---
title: "Human-in-the-Loop Patterns for Agentic Workflows"
date: 2026-04-14 08:00:00 +0530
categories: [AI, Agentic Frameworks]
tags: [agents, agentic-frameworks-series, human-in-the-loop, guardrails]
---

Every framework covered this month has some mechanism for pausing an agent and waiting for a human. What's less obvious is *when* to use it — inserting a human checkpoint everywhere kills the productivity gain of automating the task at all, and inserting one nowhere reproduces the guardrail failures from March.

## Three Points Where Humans Belong in the Loop

- **Before an irreversible action** — sending an email, charging a card, deleting a record. Covered already as an action-tier guardrail; this is the most common and most defensible checkpoint.
- **Below a confidence threshold** — when the agent itself signals uncertainty, or a judge model scores its plan as low-confidence, route to a human rather than proceeding on a guess.
- **On divergence from expected scope** — if a plan's step count, cost, or tool selection falls well outside the normal range for a task type, that's a signal something's gone wrong even before you know what.

```python
def needs_human_review(plan: list[str], confidence: float, estimated_cost: float) -> bool:
    return (
        confidence < 0.6
        or len(plan) > 8
        or estimated_cost > 2.00
        or any(is_irreversible(step) for step in plan)
    )
```

## Async Approval Queues, Not Blocking Calls

A synchronous "wait for human input" call works for a demo and fails in production — nobody is standing by to approve every agent action in real time. Route approval requests to a queue instead:

```python
def request_approval(agent_run_id: str, action: dict) -> None:
    approval_queue.enqueue({
        "run_id": agent_run_id,
        "action": action,
        "requested_at": now(),
        "status": "pending",
    })
    pause_agent_run(agent_run_id)  # checkpoint state, as in LangGraph's interrupt

def on_approval_received(run_id: str, approved: bool):
    if approved:
        resume_agent_run(run_id)
    else:
        cancel_agent_run(run_id, reason="rejected by reviewer")
```

This is exactly the LangGraph `interrupt_before` pattern from earlier this month, generalized across frameworks — the mechanism is always the same: persist state, pause, resume on external signal.

## Designing the Review Surface

A reviewer approving "send_email(to=..., subject=..., body=...)" as raw JSON will rubber-stamp it without reading it closely. Render the pending action the way a human actually needs to see it — a formatted email preview, not a function call:

```python
def render_for_review(action: dict) -> str:
    if action["tool"] == "send_email":
        return f"To: {action['args']['to']}\nSubject: {action['args']['subject']}\n\n{action['args']['body']}"
    return json.dumps(action, indent=2)
```

## Measuring Whether Your Checkpoints Are Working

Track two numbers over time: the approval rate (if it's near 100%, the checkpoint may be unnecessary friction) and the rejection reasons (if the same category of rejection recurs, that's a guardrail or prompt fix waiting to happen, not a permanent human tax). A well-tuned human-in-the-loop system should shrink its own footprint as the agent's failure modes get fixed upstream.

---

*Part of the [Agentic Frameworks series]({{ site.baseurl }}/tags/agentic-frameworks-series/) — next: [orchestration patterns]({{ site.baseurl }}/posts/agent-orchestration-sequential-parallel-hierarchical/) compared side by side.*
