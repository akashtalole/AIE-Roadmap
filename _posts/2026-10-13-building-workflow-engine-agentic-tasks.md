---
title: "Building a Workflow Engine for Agentic Tasks"
date: 2026-10-13 08:00:00 +0530
categories: [AI, Agentic Frameworks]
tags: [agents, deep-dive-series, python, workflow-engine]
mermaid: true
---

The last two posts covered Temporal (a full durable-execution platform) and event-driven queues (a lighter, hand-rolled decoupling pattern). This post builds a minimal workflow engine from scratch — useful directly for teams not ready for Temporal's operational overhead, and useful as a lens for understanding what either heavier option is actually providing.

## The Minimal Workflow Abstraction

```python
@dataclass
class WorkflowStep:
    name: str
    action: Callable
    depends_on: list[str] = field(default_factory=list)
    retry_policy: dict = field(default_factory=lambda: {"max_attempts": 3})

@dataclass
class WorkflowDefinition:
    name: str
    steps: list[WorkflowStep]
```

This is a deliberately small abstraction — a named, ordered (via `depends_on`) set of steps, each independently retryable, which is roughly the minimum needed to express everything from March's simple agent loop through this month's multi-stage pipelines.

## The Execution Engine

```python
class WorkflowEngine:
    def __init__(self, checkpoint_store):
        self.checkpoint_store = checkpoint_store

    async def run(self, definition: WorkflowDefinition, workflow_id: str, initial_context: dict):
        state = await self.checkpoint_store.load(workflow_id) or {"context": initial_context, "completed_steps": {}}
        for step in topological_order(definition.steps):
            if step.name in state["completed_steps"]:
                continue  # idempotent resume — skip already-completed steps
            result = await self._execute_with_retry(step, state["context"])
            state["completed_steps"][step.name] = result
            state["context"][step.name] = result
            await self.checkpoint_store.save(workflow_id, state)
        return state["context"]

    async def _execute_with_retry(self, step: WorkflowStep, context: dict):
        for attempt in range(step.retry_policy["max_attempts"]):
            try:
                return await step.action(context)
            except Exception as e:
                if attempt == step.retry_policy["max_attempts"] - 1:
                    raise
                await asyncio.sleep(2 ** attempt)
```

Checkpointing after every step, with a resume check at the top of the loop, gives the same crash-recovery guarantee as this month's earlier posts — a much smaller implementation than Temporal, at the cost of Temporal's more sophisticated guarantees (arbitrary-duration waits, signal handling, exactly-once semantics under more edge cases).

## Defining an Agentic Task as a Workflow

```python
research_workflow = WorkflowDefinition(
    name="research_and_report",
    steps=[
        WorkflowStep("plan", action=plan_research_step),
        WorkflowStep("gather_sources", action=gather_sources_step, depends_on=["plan"]),
        WorkflowStep("analyze", action=analyze_findings_step, depends_on=["gather_sources"]),
        WorkflowStep("write_report", action=write_report_step, depends_on=["analyze"]),
        WorkflowStep("human_review", action=request_human_review_step, depends_on=["write_report"]),
    ],
)

result = await engine.run(research_workflow, workflow_id="report-2026-10-13", initial_context={"topic": "vector DB pricing"})
```

## Parallel Step Execution

```python
async def run_parallel_ready_steps(self, definition, state):
    ready_steps = [s for s in definition.steps if s.name not in state["completed_steps"]
                   and all(dep in state["completed_steps"] for dep in s.depends_on)]
    results = await asyncio.gather(*[self._execute_with_retry(s, state["context"]) for s in ready_steps])
    for step, result in zip(ready_steps, results):
        state["completed_steps"][step.name] = result
```

Running every step whose dependencies are satisfied concurrently, rather than strictly sequentially, gives the same parallel-orchestration benefit from April's orchestration post directly from the dependency graph, with no separate parallel-branch configuration needed.

## When to Build vs Buy

```python
def build_vs_buy_workflow_engine(requirements: dict) -> str:
    if requirements["max_duration"] > timedelta(days=1) or requirements["needs_signals"]:
        return "Temporal — genuine long-duration and external-event needs exceed what a minimal engine handles well"
    if requirements["step_count"] < 10 and requirements["team_size"] < 5:
        return "minimal hand-rolled engine — sufficient and much simpler to operate"
    return "evaluate both against actual requirements before committing"
```

## Testing a Workflow Engine

```python
async def test_workflow_resumes_after_simulated_crash():
    engine = WorkflowEngine(checkpoint_store=test_store)
    await engine.run(research_workflow, "test-1", {"topic": "x"}, fail_after_step="gather_sources")
    result = await engine.run(research_workflow, "test-1", {"topic": "x"})  # resume
    assert result["write_report"] is not None
```

Testing crash-and-resume behavior explicitly, the same discipline as October's checkpointing post — a workflow engine's entire value proposition is durability, and that's exactly the property that needs direct test coverage, not just happy-path execution tests.

---

*Part of the [Agentic Framework Deep Dives series]({{ site.baseurl }}/tags/deep-dive-series/) — next: [Agent-to-Agent (A2A) protocol deep dive]({{ site.baseurl }}/posts/a2a-protocol-deep-dive/), extending April's introduction.*
