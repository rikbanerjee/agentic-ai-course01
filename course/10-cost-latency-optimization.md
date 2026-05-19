# Cost & Latency Optimization

> **When to read this**: after Module 4, then again before the capstone goes "live."
> **Goal**: Know the levers that cut cost 2–5× and p95 latency 30–60% without quality loss.

The cost and latency budgets in your system memo aren't decoration — they're real constraints that will pressure your design. This track is the toolkit.

---

## 1. The cost model: where the money actually goes

For a typical agentic system, an `ask()` call looks like:

| Step | Tokens (approx) | Calls | % of cost |
|------|-----------------|-------|-----------|
| Intent classification | 200 in / 50 out | 1 | 5% |
| Query rewrite | 250 in / 80 out | 1 | 7% |
| Retrieval (embed query) | 30 tokens | 1 | <1% |
| Retrieval judge | 1500 in / 50 out | 1 | 12% |
| Final answer generation | 4000 in / 300 out | 1 | 65% |
| Output guardrail (if LLM-based) | 400 in / 50 out | 1 | 5% |
| Tool calls (search, get_order) | n/a | 1-3 | $0 |

> 💡 **Insight**
> The final-answer call is almost always >50% of cost because it's the only one that sees the full retrieval context. Cut that context smartly and you cut total cost a lot.

---

## 2. The seven cost levers

### Lever 1 — Right-size the model per agent

Don't run every agent on your best model. A typical good mix:

| Role | Model |
|------|-------|
| Intent classifier | gpt-4o-mini or gemini-2.0-flash |
| Query rewriter | gpt-4o-mini |
| Retrieval judge | gpt-4o-mini |
| Final answer | gpt-4o (or claude-sonnet) |
| Output guardrail | gpt-4o-mini |

Rule of thumb: cheap+fast for routing/classification, premium for user-facing prose.

### Lever 2 — Prompt caching

OpenAI and Anthropic both support caching the *prefix* of a prompt. If your system prompt is 2000 tokens and never changes, after the first call it costs ~10% of normal.

```python
# OpenAI: caching is automatic for prompts > 1024 tokens, prefix unchanged
# Anthropic: explicit cache_control on a content block
from anthropic import Anthropic
client = Anthropic()
res = client.messages.create(
    model="claude-sonnet-4-6",
    max_tokens=1024,
    system=[{"type":"text","text": LONG_SYSTEM_PROMPT, "cache_control":{"type":"ephemeral"}}],
    messages=[{"role":"user","content": query}],
)
```

Savings: 50–90% on the cached portion. Set up your prompts so the static parts are at the front.

### Lever 3 — Smaller retrieval context

Retrieved chunks dominate the prompt size. Reduce by:

- Rerank to top-3 instead of top-5 (~40% fewer context tokens).
- Strip metadata from chunks before sending (often half the bytes are JSON noise).
- Summarize chunks > 600 chars before injection.

### Lever 4 — Cache embeddings and rewrites

Embeddings of the same query hash → cache. Rewrites of recurring query patterns → cache.

```python
import hashlib, json
from functools import lru_cache

@lru_cache(maxsize=10_000)
def cached_embed(query: str) -> tuple:  # tuples because lru_cache needs hashable
    return tuple(openai.embeddings.create(model="text-embedding-3-small", input=query).data[0].embedding)
```

For persistent caching, use Redis or SQLite.

### Lever 5 — Skip the LLM when you can

Many "agent" steps don't need an LLM:

- Regex for order ID extraction.
- Keyword lookup for the top 50 common questions (FAQ shortcut).
- Deterministic routing based on the URL/intent the user clicked.

```python
import re
ORDER_RE = re.compile(r"\bORD-\d{4,}\b")
def quick_order_path(query: str) -> str | None:
    m = ORDER_RE.search(query)
    return m.group(0) if m else None
```

If `quick_order_path` returns a hit, skip the intent classifier entirely.

### Lever 6 — Batch when possible

For evals, summarization, or background jobs, batch many requests into one. OpenAI's batch API is 50% off, completes in <24h.

### Lever 7 — Truncate aggressively when over budget

Set a hard token ceiling per request. If your assembled prompt exceeds it, drop the lowest-ranked chunks first, then summarize history, then return a politely degraded answer.

---

## 3. The eight latency levers

### Lever 1 — Stream

Streaming doesn't reduce time-to-completion, but it cuts **time-to-first-token** dramatically. Users perceive a streaming 4-second response as faster than a non-streaming 2-second one.

```python
from openai import OpenAI
client = OpenAI()
stream = client.chat.completions.create(model="gpt-4o", messages=[...], stream=True)
for chunk in stream:
    delta = chunk.choices[0].delta.content or ""
    print(delta, end="", flush=True)
```

