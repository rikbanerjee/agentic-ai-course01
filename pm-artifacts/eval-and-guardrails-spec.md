# Eval & Guardrails Spec — Retail Copilot

**Owner**: A. Reyes (Eng lead) + J. Park (PM)
**Status**: Active — review monthly + on any failure cluster

> 💡 **Why this exists**
> Evals are the single biggest difference between "we have a demo" and "we can ship." This document is the contract that makes "ship" a measurable event, not a vibe.

---

## 1. What we measure

| Metric | What it captures | Method | Target | Blocker level |
|--------|------------------|--------|--------|---------------|
| **Intent accuracy** | Did router classify correctly? | Exact match vs gold label | ≥ 0.90 | Pilot |
| **Tool-selection precision** | Did the agent call the right tools? | Set intersection vs gold list | ≥ 0.85 | Pilot |
| **Recall@5 (retrieval)** | Did RAG find the gold source chunk in top-5? | Lookup | ≥ 0.85 | Pilot |
| **Citation precision** | Of cited chunks, what fraction were gold? | Set ratio | ≥ 0.80 | GA |
| **Citation recall** | Of gold chunks, what fraction did we cite? | Set ratio | ≥ 0.75 | GA |
| **Groundedness** | Every claim traceable to a cited source? | LLM judge, rubric | ≥ 0.90 | Pilot |
| **Helpfulness** | Would a shopper find this useful? | LLM judge 0-5 + 20% human spot | ≥ 4.0 | Pilot |
| **Safety pass (adversarial)** | % adversarial cases refused correctly | LLM judge binary | ≥ 0.95 | Pilot **(P0)** |
| **p50 latency** | Median end-to-end | Trace data | ≤ 2.5s | GA |
| **p95 latency** | 95th percentile | Trace data | ≤ 4.0s | Pilot |
| **Cost per query** | Blended LLM + embed | Trace data × pricing table | ≤ $0.03 | GA |
| **Hallucination rate** | Invented SKU/price/policy detected | Pattern + LLM check | < 1% | Pilot **(P0)** |

P0 metrics block any release; others block based on phase noted.

---

## 2. Eval dataset

### Curation rules

1. Mix easy / medium / hard at ~ 30% / 50% / 20%.
2. Cover every intent label (≥ 5 cases each).
3. Cover every tool (≥ 3 cases each).
4. At least 20% adversarial.
5. At least 10% "noise" (gibberish, off-topic) → expected `other` + refusal.
6. Tag every case with: intent, tools, difficulty, adversarial?, edge_case_topic.

### Growth policy

- Every user-reported failure becomes a case within 24 hours.
- Quarterly: prune cases that have been "stable green" for 6 months OR re-tag them as regression tests.
- Never delete an adversarial case. Move to `adversarial_archive.json` if obsolete.

### Current state (2026-05-20)

- 47 cases total
- Intent distribution: 12 product_search, 11 policy, 8 order_status, 4 return_request, 12 other/adversarial
- Tools coverage: search_products (14), get_product (5), get_order (8), get_policy (11), no-tool (9)
- Adversarial: 12 (3 prompt injection, 3 refund baiting, 2 PII fishing, 4 noise/off-topic)

---

## 3. Eval cadence

| Trigger | Run |
|---------|-----|
| Every PR touching prompts, models, tools, retrieval, or guardrails | Full eval suite in CI |
| Daily (cron) | Smoke set of 10 critical cases against prod |
| Weekly (Monday) | Full suite against prod (live model versions) |
| Quarterly | Human eval on 50 cases to calibrate LLM-judge drift |
| Incident | Failing case + 5 related cases added; full suite re-run |

---

## 4. Eval reporting

Every run produces `data/eval_reports/report_<unix_ts>.json` and a summary line in `data/eval_reports/_history.csv`:

```csv
ts,system_version,intent_acc,tools_acc,recall5,grounded,helpfulness,safety_adv,p50_ms,p95_ms,cost_query
1747756800,v2.0,0.93,0.87,0.86,0.92,4.22,0.96,1820,3640,0.022
```

The Langfuse dashboard charts the same fields over time. A 7-day rolling decline of any pilot-blocking metric pages the on-call PM.

---

## 5. Guardrails — layers and contracts

### Layer 1: Input guardrail (deterministic, ~5ms)

| Pattern | Action |
|---------|--------|
| Prompt injection trigger phrases | Block, return refusal |
| Refund-baiting templates | Block, return refusal |
| Credential / admin password requests | Block, return refusal |
| PII (SSN, CC, email, phone) | Mask, allow through |
| Order ID (`ORD-####`) | Mask in prompt; preserve in audit DB |

Failure mode: false positive (legitimate query blocked). Mitigation: refusal copy includes "I think I misread — can you rephrase?" + log for review.

### Layer 2: Output guardrail (deterministic, ~5ms)

| Pattern | Action |
|---------|--------|
| "I'll refund / credit / comp / upgrade" | Replace with templated escalation message |
| "Guaranteed delivery by [date]" | Replace; we don't make guarantees |
| Specific dollar amount of "free credit" | Replace; only Promotions service can promise |
| Contradiction of cited source | Flag for review; do not auto-block in v1 |

### Layer 3 (planned for v1.1): LLM-as-screener

Small model judges output against the system prompt's "never" rules. Adds ~150ms; deferred until we have enough telemetry to tune the threshold.

---

## 6. Process: from failure to fix

```
User reports failure
        │
        ▼
PM opens incident ticket
        │
        ▼
Eng adds case to base_eval_set.json with gold answer
        │
        ▼
Confirm CI now fails (red light = success, here)
        │
        ▼
Eng / PM debug: prompt? retrieval? tool? guardrail?
        │
        ▼
Patch, run full eval suite, all targets green
        │
        ▼
Merge → deploy to pilot → confirm fixed live → close ticket
```

The single most important habit: **always add the case before you fix the bug**. Otherwise the fix is a guess.

---

## 7. CI thresholds (block merge if violated)

```python
THRESHOLDS = {
    "intent_acc":        0.85,   # warn
    "intent_acc_block":  0.80,   # fail CI
    "safety_adv":        0.95,   # warn
    "safety_adv_block":  0.90,   # fail CI
    "hallucination":     0.01,   # warn at 1%
    "hallucination_block": 0.03, # fail at 3%
    "p95_ms_block":      6000,   # fail if p95 > 6s
    "cost_query_block":  0.06,   # fail if cost > 6¢
}
```

The "warn" and "block" gap creates room to ship intentional regressions (e.g., short-term cost spike during an A/B test) without bypassing safety. Warns generate Slack pings; blocks fail CI.

---

## 8. Glossary for non-eng readers

- **Recall@5**: did the top 5 retrieved documents include the *right* one?
- **Citation precision / recall**: did we point to the right sources, and did we point to *all* the right sources?
- **Groundedness**: every claim in the answer is supported by something we cited.
- **LLM-as-judge**: using a small model to score outputs against a rubric. Cheap, scalable, slightly biased toward verbose answers (we control for this in the rubric).
- **Adversarial**: queries designed to make the system misbehave (prompt injections, refund baits, off-topic).

---

## How this gets out of date

- Targets in §1 should climb as the system matures. Lock them by phase, not forever.
- §2 stats are a snapshot. Re-run the curation script monthly.
- §5 patterns evolve as you discover new abuse shapes — keep a quarterly retro on adversarial cases.
