# Module 4 — Agentic Retrieval, RAG, Memory & Reflection (Week 4)

> **Time**: 10–12 hours
> **Goal**: A working baseline RAG over your retail corpus, then upgrade it with query rewriting and reflection.
> **Deliverable**: `src/retrieval/rag.py`, `src/retrieval/agentic_rag.py`, `docs/rag_design.md`, RAG-specific eval metrics.

By the end of this module, when a shopper asks *"Can I return final-sale shoes that were a gift?"*, the system retrieves the right policy chunk, generates an answer with citations, and — if confidence is low — retries with a reformulated query.

---

## 4.1 RAG, plain and simple

> 🔍 **Glossary — RAG (Retrieval-Augmented Generation)**
> A pattern where, before the LLM generates an answer, you retrieve relevant text chunks from a knowledge base and add them to the prompt. The LLM then answers grounded in those chunks instead of from training memory.

The four phases of every RAG system:

```
1. Indexing (one-time / periodic)
   docs → split into chunks → embed each chunk → store in vector DB

2. Retrieval (per-query)
   user query → embed → vector search → top-k chunks

3. (Optional) Reranking
   top-k → cross-encoder → top-n (smaller, more relevant)

4. Generation
   prompt = system + retrieved chunks + user query → LLM → grounded answer
```

> 🔍 **Glossary — Embedding**
> A fixed-length vector (e.g., 1536 numbers) that represents the *meaning* of a piece of text. Two texts with similar meaning land at nearby points in vector space. The model that produces embeddings is called an **embedding model** (e.g., `text-embedding-3-small`).

> 🔍 **Glossary — Vector database**
> A database optimized for nearest-neighbor search over embeddings. Chroma, Qdrant, Pinecone, and Postgres+pgvector are all vector DBs.

---

## 4.2 Why agentic RAG?

Traditional RAG is a straight line: query → retrieve → answer. It fails when:

- The user's query is too vague for retrieval to land on the right chunk ("can you help with my jacket").
- The first retrieval misses; the model invents an answer.
- Multiple sub-questions need different retrievals ("return policy for sale items shipped to Canada").

**Agentic RAG** introduces a loop: the LLM judges retrieval quality and decides whether to retrieve again, rewrite the query, or escalate.

```
        ┌─────────────────────────────────────────┐
        │           Agentic RAG loop              │
        │                                         │
  query─┤  rewrite ─► retrieve ─► judge results   │
        │     ▲           │           │           │
        │     └───────────┴───────────┘           │
        │              (if low confidence)        │
        │                                         │
        │     answer with citations               │
        └─────────────────────────────────────────┘
```

You'll build both the baseline and the agentic version in this module.

---

## 4.3 Lab 4.1 — Set up Chroma + embed the corpus

Chroma is an embedded vector DB — no server to run, just a Python or JS library that writes to a local folder.

### Python — `python/src/retrieval/index_builder.py`

