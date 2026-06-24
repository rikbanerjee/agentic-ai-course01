# PM Addon — Module 2: Domain & Data Preparation

> **Pairs with**: [Module 2 — Domain & Data Preparation](../course/02-domain-and-data.md)
> **Read time**: 10 minutes
> **You'll be able to**: scope data work, recognize data risks, and write a data card for your domain.

---

## The 60-second summary

The agent's quality is bounded by the data it can search. The five sources that matter for any product copilot:

1. **Catalog / product data** — your structured truth.
2. **Policy documents** — returns, shipping, warranty, etc.
3. **FAQs** — small, dense, high-signal.
4. **Reviews / user-generated content** — high noise; index with care.
5. **Transactional data** — orders, accounts; access via tools, not RAG.

The decisions you make here ripple through every later module. Bad chunking causes retrieval misses. Stale data causes hallucinations. Missing fields cause hedged answers.

---

## Why this matters for the product

Common failure pattern: a stakeholder says *"why didn't the agent know about our XYZ policy?"* — and the answer is *"because that policy isn't in the corpus."* This module is how you avoid that conversation.

Three product-level decisions you make here:

1. **Which sources to index for v1.** More is not better; noisy sources poison retrieval.
2. **What freshness contract you can offer.** "Updated nightly" vs "updated within an hour" is an engineering investment.
3. **What "I don't know" looks like.** When the data is missing, the agent should say so — not invent.

---

## What PMs decide vs what engineers decide

| PM owns | Engineering owns |
|---------|------------------|
| Which data sources are in scope | Chunking strategy per source |
| Data freshness SLA | Index update mechanism |
| Sensitive-data exclusion rules | PII detection regex set |
| The data card (sources, gaps, edge cases) | Embedding model choice |
| Whether reviews/UGC are indexed | How (when yes) |

---

## Stakeholder talking points

When merchandising asks **"why isn't the agent using customer reviews?"**:

> "Reviews are high-value but high-noise. We're holding off until we can index with quality filters (rating, recency, verified buyers) — otherwise we'd surface 2-star complaints from 2 years ago as if they're current. Planned for v1.1."

When legal asks **"what happens if a policy document changes?"**:

> "The index updates nightly. Until it updates, the agent might quote the old version. We're working on hourly updates for high-change docs; for now, every policy chunk includes a 'last updated' note the agent surfaces."

When CX asks **"why doesn't the agent know about [edge case]?"**:

> "Probably one of three reasons: the data isn't indexed, the chunking split it badly, or retrieval missed it. We log every retrieval; let me check the trace and add a test case if it's a real gap."

---

## Risk lenses

| Risk | Smell | Mitigation |
|------|-------|------------|
| **Stale data → wrong answers** | Policy changed yesterday, agent still quoting old version | Freshness SLA in the data card; "as of" caveats in prompts |
| **Sensitive data leaks** | PII in chunked corpus | Pre-index scrubber; review chunks before bulk index |
| **Poisoned corpus** | UGC indexed without quality filters | Source allowlist; minimum-rating filters |
| **Coverage holes** | Common questions consistently get "I don't know" | Track no-answer rate as a metric; investigate top 20 |
| **Chunk boundary problems** | Answer exists but agent can't retrieve it | Re-chunk with overlap; add as eval case |

---

## The data-source decision framework

For each candidate source, score:

| Dimension | High | Medium | Low |
|-----------|------|--------|-----|
| Signal density | Each chunk has answers | Mixed | Mostly noise |
| Volume manageability | < 10K chunks | 10K–100K | > 100K |
| Update frequency | Weekly+ | Monthly | Quarterly+ |
| Sensitivity (PII / legal) | None | Some | Lots |

Default rule: **High signal + Low sensitivity → index now**. Everything else needs a plan.

---

## Data card — the single most valuable doc

A "data card" is a 1-page reference for every source: what's in it, what's known to be missing, what the gotchas are. Engineers need it to chunk well; PMs need it to set expectations.

Template:

```markdown
## Source: [name]
- Size: [N records / chunks]
- Update cadence: [daily / weekly / on-edit]
- Fields: [field list]
- Known gaps: [list]
- Known errors: [list]
- Sensitive content: [yes/no, what]
- Eval expectations: [should agent say "I don't know" when X?]
```

The course's [data card example](../course/02-domain-and-data.md#26-the-data-card--your-reference-doc) is a working version.

---

## Freshness vs cost trade-off

Re-embedding the entire corpus is cheap (< $5 for the course's dataset). Re-embedding nightly is fine. Re-embedding *hourly* requires a delta-detection pipeline — that's days of engineering work. Decide based on:

- How often does the source change?
- What's the cost to the user of seeing stale data?
- Is there a "as of [date]" UX you can adopt to soften it?

For policies: nightly is usually enough. For inventory/stock: real-time via tool calls, not via RAG.

---

## Reading order in the technical module

Pages worth reading in full: §2.1 (synthetic data rationale), §2.5 (why it matters), §2.6 (data card).

Skim: §2.3 (the generation script).

Skip: TypeScript port of the dataset builder.

The deliverable — `docs/data_card.md` — is something PMs absolutely should personally draft.

---

## Worked example: scoping a real catalog integration

You're integrating your real product catalog (not the synthetic one). Apply:

1. **Source inventory**: products.csv (50K SKUs), policies/*.md (15 docs), faqs.json (80 entries), reviews (1.2M, deferred).
2. **Chunking plan**: products → one chunk per SKU; policies → 600-char windows; FAQs → one chunk per Q&A.
3. **Update cadence**: products nightly delta; policies on-edit; FAQs weekly.
4. **Sensitive exclusion**: no internal SKU cost, no margin data.
5. **Eval impact**: write 5 eval cases per source covering common questions + 2 edge cases per source.

That five-step list becomes the data section of your [RAG design doc](../pm-artifacts/rag-design.md).

---

## After this module

Move to [PM Addon 03 — GenAI & Agentic Apps](pm-addon-03-genai.md). With data sorted, you start building the agent itself.
