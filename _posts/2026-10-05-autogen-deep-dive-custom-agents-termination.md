---
title: "AutoGen Deep Dive: Custom Agents and Termination Conditions"
date: 2026-10-05 08:00:00 +0530
categories: [AI, Agentic Frameworks]
tags: [autogen, deep-dive-series, python]
---

April's AutoGen posts used `ConversableAgent` with standard configuration. This post covers subclassing for genuinely custom agent behavior and building termination logic more sophisticated than a keyword match.

## Subclassing ConversableAgent

```python
from autogen import ConversableAgent

class BudgetAwareAgent(ConversableAgent):
    def __init__(self, *args, max_cost_usd: float = 1.0, **kwargs):
        super().__init__(*args, **kwargs)
        self.spent = 0.0
        self.max_cost = max_cost_usd
        self.register_reply(ConversableAgent, self._check_budget_before_reply)

    def _check_budget_before_reply(self, messages, sender, config):
        if self.spent >= self.max_cost:
            return True, "TERMINATE: budget exceeded"
        cost = estimate_cost(messages)
        self.spent += cost
        return False, None  # let normal reply generation proceed
```

`register_reply` hooks into AutoGen's message-handling pipeline before the default LLM-calling behavior runs — this is how you inject cross-cutting concerns like April's budget guardrails directly into an agent's behavior, rather than wrapping the agent externally.

## Sophisticated Termination Logic

```python
def build_smart_termination(max_turns: int = 15) -> callable:
    def is_termination_msg(msg: dict) -> bool:
        content = msg.get("content", "")
        if "TERMINATE" in content:
            return True
        if is_repeating_previous_response(msg, conversation_history):
            return True  # loop detection, from March's guardrails post
        if looks_like_task_genuinely_complete(content):
            return True
        return False
    return is_termination_msg
```

Combining an explicit signal phrase with loop detection and a semantic "does this look complete" check produces more robust termination than any single condition alone — directly extending March's loop-detection and evaluation-of-completeness patterns into AutoGen's conversation model specifically.

## Custom Speaker Selection with State

```python
class StatefulSpeakerSelector:
    def __init__(self):
        self.turns_since_progress = 0

    def select_next_speaker(self, last_speaker, group_chat) -> ConversableAgent:
        if not made_progress(group_chat.messages[-1]):
            self.turns_since_progress += 1
        else:
            self.turns_since_progress = 0

        if self.turns_since_progress > 3:
            return group_chat.agent_by_name("escalation_agent")  # stuck — bring in a different approach
        return default_next_speaker_logic(last_speaker, group_chat)
```

A stateful selector that tracks conversation progress (not just the last message) can detect a group chat that's stalled — repeating similar exchanges without advancing toward the goal — and route to a different agent or escalate, rather than continuing an unproductive pattern until `max_round` is hit.

## Custom Human Input Handling

```python
class ApprovalGatedAgent(ConversableAgent):
    def get_human_input(self, prompt: str) -> str:
        if requires_approval(self.last_message()):
            approval = request_async_approval(self.last_message())  # August's async approval queue pattern
            return approval.decision
        return "APPROVE"  # auto-approve low-risk actions
```

Overriding `get_human_input` connects AutoGen's human-in-the-loop mechanism to the async approval queue pattern from April's human-in-the-loop post — rather than blocking synchronously for a human response (impractical in production), route to a queue and resume when a decision arrives.

## Testing Custom Agent Behavior

```python
def test_budget_aware_agent_terminates_at_limit():
    agent = BudgetAwareAgent(name="test", max_cost_usd=0.10, llm_config=mock_cheap_config)
    result = agent.initiate_chat(user_proxy, message="Long expensive task")
    assert agent.spent <= agent.max_cost + TOLERANCE
    assert "TERMINATE" in result.chat_history[-1]["content"]
```

Testing custom agent subclasses with the same mocked-LLM discipline from April's agent testing post — the budget and termination logic is deterministic control flow layered around the LLM call, and it should be tested as such, independent of actual model output variability.

## When Custom Subclassing Is Worth It

Reach for subclassing `ConversableAgent` when you need behavior that genuinely can't be expressed through configuration alone — budget enforcement, custom termination logic, specialized human-approval routing. For most standard use cases, the configuration-based approach from April's posts remains simpler and sufficient.

---

*Part of the [Agentic Framework Deep Dives series]({{ site.baseurl }}/tags/deep-dive-series/) — next: [AutoGen code execution environments]({{ site.baseurl }}/posts/autogen-code-execution-environments/) in depth.*
