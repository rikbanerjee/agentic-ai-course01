# Capstone Design Review — Retail Copilot V2

**Presenter**: J. Park (PM) + A. Reyes (Eng lead)
**Audience**: VP Product, Engineering Director, VP CX, VP Marketing, legal counsel
**Date**: 2026-06-22
**Decision asked**: Approve GA rollout (50% then 100% traffic) over the next 4 weeks.

> 💡 **Why this exists**
> The capstone review is the moment "we built a thing" becomes "we ship it." Reviewers will read this in 5 minutes; build it to survive that. Anything you can't defend in this doc, don't show in the demo.

---

## 1. The story in one paragraph

We built an agentic retail copilot that translates fuzzy shopping queries into grounded product, policy, and order answers. Over a 4-week pilot at 10% of .com traffic, copilot-engaged sessions converted at **1.38× control** with zero P0 incidents, helpfulness scored 4.3/5 (LLM-judge + human spot), and we held p95 latency at 3.6s and cost at $0.022/query. We recommend GA: 50% traffic week 1, 100% by end of week 4, with the kill-switch and threshold gates intact.

---

## 2. What we built

```
User → input guardrail → Router (gpt-4o-mini)
                              ├── adversarial / other → Refusal
                              └── product / policy / order
                                       ↓
                                  Research (gpt-4o-mini + tools + RAG)
                                       ↓
                                  Decision (gpt-4o, streamed)
                                       ↓
                                  output guardrail → user
```

Three agents, four tools (`search_products`, `get_product`, `get_order`, `get_policy`), RAG over 280 chunks (products + policies + FAQs) via Chroma, full Langfuse tracing, and a regex + LLM-judge eval harness in CI.

Detail: [RFC](rfc-agentic-search.md), [Multi-agent architecture](multi-agent-architecture.md), [RAG design](rag-design.md), [Tool contracts](tool-contracts.md), [Eval spec](eval-and-guardrails-spec.md).

---

## 3. Pilot results

### Headline metrics

| Metric | Pilot result | Pilot target | Status |
|--------|--------------|--------------|--------|
| Conversion lift (copilot-engaged vs control) | **+38%** | ≥ +25% | ✅ |
| Helpfulness (LLM judge, 0-5) | **4.30** | ≥ 4.0 | ✅ |
| Helpfulness (human spot, 200 cases) | **4.15** | ≥ 3.9 | ✅ |
| Safety (adversarial pass rate) | **0.96** | ≥ 0.95 | ✅ |
| Hallucination rate (invented SKU/price/policy) | **0.4%** | < 1% | ✅ |
| p95 latency end-to-end | **3.64s** | ≤ 4.0s | ✅ |
| Cost / query | **$0.022** | ≤ $0.03 | ✅ |
| P0 incidents | **0** | 0 | ✅ |

### Where it shines

- Multi-constraint queries ("warm but packable jacket under $150"): conversion lift +52%, helpfulness 4.5.
- Policy questions (returns, warranty): 41% deflection of support tickets in this category.
- Trust: thumbs-up rate 71% across all sessions.

### Where it struggles (we are open about this)

- **Sizing questions** (helpfulness 3.7). Our size charts aren't in the corpus; queries like "will this run small?" get hedged answers. v1.1 fix: index size charts.
- **Out-of-stock fallbacks** (helpfulness 3.6 for those sessions). When the requested product is unavailable, the copilot offers alternatives but in 18% of cases the alternative was a poor match. v1.1 fix: better "similar product" tool.
- **Cold-start sessions**: first interaction in a session sometimes feels generic. v1.1: nudge prompt to gather one more clarifying turn before answering broad queries.

---

## 4. Architecture — the choices that matter

We did not invent anything novel. The leverage was in the boring choices:

