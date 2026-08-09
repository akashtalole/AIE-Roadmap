---
title: "Debugging Agents: Tracing Every Tool Call and Thought"
date: 2026-04-19 08:00:00 +0530
categories: [AI, Agentic Frameworks]
tags: [agents, agentic-frameworks-series, debugging, observability, python]
---

When a deterministic function fails, you get a stack trace pointing at a line of code. When an agent fails, the "bug" might be in the third of nine reasoning steps, in a tool that returned malformed data, or in a prompt that was ambiguous only for this specific input. You need the full trace, not just the final error.

## What a Useful Trace Captures

```python
@dataclass
class TraceStep:
    step_number: int
    type: Literal["thought", "tool_call", "tool_result", "final_answer"]
    content: str
    tool_name: str | None = None
    tool_args: dict | None = None
    latency_ms: float = 0
    tokens_used: int = 0
    timestamp: datetime = field(default_factory=now)
```

Capture this at every step, not just on failure — you don't know a run will fail until it does, and reconstructing a trace after the fact from logs alone is much harder than recording it as you go.

## Instrumenting the Loop

```python
def run_agent_with_trace(goal: str, tools: dict) -> tuple[str, list[TraceStep]]:
    trace = []
    messages = [{"role": "user", "content": goal}]
    for i in range(MAX_STEPS):
        response = llm.chat(messages, tools=list(tools.values()))
        trace.append(TraceStep(i, "thought", response.reasoning or "", tokens_used=response.usage.total))
        if not response.tool_calls:
            trace.append(TraceStep(i, "final_answer", response.content))
            return response.content, trace
        for call in response.tool_calls:
            start = time.monotonic()
            result = tools[call.name](**call.arguments)
            trace.append(TraceStep(i, "tool_call", "", call.name, call.arguments, (time.monotonic() - start) * 1000))
            trace.append(TraceStep(i, "tool_result", str(result)[:500]))
            messages.append({"role": "tool", "content": str(result)})
    return "max steps reached", trace
```

## Reading a Failed Trace Systematically

When an agent produces a wrong or incomplete answer, walk the trace in order and ask, at each step: was the *reasoning* wrong, or was the *information it had* wrong? These have completely different fixes — a reasoning failure needs a prompt or model change, an information failure needs a tool or retrieval fix. Conflating the two leads to fixing the wrong layer.

```python
def diagnose_failure(trace: list[TraceStep]) -> str:
    tool_results = [s for s in trace if s.type == "tool_result"]
    if any("error" in s.content.lower() or "not found" in s.content.lower() for s in tool_results):
        return "likely a tool/data issue — check tool_result steps for errors"
    return "tool results look clean — likely a reasoning issue in the thought steps"
```

## Visual Trace Viewers

For anything beyond a handful of debugging sessions, a text log stops scaling. The observability tools covered in June — Langfuse, LangSmith, Arize Phoenix — all render traces as an expandable tree, letting you jump straight to the step where things diverged instead of scrolling through a flat log.

## Correlating Traces Across a Multi-Agent Run

For multi-agent systems, tag every trace step with the agent that produced it and a shared `run_id`, so a failure in a worker agent can be traced back to the manager decision that dispatched it — losing that correlation is the fastest way to turn a multi-agent debugging session into guesswork.

---

*Part of the [Agentic Frameworks series]({{ site.baseurl }}/tags/agentic-frameworks-series/) — next: [cost-aware agent design]({{ site.baseurl }}/posts/cost-aware-agent-design-circuit-breakers/), because a trace like this is also how you find where your budget is going.*
