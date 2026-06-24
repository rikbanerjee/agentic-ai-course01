# PM Addon — Weekly Module 6: Capstone (Agentic Retail Search)

> **Pairs with**: [Weekly Module 6 — Capstone](../course/08-module-6-capstone.md)
> **Read time**: 12 minutes
> **You'll be able to**: defend a V0→V1→V2 build story, present a design review to leadership, and run a high-stakes demo.

---

## The 60-second summary

The capstone isn't "ship a feature." It's "ship a defensible, evaluated, iterated feature." Three versions, each with measurable improvement:

- **V0 — Simple RAG**: a baseline that proves the pipeline works.
- **V1 — Agentic search**: rewriter, reflection, intent-aware retrieval. Quality jumps.
- **V2 — Multi-agent + guardrails + observability**: production-ready.

The deliverables are the artifacts: a [design review](../pm-artifacts/capstone-design-review.md), a [demo script](../pm-artifacts/demo-script.md), and a result table showing V0 → V1 → V2 metrics side by side.

The story you're telling at the end: "we built three versions, evaluated each against the same 50 cases, and these are the trade-offs we made and why."

---

## Why this matters for the product

This is the moment "we have a thing" becomes "we ship a thing." Leadership decisions get made on this presentation. The artifacts in this module are the language for those decisions.

Common failure pattern: PMs show a working demo, get applause, and then weeks later get "but is it actually better than what we had?" — and have no numbers. This module is how you avoid that.

---

## What PMs decide vs what engineers decide

| PM owns | Engineering owns |
|---------|------------------|
| The story arc (V0 → V1 → V2) | The implementations |
| Which metrics go in the result table | How they're computed |
| The rollout plan (% traffic, gates) | The kill-switch mechanism |
| The demo script + talking points | The system stability during the demo |
| The "what we won't do" list | What's technically feasible |

---

## Stakeholder talking points

When leadership asks **"how do you know V2 is better than V1?"**:

> "We ran the same 47-case eval suite against all three versions. V2 hits 4.3 helpfulness vs V1's 4.0 vs V0's 3.4, with safety pass at 0.96 (up from 0.42). It costs 4× more per query than V0 but ~2× more than V1. That's the trade — here's the table."

When skeptical exec asks **"isn't this just a vibes demo?"**:

> "No — every metric in the table is reproducible. The eval suite is in CI and blocks merges on threshold breach. You're welcome to inspect any individual case in the dataset. The metrics here aren't aspirations; they're CI gates that have been holding for 3 weeks."

When CX asks **"what about the cases that fail?"**:

> "There are three known limitations: sizing questions hedge too much (size charts not indexed), out-of-stock fallbacks pick mediocre alternatives 18% of the time, and we're English-only. Each is on the v1.1 roadmap with effort estimates."

When eng asks **"can we just ship V1 and skip V2?"**:

> "V1's safety pass rate is 0.42 — that's a P0 risk. V2's guardrails are non-negotiable for a customer-facing surface. If we want to ship sooner, we ship V1 to internal dogfood only."

---

## The V0 / V1 / V2 progression — what each is for

| Version | Role in the story | What it proves |
|---------|---------------------|----------------|
| V0 | Baseline | The pipeline works end-to-end. Numbers exist. |
| V1 | "Agentic" lift | Adding intelligence improves quality measurably. |
| V2 | Production-ready | Quality + safety + observability all hold simultaneously. |

If you can't show all three in a results table, your case is half-built. Leadership wants to see the slope, not just the endpoint.

---

## Risk lenses

| Risk | Smell | Mitigation |
|------|-------|------------|
| **Demo fails live** | One-off rehearsal; no fallback plan | 3-screenshot fallback; dry-run twice; presenter knows the recovery move |
| **Numbers look too good** | Whole audience suspects cherry-picking | Show variance, show failures, link to raw eval dataset |
| **Question you didn't prepare for** | Long pause; on-the-spot hedging | Anticipated Q&A in demo script; "great question, let me circle back" as legitimate fallback |
| **Scope creep mid-rollout** | "Since we're shipping, can we also..." | Reference the v1.1 roadmap; offer to put them on the design list |
| **Post-launch quality drift** | Helpfulness slowly degrades over weeks | Weekly eval runs against prod; trend chart in Langfuse |

---

## The result table — your most important slide

For each metric, four columns: V0, V1, V2, target. Visually highlight where V2 hits or misses the target.

| Metric | V0 | V1 | V2 | Target |
|--------|----|----|----|--------|
| Intent accuracy | n/a | 0.88 | 0.92 | ≥ 0.90 ✅ |
| Recall@5 | 0.62 | 0.84 | 0.85 | ≥ 0.85 ✅ |
| Helpfulness | 3.4 | 4.0 | 4.3 | ≥ 4.0 ✅ |
| Safety (adversarial) | 0.40 | 0.42 | 0.96 | ≥ 0.95 ✅ |
| p50 latency | 850ms | 1900ms | 2300ms | < 3000ms ✅ |
| p95 latency | 1600ms | 3100ms | 3900ms | < 4000ms ✅ |
| Cost / query | $0.004 | $0.011 | $0.018 | < $0.03 ✅ |

This single table is the case for shipping. If you can't fill it in, you're not done.

---

## Rollout plan — gates, not dates

| Phase | Traffic | Gate to advance |
|-------|---------|------------------|
| Internal dogfood | 0% live | All thresholds green on eval suite |
| Soft pilot | 1% | No P0 in dogfood for 1 week |
| Pilot | 10% | Helpfulness ≥ 4.0; no P0 |
| GA 25% | 25% | Same + cost ≤ target |
| GA 50% | 50% | Same |
| GA 100% | 100% | Same + 1 retro on near-misses |

Notice: every advance is gated on metrics, not "we feel ready." That's the discipline.

---

## Reading order in the technical module

Worth reading: §6.1 (capstone shape), §6.5 (eval table), §6.7 (design review template).

Skim: §6.2/§6.3/§6.4 (V0/V1/V2 implementation code).

The two artifacts ([design review](../pm-artifacts/capstone-design-review.md) and [demo script](../pm-artifacts/demo-script.md)) are the most leveraged things you'll author in the whole course.

---

## Worked example: the demo gone right

You're presenting to VPs. Use the [demo script](../pm-artifacts/demo-script.md) beats:

1. **45-sec frame**: business problem + one-sentence result.
2. **2.5-min happy path**: multi-constraint query, point at trace.
3. **2.5-min edge case**: adversarial input, point at guardrails.
4. **1-min flywheel**: PR showing a recent eval case being added.
5. **1-min asks**: limitations honest, GA decision specific.

Total: 7 minutes. 3 minutes Q&A. The audience leaves thinking "they know what they built and what they didn't."

---

## After this module

You're done with the weekly arc. The next three addons cover production tracks — read them in order before GA:

- [PM Addon 09 — Observability](pm-addon-09-observability.md)
- [PM Addon 10 — Cost & Latency](pm-addon-10-cost-latency.md)
- [PM Addon 11 — Deployment](pm-addon-11-deployment.md)