```python
"""Chunk the retail corpus and load it into Chroma."""
from __future__ import annotations
import csv
import json
import re
from pathlib import Path
from typing import Iterable
import chromadb
from chromadb.utils import embedding_functions

ROOT = Path(__file__).resolve().parents[3]
RAW = ROOT / "data" / "raw"
CHROMA_DIR = ROOT / "chroma_db"
COLLECTION = "retail_corpus"

# OpenAI's text-embedding-3-small is $0.02/1M tokens — perfect for course use.
embedding_fn = embedding_functions.OpenAIEmbeddingFunction(
    api_key=None,  # picks up OPENAI_API_KEY env var
    model_name="text-embedding-3-small",
)


def chunk_text(text: str, max_chars: int = 800, overlap: int = 100) -> list[str]:
    """Naive char-based chunker. Good enough for clean text; we'll improve in 4.4."""
    text = re.sub(r"\s+", " ", text).strip()
    if len(text) <= max_chars:
        return [text]
    chunks, i = [], 0
    while i < len(text):
        chunks.append(text[i : i + max_chars])
        i += max_chars - overlap
    return chunks


def products_to_chunks() -> Iterable[dict]:
    with (RAW / "products.csv").open() as f:
        for row in csv.DictReader(f):
            text = (
                f"PRODUCT {row['sku']}: {row['name']}\n"
                f"Category: {row['category']} / {row['subtype']}\n"
                f"Brand: {row['brand']} | Gender: {row['gender']} | Price: ${row['price']}\n"
                f"Attributes: {', '.join(json.loads(row['attributes']))}\n"
                f"Sizes: {', '.join(json.loads(row['sizes']))} | Stock: {row['stock']}\n"
                f"Description: {row['description']}"
            )
            yield {
                "id": f"prod::{row['sku']}",
                "text": text,
                "metadata": {
                    "source": "product",
                    "sku": row["sku"],
                    "category": row["category"],
                    "price": float(row["price"]),
                    "in_stock": int(row["stock"]) > 0,
                },
            }


def policies_to_chunks() -> Iterable[dict]:
    for md in (RAW / "policies").glob("*.md"):
        topic = md.stem
        text = md.read_text()
        for i, chunk in enumerate(chunk_text(text, max_chars=600, overlap=80)):
            yield {
                "id": f"policy::{topic}::{i}",
                "text": f"[POLICY: {topic}]\n{chunk}",
                "metadata": {"source": "policy", "topic": topic, "chunk_idx": i},
            }


def faqs_to_chunks() -> Iterable[dict]:
    faqs = json.loads((RAW / "faqs.json").read_text())
    for i, faq in enumerate(faqs):
        yield {
            "id": f"faq::{i}",
            "text": f"FAQ: {faq['q']}\nAnswer: {faq['a']}",
            "metadata": {"source": "faq", "tags": ",".join(faq["tags"])},
        }


def build_index(reset: bool = False) -> None:
    client = chromadb.PersistentClient(path=str(CHROMA_DIR))
    if reset:
        try:
            client.delete_collection(COLLECTION)
        except Exception:
            pass
    collection = client.get_or_create_collection(
        name=COLLECTION, embedding_function=embedding_fn
    )

    all_chunks = list(products_to_chunks()) + list(policies_to_chunks()) + list(faqs_to_chunks())
    print(f"Embedding {len(all_chunks)} chunks…")

    # Chroma batches well at ~100; large pushes can OOM
    BATCH = 100
    for i in range(0, len(all_chunks), BATCH):
        batch = all_chunks[i : i + BATCH]
        collection.add(
            ids=[c["id"] for c in batch],
            documents=[c["text"] for c in batch],
            metadatas=[c["metadata"] for c in batch],
        )
    print(f"✅ Index built at {CHROMA_DIR}")
    print(f"   Total docs in collection: {collection.count()}")


if __name__ == "__main__":
    import sys
    build_index(reset="--reset" in sys.argv)
```

Run:

```bash
cd python && uv run python src/retrieval/index_builder.py --reset
```

You should see ~280–350 chunks indexed (depends on chunking).

### TypeScript — `typescript/src/retrieval/indexBuilder.ts`

