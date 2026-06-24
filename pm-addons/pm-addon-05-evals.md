# PM Addon — Weekly Module 3: Evaluation & Guardrails

> **Pairs with**: [Weekly Module 3](../course/05-module-3-evaluation-guardrails.md)
> **Read time**: 14 minutes
> **You'll be able to**: define quality metrics, run the eval flywheel, and resist pressure to ship without evals.

---

## The 60-second summary

**Evals are the spine of an LLM product.** Without them: every prompt change is a guess, every release is a coin flip, every user complaint is impossible to learn from.

Three habits to enforce:

1. **Every user-reported failure becomes a new eval case before it's fixed.** Otherwise the fix is a guess.
2. **CI runs evals on every PR.** Quality regressions block merges.
3. **Quality, cost, latency, and safety are tracked together over time.** A 0.05 lift in helpfulness at 2× cost is a different story than at the same cost.

Two layers of guardrails:

- **Pre-LLM (input)**: regex catches prompt injection, refund-baiting, PII before it hits the model.
- **Post-LLM (output)**: regex catches forbidden promises before they reach the user.

The PM artifact: an [eval & guardrails spec](../pm-artifacts/eval-and-guardrails-spec.md).

---

## Why this matters for the product

This is the module where shippable becomes a measurable thing. PMs who skip evals end up in two places:

1. **"It feels worse" arguments** — endless, unresolvable, demoralizing.
2. **Whack-a-mole post-launch** — fixing the same class of bug six different ways.

PMs who invest in evals get:

1. **A merge-blocking quality gate** that protects launches.
2. **A defensible launch story** ("here are 47 cases at these thresholds").
3. **A backlog of real failure cases** that becomes the highest-leverage roadmap input.

---

## What PMs decide vs what engineers decide

| PM owns | Engineering owns |
|---------|------------------|
| Eval target thresholds | LLM-judge prompt + harness code |
| Case curation rules | Test infrastructure (CI integration) |
| Which metrics block release | Where eval reports are stored |
| Quarterly human calibration | How to score automatically |
| Guardrail policy ("never promise X") | Regex patterns / classifier implementation |

The eval spec is co-authored. PMs sign off on thresholds; eng signs off on feasibility.

---

## Stakeholder talking points

When leadership asks **"how do you know it works?"**:

> "We have 47 labeled eval cases covering every intent and tool, including 12 adversarial cases. Current pass rates are [list]. The CI fails any PR that drops these below thresholds. I can show you the dashboard."

When eng asks **"can we skip evals on this small change?"**:

> "No. Two reasons. One: small changes are exactly where regressions hide — they don't trigger review attention. Two: if we skip once, we skip again. Let's just run them."

When CX brings a user complaint **"the agent said something wrong"**:

> "Great — let me see the trace. We'll create an eval case for it before patching. That way the fix is provable and the bug can never silently come back."

When finance asks **"why does eval cost LLM API calls?"**:

> "Every eval run costs $5-10 in API calls (47 cases × 2 LLM calls per case = ~100 calls, mostly gpt-4o-mini). Per release we run it 1-5 times. Annual eval spend is maybe $2K. That's cheap insurance against a P0."

---

## The eval flywheel — internalize this picture

```
[User complaint] → [PM opens ticket] → [Eng adds eval case]
                                              ↓
                          [Confirm CI fails on new case]
                                              ↓
                            [Patch the bug + verify CI green]
                                              ↓
                            [Merge → deploy → close ticket]
```

The critical insight: **case added before fix**. Otherwise the fix is post-hoc rationalization.

---

## Metrics framework

Four metric families. Every product should track all four.

| Family | What it answers | Example | Method |
|--------|------------------|---------|--------|
| **Correctness** | Did we give the right answer? | Intent accuracy, citation precision | Exact match + LLM judge |
| **Helpfulness** | Would the user value this? | Helpfulness 0-5 | LLM judge + human spot |
| **Safety** | Did we refuse what we should? | Adversarial pass rate | LLM judge + rule check |
| **Cost/perf** | Can we sustain this at scale? | p95 latency, $/query | Trace data |

Skip any family → you're shipping blind on that dimension.

---

## Risk lenses

| Risk | Smell | Mitigation |
|------|-------|------------|
| **Threshold drift** | Quietly lowering targets when builds miss | Thresholds in source-controlled config; require PM+Eng sign-off to change |
| **LLM-judge bias** | Judge rewards verbosity / matches its own writing | Quarterly human calibration; rubric forbids verbosity reward |
| **Case set staleness** | Same 30 cases for 6 months | Growth policy: ≥ 2 new cases per week during active development |
| **Skipping evals on "trivial" changes** | "It's just a prompt tweak" | CI runs evals on every PR touching prompts/models/tools/retrieval |
| **Adversarial set lag** | Real abuse patterns not yet in eval set | Monthly red-team review of recent logs |

---

## Guardrails layered framework

```
                                ┌─ blocked (input filter) ─┐
                                │                          │
user input ─► input guardrail ──┤                          ├─► refusal handler
                                │                          │
                                └─ allowed (masked) ───────┘
                                            ↓
                                      LLM pipeline
                                            ↓
                                     model output
                                            ↓
                                output guardrail
                                ┌─ flagged (replaced) ─┐
                                │                      │
                                └─ allowed ────────────┘
                                            ↓
                                      user sees
```

Layer choices:

| Layer | Tech | Pros | Cons |
|-------|------|------|------|
| Pre-LLM regex | Deterministic | Fast, no false negatives on listed patterns | Misses paraphrased attacks |
| Pre-LLM classifier | Small LLM | Catches paraphrases | Adds 150-500ms |
| Post-LLM regex | Deterministic | Fast | Same paraphrase limits |
| Post-LLM classifier | Small LLM | Robust | Adds latency and cost |

Default v1: regex on both ends. Add classifier in v1.1 if needed based on telemetry.

---

## Reading order in the technical module

Worth reading in full: §3.1 (benchmarks vs product evals), §3.2 (flywheel), §3.6 (reading results), §3.10 (eval spec assignment).

Skim: §3.4 (eval dataset construction lab), §3.7 (guardrails code).

Skip: code internals if not implementing.

---

## Worked example: setting eval thresholds with leadership

VP P asks: *"What does 'good enough to launch' look like?"*

PM playbook:

1. **Anchor on hard constraints first.** Safety = 0.95 pass rate (non-negotiable). Hallucination < 1% (P0 risk).
2. **Anchor on user perception.** Helpfulness ≥ 4.0/5 — that's the threshold where LLM-judge + human-spot agree.
3. **Anchor on business.** Conversion lift ≥ 1.25× control on pilot.
4. **Soft metrics.** Intent accuracy ≥ 0.90 (warm gate; doesn't block but warns).
5. **Counter-metrics.** Don't regress CSAT, p95, $/query.

Drop these in the eval spec. Sign-off ritual: PM + Eng + (Safety/Legal where relevant). When pressure builds to lower a threshold, point at the sign-off.

---

## After this module

Move to [PM Addon 06 — Agentic RAG](pm-addon-06-rag.md). RAG is where most retrieval failures live.
