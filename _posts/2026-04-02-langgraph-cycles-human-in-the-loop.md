---
title: "LangGraph Cycles, Branching, and Human-in-the-Loop"
date: 2026-04-02 08:00:00 +0530
categories: [AI, Agentic Frameworks]
tags: [langgraph, agentic-frameworks-series, python, human-in-the-loop]
---

Yesterday's graph had one cycle: agent, tools, back to agent. Real workflows need more than that — parallel branches that reconverge, and points where the graph must pause and wait for a human before continuing.

## Interrupting a Graph for Human Approval

LangGraph's checkpointer makes this straightforward: compile with a checkpointer, mark a node to interrupt before it runs, and the graph simply pauses — its full state persisted — until you resume it.

```python
from langgraph.checkpoint.memory import MemorySaver

checkpointer = MemorySaver()
app = graph.compile(checkpointer=checkpointer, interrupt_before=["send_email"])

config = {"configurable": {"thread_id": "run-42"}}
result = app.invoke(initial_state, config=config)
# graph pauses right before the send_email node

pending_state = app.get_state(config)
print(pending_state.values["messages"][-1])  # show the human what's about to happen

# after a human approves:
app.invoke(None, config=config)  # resumes exactly where it paused
```

The `thread_id` is what makes this durable — the graph's entire state lives under that ID, so approval can come seconds or days later, from an entirely different process, and resumption picks up with full context intact.

## Parallel Branches That Reconverge

When two subtasks don't depend on each other, running them as parallel graph branches cuts latency without changing the logic:

```python
graph.add_node("research_pricing", research_pricing_node)
graph.add_node("research_features", research_features_node)
graph.add_node("compare", compare_node)

graph.add_edge("start", "research_pricing")
graph.add_edge("start", "research_features")
graph.add_edge("research_pricing", "compare")
graph.add_edge("research_features", "compare")
```

LangGraph waits for both `research_pricing` and `research_features` to complete before running `compare` — you don't write any explicit synchronization code, the graph structure encodes it.

## Conditional Branching on More Than Tool Calls

Branch logic isn't limited to "did the model call a tool" — route on any property of the state:

```python
def route_by_confidence(state: AgentState) -> str:
    if state["confidence"] < 0.6:
        return "escalate_to_human"
    if state["needs_more_research"]:
        return "research"
    return "finalize"

graph.add_conditional_edges("assess", route_by_confidence, {
    "escalate_to_human": "human_review",
    "research": "research",
    "finalize": "finalize",
})
```

## Why Durable State Matters in Production

A graph that only holds state in memory dies with the process. Swap `MemorySaver` for a Postgres or SQLite-backed checkpointer and the same interrupt/resume pattern survives deploys, crashes, and horizontal scaling — a request that paused for approval on one server instance can resume on a completely different one.

---

*Part of the [Agentic Frameworks series]({{ site.baseurl }}/tags/agentic-frameworks-series/) — next up: the same coordination problem from a different angle, with [CrewAI's role-based crews]({{ site.baseurl }}/posts/crewai-basics-roles-tasks-crews/).*
