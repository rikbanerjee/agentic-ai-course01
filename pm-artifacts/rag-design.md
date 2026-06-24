# RAG Design — Retail Copilot

**Owner**: A. Reyes (Eng lead)
**Status**: Active v1; v1.1 reranker work in progress
**Companion to**: [RFC](rfc-agentic-search.md), [Context policy](context-policy.md)

> 💡 **Why this exists**
> Retrieval is where most agentic systems silently fail. This doc nails down chunking, embedding, index, retrieval, and reranking decisions — each with the eval evidence that backed the choice.

---

## 1. Corpus

| Source | Count | Avg size | Update cadence |
|--------|-------|----------|----------------|
| Products | 200 | ~280 chars/desc | Nightly delta |
| Policies | 5 docs | ~600 chars | Quarterly |
| FAQs | 10 entries | ~80 chars | Monthly |
| Reviews | ~1500 | ~80 chars | NOT indexed v1 |

Reviews intentionally excluded from v1 because:
- High noise (mixed quality, formatting).
- Risk of surfacing dated complaints as if current.
- v1.1: index reviews with rating + recency filters.

---

## 2. Chunking strategy

| Source | Strategy | Why |
|--------|----------|-----|
| Products | One chunk per record | Preserves SKU↔name↔price↔stock linkage; we don't want the model to see "$129" without the SKU |
| Policies | 600-char windows, 80-char overlap | Most policies < 1200 chars; overlap catches multi-paragraph rules |
| FAQs | One Q&A per chunk | Each is a unit of meaning |

### Why not header-aware splitting?

We tried `MarkdownHeaderTextSplitter` on policies in week 3. Result: 2 of 5 policies have headings only at the top; the rest are flat prose. Header splitting created lopsided chunks. Fixed-window was better on Recall@5 (0.86 vs 0.82) and simpler.

### Why no sentence-window splitting?

Tested. For policy text it cost 30% more chunks for no meaningful Recall@5 gain (+0.01). Deferred.

---

## 3. Embedding

- Model: **text-embedding-3-small** (1536d).
- Reason: $0.02/1M tokens; eval-equivalent to text-embedding-3-large on our corpus (Recall@5 within 0.01).
- Distance metric: cosine (Chroma default).

### Why not local embeddings (sentence-transformers)?

Tested `all-MiniLM-L6-v2`. Recall@5 dropped 0.08 vs OpenAI. Local model is faster per call but the quality trade wasn't worth it for our small corpus.

### Re-embedding policy

- On chunk text change: re-embed that chunk only.
- On embedding-model change: rebuild full collection (one-time, ~5min for 280 chunks).
- Nightly delta job watches `data/raw/products.csv` for diffs.

---

## 4. Vector index

| Choice | v1 (pilot) | GA |
|--------|-----------|-----|
| Engine | Chroma (embedded) | Qdrant Cloud |
| Persistence | Local disk in container | Managed |
| Backup | Daily snapshot to S3 | Built-in |
| Why the switch | Chroma is great for dev; Qdrant is operationally easier at scale | |

We chose Chroma for v1 because we re-build the index on every release; setup is one Python install. Qdrant becomes worth it when we want multi-region replication, RBAC on collections, and managed snapshots.

---

## 5. Retrieval

### Query path

1. Embed the (possibly rewritten) query with the same model used for indexing.
2. `top_k=5` ANN search.
3. Apply metadata filters (in-stock for product_search; source allowlist).
4. Drop chunks above distance threshold 0.55.
5. (v1.1) Optional cross-encoder rerank to top-3.

### Why k=5

Tested k ∈ {3, 5, 10}:

| k | Recall@k | Median tokens in context | Cost / answer |
|---|----------|---------------------------|---------------|
| 3 | 0.78 | 1.1K | $0.012 |
| 5 | 0.86 | 1.9K | $0.018 |
| 10 | 0.88 | 3.7K | $0.031 |

k=5 was the knee of the curve. k=10 doubled cost for 0.02 gain.

### Why distance threshold 0.55

Histogram of distances on the eval set showed bimodal distribution: real matches clustered < 0.45; noise > 0.6. 0.55 captures real matches with a tolerance band and rejects most noise.

---

## 6. Agentic behaviors

### Query rewriting

When triggered: always for short queries (< 6 tokens) and for queries the router flags as ambiguous (confidence < 0.7).

Why: vague queries like "is there a thing for this" don't embed near useful chunks. Rewriting produces 1-3 specific queries; we union the retrievals.

Cost: +1 LLM call (~400ms, ~$0.0004). Benefit: Recall@5 lifts from 0.68 → 0.84 on the ambiguous slice.

### Retrieval judge

A small LLM call after retrieval: "Do these chunks answer the query?" Returns sufficient + confidence + missing.

If sufficient & confidence ≥ 0.7: proceed to generation.
Else: re-run rewriter with the `missing` field as a hint; union with prior hits; max 2 attempts.

Cost: +1 LLM call (~600ms, ~$0.0006). Benefit: catches ~10% of cases where rewriter alone wasn't enough.

### Reflection (deferred to v1.1)

Have the generation model critique its own draft and retry if low confidence. In dev tests, this added ~1500ms and improved helpfulness 0.2 on the eval — not yet worth the latency cost.

---

## 7. Generation prompt (excerpt)

```
You are a retail copilot. Answer the shopper's question using ONLY the
provided context. Every factual claim must cite [chunk_id]. If the context
does not contain the answer, say "I don't have that information in the catalog."

NEVER invent SKUs, prices, stock counts, or policy wording.
NEVER promise refunds, credits, or shipping upgrades — escalate instead.

Format:
  Brief direct answer (1-2 sentences).
  Then a bulleted list of supporting facts with [chunk_id] citations.
```

The instruction "ONLY the provided context" + "never invent" is the most impactful pair of constraints we found. Removing either dropped groundedness 0.1+ on evals.

---

## 8. Numbers (current, 2026-05-20)

| Metric | v0 (baseline RAG) | v1 (agentic) | Target |
|--------|---------------|---------------|--------|
| Recall@5 | 0.62 | 0.86 | ≥ 0.85 |
| Groundedness | 0.71 | 0.92 | ≥ 0.90 |
| Citation precision | 0.71 | 0.88 | ≥ 0.85 |
| Avg cost / answer | $0.004 | $0.018 | ≤ $0.03 |
| p95 latency | 1.6s | 3.6s | ≤ 4.0s |

The agentic path costs 4× more and adds 2s — we considered this a fair price for the +0.21 helpfulness gain.

---

## 9. Open questions

| # | Question | Owner | Resolve by |
|---|----------|-------|------------|
| 1 | Should v1.1 include reranking with bge-base? | Eng | Pre-GA |
| 2 | Should we surface "I'm not sure" rather than "I don't have that"? | Design + PM | Pilot week 3 |
| 3 | Index reviews with what filters? | Eng + Merch | v1.1 |
| 4 | Switch to text-embedding-3-large for policies only? | Eng | After GA |

---

## How this gets out of date

- Recall numbers (§8) drift as the corpus grows. Re-run weekly.
- Cost numbers shift with model pricing. Re-baseline before any GA decision.
- Chunking choices (§2) need revisit if a new source type is added (reviews, blog posts).
