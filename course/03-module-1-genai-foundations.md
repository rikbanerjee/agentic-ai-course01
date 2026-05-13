# Module 1 — GenAI & Agentic Apps (Week 1)

> **Time**: 10–12 hours
> **Goal**: Build a traceable GenAI micro-app for retail intent + first system memo.
> **Deliverable**: `docs/system_memo.md`, working CLI app, logs you can query.

This is the first "production-flavored" week. You'll write code that *would actually be put in front of users*, with the logging, error handling, and structured output discipline that real shipping demands.

---

## 1.1 The mental shift from software to GenAI software

Traditional software:

```
input → function → deterministic output → same input always returns same output
```

GenAI software:

```
input → LLM → probabilistic output → same input can return different outputs
```

This breaks five things you take for granted as a software engineer:

| Traditional assumption | What changes with LLMs |
|------------------------|------------------------|
| Unit tests are deterministic | You need **evals** that measure quality with tolerance |
| Bugs reproduce locally | A failure might happen 3% of the time and never on your machine |
| Performance is measurable in CPU/RAM | You measure **token cost, latency, quality** together |
| Changes are debuggable via stack traces | You debug by inspecting prompts, retrievals, and outputs |
| Output format is guaranteed by code | The model can return malformed JSON 1% of the time |

> 💡 **Why this matters**
> If you keep applying traditional-software intuitions, you'll over-engineer the deterministic parts and under-engineer the probabilistic parts. The discipline of this course is the inverse.

---

## 1.2 The five failure surfaces in retail GenAI

For your domain (retail), here are the failures you have to assume happen:

