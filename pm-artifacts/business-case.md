# Business Case — Retail Copilot

**Author**: J. Park (PM, Discovery)
**Reviewers**: VP Product, CFO office, Engineering Director, Head of CX
**Decision asked**: Approve a 6-week build + 4-week pilot of an agentic retail copilot for the .com storefront.
**Date**: 2026-05-04
**Status**: Draft v0.3 — circulating for comment

> 💡 **Why this section exists**: every reader wants to know who to push back on, who's already bought in, and what specifically they're being asked to approve. Put it on top.

---

## TL;DR

We propose a 10-week investment (6 weeks build + 4 weeks pilot) to launch a natural-language shopping copilot on northcrest.com. The hypothesis: shoppers using the copilot will convert at 1.4× the rate of shoppers who use only faceted search, paying back the build cost within 90 days of GA at our current traffic.

If we hit our pilot success criteria, we go to GA in Q4. If we miss, we sunset and document learnings — total exposure is ~$180K of engineering time and ~$8K of LLM costs.

---

## Problem we're solving

Outdoor shoppers describe products in fuzzy, multi-constraint terms: *"a warm jacket that's not too bulky for layering, under $200, in stock for delivery before Friday."* Our current experience forces them to translate that into facets — category, price range, attribute checkboxes — and ~38% bounce before completing a discovery.

Three signals tell us this is real:

1. **Bounce data**: 38% bounce within the first 90s of arriving on a category page (up from 31% two years ago — faceted UX is decaying as catalog grows).
2. **Support tickets**: 18% of inbound tickets are *"can you help me find…"* questions that a copilot could deflect.
3. **Customer interviews**: 11 of 14 shoppers in last quarter's research session said they'd "rather ask a person" than fight the filters.

---

## What we're proposing

A copilot embedded in the storefront that:

- Accepts natural-language shopping queries.
- Searches the catalog with attribute + price + stock filters inferred from the query.
- Answers policy questions (returns, shipping, warranty) with citations.
- Looks up order status for logged-in shoppers.
- Escalates anything outside scope to existing human support.

What it explicitly **does not do** in v1:

- Promise refunds, credits, or shipping upgrades.
- Modify orders or account data.
- Speak non-English.
- Handle image search.

---

## Expected impact (hypotheses, not promises)

| Hypothesis | Source | Risk if wrong |
|------------|--------|---------------|
| 1.4× conversion lift on copilot-engaged sessions | Industry comps + our own A/B tests of guided shopping flows | Low — even at 1.1× the cost recovers |
| 15% deflection of policy/return tickets | CX historical data; mid-market peer benchmarks | Medium — depends on answer quality and trust |
| ≤ $0.03 LLM cost per query at GA scale | Course module 10 budgeting; current Gemini/OpenAI pricing | Medium — model prices trend down but can spike |
| p95 latency ≤ 4s | Course module 10 latency budget; pilot traffic ~3 QPS p99 | Low |

---

## Costs

### One-time (pilot)

| Item | Cost | Notes |
|------|------|-------|
| Engineering (2 FTE × 10 weeks) | $130K | 1 backend, 1 full-stack |
| PM + design (0.5 FTE × 10 weeks) | $35K | |
| LLM API spend (pilot) | $5–8K | gpt-4o + gemini-2.0-flash; 4-week pilot, ~50K queries |
| Vendor: Langfuse (observability) | $0 | Self-host on existing infra |
| Vendor: Vector DB | $0 | Chroma local for pilot; revisit at GA |
| Contingency | $15K | |
| **Total pilot** | **~$185K** | |

### Recurring (post-GA)

| Item | Annual | Notes |
|------|--------|-------|
| LLM API at projected GA volume (500K queries/mo) | $180K | $0.03/query × 500K × 12 |
| Hosted vector DB upgrade (Qdrant Cloud) | $24K | |
| Observability (Langfuse hosted) | $12K | |
| Maintenance engineering (0.5 FTE) | $90K | Evals upkeep, prompt iteration, incident response |
| **Total annual** | **~$306K** | |

### Payback

Assumed $42 AOV, 11% baseline conversion, 1.4× conversion lift on engaged sessions, 25% of traffic engaging:

- Incremental conversion: 0.25 × (0.11 × 1.4 − 0.11) = 1.1pp of total traffic
- Incremental revenue/month at 1M sessions: 1M × 0.011 × $42 = **$462K/month**
- Payback on pilot ($185K): **~13 days post-GA**
- Payback on annual recurring ($306K): **~20 days/year**

These assume conservative deltas; sensitivity analysis appended below.

---

## Risks & mitigations

| Risk | Likelihood | Impact | Mitigation |
|------|------------|--------|------------|
| Copilot hallucinates SKUs/prices and embarrasses brand | Medium | High | Guardrails layer (no claim without a tool result), eval threshold ≥ 0.95 on grounding before launch, kill switch |
| LLM cost overrun (3× projected) | Low–Medium | Medium | Daily spend cap; alerts at 50%/80%; fallback to gemini-flash-only mode |
| Provider outage during peak shopping | Medium | High | Multi-provider failover (OpenAI ↔ Gemini); cached graceful degradation message |
| Prompt injection / abuse | High | Low–Medium | Input guardrails; output filtering; rate limits; abuse logs reviewed weekly |
| Conversion lift fails to materialize | Medium | High (sunset) | 4-week pilot with explicit go/no-go criteria |
| Compliance: PII leakage to LLM provider | Low | High | Order IDs masked in prompts; data processing agreements signed before pilot |

---

## Success criteria for the pilot

**Go to GA if all three:**
1. Helpfulness score (LLM-judge, human-spot-checked) ≥ 4.0/5 on 200+ live sessions.
2. Conversion lift on copilot-engaged sessions ≥ 1.25× control.
3. Zero P0 incidents (hallucinated price, false refund promise, PII leak).

**No-go (sunset or rebuild) if any:**
1. Helpfulness < 3.5 after 2 weeks of iteration.
2. Conversion lift < 1.1× (insufficient ROI at projected costs).
3. ≥ 1 P0 incident not resolvable by guardrails.

---

## Alternatives we considered

1. **Improve faceted search only** (no LLM). Cheaper, but addresses symptoms not the root translation gap. Estimated 5–10% conversion lift at most.
2. **Buy a third-party shopping assistant SaaS.** $60K/year subscription; less control over prompts, evals, data residency. Worth revisiting if internal build slips.
3. **Wait for native commerce features in our existing search vendor.** Vendor's roadmap is opaque; 12-month wait minimum.

---

## Decision asked

Approve $185K pilot budget + 2 FTE engineering allocation. Decision needed by **2026-05-15** to align with engineering capacity in the next planning cycle.

---

## Appendix: sensitivity analysis

| Conversion lift on engaged sessions | Engagement rate | Monthly incremental revenue | Payback period |
|-------------------------------------|------------------|------------------------------|----------------|
| 1.1× | 25% | $115K | ~2 months |
| 1.25× | 25% | $289K | ~3 weeks |
| 1.4× (base case) | 25% | $462K | ~13 days |
| 1.4× | 15% | $277K | ~3 weeks |
| 1.4× | 40% | $740K | ~8 days |

Break-even on annual recurring at 1.15× lift with 20% engagement.

---

## How this gets out of date

- Model prices change quarterly. Re-run the cost columns before any GA decision.
- Conversion baseline shifts with seasonality. Lock pilot dates against a low-variance period (post-holiday, pre-summer).
- New competitive launches in agentic commerce may change the alternative comparisons.