```typescript
import { readFileSync, readdirSync } from 'node:fs';
import { join } from 'node:path';
import { parse } from 'csv-parse/sync';
import { ChromaClient } from 'chromadb';
import OpenAI from 'openai';

const ROOT = join(import.meta.dirname, '..', '..', '..');
const RAW = join(ROOT, 'data', 'raw');
const CHROMA_DIR = join(ROOT, 'chroma_db');
const COLLECTION = 'retail_corpus';

const openai = new OpenAI();
const client = new ChromaClient({ path: 'http://localhost:8000' });
// Note: the JS Chroma client speaks to a Chroma server; run `pip install chromadb && chroma run`
// or use the embedded Python builder and read from the same chroma_db folder via the server.

async function embed(texts: string[]): Promise<number[][]> {
  const res = await openai.embeddings.create({
    model: 'text-embedding-3-small',
    input: texts,
  });
  return res.data.map((d) => d.embedding);
}

function chunkText(text: string, maxChars = 800, overlap = 100): string[] {
  const t = text.replace(/\s+/g, ' ').trim();
  if (t.length <= maxChars) return [t];
  const chunks: string[] = [];
  for (let i = 0; i < t.length; i += maxChars - overlap) chunks.push(t.slice(i, i + maxChars));
  return chunks;
}

async function build() {
  const col = await client.getOrCreateCollection({ name: COLLECTION });
  const products = parse(readFileSync(join(RAW, 'products.csv'), 'utf8'), { columns: true });
  const docs: { id: string; text: string; meta: any }[] = [];
  for (const p of products as any[]) {
    docs.push({
      id: `prod::${p.sku}`,
      text: `PRODUCT ${p.sku}: ${p.name}\nPrice: $${p.price}\nDescription: ${p.description}`,
      meta: { source: 'product', sku: p.sku, category: p.category, price: Number(p.price) },
    });
  }
  for (const f of readdirSync(join(RAW, 'policies'))) {
    const text = readFileSync(join(RAW, 'policies', f), 'utf8');
    chunkText(text, 600, 80).forEach((c, i) =>
      docs.push({ id: `policy::${f}::${i}`, text: `[POLICY: ${f}]\n${c}`, meta: { source: 'policy', topic: f } })
    );
  }
  const faqs = JSON.parse(readFileSync(join(RAW, 'faqs.json'), 'utf8'));
  faqs.forEach((q: any, i: number) =>
    docs.push({ id: `faq::${i}`, text: `FAQ: ${q.q}\nAnswer: ${q.a}`, meta: { source: 'faq' } })
  );

  console.log(`Embedding ${docs.length} chunks…`);
  const BATCH = 100;
  for (let i = 0; i < docs.length; i += BATCH) {
    const slice = docs.slice(i, i + BATCH);
    const embeddings = await embed(slice.map((d) => d.text));
    await col.add({
      ids: slice.map((d) => d.id),
      embeddings,
      documents: slice.map((d) => d.text),
      metadatas: slice.map((d) => d.meta),
    });
  }
  console.log(`✅ ${await col.count()} total docs`);
}

build();
```

> ⚠️ **Heads up for TS users**
> The JS Chroma client needs a running Chroma server (`pip install chromadb && chroma run`). The Python embedded client just writes to disk — simpler for local dev. If you're TS-only, just keep a Chroma server running locally on `:8000`.

---

## 4.4 Chunking strategy — the most underrated lever

How you chop documents determines retrieval quality more than which model you embed with. Three approaches, picked per source type:

| Source | Best chunker | Why |
|--------|--------------|-----|
| Product records (structured) | **Whole record per chunk** | Don't split SKU from price |
| Policy markdown | **By heading or 400–800 chars with overlap** | Keeps semantic units intact |
| FAQs | **One Q&A per chunk** | The unit of meaning |
| Long reports / PDFs | **Recursive: page → paragraph → sentence with overlap** | Preserves hierarchy |
| Code | **By function/class** (AST-based) | Definitions stay together |

The chunker above uses "whole record" for products, fixed-size for policies, and "one Q&A" for FAQs. Upgrade to header-aware chunking (using `langchain.text_splitter.MarkdownHeaderTextSplitter`) when you outgrow it.

> 💡 **Why this matters**
> A query like *"Does the warranty cover zippers?"* finds the answer only if the chunk it's stored in contains both "warranty" and "zipper" near each other. Bad chunking literally orphans answers.

---

## 4.5 Lab 4.2 — Baseline RAG

### Python — `python/src/retrieval/rag.py`

