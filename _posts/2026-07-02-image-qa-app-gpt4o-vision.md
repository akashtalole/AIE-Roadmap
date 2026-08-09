---
title: "Building an Image Q&A App with GPT-4o Vision"
date: 2026-07-02 08:00:00 +0530
categories: [AI, Multimodal]
tags: [multimodal, multimodal-series, openai, python, fastapi]
---

A worked example, end to end: a FastAPI endpoint that accepts an image and a question, and returns a grounded answer — the multimodal equivalent of the FastAPI integration patterns from the LLM engineering series.

## The Endpoint

```python
from fastapi import FastAPI, UploadFile
import base64

app = FastAPI()

@app.post("/ask-about-image")
async def ask_about_image(image: UploadFile, question: str):
    image_bytes = await image.read()
    image_b64 = base64.b64encode(image_bytes).decode()

    response = openai_client.chat.completions.create(
        model="gpt-4.1",
        messages=[{
            "role": "user",
            "content": [
                {"type": "text", "text": question},
                {"type": "image_url", "image_url": {"url": f"data:image/jpeg;base64,{image_b64}", "detail": "high"}},
            ],
        }],
    )
    return {"answer": response.choices[0].message.content}
```

The `detail: "high"` parameter controls whether OpenAI processes the image at full resolution (more tokens, better fine-detail accuracy) or a downsampled low-detail pass — set explicitly rather than relying on the default, since the right choice depends on whether the question needs fine visual detail.

## Handling Multiple Images in One Request

```python
@app.post("/compare-images")
async def compare_images(images: list[UploadFile], question: str):
    content = [{"type": "text", "text": question}]
    for img in images:
        img_bytes = await img.read()
        img_b64 = base64.b64encode(img_bytes).decode()
        content.append({"type": "image_url", "image_url": {"url": f"data:image/jpeg;base64,{img_b64}"}})

    response = openai_client.chat.completions.create(model="gpt-4.1", messages=[{"role": "content", "content": content}])
    return {"answer": response.choices[0].message.content}
```

Multi-image requests are what makes "compare these two screenshots" or "which of these three photos matches the description" possible — the model reasons over all images in the same context, in the order provided.

## Validating and Constraining Input

```python
MAX_IMAGE_SIZE_MB = 10
ALLOWED_TYPES = {"image/jpeg", "image/png", "image/webp"}

async def validate_image(image: UploadFile):
    if image.content_type not in ALLOWED_TYPES:
        raise HTTPException(400, "Unsupported image type")
    contents = await image.read()
    if len(contents) > MAX_IMAGE_SIZE_MB * 1024 * 1024:
        raise HTTPException(400, "Image too large")
    await image.seek(0)
    return contents
```

## Streaming the Response

```python
@app.post("/ask-about-image-stream")
async def ask_about_image_stream(image: UploadFile, question: str):
    image_b64 = base64.b64encode(await image.read()).decode()
    async def generate():
        stream = await openai_client.chat.completions.create(
            model="gpt-4.1", messages=[...], stream=True,
        )
        async for chunk in stream:
            if chunk.choices[0].delta.content:
                yield chunk.choices[0].delta.content
    return StreamingResponse(generate(), media_type="text/plain")
```

This reuses the streaming patterns from the LLM engineering series directly — the vision request just has a richer content payload feeding into the same streaming mechanics.

## Cost-Conscious Defaults

Default to `detail: "low"` unless the question specifically needs fine visual detail, and resize oversized uploads server-side before sending to the API — a user uploading a 20-megapixel photo for a simple "what's in this image" question is paying (in tokens and latency) for resolution the answer doesn't need.

---

*Part of the [Multimodal AI series]({{ site.baseurl }}/tags/multimodal-series/) — next: the same pattern with [Claude Vision]({{ site.baseurl }}/posts/image-qa-app-claude-vision/), including where the two providers' behavior differs.*
