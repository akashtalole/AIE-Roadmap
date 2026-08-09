---
title: "Speech-to-Text: Comparing Whisper, Deepgram, and AssemblyAI"
date: 2026-07-12 08:00:00 +0530
categories: [AI, Multimodal]
tags: [multimodal, multimodal-series, speech, comparison, python]
---

April's voice-agent post deferred the STT provider comparison to this series — here it is. The right choice depends heavily on whether your use case is real-time streaming or batch transcription, more than on raw accuracy alone.

## The Three Options at a Glance

| Provider | Type | Strength | Best fit |
|---|---|---|---|
| OpenAI Whisper (API or self-hosted) | Batch (streaming via workarounds) | Strong accuracy, many languages, open-weight self-hosting option | Batch transcription, cost-sensitive self-hosted deployments |
| Deepgram | Native streaming | Very low latency, purpose-built for real-time | Real-time voice agents, live captioning |
| AssemblyAI | Both batch and streaming | Strong accuracy plus built-in features (speaker diarization, sentiment) | Feature-rich transcription pipelines, meeting/call analysis |

## Batch Transcription with Whisper

```python
def transcribe_batch(audio_path: str) -> str:
    with open(audio_path, "rb") as f:
        response = openai_client.audio.transcriptions.create(model="whisper-1", file=f)
    return response.text
```

Self-hosting Whisper (via `faster-whisper` or similar) is a legitimate option when data residency matters or volume makes the per-minute API cost add up — the same self-hosting tradeoff from May's fine-tuning economics post applies here: infrastructure ownership cost versus per-request API cost.

## Real-Time Streaming with Deepgram

```python
import websockets

async def stream_transcribe(audio_stream):
    async with websockets.connect(
        "wss://api.deepgram.com/v1/listen?model=nova-3&interim_results=true",
        extra_headers={"Authorization": f"Token {DEEPGRAM_API_KEY}"},
    ) as ws:
        async def send_audio():
            async for chunk in audio_stream:
                await ws.send(chunk)
        asyncio.create_task(send_audio())
        async for message in ws:
            result = json.loads(message)
            if result["is_final"]:
                yield result["channel"]["alternatives"][0]["transcript"]
```

`interim_results=true` streams partial transcripts as the user is still speaking, not just the final confirmed text — this is what April's voice-agent pipeline needs for low-perceived-latency turn-taking, distinct from waiting for a fully finalized transcript.

## AssemblyAI's Built-In Analysis Features

```python
transcript = assemblyai_client.transcribe(audio_url, config={
    "speaker_labels": True,        # who said what
    "sentiment_analysis": True,
    "auto_chapters": True,         # useful for long recordings
})
```

For use cases beyond raw transcription — meeting notes (covered later this month), call center analytics — AssemblyAI's built-in diarization and analysis features can replace what would otherwise be a separate post-processing LLM call, at the cost of being locked to their specific feature set and pricing.

## Benchmarking for Your Specific Audio

Word error rate (WER) varies significantly by accent, background noise, domain-specific vocabulary, and audio quality — public WER benchmarks are a weak predictor of performance on your actual audio. Build a small labeled test set from real audio representative of your use case and measure WER directly:

```python
def word_error_rate(reference: str, hypothesis: str) -> float:
    ref_words, hyp_words = reference.split(), hypothesis.split()
    distance = levenshtein_distance(ref_words, hyp_words)
    return distance / len(ref_words)
```

## Latency vs Accuracy: The Real Tradeoff

For real-time voice agents, a slightly less accurate but much lower-latency streaming provider usually beats a more accurate batch-oriented one — April's voice-pipeline post established that conversational feel depends heavily on round-trip latency, and that constraint should drive the provider choice more than a marginal WER difference would.

---

*Part of the [Multimodal AI series]({{ site.baseurl }}/tags/multimodal-series/) — next: [text-to-speech]({{ site.baseurl }}/posts/text-to-speech-natural-voice-output/), the other half of a voice pipeline.*