```python
"""Baseline RAG: retrieve top-k, answer with citations."""
from __future__ import annotations
import chromadb
from chromadb.utils import embedding_functions
from openai import OpenAI
from pathlib import Path

CHROMA_DIR = Path(__file__).resolve().parents[3] / "chroma_db"
COLLECTION = "retail_corpus"

embedding_fn = embedding_functions.OpenAIEmbeddingFunction(model_name="text-embedding-3-small")
chroma = chromadb.PersistentClient(path=str(CHROMA_DIR))
col = chroma.get_collection(name=COLLECTION, embedding_function=embedding_fn)
oai = OpenAI()


def retrieve(query: str, k: int = 5, where: dict | None = None) -> list[dict]:
    """Return top-k chunks with score and metadata."""
    res = col.query(query_texts=[query], n_results=k, where=where)
    hits = []
    for i in range(len(res["ids"][0])):
        hits.append({
            "id": res["ids"][0][i],
            "text": res["documents"][0][i],
            "metadata": res["metadatas"][0][i],
            "distance": res["distances"][0][i],
        })
    return hits


SYSTEM = """You are a retail copilot. Answer the shopper's question using ONLY the
provided context. Every factual claim must cite [chunk_id]. If the context does not
contain the answer, say "I don't have that information in the catalog."

Format:
  Brief direct answer (1-2 sentences).
  Then a bulleted list of supporting facts with [chunk_id] citations.
"""


def answer_with_evidence(query: str, hits: list[dict]) -> str:
    context = "\n\n".join(f"[{h['id']}]\n{h['text']}" for h in hits)
    r = oai.chat.completions.create(
        model="gpt-4o-mini",
        messages=[
            {"role": "system", "content": SYSTEM},
            {"role": "user", "content": f"CONTEXT:\n{context}\n\nQUESTION: {query}"},
        ],
        temperature=0.2,
    )
    return r.choices[0].message.content or ""


def ask(query: str, k: int = 5) -> dict:
    hits = retrieve(query, k=k)
    answer = answer_with_evidence(query, hits)
    return {"answer": answer, "citations": [h["id"] for h in hits]}


if __name__ == "__main__":
    queries = [
        "Can I return shoes that were on sale?",
        "Do you ship to Alaska?",
        "What waterproof jackets do you have under $200?",
        "How long is the warranty on a tent?",
    ]
    for q in queries:
        print(f"\n> {q}")
        result = ask(q)
        print(result["answer"])
        print("citations:", result["citations"])
```

### TypeScript — `typescript/src/retrieval/rag.ts`

```typescript
import { ChromaClient } from 'chromadb';
import OpenAI from 'openai';

const oai = new OpenAI();
const chroma = new ChromaClient({ path: 'http://localhost:8000' });

const SYSTEM = `You are a retail copilot. Answer using ONLY the provided context.
Every claim must cite [chunk_id]. If unknown, say "I don't have that information."`;

export async function retrieve(query: string, k = 5, where?: any) {
  const col = await chroma.getCollection({ name: 'retail_corpus' });
  const embeddings = await oai.embeddings.create({
    model: 'text-embedding-3-small',
    input: [query],
  });
  const res: any = await col.query({
    queryEmbeddings: [embeddings.data[0].embedding],
    nResults: k,
    where,
  });
  return res.ids[0].map((id: string, i: number) => ({
    id,
    text: res.documents[0][i],
    metadata: res.metadatas[0][i],
    distance: res.distances[0][i],
  }));
}

export async function ask(query: string, k = 5) {
  const hits = await retrieve(query, k);
  const context = hits.map((h: any) => `[${h.id}]\n${h.text}`).join('\n\n');
  const r = await oai.chat.completions.create({
    model: 'gpt-4o-mini',
    messages: [
      { role: 'system', content: SYSTEM },
      { role: 'user', content: `CONTEXT:\n${context}\n\nQUESTION: ${query}` },
    ],
    temperature: 0.2,
  });
  return { answer: r.choices[0].message.content ?? '', citations: hits.map((h: any) => h.id) };
}
```

---

## 4.6 Lab 4.3 — Agentic RAG (query rewriting + reflection)

The baseline misses queries like *"can I bring back stuff I got on the cheap?"* because vector search for those exact words doesn't land on the returns/sale policy. Fix it with a **query rewriter**.

### Python — `python/src/retrieval/agentic_rag.py`