1. **Hallucination** — "We carry a Patagonia Nano Puff in lavender for $89." (We don't.)
2. **Retrieval failure** — "I don't have a returns policy for sale items." (You do; RAG missed the chunk.)
3. **Wrong tool / no tool** — User asks order status, agent answers from training knowledge instead of calling `get_order`.
4. **Policy violation** — "Sure, I'll give you a 50% refund." (Agent has no authority to promise that.)
5. **Format break** — Returns malformed JSON, breaks downstream UI.

Every lab in this module helps you detect or prevent one of these.

---

## 1.3 Lab 1.5 — A traceable intent classifier (production-style)

You built a tiny version in Module 1 setup. Now make it production-flavored: structured output via Pydantic/Zod, fallbacks, logging, error handling.

### Python — `python/src/agents/intent_v1.py`

```python
"""Production-style intent classifier with structured output, retries, and tracing."""
from __future__ import annotations
import json
import logging
import time
from typing import Literal
from pydantic import BaseModel, Field, ValidationError
from openai import OpenAI
from logging_setup import log_event, new_request_id

logger = logging.getLogger(__name__)

Intent = Literal["product_search", "order_status", "return_request", "policy_question", "other"]


class IntentResult(BaseModel):
    classification: Intent
    confidence: float = Field(ge=0, le=1)
    rationale: str
    suggested_tools: list[str] = Field(default_factory=list)


SYSTEM_PROMPT = """You are a retail intent classifier for an e-commerce copilot.

Given a shopper's query, return JSON with these fields:
  classification: one of [product_search, order_status, return_request, policy_question, other]
  confidence: float 0-1, your calibrated confidence
  rationale: one short sentence
  suggested_tools: list of tool names from this set:
    [search_products, get_product, get_order, get_policy, get_faq]

Rules:
- If the query is ambiguous, set confidence < 0.6 and classification = "other".
- Never invent tool names not in the allowed set.
- Output JSON only — no markdown, no prose."""


def classify_intent(query: str, model: str = "gpt-4o-mini", max_retries: int = 2) -> IntentResult:
    client = OpenAI()
    rid = new_request_id()
    log_event(logger, "intent.request", request_id=rid, query=query, model=model)

    last_error: str | None = None
    for attempt in range(max_retries + 1):
        t0 = time.perf_counter()
        try:
            response = client.chat.completions.create(
                model=model,
                messages=[
                    {"role": "system", "content": SYSTEM_PROMPT},
                    {"role": "user", "content": query},
                ],
                response_format={"type": "json_object"},
                temperature=0.1,
            )
            raw = response.choices[0].message.content or "{}"
            parsed = IntentResult.model_validate_json(raw)
            latency_ms = int((time.perf_counter() - t0) * 1000)
            log_event(
                logger,
                "intent.success",
                request_id=rid,
                latency_ms=latency_ms,
                classification=parsed.classification,
                confidence=parsed.confidence,
                prompt_tokens=response.usage.prompt_tokens,
                completion_tokens=response.usage.completion_tokens,
                attempt=attempt,
            )
            return parsed
        except (json.JSONDecodeError, ValidationError) as e:
            last_error = str(e)
            log_event(logger, "intent.parse_error", request_id=rid, attempt=attempt, error=last_error)
            continue

    # Fallback: low-confidence "other" so the downstream agent can ask a clarifying question
    log_event(logger, "intent.fallback", request_id=rid, last_error=last_error)
    return IntentResult(
        classification="other",
        confidence=0.0,
        rationale=f"Parse failure after {max_retries + 1} attempts: {last_error}",
        suggested_tools=[],
    )


if __name__ == "__main__":
    from logging_setup import setup_logging
    setup_logging()
    queries = [
        "Do you have waterproof hiking jackets in size M under $150?",
        "Where is my order #ORD-4456?",
        "What's the return policy for sale items?",
        "kjsdhfksjdh",
    ]
    for q in queries:
        result = classify_intent(q)
        print(f"\n> {q}\n  {result.model_dump()}")
```

### TypeScript — `typescript/src/agents/intentV1.ts`

```typescript
import OpenAI from 'openai';
import { z } from 'zod';
import { logger, newRequestId } from '../logging.js';

const IntentSchema = z.object({
  classification: z.enum([
    'product_search',
    'order_status',
    'return_request',
    'policy_question',
    'other',
  ]),
  confidence: z.number().min(0).max(1),
  rationale: z.string(),
  suggested_tools: z.array(z.string()).default([]),
});
export type IntentResult = z.infer<typeof IntentSchema>;

const SYSTEM_PROMPT = `You are a retail intent classifier for an e-commerce copilot.

Given a shopper's query, return JSON with these fields:
  classification: one of [product_search, order_status, return_request, policy_question, other]
  confidence: float 0-1, your calibrated confidence
  rationale: one short sentence
  suggested_tools: list of tool names from this set:
    [search_products, get_product, get_order, get_policy, get_faq]

Rules:
- If the query is ambiguous, set confidence < 0.6 and classification = "other".
- Never invent tool names not in the allowed set.
- Output JSON only — no markdown, no prose.`;

export async function classifyIntent(
  query: string,
  model = 'gpt-4o-mini',
  maxRetries = 2
): Promise<IntentResult> {
  const client = new OpenAI();
  const rid = newRequestId();
  logger.info({ rid, query, model }, 'intent.request');

  let lastError: string | null = null;
  for (let attempt = 0; attempt <= maxRetries; attempt++) {
    const t0 = performance.now();
    try {
      const res = await client.chat.completions.create({
        model,
        messages: [
          { role: 'system', content: SYSTEM_PROMPT },
          { role: 'user', content: query },
        ],
        response_format: { type: 'json_object' },
        temperature: 0.1,
      });
      const raw = res.choices[0].message.content ?? '{}';
      const parsed = IntentSchema.parse(JSON.parse(raw));
      logger.info(
        {
          rid,
          latencyMs: Math.round(performance.now() - t0),
          ...parsed,
          promptTokens: res.usage?.prompt_tokens,
          completionTokens: res.usage?.completion_tokens,
          attempt,
        },
        'intent.success'
      );
      return parsed;
    } catch (e) {
      lastError = e instanceof Error ? e.message : String(e);
      logger.warn({ rid, attempt, error: lastError }, 'intent.parse_error');
    }
  }

  logger.warn({ rid, lastError }, 'intent.fallback');
  return {
    classification: 'other',
    confidence: 0,
    rationale: `Parse failure after retries: ${lastError}`,
    suggested_tools: [],
  };
}
```

Run both and read your logs. **Read your logs.** The most common beginner mistake is not looking at the structured logs you're already emitting.

---

## 1.4 Lab 1.6 — A CLI wrapper

Build a Click/Commander CLI so you can run queries by hand.

### Python — `python/src/app.py`

```python
import sys
from logging_setup import setup_logging
from agents.intent_v1 import classify_intent

def main():
    setup_logging()
    if len(sys.argv) < 2:
        print("Usage: python src/app.py 'your query here'")
        sys.exit(1)
    query = " ".join(sys.argv[1:])
    result = classify_intent(query)
    print(result.model_dump_json(indent=2))

if __name__ == "__main__":
    main()
```

Run:

```bash
uv run python src/app.py "Do you have men's hiking boots size 11?"
```

### TypeScript — `typescript/src/app.ts`

```typescript
import { classifyIntent } from './agents/intentV1.js';

const query = process.argv.slice(2).join(' ');
if (!query) {
  console.error("Usage: pnpm tsx src/app.ts 'your query here'");
  process.exit(1);
}
const result = await classifyIntent(query);
console.log(JSON.stringify(result, null, 2));
```

Run:

```bash
pnpm tsx src/app.ts "Do you have men's hiking boots size 11?"
```

---

## 1.5 Lab 1.7 — A tracing notebook

Open `notebooks/01_trace_analysis.ipynb`:

```python
import json, pandas as pd
from pathlib import Path

# Assume you piped logs to a file: python src/app.py "..." 2>&1 | tee logs.jsonl
logs = []
with open("../logs.jsonl") as f:
    for line in f:
        try:
            logs.append(json.loads(line))
        except json.JSONDecodeError:
            pass

df = pd.DataFrame(logs)
print("Total events:", len(df))
print("\nEvent types:\n", df["msg"].value_counts())

successes = df[df["msg"] == "intent.success"]
print("\nLatency p50/p95:", successes["latency_ms"].quantile([0.5, 0.95]).to_dict())
print("Avg tokens in/out:", successes[["prompt_tokens","completion_tokens"]].mean().to_dict())
```

> 💡 **Why this matters**
> You just built a *minimum viable observability stack* with `tee` and pandas. In Module 9 you'll graduate to LangSmith/Langfuse, but the discipline of "every interesting event is a structured log line" is the foundation.

---

## 1.6 Same lab with LangChain (Python + TS) — preview

You'll meet LangChain properly in Module 4, but here's the same intent classifier in LangChain so you see the abstraction.

### Python (LangChain)

```python
from langchain_openai import ChatOpenAI
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.output_parsers import JsonOutputParser
from pydantic import BaseModel, Field
from typing import Literal

class IntentResult(BaseModel):
    classification: Literal["product_search","order_status","return_request","policy_question","other"]
    confidence: float = Field(ge=0, le=1)
    rationale: str

parser = JsonOutputParser(pydantic_object=IntentResult)
prompt = ChatPromptTemplate.from_messages([
    ("system", "You are a retail intent classifier. {format_instructions}"),
    ("user", "{query}"),
]).partial(format_instructions=parser.get_format_instructions())

llm = ChatOpenAI(model="gpt-4o-mini", temperature=0.1)
chain = prompt | llm | parser

result = chain.invoke({"query": "Where is my order ORD-4456?"})
print(result)
```

### TypeScript (LangChain)

```typescript
import { ChatOpenAI } from '@langchain/openai';
import { ChatPromptTemplate } from '@langchain/core/prompts';
import { JsonOutputParser } from '@langchain/core/output_parsers';

const parser = new JsonOutputParser<{ classification: string; confidence: number; rationale: string }>();
const prompt = ChatPromptTemplate.fromMessages([
  ['system', 'You are a retail intent classifier. Return JSON with fields classification, confidence, rationale.'],
  ['user', '{query}'],
]);
const llm = new ChatOpenAI({ model: 'gpt-4o-mini', temperature: 0.1 });
const chain = prompt.pipe(llm).pipe(parser);
console.log(await chain.invoke({ query: 'Where is my order ORD-4456?' }));
```

**What you gain with LangChain**: composable chains, swap providers in one line, built-in retry/parse helpers.
**What you give up**: a small layer of magic. When something goes wrong, you have to know LangChain's abstractions, not just OpenAI's API.

The Framework Cheatsheet covers this tradeoff in depth.

---

## 1.7 Assignment 1 — System Memo

Create `docs/system_memo.md`. Target 1–2 pages. Use this template:

```markdown
# System Memo — Retail Agentic Copilot v0

## 1. Target user
- Primary: online shoppers visiting our outdoor retail site
- Secondary: support agents handling escalations

## 2. Job to be done
- Shoppers want to find products matching fuzzy constraints ("warm for winter, not too bulky")
  and resolve simple order/return questions without waiting for a human.

## 3. Current non-AI baseline
- Faceted search with rigid filters
- Static FAQ page
- Email support (24h SLA)

## 4. Where AI adds value
- Interpreting natural-language constraints
- Synthesizing info across product, policy, and review sources
- Drafting personalized responses (with human-in-loop for refunds)

## 5. Where AI is explicitly out of bounds
- Final pricing or discounts (deterministic system of record)
- Promising refunds, credits, or shipping upgrades
- Modifying inventory or customer records

## 6. Constraints
- Latency: p95 < 3 seconds for shopping; < 5s for policy questions
- Cost: < $0.03 per query at scale
- Safety: must not invent SKUs, prices, or policies
- Compliance: PII must not leak to LLM providers (mask order IDs in prompts where possible)

## 7. Success metrics (will refine in Module 3)
- Intent classification accuracy ≥ 90% on labeled set
- Hallucination rate (invented SKU/price/policy) ≤ 1%
- Helpful-and-grounded rate (human eval) ≥ 80%
```

This memo will be your reference for every later module. Update it when something fundamental changes.

---

## 1.8 Checklist before moving on

- [ ] `intent_v1` works in both Python and TypeScript.
- [ ] Structured output passes validation; bad inputs trigger fallback (not crash).
- [ ] You can read your structured logs and answer: "what was the p95 latency for the last 20 calls?"
- [ ] `docs/system_memo.md` exists, is 1–2 pages, and you'd feel okay showing it to a PM.
- [ ] You've called both OpenAI and Gemini with the same prompt and *qualitatively* compared the outputs.

Head to [Module 2 — Context Engineering](04-module-2-context-engineering.md).

---

## Decision-log prompts

- What's your top concern about putting an agent in front of real shoppers? Is it covered in the memo?
- Did you choose JSON-mode (`response_format`) or function calling for structured output? Why?
- Where did you set `temperature`? Higher (creative) vs lower (deterministic) — defend your choice.
