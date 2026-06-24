# PM Addon — Cost & Latency Optimization

> **Pairs with**: [Cost & Latency Optimization track](../course/10-cost-latency-optimization.md)
> **Read time**: 10 minutes
> **You'll be able to**: build a cost/latency budget, decide which levers to pull, and defend the trade-offs to finance and eng.

---

## The 60-second summary

For a typical agentic system, the final-answer LLM call eats >50% of cost and ~40% of latency. Knowing this changes how you spend optimization effort.

Seven cost levers (cheapest first to consider):

1. Right-size the model per agent (cheap classifier + premium synthesizer)
2. Prompt caching (50-90% off cached prefix)
3. Smaller retrieval context (rerank to top-3 instead of top-5)
4. Cache embeddings + repeated rewrites
5. Skip the LLM when deterministic logic works (regex, lookup)
6. Batch when possible (50% off on async)
7. Truncate aggressively when over budget

Eight latency levers:

1. Stream (cuts time-to-first-token dramatically)
2. Parallelize independent calls
3. Skip steps when confident
4. Smaller models on hot paths
5. Avoid unnecessary roundtrips (parallel tool calls)
6. Warm the path at startup
7. Edge deploy hot paths
8. Speculative execution

You don't optimize all three of quality, cost, and latency simultaneously. Pick what to fix.

---

## Why this matters for the product

Cost and latency are not "tech debt" — they directly determine GA viability.

- **At pilot scale**, cost feels manageable. At 100× volume, a $0.02 query becomes $50K/month.
- **A 1-second latency improvement** can lift conversion 5-10% on commerce surfaces.
- **A 50% cost reduction** can be the difference between a green or red P&L.

PMs who don't engage here surrender major levers to engineering judgment alone.

---

## What PMs decide vs what engineers decide

| PM owns | Engineering owns |
|---------|------------------|
| Cost budget per query | Implementation of cost controls |
| Latency budget per stage | How to meet them |
| Quality/cost trade-offs at the feature level | Quality/cost trade-offs at the prompt level |
| Whether to cache (privacy implications) | Cache implementation |
| Whether to A/B model choices | A/B mechanics |

---

## Stakeholder talking points

When finance asks **"why is each query $0.02?"**:

> "Three model calls per query: a cheap router, a cheap research model, a premium synthesizer. The synthesizer is 65% of the cost — we use it because helpfulness drops 0.4 without it. Here's the full breakdown by stage."

When eng asks **"can we use a smaller model on the synthesizer?"**:

> "Tried it — helpfulness drops from 4.3 to 3.7. Material regression. We could route by query complexity: simple queries to mini, hard queries to 4o. Let's eval the routing logic before deciding."

When user research surfaces **"the agent feels slow"**:

> "p95 is 3.6s — within target. But time-to-first-token is 1.5s; that's what users feel as 'slow.' We can drop TTFT to 400ms with streaming + a 'thinking' indicator. Want me to prioritize?"

When finance asks **"can we negotiate provider pricing?"**:

> "At our spend level, no. At GA scale ($150K/year), maybe — usually 10-15% off enterprise tier. The bigger lever is the cost levers themselves: prompt caching saved us 40% on the system prompt portion."

---

## The cost / latency / quality triangle

You usually can't optimize all three. Decide which to fix and which to flex.

| Product type | Pick |
|--------------|------|
| Latency-sensitive (chat, voice) | Fix latency, flex cost slightly |
| High-stakes single answers (legal, medical) | Fix quality, flex cost AND latency |
| Bulk processing (summaries, batched eval) | Fix cost, flex latency |

For retail shopping copilot: latency-sensitive. For a merchandiser research tool: bulk.

---

## Risk lenses

| Risk | Smell | Mitigation |
|------|-------|------------|
| **Cost spike** | 3× projected; budget alarm fires | Daily spend cap with auto-shutoff; per-stage cost limits |
| **Latency regression** | p95 creeping over weeks | Per-stage latency budgets monitored; alarm on drift |
| **Caching stale data** | Cached embeddings on a refreshed catalog give wrong answers | TTL + invalidation on source change |
| **Provider price hike** | Sudden 50% increase | Multi-provider routing pre-built; runbook to switch in < 1 hour |
| **Quality regression from cost cuts** | Eval metrics drop after a model swap | Tie cost changes to eval re-run in CI |

---

## The cost budget table — a PM artifact

For every PM-owned LLM feature, build this:

| Stage | Tokens (in/out) | Calls | Cost / query | % of total |
|-------|------------------|-------|--------------|------------|
| Intent classifier | 200 / 50 | 1 | $0.0001 | 0.5% |
| Query rewriter | 250 / 80 | 1 | $0.0001 | 0.5% |
| Retrieval embed | 30 | 1 | $0.00001 | <0.1% |
| Retrieval judge | 1500 / 50 | 1 | $0.0003 | 1.5% |
| Final answer | 4000 / 300 | 1 | $0.013 | 65% |
| Output guard | 400 / 50 | 0-1 | $0.0001 | 0.5% |

This table is the conversation. When eng proposes an optimization, you can immediately see "is this attacking the right column?"

---

## The latency budget table — also a PM artifact

| Stage | Budget (p95) | Today | Status |
|-------|--------------|-------|--------|
| Network ingress | 50ms | 40ms | ✅ |
| Input guardrail | 5ms | 3ms | ✅ |
| Intent classifier | 600ms | 580ms | ✅ |
| Retrieval | 200ms | 180ms | ✅ |
| Query rewrite (parallel) | 0ms | 0ms | ✅ |
| Retrieval judge | 600ms | 720ms | ⚠️ |
| Final answer (streamed) | 1800ms | 1850ms | ⚠️ |
| Output guard | 5ms | 4ms | ✅ |
| **Total p95** | **3300ms** | **3640ms** | ⚠️ |

Yellow flags tell engineering exactly where to attack first.

---

## Reading order in the technical module

Worth reading: §1 (cost model), §2 (seven levers), §3 (eight latency levers), §6 (triangle).

Skim: code examples.

The two budget tables (§2 / §4) are PM-co-authored artifacts.

---

## Worked example: a "make it cheaper" mandate

CFO asks: *"Can we cut copilot cost 30%?"*

PM playbook:

1. **Pull the cost table.** Identify the column eating most cost (synthesizer at 65%).
2. **Eval the obvious lever.** Test gpt-4o-mini on the synthesizer. Measure helpfulness drop.
3. **If quality holds**: ship the swap. ~50% cost reduction.
4. **If quality drops**: try routing — mini for simple queries, 4o for hard. Measure.
5. **Worst case**: enable prompt caching on the long system prompt. ~25% cost reduction without quality risk.
6. **Frame to CFO**: "Here are three options at 25%, 40%, and 55% cost reduction with these quality trade-offs."

That's a 2-week eval cycle, not a 1-week dev cycle. Set expectations accordingly.

---

## After this module

Read [PM Addon 11 — Deployment](pm-addon-11-deployment.md). Cost and latency optimization assume you're shipping; deployment is how you actually do that.