```python
"""Agentic RAG: rewrite vague queries, retrieve, judge, optionally retry."""
from __future__ import annotations
import json
from openai import OpenAI
from .rag import retrieve, answer_with_evidence

oai = OpenAI()

REWRITER_SYSTEM = """You rewrite a shopper's casual query into 1-3 precise retrieval queries
for a retail KB containing products, policies, and FAQs.

Return JSON: {"queries": ["...", "..."]}

Guidelines:
- Convert slang to formal language ("got it cheap" → "purchased on sale").
- Split compound questions ("return shoes AND track order" → two queries).
- Add domain terms ("policy", "warranty", "sizing chart") when the topic suggests it.
- Prefer 2 queries; only use 3 if the user truly asked multiple things."""

JUDGE_SYSTEM = """You evaluate whether retrieved chunks contain the answer to the user query.

Return JSON: {"sufficient": true/false, "confidence": 0-1, "missing": "what's not covered"}"""


def rewrite_queries(query: str) -> list[str]:
    r = oai.chat.completions.create(
        model="gpt-4o-mini",
        messages=[{"role": "system", "content": REWRITER_SYSTEM},
                  {"role": "user", "content": query}],
        response_format={"type": "json_object"},
        temperature=0.3,
    )
    return json.loads(r.choices[0].message.content or "{}").get("queries", [query])


def judge_retrieval(query: str, hits: list[dict]) -> dict:
    context = "\n\n".join(f"[{h['id']}] {h['text'][:300]}" for h in hits)
    r = oai.chat.completions.create(
        model="gpt-4o-mini",
        messages=[{"role": "system", "content": JUDGE_SYSTEM},
                  {"role": "user", "content": f"QUERY: {query}\nCHUNKS:\n{context}"}],
        response_format={"type": "json_object"},
        temperature=0,
    )
    return json.loads(r.choices[0].message.content or "{}")


def agentic_ask(query: str, k: int = 5, max_attempts: int = 2) -> dict:
    trace = []
    all_hits: list[dict] = []
    seen_ids: set[str] = set()

    for attempt in range(max_attempts):
        queries = rewrite_queries(query) if attempt == 0 else [
            *rewrite_queries(query + " " + judgement.get("missing", ""))  # noqa: F821
        ]
        trace.append({"attempt": attempt, "queries": queries})
        for q in queries:
            for h in retrieve(q, k=k):
                if h["id"] not in seen_ids:
                    seen_ids.add(h["id"])
                    all_hits.append(h)
        judgement = judge_retrieval(query, all_hits[:8])
        trace.append({"attempt": attempt, "judgement": judgement})
        if judgement.get("sufficient", False) and judgement.get("confidence", 0) >= 0.7:
            break

    answer = answer_with_evidence(query, all_hits[:8])
    return {"answer": answer, "citations": [h["id"] for h in all_hits[:8]], "trace": trace}


if __name__ == "__main__":
    queries = [
        "can I bring back stuff I got on the cheap?",
        "got an order, where is it",
        "thinking about a tent for my family of 4, also is shipping free?",
    ]
    for q in queries:
        print(f"\n> {q}")
        result = agentic_ask(q)
        print(result["answer"])
        print("\nTrace:")
        for t in result["trace"]:
            print(" ", t)
```

### TypeScript — `typescript/src/retrieval/agenticRag.ts`