```typescript
const stream = await client.chat.completions.create({ model: 'gpt-4o', messages, stream: true });
for await (const chunk of stream) process.stdout.write(chunk.choices[0].delta.content ?? '');
```

### Lever 2 — Parallelize independent calls

If you need to call `search_products` AND `get_policy`, do them in parallel.

```python
import asyncio
results = await asyncio.gather(search_products(...), get_policy(...))
```

```typescript
const [products, policy] = await Promise.all([searchProducts(...), getPolicy(...)]);
```

The router → research handoff *is* sequential, but research can fan out internally.

### Lever 3 — Skip steps when confident

The query rewriter adds ~400ms. Skip it for queries longer than 8 words (likely already specific). Skip the retrieval judge if Recall@1 score from the vector search > 0.85.

### Lever 4 — Smaller models on the hot path

A `gpt-4o-mini` call at 600ms vs `gpt-4o` at 1500ms can compound to 2-3s saved across a multi-agent flow. Use big models only where quality demands.

### Lever 5 — Avoid unnecessary roundtrips

LangGraph and modern OpenAI SDKs can return multiple tool calls in one response. Use it. Sequential `[tool call] → [completion] → [tool call] → [completion]` is 2× the latency of parallel tool calls.

### Lever 6 — Warm the path

Pre-load models, embeddings, and DB connections at startup, not per-request:

```python
# in app startup
from sentence_transformers import CrossEncoder
RERANKER = CrossEncoder("BAAI/bge-reranker-base")  # 800ms to load; do once
```

### Lever 7 — Edge deploy hot paths

For TS apps, deploy the intent classifier and refusal handler at the edge (Vercel Edge, Cloudflare Workers). The user gets ~50ms TTFB before the heavy lifting begins.

### Lever 8 — Speculative execution

While the router classifies, start the most likely retrieval in parallel. If the classification ends up agreeing, you've shaved ~700ms. If not, you've wasted a cheap call.

---

## 4. The latency budget table

For the V2 retail copilot:

| Stage | Budget (p95) |
|-------|--------------|
| Network ingress | 50 ms |
| Input guardrail | 5 ms |
| Intent classifier (mini) | 600 ms |
| Retrieval (vector search) | 200 ms |
| Query rewrite (parallel with retrieval) | 0 (overlapped) |
| Retrieval judge | 600 ms |
| Final answer (4o, streamed) | 1800 ms (TTFB ~400 ms) |
| Output guardrail | 5 ms |
| Network egress | 50 ms |
| **Total p95** | **~3300 ms** |

Build a profile flame graph (Langfuse trace spans give you this for free). Anywhere a span exceeds budget, attack with the levers above.

---

## 5. Lab — Measure and improve

Add latency tracking per span:

```python
import time
from contextlib import contextmanager

@contextmanager
def span(name: str, trace: dict):
    t0 = time.perf_counter()
    yield
    trace[name] = int((time.perf_counter() - t0) * 1000)
```

```python
trace: dict[str, int] = {}
with span("intent", trace):
    intent = classify_intent(q)
with span("retrieval", trace):
    hits = retrieve(q)
print(trace)
# {"intent": 612, "retrieval": 187}
```

Run 50 queries, aggregate, pick the top span, attack with a lever, re-measure. Repeat.

---

## 6. The cost / latency / quality triangle

You usually can't optimize all three. Decide which to fix and which to flex:

- **Latency-sensitive product** (chat, voice): pick cheap + fast models; tolerate slightly lower quality; cache aggressively.
- **High-stakes single answers** (legal, medical): pick best model; tolerate cost + latency; require human review.
- **Bulk processing** (analytics, summarization): batch; tolerate latency; minimize cost per item.

For retail shopping: latency-sensitive. For merchandising/research copilots: bulk.

---

## 7. Checklist

- [ ] You know your p50/p95/p99 latency per stage.
- [ ] You know your cost per query and what % each stage contributes.
- [ ] Streaming is enabled for user-facing output.
- [ ] Prompt caching is on where supported.
- [ ] At least one "skip the LLM" shortcut exists for a common pattern.
- [ ] You've tried a smaller model on at least one agent and measured the quality drop.

---

## Common anti-patterns

- **One model for everything.** Always evaluate cheaper models on non-headline agents.
- **No streaming.** Users hate a blank screen for 3 seconds.
- **No caching.** Embedding the same FAQ query 10,000 times costs 100× more than it should.
- **Optimizing without measuring.** Don't tune what you haven't profiled.
- **Caching forever.** Stale embeddings on a refreshed catalog give very wrong answers.
