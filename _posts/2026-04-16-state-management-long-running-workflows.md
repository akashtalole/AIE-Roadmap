---
title: "State Management in Long-Running Agent Workflows"
date: 2026-04-16 08:00:00 +0530
categories: [AI, Agentic Frameworks]
tags: [agents, agentic-frameworks-series, state-management, python]
---

A demo agent lives and dies within one process. A production agent handling a multi-hour research task, or waiting days for a human approval, needs its state to outlive the process that started it. This is the problem every framework's checkpointer is solving.

## What "State" Actually Means for an Agent

It's more than the message history — a complete agent checkpoint needs:

```python
@dataclass
class AgentCheckpoint:
    run_id: str
    messages: list[dict]
    plan: list[str]
    completed_steps: list[str]
    memory_refs: list[str]  # pointers into long-term memory, not the memory itself
    budget_used: float
    status: Literal["running", "paused", "completed", "failed"]
    updated_at: datetime
```

Note `memory_refs`, not the memory contents — checkpoint the pointer, not a copy, so a memory update after checkpointing is still visible when the run resumes.

## Persisting to a Real Store

```python
def save_checkpoint(checkpoint: AgentCheckpoint):
    db.execute(
        "INSERT INTO agent_checkpoints (run_id, state, updated_at) "
        "VALUES (%s, %s, %s) ON CONFLICT (run_id) DO UPDATE SET state = %s, updated_at = %s",
        (checkpoint.run_id, json.dumps(asdict(checkpoint)), now(), json.dumps(asdict(checkpoint)), now()),
    )

def load_checkpoint(run_id: str) -> AgentCheckpoint:
    row = db.query_one("SELECT state FROM agent_checkpoints WHERE run_id = %s", (run_id,))
    return AgentCheckpoint(**json.loads(row["state"]))
```

Postgres with a JSONB column is enough for most workloads — you don't need a specialized workflow engine until you're dealing with hundreds of concurrent long-running runs, which is the territory October's post on Temporal covers.

## Resuming Correctly

Resumption isn't just "load the state and call the loop again" — you need to detect and skip already-completed side effects, or a resumed run can double-send an email or double-charge a card:

```python
def resume_run(run_id: str):
    checkpoint = load_checkpoint(run_id)
    for step in checkpoint.plan:
        if step in checkpoint.completed_steps:
            continue  # already done before the pause — don't redo it
        execute_step(step, checkpoint)
```

This is the same idempotency concern that shows up in distributed systems generally — every side-effecting tool call an agent makes should be safe to skip on replay, which usually means checking "has this already happened" before acting, not just trusting the checkpoint's completed-steps list blindly.

## Concurrency: Multiple Runs, Shared Resources

Long-running workflows also need to handle concurrent access to shared state — two agent runs writing to the same long-term memory store, or racing to update the same database row. Standard techniques apply: optimistic locking on checkpoint updates, and scoping shared memory writes with the same care you'd give any concurrent database access.

## The Payoff

Correct state management is what turns an agent from "a script you run and watch" into a system you can trust to keep working across restarts, deploys, and days-long approval delays — the actual bar for something you'd call production-ready.

---

*Part of the [Agentic Frameworks series]({{ site.baseurl }}/tags/agentic-frameworks-series/) — next: [streaming agent output]({{ site.baseurl }}/posts/streaming-agent-output-real-time/) to a UI as it happens, rather than only after a run completes.*