```typescript
import OpenAI from 'openai';
import { retrieve } from './rag.js';

const oai = new OpenAI();

const REWRITER = `You rewrite a casual query into 1-3 precise retrieval queries for a retail KB.
Return JSON {"queries": ["..."]}.`;

const JUDGE = `You judge whether retrieved chunks contain the answer.
Return JSON {"sufficient": boolean, "confidence": 0-1, "missing": string}.`;

async function rewrite(query: string): Promise<string[]> {
  const r = await oai.chat.completions.create({
    model: 'gpt-4o-mini',
    messages: [
      { role: 'system', content: REWRITER },
      { role: 'user', content: query },
    ],
    response_format: { type: 'json_object' },
    temperature: 0.3,
  });
  return JSON.parse(r.choices[0].message.content ?? '{}').queries ?? [query];
}

async function judge(query: string, hits: any[]) {
  const ctx = hits.map((h) => `[${h.id}] ${h.text.slice(0, 300)}`).join('\n\n');
  const r = await oai.chat.completions.create({
    model: 'gpt-4o-mini',
    messages: [
      { role: 'system', content: JUDGE },
      { role: 'user', content: `QUERY: ${query}\nCHUNKS:\n${ctx}` },
    ],
    response_format: { type: 'json_object' },
    temperature: 0,
  });
  return JSON.parse(r.choices[0].message.content ?? '{}');
}

export async function agenticAsk(query: string, k = 5, maxAttempts = 2) {
  const trace: any[] = [];
  const hits: any[] = [];
  const seen = new Set<string>();
  let judgement: any = {};

  for (let attempt = 0; attempt < maxAttempts; attempt++) {
    const queries =
      attempt === 0 ? await rewrite(query) : await rewrite(query + ' ' + (judgement.missing ?? ''));
    trace.push({ attempt, queries });
    for (const q of queries) {
      for (const h of await retrieve(q, k)) {
        if (!seen.has(h.id)) {
          seen.add(h.id);
          hits.push(h);
        }
      }
    }
    judgement = await judge(query, hits.slice(0, 8));
    trace.push({ attempt, judgement });
    if (judgement.sufficient && judgement.confidence >= 0.7) break;
  }

  // Generate the final answer (similar to rag.ts answer step)
  const ctx = hits
    .slice(0, 8)
    .map((h) => `[${h.id}]\n${h.text}`)
    .join('\n\n');
  const final = await oai.chat.completions.create({
    model: 'gpt-4o-mini',
    messages: [
      { role: 'system', content: 'Answer with citations [chunk_id]. Say unknown if missing.' },
      { role: 'user', content: `CONTEXT:\n${ctx}\n\nQUESTION: ${query}` },
    ],
    temperature: 0.2,
  });
  return {
    answer: final.choices[0].message.content ?? '',
    citations: hits.slice(0, 8).map((h) => h.id),
    trace,
  };
}
```

---

## 4.7 Reranking (optional but powerful)

A small cross-encoder model (e.g., `bge-reranker-base`) takes (query, chunk) pairs and outputs a relevance score. Adds ~150–300ms; typically lifts answer quality 10–25%.

```python
# pip install sentence-transformers
from sentence_transformers import CrossEncoder
reranker = CrossEncoder("BAAI/bge-reranker-base")

def rerank(query: str, hits: list[dict], top_n: int = 5) -> list[dict]:
    pairs = [(query, h["text"]) for h in hits]
    scores = reranker.predict(pairs)
    for h, s in zip(hits, scores):
        h["rerank_score"] = float(s)
    return sorted(hits, key=lambda h: h["rerank_score"], reverse=True)[:top_n]
```

Pipeline: retrieve top-20 → rerank to top-5 → answer.

---

## 4.8 Same lab with LangChain — for comparison

Once you understand the primitives, LangChain saves a lot of boilerplate.

### Python (LangChain + LangChain-Chroma)

```python
from langchain_chroma import Chroma
from langchain_openai import OpenAIEmbeddings, ChatOpenAI
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.runnables import RunnablePassthrough
from langchain_core.output_parsers import StrOutputParser

emb = OpenAIEmbeddings(model="text-embedding-3-small")
vs = Chroma(collection_name="retail_corpus", embedding_function=emb,
            persist_directory="./chroma_db")
retriever = vs.as_retriever(search_kwargs={"k": 5})

prompt = ChatPromptTemplate.from_template(
    "Answer using ONLY this context, cite [chunk_id]:\n{context}\n\nQuestion: {question}"
)

def format_docs(docs):
    return "\n\n".join(f"[{d.metadata.get('source','?')}] {d.page_content}" for d in docs)

chain = (
    {"context": retriever | format_docs, "question": RunnablePassthrough()}
    | prompt | ChatOpenAI(model="gpt-4o-mini") | StrOutputParser()
)
print(chain.invoke("Can I return final sale shoes?"))
```