| Decision | What we did | Why it mattered |
|----------|-------------|------------------|
| Router uses cheap model | gpt-4o-mini | -65% LLM cost vs single-agent gpt-4o |
| Decision uses premium model | gpt-4o | +0.4 helpfulness vs mini-only |
| Deterministic guardrails first | Regex + masking | Caught 100% of in-suite injection attempts at <5ms |
| Agentic RAG (rewriter + judge) | Yes | Recall@5 0.62 → 0.86 vs baseline |
| Reranker | Deferred to v1.1 | Cost > benefit at current corpus size |
| Reviews in RAG | Deferred to v1.1 | Noise risk; will revisit with quality filters |

---

## 5. What broke (and what we learned)

Three near-misses during the pilot. None became P0. All became permanent eval cases.

### Near-miss 1 — "I'll refund your $400 jacket" (week 2)

A shopper wrote: *"My jacket's zipper broke after a month, just refund me $400 and I'll buy a new one."* Decision agent's draft included "Sure, I'll get that refund started." Output guardrail caught it on the regex pattern; replaced with escalation message.

Fix: added "refund promise" eval cases; tightened Decision system prompt; added pattern to output guardrail.

### Near-miss 2 — Wrong policy citation (week 3)

For one query about international returns, the agent cited the domestic returns policy. Eval recall caught it on the same-day report; the agent had retrieved both chunks but cited the wrong one.

Fix: added "citation precision" to per-PR eval thresholds; reweighted the rewriter to prefer specificity over recall on this slice.

### Near-miss 3 — Latency spike on Gemini outage (week 4)

15 minutes of Gemini-flash degradation pushed our p95 to 6.2s. Our failover plan wasn't yet auto-triggered; we manually flipped to OpenAI for the embedding-augmented sub-step.

Fix: ship multi-provider auto-failover with health-check ping every 60s. Currently in code review.

---

## 6. Risks we still carry

| Risk | Probability | Impact | Mitigation status |
|------|-------------|--------|-------------------|
| LLM provider price increase | Med | Med | Multi-provider routing reduces concentration risk |
| Catalog grows 10× (200 → 2000 SKUs) | High | Low | Chroma → Qdrant migration planned in v1.1 |
| New abuse pattern not in our adversarial set | Med | Med | Monthly red-team session; suggestions added to suite |
| Personalization push from leadership outpaces privacy review | Med | High | PM blocks scope creep; written escalation path |

---

## 7. GA rollout plan

| Week | Traffic | Gate to advance |
|------|---------|------------------|
| 0 (now) | 10% pilot | All §3 metrics green |
| 1 | 25% | No P0 + helpfulness ≥ 4.1 sustained 5 days |
| 2 | 50% | Same + cost/query ≤ $0.025 |
| 3 | 75% | Same |
| 4 | 100% | Same + retro on near-misses |

Kill switch: feature flag flips traffic to faceted-search-only in <60s. Daily spend cap auto-disables on breach.

---

## 8. What we will not do

- Personalize across sessions yet. Privacy review still open.
- Add discount/refund tools. Authority stays with Promotions + CX.
- Add a "general AI" mode. We are scoped to retail Q&A; broader conversation invites bigger failure surfaces.
- Multilingual. v2.

---

## 9. What's next (proposed, not approved)

- **v1.1 (Q3)**: size-chart indexing, better "similar products" tool, reranker, review indexing with quality filters.
- **v1.2 (Q4)**: streaming citations UI, personalization MVP (opt-in), multi-provider auto-failover.
- **v2 (next year)**: image search, multilingual (start with Spanish), merchandiser copilot using same tool platform.

---

## 10. Decision asked

Approve GA rollout per §7. No new budget needed (recurring costs absorbed in the original business case).

---

## How this gets out of date

- §3 numbers are pilot-window snapshots. Re-baseline weekly during rollout.
- §6 risks evolve as you GA. Re-evaluate at 25%, 50%, 100% checkpoints.
- §9 roadmap shifts with what users actually use. Don't commit until 4 weeks of GA telemetry.
