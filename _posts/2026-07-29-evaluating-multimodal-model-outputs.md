---
title: "Evaluating Multimodal Model Outputs"
date: 2026-07-29 08:00:00 +0530
categories: [AI, Multimodal]
tags: [multimodal, multimodal-series, evaluation, python]
---

June's evaluation series was text-first throughout. This post is the explicit bridge — every principle from that series (golden datasets, LLM-as-judge, reference-based vs reference-free metrics) applies to multimodal outputs, with modality-specific implementation details.

## Golden Datasets for Multimodal Tasks

```python
@dataclass
class MultimodalGoldenExample:
    input_image: bytes | None
    input_audio: bytes | None
    input_text: str
    expected_output: str | dict
    acceptable_criteria: list[str]
    difficulty: str  # include "low_quality_input" as a difficulty tier specific to multimodal
```

The `low_quality_input` difficulty tier deserves deliberate representation — blurry photos, noisy audio, low-resolution screenshots — since real-world multimodal input quality varies far more than typical text input does, and a golden set built only from clean examples will systematically overestimate production reliability.

## Reference-Based Evaluation for Structured Extraction

For document/table/invoice extraction (this month's earlier posts), reference-based scoring is straightforward — exact or fuzzy field-level match against a known-correct extraction:

```python
def score_extraction_accuracy(extracted: dict, reference: dict) -> dict:
    field_scores = {
        field: (extracted.get(field) == reference[field]) if not is_numeric(reference[field])
               else abs(extracted.get(field, 0) - reference[field]) < TOLERANCE
        for field in reference
    }
    return {"field_accuracy": mean(field_scores.values()), "field_scores": field_scores}
```

## Reference-Free Evaluation for Generated Content

For image generation (prompt adherence) or open-ended visual description, a VLM-as-judge comparing output against the *source*, not a fixed reference, is the right tool — directly extending June's LLM-as-judge pattern to accept image input:

```python
def judge_image_generation(prompt: str, generated_image: bytes) -> dict:
    response = client.messages.create(model="claude-sonnet-5", max_tokens=512, messages=[{
        "role": "user", "content": [
            {"type": "image", "source": {"type": "base64", "media_type": "image/png", "data": base64.b64encode(generated_image).decode()}},
            {"type": "text", "text": f"Does this image match the prompt: '{prompt}'? Score adherence 0-1 and list any discrepancies."},
        ],
    }])
    return json.loads(response.content[0].text)
```

## Audio-Specific Evaluation

For TTS output, combine objective metrics (pronunciation accuracy against a phonetic reference) with the MOS-style subjective scoring from July's TTS post; for STT, word error rate against a golden transcript is the direct reference-based metric, following the earlier speech-comparison post's methodology.

## Multimodal Faithfulness: Extending RAGAS's Concept

The faithfulness metric from March's RAG series — does the response only claim what's supported by context — extends naturally to visual context:

```python
def visual_faithfulness(response_text: str, source_image: bytes) -> float:
    claims = extract_atomic_claims(response_text)
    response = client.messages.create(model="claude-sonnet-5", max_tokens=512, messages=[{
        "role": "user", "content": [
            {"type": "image", "source": {"type": "base64", "media_type": "image/png", "data": base64.b64encode(source_image).decode()}},
            {"type": "text", "text": f"For each claim, is it visually supported by this image? Claims: {claims}"},
        ],
    }])
    results = json.loads(response.content[0].text)
    return sum(r["supported"] for r in results) / len(results)
```

## Building Multimodal Cases Into the CI Regression Gate

Extend June's CI gate to include multimodal test cases directly — a prompt or model change that regresses image understanding or extraction accuracy should block a merge exactly the way a text regression would, using the same infrastructure with multimodal-aware scorers plugged in.

## Sampling Production Multimodal Traffic

The same continuous-evaluation discipline from June applies, with attention to modality-specific drift — image quality distribution shifting (users uploading from different device types over time), or audio characteristics changing (different microphone quality, background noise patterns) are drift signals a text-only monitoring setup wouldn't surface.

---

*Part of the [Multimodal AI series]({{ site.baseurl }}/tags/multimodal-series/) — next: [multimodal agents that see, hear, and act]({{ site.baseurl }}/posts/multimodal-agents-see-hear-act/), bringing every modality together in one agent.*
