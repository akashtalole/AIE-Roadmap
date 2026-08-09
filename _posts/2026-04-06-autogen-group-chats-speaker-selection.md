---
title: "AutoGen Group Chats and Custom Speaker Selection"
date: 2026-04-06 08:00:00 +0530
categories: [AI, Agentic Frameworks]
tags: [autogen, agentic-frameworks-series, python, multi-agent]
---

Two agents in a conversation is simple: whoever didn't just speak, speaks next. Add a third agent and that rule breaks down — you need something deciding *who* talks next, turn by turn. That's what AutoGen's `GroupChat` and `GroupChatManager` exist for.

## Setting Up a Group Chat

```python
from autogen import GroupChat, GroupChatManager, ConversableAgent

planner = ConversableAgent(name="planner", system_message="Break the task into steps.", llm_config=cfg)
coder = ConversableAgent(name="coder", system_message="Write code for the current step.", llm_config=cfg)
reviewer = ConversableAgent(name="reviewer", system_message="Review code for bugs before it's accepted.", llm_config=cfg)

group_chat = GroupChat(agents=[planner, coder, reviewer], messages=[], max_round=15)
manager = GroupChatManager(groupchat=group_chat, llm_config=cfg)

planner.initiate_chat(manager, message="Build a CLI tool that dedupes CSV rows by a given column.")
```

The `GroupChatManager` doesn't do the task itself — its only job is deciding, after each message, which agent speaks next.

## Speaker Selection Strategies

By default, the manager uses an LLM call to pick the next speaker based on the conversation so far, but you can override this with a rule when the ordering should be deterministic:

```python
def select_next_speaker(last_speaker, group_chat):
    order = {"planner": coder, "coder": reviewer, "reviewer": planner}
    return order.get(last_speaker.name, planner)

group_chat = GroupChat(
    agents=[planner, coder, reviewer],
    messages=[],
    speaker_selection_method=select_next_speaker,
)
```

Custom selection functions are worth writing whenever you know the real workflow shape in advance — letting an LLM guess "who should talk next" adds latency and unpredictability for a decision you could just hardcode.

## Constrained Selection: Round-Robin and Allowed-Next Lists

Two built-in alternatives to full LLM-driven selection:

- `speaker_selection_method="round_robin"` — cycles through agents in a fixed order, useful for structured review pipelines
- `allowed_or_disallowed_speaker_transitions` — a graph of which agent may follow which, letting the LLM still choose *when* multiple options are valid but preventing it from picking an agent that shouldn't speak next

```python
allowed_transitions = {
    planner: [coder],
    coder: [reviewer],
    reviewer: [planner, coder],  # reviewer can send back to coder or to planner
}
group_chat = GroupChat(agents=[planner, coder, reviewer], messages=[], allowed_or_disallowed_speaker_transitions=allowed_transitions, speaker_transitions_type="allowed")
```

## Termination in a Group Setting

With more than two agents, the "does the conversation end" question also needs a clear owner — usually the manager checking for a specific phrase from a designated agent (e.g., the reviewer saying "APPROVED"), combined with the `max_round` cap as a hard backstop against a group that never converges.

## Group Chats vs a Fixed Graph

A group chat converges toward the same territory as a LangGraph graph with several nodes — the difference is where the control logic lives. LangGraph makes you declare every transition explicitly; group chat can let the model improvise transitions within constraints you define. Pick declarative graphs when the workflow is well-understood; group chats when the right sequence genuinely depends on what agents discover mid-conversation.

---

*Part of the [Agentic Frameworks series]({{ site.baseurl }}/tags/agentic-frameworks-series/) — next: [OpenAI's Agents SDK]({{ site.baseurl }}/posts/openai-agents-sdk-handoffs-guardrails/) and its handoff model.*
