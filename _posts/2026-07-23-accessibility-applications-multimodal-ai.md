---
title: "Accessibility Applications of Multimodal AI"
date: 2026-07-23 08:00:00 +0530
categories: [AI, Multimodal]
tags: [multimodal, multimodal-series, accessibility, python]
---

Multimodal AI's ability to translate between modalities — image to text, speech to text, text to speech — maps directly onto some of the most impactful accessibility use cases available to an AI engineer today, and it's worth a dedicated post rather than treating it as an afterthought.

## Image Description for Screen Readers

```python
def generate_alt_text(image_bytes: bytes, context: str = "") -> str:
    response = client.messages.create(
        model="claude-sonnet-5", max_tokens=200,
        messages=[{"role": "user", "content": [
            {"type": "image", "source": {"type": "base64", "media_type": "image/png", "data": base64.b64encode(image_bytes).decode()}},
            {"type": "text", "text": f"Write concise, informative alt text for a screen reader. Context: {context}. "
                                     f"Describe what's meaningful, not every visual detail."},
        ]}],
    )
    return response.content[0].text
```

Good alt text prioritizes *meaning* over exhaustive visual description — "a bar chart showing Q3 revenue up 15% from Q2" is far more useful to a screen reader user than a literal pixel-level description of colors and shapes, a distinction worth encoding directly in the prompt.

## Real-Time Scene Description for Visually Impaired Users

Combining the visual agent loop from yesterday with a continuous camera feed enables a "describe what's in front of me" assistive application:

```python
async def continuous_scene_description(camera_stream, description_interval_s: float = 3):
    async for frame in camera_stream:
        description = await generate_scene_description(frame)
        if is_meaningfully_different(description, last_description):
            await speak_description(description)  # TTS from earlier this month
        await asyncio.sleep(description_interval_s)
```

`is_meaningfully_different` — avoiding re-narrating an unchanged scene — matters enormously for usability here; constant redundant narration is worse than no narration at all for a user relying on this as their primary information channel.

## Live Captioning and Transcription

The streaming STT pipeline from earlier this month, applied directly to real-time captioning for deaf and hard-of-hearing users, with speaker labels (from AssemblyAI's diarization) making multi-speaker conversations followable:

```python
async def live_captioning(audio_stream):
    async for transcript_segment in stream_transcribe(audio_stream):
        display_caption(transcript_segment["text"], speaker=transcript_segment.get("speaker"))
```

## Voice Interfaces for Motor-Impairment Accessibility

The full voice assistant pipeline from earlier this month directly serves users for whom typing or precise pointer control is difficult — a voice-first interface isn't a novelty feature for this use case, it's the primary accessible path to the product's functionality.

## Document Accessibility: Structure Extraction for Screen Readers

```python
def make_document_accessible(pdf_path: str) -> dict:
    structure = extract_document_structure(pdf_path)  # headings, reading order, tables (from this month's document posts)
    return {
        "reading_order": structure["reading_order"],
        "headings": structure["headings"],
        "table_descriptions": [describe_table_for_screen_reader(t) for t in structure["tables"]],
        "alt_texts": [generate_alt_text(img) for img in structure["images"]],
    }
```

Reusing the document extraction pipeline from earlier this month for accessibility purposes — extracting proper reading order and structure — turns a scanned or poorly-tagged PDF into something a screen reader can navigate meaningfully, not just read as an undifferentiated text blob.

## Evaluating for This Use Case Specifically

Accessibility features need evaluation that includes actual users of assistive technology, not just automated quality metrics — the golden-set and judge-based evaluation from June is a useful first-pass filter, but real usability for accessibility applications requires validation with people who depend on these tools daily, whose needs a generic quality judge won't fully capture.

---

*Part of the [Multimodal AI series]({{ site.baseurl }}/tags/multimodal-series/) — next: [multimodal guardrails]({{ site.baseurl }}/posts/multimodal-guardrails-unsafe-images-audio/), the safety considerations specific to non-text content.*
