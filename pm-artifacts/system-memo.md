# System Memo — Retail Copilot v1

**Owner**: J. Park (PM, Discovery)
**Engineering lead**: A. Reyes
**Status**: Approved 2026-05-12
**Companion to**: [Business case](business-case.md), [RFC: Agentic Search v1](rfc-agentic-search.md)

> 💡 **Why this exists**
> The business case justifies *why*. The system memo locks down *what* and *what not*. It is the one-page contract between PM, engineering, design, and CX before any code is written.

---

## 1. Target user

**Primary**: returning shoppers on northcrest.com who arrive with intent but not a specific SKU. They know they want "a warm jacket for layering" — they don't know that's a midweight insulated piece in our taxonomy.

**Secondary**: first-time shoppers who'd otherwise bounce because faceted filters intimidate them.

**Explicitly out of scope for v1**: bulk wholesale buyers, B2B accounts, internal merchandisers.

---

## 2. Job to be done

> When I'm shopping for outdoor gear with fuzzy constraints,
> I want to describe what I need in my own words,
> so I can find a product that fits without learning your filter taxonomy.

The job is **translation**: shopper's mental model → our catalog model.

---

## 3. Current non-AI baseline

- Faceted search returns 60–250 SKUs per query; shopper hand-filters.
- Static FAQ page (read-only).
- Email + chat support, 24h SLA on email, ~5min on chat during business hours.

Conversion on copilot-target queries today: **6.4%** (vs 11% site average).

---

## 4. Where AI adds value

| Capability | Mechanism |
|------------|-----------|
| Interpret fuzzy intent | LLM extracts structured constraints (price, attributes, size, use case) |
| Synthesize across catalog + policy + reviews | Agentic RAG with citations |
| Personalize within session | Memory of recent turns: size, gender, location |
| Deflect simple policy questions | RAG over 5 policy docs + FAQ |
| Triage order lookups for logged-in users | Tool call to internal Order API |

---

## 5. Where AI is explicitly out of bounds

| Action | Why excluded |
|--------|--------------|
| Final pricing or discount codes | Must come from deterministic system of record (Promotions service) |
| Promising refunds, credits, shipping upgrades | Authority sits with CX; legal/compliance review pending |
| Modifying inventory or customer records | Read-only in v1 |
| Speaking non-English | Localization defer to v2 |
| Image search ("find me one like this photo") | Defer to v2 |
| Replacing chat support | We deflect; we never refuse to escalate |

> ⚠️ **For PMs**: this section is the most disputed and the most valuable. Every "wouldn't it be cool if…" suggestion lives or dies here. Write it before someone asks.

---

## 6. Constraints

| Constraint | Target | Why this number |
|------------|--------|-----------------|
| First-token latency | < 1.5s p95 | Below the threshold where users perceive "waiting" |
| Full-response latency | < 4s p95 | Streaming masks the long tail |
| Cost per query | < $0.03 | Holds gross margin on incremental conversion |
| Hallucination rate (invented SKU/price/policy) | < 1% | Below threshold for brand risk in pilot |
| Safety pass rate (adversarial set) | > 95% | Pilot launch gate |
| English-only | 100% | Spanish/French in v2 |
| Browser support | Chrome/Safari/Firefox latest 2 versions | Aligns with current site support matrix |

---

## 7. Success metrics

### Leading indicators (weekly)
- Copilot engagement rate (% of sessions that interact)
- Median turns per session
- Thumbs-up rate
- Eval suite pass rate

### Lagging indicators (4-week rolling)
- Conversion lift vs control
- Add-to-cart rate on copilot-recommended SKUs
- Support ticket deflection (policy/returns category)

### Counter-metrics (we monitor these don't degrade)
- p95 latency
- Cost per query
- Rate of "I don't have that information" responses
- Customer satisfaction (CSAT) on support tickets (no decay vs baseline)

---

## 8. Open questions (tracked, not blocking)

| Question | Owner | Resolution by |
|----------|-------|---------------|
| Personalization scope: use customer history or session-only? | Privacy review | Week 2 |
| Should the copilot mention competitors when asked? | Brand + Legal | Week 3 |
| Show citations to shoppers or only in logs? | Design + Trust research | Week 4 |
| Failover behavior when LLM provider is degraded | Eng + Ops | Week 4 |

---

## 9. Approvals

| Role | Name | Approved on |
|------|------|-------------|
| VP Product | M. Liu | 2026-05-11 |
| Engineering Director | S. Patel | 2026-05-12 |
| Head of CX | R. Diaz | 2026-05-12 |
| Legal | T. Okonkwo | 2026-05-12 (with note on §5 PII handling) |

---

## How this gets out of date

- Section 5 (out of bounds) expands once we've earned trust — review after pilot.
- Section 6 (constraints) re-baselines when GA traffic exceeds 10× pilot.
- Section 7 (metrics) gains new counter-metrics as we discover failure modes in the wild.
