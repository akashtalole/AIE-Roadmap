---
title: "Capstone Project 6: Build a Voice-Enabled Support Agent"
date: 2026-12-15 08:00:00 +0530
categories: [AI, Career]
tags: [career, career-series, capstone, voice, python]
---

Sixth capstone: a complete voice-driven agent combining July's multimodal/voice series with March/April's agent fundamentals — the most technically demanding capstone in terms of number of integrated components.

## Project Brief

Build an agent that accepts spoken input, reasons and takes action using tools, and responds with natural speech — a real-time pipeline demonstrating July's full voice architecture.

## Requirements

```python
capstone_6_requirements = {
    "streaming_stt": "real-time transcription, not batch (July's STT comparison post)",
    "agent_loop": "at least 2 tools the agent can call based on the spoken request",
    "streaming_tts": "sentence-chunked streaming response, not wait-for-complete-then-speak",
    "turn_taking": "basic VAD and interruption handling (July's turn-taking post)",
    "latency_measurement": "an actual measured end-to-end latency budget breakdown (July's pipeline post)",
}
```

## Suggested Scope

```python
suggested_scope = {
    "domain": "pick something with clear, bounded tool needs — a personal task manager, a simple Q&A over your own notes",
    "keep_the_tool_set_small": "2-3 well-designed tools beats many shallow ones, echoing April's tool-design principle",
}
```

Given how many components this project integrates (STT, agent loop, TTS, turn-taking), keeping the domain scope narrow is what makes the project completable within a reasonable timeframe while still doing each component justice.

## Milestones

```python
milestones = {
    "week_1": "STT + basic agent loop working with text output first (de-risk the harder voice parts)",
    "week_2": "add TTS, get the full pipeline working end to end without streaming",
    "week_3": "add streaming and turn-taking/interruption handling",
    "week_4": "measure and optimize latency, record a demo video, write up the architecture",
}
```

Building text-first before adding voice is a deliberate risk-management sequencing choice — it isolates whether a problem is in the agent logic or the voice pipeline specifically, much easier to debug than building everything simultaneously.

## The Latency Budget as a Concrete Deliverable

```python
def measure_capstone_6_latency() -> dict:
    return {
        "stt_final_transcript_ms": measure_stage("stt"),
        "agent_reasoning_ms": measure_stage("agent"),
        "first_audio_chunk_ms": measure_stage("tts_first_chunk"),
        "total_perceived_latency_ms": measure_total(),
    }
```

Reporting this breakdown, following July's latency-budget post exactly, is one of the strongest signals in this capstone's write-up — it demonstrates you understand *where* time goes in a voice pipeline, not just that you got one working.

## Evaluation Rubric

```python
def self_evaluate_capstone_6(project: dict) -> dict:
    return {
        "handles_interruption": project.get("barge_in_tested", False),
        "measured_latency_breakdown": "latency_breakdown" in project,
        "tools_actually_get_called_correctly": project.get("tool_call_accuracy", 0) > 0.8,
        "has_recorded_demo": project.get("demo_video_url") is not None,  # essential for a voice project — text can't convey it
    }
```

A recorded demo video is close to mandatory for this specific capstone — a voice interaction's quality is very hard to convey through a written description or static screenshots alone, unlike most of this roadmap's other project types.

## Stretch Goals

```python
stretch_goals = {
    "multi_speaker_handling": "basic diarization if multiple people might interact with it",
    "graceful_stt_error_recovery": "handling a misheard request gracefully, per July's guidance",
    "deploy_on_actual_hardware": "a Raspberry Pi or similar, closer to November 28's real-world case study",
}
```

## Why This Is the Hardest Capstone

Combining streaming across three distinct subsystems (STT, reasoning, TTS) with real-time constraints is genuinely more complex than any single-modality capstone — completing this one well is a strong signal of comfort integrating multiple real-time systems together, a skill few candidates can demonstrate concretely.

---

*Part of the [Career series]({{ site.baseurl }}/tags/career-series/) — next: [Capstone 7: a document AI pipeline]({{ site.baseurl }}/posts/capstone-7-document-ai-pipeline/).*
