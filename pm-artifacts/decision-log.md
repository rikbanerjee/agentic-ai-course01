# Decision Log — Retail Copilot

A running record of every significant choice, the rationale behind it, and the experiment / data that backed it.

> 💡 **Why this exists**
> Six months from now, you (or a teammate) will look at a piece of code and ask "why is this written this way?" The decision log is the answer. It's also how new joiners get up to speed without scheduling six meetings.

---

## Conventions

- Newest entries on top.
- Each entry: date, decision, alternatives considered, evidence, owner, status.
- Reverse a decision by adding a new entry that supersedes the old one (don't delete history).
- Link to PRs, eval reports, traces wherever possible.

---

## 2026-06-15 — Index size charts into RAG corpus

**Decision**: Add product size charts as a new source in the RAG index for v1.1.

**Alternatives considered**:

- Pass charts as structured tool output (rejected: low recall on natural-language sizing questions).
- Keep them out (rejected: helpfulness on sizing queries stuck at 3.7).

**Evidence**:

- 14 cases in eval suite tagged "sizing"; current pass rate 0.42.
- Pilot users explicitly mention size questions in 11% of sessions; most current responses hedge ("refer to the size chart").

**Owner**: A. Reyes
**Status**: Approved, scheduled v1.1.

---

## 2026-06-08 — Multi-provider auto-failover

**Decision**: Implement automatic failover from primary (OpenAI) to secondary (Gemini) on health-check failures or sustained 429s.

**Alternatives considered**:

- Manual flip on PagerDuty page (current state) — rejected after week-4 incident showed 15-min MTTR.
- Always run both in parallel (rejected: doubles cost; no quality lift).

**Evidence**:

- 4-week pilot saw one 15-min Gemini degradation that spiked p95 to 6.2s.
- Health-check + Tenacity-style routing rolled out in dev added 12ms median overhead.

**Owner**: A. Reyes
**Status**: In code review.

---

## 2026-06-01 — Defer reranker to v1.1

**Decision**: Skip cross-encoder reranking for v1; revisit after GA at 50%.

**Alternatives considered**:

- Ship with `BAAI/bge-reranker-base` (rejected: +280ms p95, +0.04 Recall@5 not worth the latency).
- Skip permanently (no — corpus will grow; rerank becomes valuable).

**Evidence**:

- A/B in dev: reranker added 280ms p95, lifted helpfulness by 0.06 (not statistically significant on n=47 cases).
- Cost neutral.

**Owner**: A. Reyes
**Status**: Approved.

---

## 2026-05-28 — Switch Decision agent to gpt-4o

**Decision**: Replace gpt-4o-mini with gpt-4o for the Decision agent only.

**Alternatives considered**:

- Keep mini everywhere (rejected: prose felt flat in human spot-check; helpfulness stuck at 3.7).
- Use Claude Sonnet (deferred: provider not yet legally cleared).

**Evidence**:

- Side-by-side A/B (mini vs 4o on Decision only): helpfulness 3.7 → 4.2; cost rose from $0.011 to $0.018/query.
- Latency rose 600ms; still within budget.

**Owner**: J. Park + A. Reyes
**Status**: Live.

---

## 2026-05-20 — Add output regex guardrail

**Decision**: Add deterministic regex layer post-LLM to catch forbidden promise patterns.

**Alternatives considered**:

- LLM-based output classifier only (deferred to v1.1; needed telemetry to tune).
- Prompt instructions only (rejected: dogfood week showed leaks even with strict prompts).

**Evidence**:

- Dogfood week: 3/47 cases leaked refund/credit phrases despite system-prompt rules.
- Regex shipped: 0/47 leaks on next run; p95 latency added <5ms.

**Owner**: A. Reyes
**Status**: Live.

---

## 2026-05-15 — Pilot at 10%, not 5%, of .com traffic

**Decision**: Run pilot at 10% rather than the originally proposed 5%.

**Alternatives considered**:

- 5% (rejected: insufficient sample size for adversarial coverage).
- 25% (rejected: too much exposure pre-pilot).

**Evidence**:

- Power analysis: 5% would have required 5 weeks to reach significance; 10% reached significance in 2.

**Owner**: J. Park
**Status**: Approved (RFC §rollout).

---

## 2026-05-10 — Chroma for pilot, Qdrant for GA

**Decision**: Use embedded Chroma for v1 pilot; plan Qdrant Cloud migration for GA.

**Alternatives considered**:

- Pinecone (rejected: $$$ for our volume).
- pgvector on existing Postgres (rejected: ops team wanted to keep Postgres for transactional data only).
- Qdrant from day 1 (rejected: setup friction in pilot phase).

**Evidence**:

- Chroma: 5-min setup; 200ms p95 retrieval on 280 chunks.
- Qdrant Cloud quote: ~$2K/mo at projected GA scale; well within budget.

**Owner**: A. Reyes
**Status**: Pilot live on Chroma; Qdrant migration scheduled v1.1.

---

## 2026-05-05 — gpt-4o-mini for Router

**Decision**: Use gpt-4o-mini, not 4o, for the Router agent.

**Alternatives considered**:

- gpt-4o (rejected: 4× cost; intent classification is a small-model job).
- gemini-2.0-flash (kept as documented fallback).
- Local fine-tuned classifier (rejected: corpus too small to train, maintenance burden).

**Evidence**:

- 47-case eval with mini: intent accuracy 0.92.
- Same eval with 4o: 0.93 (within noise).
- Cost difference per Router call: $0.0004 vs $0.0024.

**Owner**: A. Reyes
**Status**: Live.

---

## 2026-05-04 — Course of action: build internally rather than buy

**Decision**: Build the copilot internally rather than license a third-party shopping assistant.

**Alternatives considered**:

- Vendor A (declined: per-query pricing 5× internal cost; can't own eval).
- Vendor B (declined: data residency unclear; locked into proprietary observability).

**Evidence**:

- Business case payback in 13 days post-GA vs 6 months for vendor option.
- Internal build owns the eval suite — strategic asset.

**Owner**: J. Park + VP P
**Status**: Approved; business case approved 2026-05-04.

---

## How this gets out of date

Decision logs don't get out of date — they accumulate. But they get hard to navigate above ~50 entries.

When that happens:

- Tag entries with categories (eg `[architecture]`, `[model-choice]`, `[guardrails]`).
- Add a per-category index at the top.
- Archive entries older than 18 months to `decision-log-archive.md`.
