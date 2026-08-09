---
title: "AutoGen: Multi-Agent Conversations That Get Things Done"
date: 2026-04-05 08:00:00 +0530
categories: [AI, Agentic Frameworks]
tags: [autogen, agentic-frameworks-series, python, multi-agent]
---

LangGraph models coordination as a graph; CrewAI models it as roles and tasks. AutoGen models it as a **conversation** — agents are chat participants that send messages to each other, and the "workflow" emerges from that conversation rather than being declared up front.

## The Core Abstraction: ConversableAgent

```python
from autogen import ConversableAgent

assistant = ConversableAgent(
    name="assistant",
    system_message="You solve coding problems. Write Python and explain your reasoning.",
    llm_config={"model": "claude-sonnet-5"},
)

user_proxy = ConversableAgent(
    name="user_proxy",
    human_input_mode="NEVER",
    code_execution_config={"work_dir": "coding", "use_docker": True},
)

user_proxy.initiate_chat(assistant, message="Write a function that dedupes a list while preserving order.")
```

`user_proxy` here isn't a human — it's an agent configured to execute code the assistant writes and report the result back, closing the loop automatically. Setting `human_input_mode="ALWAYS"` instead turns it into a genuine human-in-the-loop checkpoint, pausing for real input at each turn.

## Why Code Execution Is a First-Class Citizen

AutoGen's `code_execution_config` reflects a specific philosophy: for a large class of tasks, the most reliable "tool" isn't a hand-written function — it's letting the model write and run arbitrary code in a sandbox, and feeding the actual output back as ground truth. This matters enough on its own that we'll dedicate a full post to sandboxing code-generating agents safely later this month.

## Termination Conditions

A conversation between two agents needs an explicit reason to stop, or it runs forever:

```python
user_proxy = ConversableAgent(
    name="user_proxy",
    human_input_mode="NEVER",
    is_termination_msg=lambda msg: "TERMINATE" in msg.get("content", ""),
    max_consecutive_auto_reply=10,
)
```

Two independent stop conditions here: a content-based signal (the assistant says "TERMINATE" when it judges the task done) and a hard cap on turns as a backstop — the same budget-guard principle from March's guardrails post, applied to conversation turns instead of tool calls.

## Nested Chats: Composing Conversations

AutoGen lets one agent's turn trigger an entirely separate nested conversation, then fold the result back into the parent chat — useful for a "consult a specialist, then continue" pattern without flattening everything into one long transcript:

```python
assistant.register_nested_chats(
    trigger=lambda sender: sender is user_proxy,
    chat_queue=[{"recipient": specialist_agent, "message": "Review this approach for correctness."}],
)
```

## Where Conversation-as-Workflow Shines (and Where It Doesn't)

The conversational model is a natural fit for tasks that genuinely resemble a back-and-forth discussion — code review, debate, iterative refinement. It's a worse fit for workflows with strict, known structure, where the extra freedom just adds unpredictability that a fixed graph or task sequence wouldn't have. Tomorrow's post on group chats and speaker selection covers how AutoGen scales this pattern past two participants.

---

*Part of the [Agentic Frameworks series]({{ site.baseurl }}/tags/agentic-frameworks-series/) — continues with [AutoGen group chats]({{ site.baseurl }}/posts/autogen-group-chats-speaker-selection/).*