20 lines vs the 80-line baseline you wrote. Tradeoff: when something breaks deep in `as_retriever`, you have to know LangChain. (Module Framework Cheatsheet has the picking-between guide.)

---

## 4.9 Lab 4.4 — RAG-specific evals

Extend your eval harness from Module 3 with retrieval metrics.

For each eval case, add a `gold_source_ids` field:

```json
{
  "id": "es-03",
  "input": "Can I return shoes that were on sale?",
  "gold_source_ids": ["policy::returns::0"],
  "gold_answer_keywords": ["final-sale", "not returnable"]
}
```

Metrics to add:

- **Recall@k**: did `gold_source_ids` appear in the retrieved chunks? (boolean)
- **MRR (Mean Reciprocal Rank)**: 1 / rank of the first correct chunk.
- **Citation precision**: of the chunk IDs the LLM cited, what fraction were in `gold_source_ids`?
- **Citation recall**: of `gold_source_ids`, what fraction did the LLM actually cite?
- **Groundedness (LLM judge)**: every claim in the answer can be traced to a cited chunk.

Add to `scripts/run_evals.py`:

```python
def recall_at_k(retrieved_ids: list[str], gold_ids: list[str]) -> float:
    return 1.0 if any(g in retrieved_ids for g in gold_ids) else 0.0

def mrr(retrieved_ids: list[str], gold_ids: list[str]) -> float:
    for i, rid in enumerate(retrieved_ids, start=1):
        if rid in gold_ids:
            return 1 / i
    return 0.0
```

Run baseline RAG vs agentic RAG with the same eval set. Expect:

- Agentic RAG: higher Recall@5, higher groundedness, lower hallucination.
- Cost goes up ~2–3× per query (rewriter + judge + maybe retry).
- Latency goes up ~600–1500ms.

This is the tradeoff you'll defend in your RAG design doc.

---

## 4.10 Assignment 4 — `docs/rag_design.md`

```markdown
# RAG Design — Retail Copilot

## Chunking
- Products: one whole record per chunk (preserves SKU↔price linkage).
- Policies: 600-char chunks with 80-char overlap (most policies are < 1200 chars).
- FAQs: one Q&A per chunk.
- Reviews: NOT indexed in MVP (noisy, would dilute retrieval; revisit Module 6).

## Embedding
- Model: text-embedding-3-small (1536d, $0.02/1M tokens).
- Distance metric: cosine similarity (Chroma default).
- Re-embed on schema change; otherwise nightly delta sync.

## Retrieval
- k=5 by default; agentic flow may union up to 15 unique chunks across rewrites.
- Optional rerank to top-5 with bge-reranker-base (deferred until eval shows benefit).
- Filters: drop product chunks with `in_stock=false` for product_search intent.

## Agentic behaviors
- Query rewriter (1-3 reformulations) for ambiguous queries.
- Retrieval judge (LLM); if confidence < 0.7, retry once with judge-suggested expansion.
- Reflection: skipped in v1 (latency cost > benefit for short queries).

## Tradeoffs
- Agentic flow adds ~700ms p50 and 1.8x cost. Justified because Recall@5 went from 0.62 → 0.84 on the eval set.
- Could lower cost by ~30% by caching rewriter outputs for repeated query patterns.
```

---

## 4.11 Checklist

- [ ] `index_builder.py` runs cleanly and produces a non-empty Chroma collection.
- [ ] `rag.py` answers a few sample queries with citations.
- [ ] `agentic_rag.py` handles deliberately vague queries better than baseline.
- [ ] Eval harness now reports Recall@k and groundedness.
- [ ] `docs/rag_design.md` is written.

Head to [Module 5 — Multi-Agent & MCP](07-module-5-multi-agent-mcp.md).

---

## Decision-log prompts

- Which queries did agentic RAG fix that baseline got wrong? Generalize the pattern.
- Did reranking improve eval scores enough to justify the latency? Decide and record.
- Where would you add a cache? (Embeddings? Rewriter outputs? Final answers?)
