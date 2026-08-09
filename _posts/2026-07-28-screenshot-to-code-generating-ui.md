---
title: "Screenshot-to-Code: Generating UI from Images"
date: 2026-07-28 08:00:00 +0530
categories: [AI, Multimodal]
tags: [multimodal, multimodal-series, code-generation, python]
---

A VLM reading a UI mockup or screenshot and generating working code for it combines this month's vision understanding with the coding-agent techniques from April — a genuinely practical use case for design-to-development handoff.

## Basic Screenshot-to-Code

```python
def generate_ui_code(screenshot_bytes: bytes, framework: str = "React + Tailwind") -> str:
    response = client.messages.create(
        model="claude-sonnet-5", max_tokens=4096,
        messages=[{"role": "user", "content": [
            {"type": "image", "source": {"type": "base64", "media_type": "image/png", "data": base64.b64encode(screenshot_bytes).decode()}},
            {"type": "text", "text": f"Generate {framework} code that recreates this UI as closely as possible. "
                                     f"Match layout, spacing, colors, and text content exactly."},
        ]}],
    )
    return extract_code_block(response.content[0].text)
```

## Iterative Refinement Against a Rendered Screenshot

The highest-value technique here is closing the loop visually — render the generated code, screenshot the result, and compare it back against the original:

```python
def iterative_screenshot_to_code(original_screenshot: bytes, max_rounds: int = 3) -> str:
    code = generate_ui_code(original_screenshot)
    for _ in range(max_rounds):
        rendered = render_and_screenshot(code)  # headless browser render, from the run skill's playwright pattern
        diff_analysis = compare_screenshots(original_screenshot, rendered)
        if diff_analysis["match_score"] > 0.9:
            return code
        code = refine_code(code, original_screenshot, diff_analysis["issues"])
    return code

def compare_screenshots(original: bytes, rendered: bytes) -> dict:
    response = client.messages.create(model="claude-sonnet-5", max_tokens=1024, messages=[{
        "role": "user", "content": [
            {"type": "text", "text": "Original:"},
            {"type": "image", "source": {"type": "base64", "media_type": "image/png", "data": base64.b64encode(original).decode()}},
            {"type": "text", "text": "Generated:"},
            {"type": "image", "source": {"type": "base64", "media_type": "image/png", "data": base64.b64encode(rendered).decode()}},
            {"type": "text", "text": "List specific visual differences and rate overall match 0-1."},
        ],
    }])
    return json.loads(response.content[0].text)
```

This visual diff-and-refine loop is what separates a rough first pass from production-usable output — a single-shot generation rarely matches spacing and alignment precisely, but two or three refinement rounds against actual rendered output closes most of that gap.

## Extracting Design Tokens Separately

For consistency across multiple screens, extract reusable design tokens (colors, spacing scale, typography) once and reference them in every subsequent generation, rather than re-deriving them from scratch per screenshot:

```python
def extract_design_tokens(screenshot_bytes: bytes) -> dict:
    response = client.messages.create(model="claude-sonnet-5", max_tokens=1024, messages=[{
        "role": "user", "content": [
            {"type": "image", "source": {"type": "base64", "media_type": "image/png", "data": base64.b64encode(screenshot_bytes).decode()}},
            {"type": "text", "text": "Extract design tokens as JSON: colors (hex), spacing scale, font sizes."},
        ],
    }])
    return json.loads(response.content[0].text)
```

## Handling Component Reuse

For a full multi-screen mockup set, identify repeated components (buttons, cards, nav bars) across screenshots first, generate each once as a reusable component, then compose screens from them — a naive per-screenshot generation approach produces duplicated, inconsistent implementations of what should be the same shared component.

## Where This Fits in a Real Design-to-Dev Workflow

Screenshot-to-code is strongest as an accelerant for a first-draft implementation a developer refines, not a fully automated replacement for design implementation — treat generated code the way you'd treat the coding-agent output from April's self-review post: a strong starting point still needing human review before merging, particularly for accessibility attributes and responsive behavior a static screenshot can't fully specify.

---

*Part of the [Multimodal AI series]({{ site.baseurl }}/tags/multimodal-series/) — next: [evaluating multimodal model outputs]({{ site.baseurl }}/posts/evaluating-multimodal-model-outputs/), extending June's evaluation series to this month's content types.*
