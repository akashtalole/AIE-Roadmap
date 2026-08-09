---
title: "Deploying Your Capstone Project for Free"
date: 2026-12-19 08:00:00 +0530
categories: [AI, Career]
tags: [career, career-series, deployment]
---

Every capstone brief this month referenced "a working, deployed demo" as a requirement. This post covers actually doing that without incurring meaningful cost — a live demo is dramatically more persuasive in a portfolio than a "clone and run locally" README.

## Why a Live Demo Matters So Much More Than Code Alone

A reviewer skimming dozens of portfolios will click a live link far more readily than clone a repo and run setup steps — the friction difference is enormous, and a live, working demo is often the single highest-leverage thing you can add to a capstone project's presentation.

## Free-Tier Hosting Options

```python
free_hosting_options = {
    "frontend_static": "Vercel, Netlify, GitHub Pages — free tiers comfortably handle portfolio-scale traffic",
    "backend_api": "Railway, Render, Fly.io free/hobby tiers — sufficient for a low-traffic demo",
    "serverless_functions": "November 10's serverless post applies directly — very low cost for intermittent demo traffic",
}
```

## Managing Model API Costs for a Public Demo

```python
demo_cost_controls = {
    "rate_limit_aggressively": "August's rate-limiting post — a public demo needs much stricter limits than a real product",
    "use_a_cheap_model": "a demo doesn't need your most capable model — route to a fast, cheap tier by default",
    "add_a_hard_daily_budget_cap": "March's budget-guardrail pattern, applied to protect your own wallet",
}
```

```python
def demo_budget_guard(daily_spend: float, daily_cap: float = 2.00) -> bool:
    if daily_spend >= daily_cap:
        disable_demo_temporarily()
        return False
    return True
```

A hard daily spend cap that gracefully disables the demo (with a clear "demo temporarily unavailable, check back tomorrow" message, following November 23's graceful-failure UI principles) rather than accumulating unbounded cost is essential — a portfolio demo going viral or getting hit by a bot shouldn't produce a surprise bill.

## Handling Secrets Safely in a Public Deployment

```python
deployment_secrets_checklist = {
    "never_commit_api_keys": "use the hosting platform's environment variable management",
    "use_a_demo_specific_api_key_with_a_low_limit": "not your primary personal API key",
    "rotate_if_a_repo_was_ever_public_with_a_key_exposed": "check git history, not just the current state",
}
```

## Making the Demo Resilient to Abuse

```python
def protect_public_demo(request: dict) -> bool:
    if is_likely_bot_traffic(request):
        return False
    if exceeds_per_ip_rate_limit(request["ip"]):
        return False
    return True
```

A public demo link, once shared, will attract some abusive or bot traffic — basic protections (rate limiting per IP, a lightweight bot-detection check) prevent this from either running up costs or degrading the experience for genuine reviewers.

## What to Include Alongside the Live Link

```python
demo_presentation_checklist = {
    "live_link": "the primary deliverable",
    "a_short_screen_recording": "as a fallback if the live demo is ever down, and useful for quick skimming",
    "clear_instructions_for_what_to_try": "a reviewer with limited time needs a few suggested example queries",
}
```

A short (60-90 second) screen recording as a backup ensures the project still demonstrates well even if the live demo happens to be down during review — belt-and-suspenders presentation that respects a reviewer's limited time and patience for troubleshooting a broken link.

## Sunsetting a Demo Responsibly

```python
def sunset_demo(project: dict):
    update_portfolio_entry(project, note="Live demo retired — see recording and code for reference")
    disable_hosting()
```

When a demo is no longer worth maintaining (an old capstone superseded by newer work), retire it cleanly with a note and the fallback recording rather than letting it silently rot into a broken link — a dead link in a portfolio is worse than no link at all.

---

*Part of the [Career series]({{ site.baseurl }}/tags/career-series/) — next: [writing a case study about your capstone project]({{ site.baseurl }}/posts/case-study-capstone-project-writing/).*
