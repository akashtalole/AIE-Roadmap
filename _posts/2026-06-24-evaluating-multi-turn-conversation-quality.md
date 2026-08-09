---
title: "Evaluating Multi-Turn Conversation Quality"
date: 2026-06-24 08:00:00 +0530
categories: [AI, Evaluation]
tags: [evaluation, evaluation-series, multi-turn, python]
---

Nearly every metric this month has scored a single input-output pair. Real conversations span many turns, and a conversation can fail in ways no single-turn evaluation would catch — even when every individual response looks fine in isolation.

## Failure Modes Specific to Multi-Turn Conversations

- **Context loss** — the model forgets or contradicts something established earlier in the conversation
- **Repetition** — asking the user for information they already provided, or repeating a previous response nearly verbatim
- **Tone drift** — covered in June's tone post, but specifically the within-conversation version
- **Failure to build on progress** — treating each turn as fresh rather than advancing toward the conversation's actual goal
- **Loop behavior** — the conversation cycles without making progress, the multi-turn analog of the agent loop-detection problem from March

## Evaluating a Full Conversation, Not Just Its Last Turn

```python
def evaluate_conversation(turns: list[dict]) -> dict:
    return {
        "context_consistency": check_no_contradictions(turns),
        "no_repeated_questions": check_no_redundant_asks(turns),
        "tone_consistency": check_tone_drift(turns),
        "progress_toward_goal": judge_conversation_progress(turns),
        "resolved": judge_final_resolution(turns),
    }

def check_no_contradictions(turns: list[dict]) -> bool:
    facts_stated = extract_stated_facts(turns)
    return not any(contradicts(f1, f2) for f1, f2 in itertools.combinations(facts_stated, 2))
```

## LLM-as-Judge for Conversation-Level Quality

```python
def judge_conversation_progress(turns: list[dict]) -> dict:
    transcript = format_conversation(turns)
    resp = llm.chat([{
        "role": "user",
        "content": f"Conversation transcript:\n{transcript}\n\n"
                    f"Did the assistant make consistent progress toward resolving the user's need, "
                    f"without repeating itself or contradicting earlier statements? "
                    f"Return JSON: {{progress_score: 0-1, issues: [...]}}"
    }], temperature=0)
    return json.loads(resp.content)
```

Judging the full transcript at once, rather than each turn independently, is what lets the judge actually catch context-loss and repetition — these are properties of the *sequence*, invisible to any single-turn evaluation no matter how good.

## Simulated Multi-Turn Testing

For systematic testing beyond real conversation logs, simulate a user turn-by-turn using a second LLM playing the user role, letting you generate and evaluate many conversation trajectories automatically:

```python
def simulate_conversation(system_under_test, user_simulator, scenario: dict, max_turns: int = 6) -> list[dict]:
    turns = []
    user_message = scenario["opening_message"]
    for _ in range(max_turns):
        assistant_response = system_under_test.respond(user_message, history=turns)
        turns.append({"role": "assistant", "content": assistant_response})
        if scenario_resolved(scenario, turns):
            break
        user_message = user_simulator.respond(assistant_response, persona=scenario["user_persona"])
        turns.append({"role": "user", "content": user_message})
    return turns
```

Varying the `user_persona` (impatient, confused, providing incomplete information, changing their mind mid-conversation) generates coverage of realistic conversational dynamics a golden set of isolated single-turn examples can't represent.

## Building Multi-Turn Cases Into the Golden Set

Extend the golden dataset structure from earlier this month with full conversation trajectories, not just single input-output pairs, specifically targeting the failure modes above — this is one of the highest-value additions to an evaluation practice that starts single-turn-only, since conversational products live or die on exactly these dynamics.

---

*Part of the [Evaluation series]({{ site.baseurl }}/tags/evaluation-series/) — next: [statistical significance]({{ site.baseurl }}/posts/statistical-significance-llm-evaluation/), for trusting whether an observed quality difference is real.*
