# PM Addon — Weekly Module 4: Agentic RAG

> **Pairs with**: [Weekly Module 4](../course/06-module-4-agentic-rag.md)
> **Read time**: 12 minutes
> **You'll be able to**: judge a RAG design, frame the cost/quality trade-offs in retrieval, and write a RAG design doc.

---

## The 60-second summary

RAG = Retrieval-Augmented Generation. Before the model answers, you find relevant chunks and put them in the prompt. The model answers grounded in those chunks instead of from training memory.

Four stages:

1. **Indexing** — chunk docs, embed each chunk, store in a vector DB.
2. **Retrieval** — embed the user query, find similar chunks.
3. **(Optional) Reranking** — second-stage scoring to surface the most relevant.
4. **Generation** — prompt the model with chunks + query.

**Agentic RAG** adds a loop: the agent judges whether retrieval was sufficient. If not, rewrite the query and try again.

PM artifact: [RAG design doc](../pm-artifacts/rag-design.md).

---

## Why this matters for the product

Bad RAG is the silent killer of agent quality. Symptoms PMs see:

- "Why doesn't the agent know about our X policy?" (chunked badly)
- "It cited the wrong product page." (similar but wrong)
- "It says 'I don't have that info' when we definitely do." (recall miss)

The fix is rarely "use a smarter model." It's almost always: better chunking, better embedding, smarter retrieval. RAG is an engineering discipline more than an ML discipline.

---

## What PMs decide vs what engineers decide

| PM owns | Engineering owns |
|---------|------------------|
| Which sources are indexed | Chunking algorithm per source |
| Freshness SLA | Index update mechanism |
| Acceptance criteria (Recall@k targets) | Embedding model choice |
| Trade-off between cost and quality | k value, threshold tuning |
| Whether to add agentic behaviors | Rewriter / judge implementation |

---

## Stakeholder talking points

When eng proposes **"let's use a bigger embedding model"**:

> "Show me the eval delta. Bigger embeddings cost more, take more storage, and rarely beat small embeddings on small corpora. If the lift is < 0.05 on Recall@5, we keep what we have."

When CX asks **"can the agent answer questions about [new topic]"**:

> "If we index the source, yes. Let's check the data card and see what's in scope. Adding a new source is a 1-2 day engineering task plus a few new eval cases."

When PM peer asks **"is RAG enough, or do we need fine-tuning?"**:

> "RAG first, every time. Fine-tuning is 10× the effort and only helps when you need the model to adopt a style or a structured task the base model can't do. We haven't hit that ceiling."

When finance asks **"why is RAG more expensive than just asking the model?"**:

> "RAG sends more tokens (the retrieved chunks). That's typically 60-80% of per-query cost. The trade is quality: groundedness rises from ~70% to ~92%, hallucination drops from ~5% to under 1%."

---

## The five RAG quality levers

| Lever | What it does | When to pull |
|-------|--------------|--------------|
| Chunking strategy | How docs are split | When eval shows the right doc retrieved but answer wrong |
| Embedding model | Quality of similarity match | When eval shows recall low across many cases |
| k (top-k retrieved) | Breadth of context | When eval shows answer was outside top-k |
| Reranking | Sharper second-stage scoring | After k tuning is exhausted |
| Agentic behaviors (rewriting, reflection) | Smarter retrieval loop | When recall is good but precision low |

Pull in this order. Each lever costs more than the last; later levers also add latency.

---

## Agentic RAG — when it's worth the cost

| Trigger | Agentic RAG helps? |
|---------|--------------------|
| Short, vague queries ("got a thing") | Yes (rewriter expands) |
| Multi-part questions ("returns AND shipping") | Yes (rewriter splits) |
| Domain slang vs catalog vocabulary | Yes (rewriter normalizes) |
| Already-specific queries | No (overhead, no lift) |
| Single-tool path (order ID extraction) | No (skip RAG entirely) |

In production, route queries: agentic RAG for ambiguous, direct for specific. The course's intent classifier (Module 1) is the routing layer.

---

## Risk lenses

| Risk | Smell | Mitigation |
|------|-------|------------|
| **Stale embeddings** | Catalog updated; agent recommends discontinued items | Nightly delta re-index; freshness alarms |
| **Wrong-chunk citation** | Agent cites doc A but the claim is from doc B | Citation precision metric + LLM-judge check |
| **Recall floor too low** | "I don't have that info" rate climbs | Track no-answer rate weekly; investigate top 20 |
| **Index size sprawl** | Adding every source "just in case" | Source allowlist with eval-driven justification |
| **Latency creep from agentic behavior** | p95 drift over weeks as queries get more complex | Per-stage latency budgets, monitored |

---

## RAG design doc — what's in it

The [RAG design doc](../pm-artifacts/rag-design.md) covers:

| Section | Why it's there |
|---------|----------------|
| Corpus | What's in vs what's deferred |
| Chunking strategy per source | Engineering choices with rationale |
| Embedding | Model, dimensions, distance metric |
| Vector index | Engine choice (Chroma → Qdrant) |
| Retrieval | k, filters, thresholds |
| Agentic behaviors | Rewriter, judge, reflection — which ones, when |
| Generation prompt | The grounding constraints |
| Numbers | Eval results today vs targets |
| Open questions | What's TBD |

PMs co-author the corpus and numbers sections. Engineers own the rest, but PMs sign off.

---

## Reading order in the technical module

Worth reading: §4.1 (RAG, plainly), §4.2 (why agentic RAG), §4.4 (chunking — the most underrated lever), §4.10 (RAG design assignment).

Skim: §4.3/§4.5/§4.6 (lab code).

Skip: TS implementations unless you co-own.

---

## Worked example: scoping a "knowledge expansion" ask

CX leadership asks: *"Can the agent learn from our most recent training docs?"*

PM playbook:

1. **Inventory the docs.** Format (markdown? PDF? Confluence?), count, update cadence, sensitivity.
2. **Score with the data-source framework** (PM Addon 02).
3. **Estimate chunking work.** PDFs need parsing; markdown is easy. Roughly 1 day per source type.
4. **Estimate eval impact.** ≥ 5 new eval cases per new source.
5. **Pilot plan.** Index 10% of the corpus, measure Recall@5 on the new eval cases, then commit.

That's a 1-week investigation, not a 1-week build. Important to frame correctly.

---

## After this module

Move to [PM Addon 07 — Multi-Agent & MCP](pm-addon-07-multiagent.md). Multi-agent is where coordination tax becomes real.
