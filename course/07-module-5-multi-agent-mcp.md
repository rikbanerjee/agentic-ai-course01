# Module 5 — Multi-Agent Systems, Tools & MCP (Week 5)

> **Time**: 10–12 hours
> **Goal**: Turn your single-prompt classifier into a coordinated multi-agent system with explicit tool contracts.
> **Deliverable**: A router + 2 specialist agents, JSON-schema-typed tools, `docs/multi_agent_architecture.md`.

This module is where "AI feature" becomes "agentic system." The keys to it not turning into spaghetti are: small typed contracts, explicit handoffs, and one agent doing one thing.

---

## 5.1 When to add a second agent

Don't until you've felt the pain of *not* having one. Signs your single agent is over-scoped:

- The system prompt is over ~500 tokens and full of conditionals ("if intent is X, then... if intent is Y, then...").
- One bad prompt change breaks unrelated capabilities.
- You can't measure per-capability quality independently.
- Different capabilities want different models (cheap classifier + expensive synthesizer).

When all three are true, split.

---

## 5.2 The classic three-agent retail topology

```
                ┌──────────────────────────┐
   user ──────▶ │       Router agent       │
                │   (intent + dispatch)    │
                └──────┬───────────────────┘
                       │
        ┌──────────────┼──────────────────────┐
        ▼              ▼                      ▼
┌─────────────┐  ┌────────────────┐  ┌──────────────────┐
│  Research   │  │   Decision     │  │    Refusal       │
│  agent      │  │   agent        │  │    handler       │
│ (RAG, tools)│  │ (business      │  │ (adversarial,    │
│             │  │  rules, format)│  │  PII, off-topic) │
└──────┬──────┘  └────────┬───────┘  └──────────────────┘
       │                  │
       └──────────┬───────┘
                  ▼
            ┌───────────┐
            │  Output   │
            │ (streamed)│
            └───────────┘
```

- **Router** — cheap, fast model. Classifies intent, picks the next agent, normalizes input.
- **Research** — does RAG (Module 4), calls catalog/order tools. The "what is true" agent.
- **Decision** — applies business rules (loyalty tier, promotions), formats user-facing copy. The "what to say" agent.
- **Refusal handler** — if guardrails flagged the input, returns a templated decline rather than burning a model call.

---

## 5.3 Lab 5.1 — Define tools with schemas

Every tool gets:

1. A **name** (verb + noun, e.g., `search_products`).
2. A **description** the LLM will read.
3. A **JSON schema** for inputs.
4. A **typed function** in code.
5. **Error contracts**: what shape do errors take?

### Python — `python/src/tools/registry.py`

```python
"""Typed tool registry the agent can call. Each tool: schema for LLM, fn for execution."""
from __future__ import annotations
import csv, json
from pathlib import Path
from typing import Any, Callable
from pydantic import BaseModel, Field, ValidationError

ROOT = Path(__file__).resolve().parents[3]
DATA = ROOT / "data" / "raw"


# ---------- Input models ----------
class SearchProductsInput(BaseModel):
    query: str = Field(description="Free-text product description (e.g., 'waterproof hiking jacket')")
    max_price: float | None = Field(None, description="Optional price ceiling in USD")
    in_stock_only: bool = Field(True, description="Exclude out-of-stock items")
    limit: int = Field(5, ge=1, le=20)


class GetProductInput(BaseModel):
    sku: str = Field(description="Product SKU like 'SKU-1234'")


class GetOrderInput(BaseModel):
    order_id: str = Field(description="Order ID like 'ORD-4456'")


class GetPolicyInput(BaseModel):
    topic: str = Field(description="One of: returns, shipping, warranty, price-match, loyalty")


# ---------- Implementations ----------
_products_cache: list[dict] | None = None
def _products() -> list[dict]:
    global _products_cache
    if _products_cache is None:
        with (DATA / "products.csv").open() as f:
            _products_cache = list(csv.DictReader(f))
    return _products_cache


def search_products(inp: SearchProductsInput) -> dict:
    q = inp.query.lower()
    hits = []
    for p in _products():
        text = f"{p['name']} {p['description']} {p['attributes']}".lower()
        if all(w in text for w in q.split()[:3]):  # naive — replace with RAG in production
            if inp.max_price is not None and float(p["price"]) > inp.max_price:
                continue
            if inp.in_stock_only and int(p["stock"]) == 0:
                continue
            hits.append({
                "sku": p["sku"], "name": p["name"], "price": float(p["price"]),
                "stock": int(p["stock"]), "category": p["category"],
            })
    return {"results": hits[: inp.limit], "total_matched": len(hits)}


def get_product(inp: GetProductInput) -> dict:
    for p in _products():
        if p["sku"] == inp.sku:
            return {"sku": p["sku"], "name": p["name"], "price": float(p["price"]),
                    "stock": int(p["stock"]), "description": p["description"]}
    return {"error": "not_found", "sku": inp.sku}


def get_order(inp: GetOrderInput) -> dict:
    with (DATA / "orders.csv").open() as f:
        for o in csv.DictReader(f):
            if o["order_id"] == inp.order_id:
                return o
    return {"error": "not_found", "order_id": inp.order_id}


def get_policy(inp: GetPolicyInput) -> dict:
    path = DATA / "policies" / f"{inp.topic}.md"
    if not path.exists():
        return {"error": "unknown_topic", "topic": inp.topic}
    return {"topic": inp.topic, "content": path.read_text()}


# ---------- Registry ----------
class Tool(BaseModel):
    name: str
    description: str
    input_model: type[BaseModel]
    fn: Callable[..., dict]

    class Config:
        arbitrary_types_allowed = True

    def schema(self) -> dict:
        return {
            "type": "function",
            "function": {
                "name": self.name,
                "description": self.description,
                "parameters": self.input_model.model_json_schema(),
            },
        }

    def run(self, args_json: str | dict) -> dict:
        args = json.loads(args_json) if isinstance(args_json, str) else args_json
        try:
            inp = self.input_model(**args)
        except ValidationError as e:
            return {"error": "validation_failed", "details": e.errors()}
        try:
            return self.fn(inp)
        except Exception as e:
            return {"error": "tool_exception", "type": type(e).__name__, "message": str(e)}


REGISTRY: dict[str, Tool] = {
    "search_products": Tool(
        name="search_products",
        description=("Search the retail catalog for products matching a natural-language description. "
                     "Use whenever the user expresses a shopping need with attributes (color, material, use case)."),
        input_model=SearchProductsInput,
        fn=search_products,
    ),
    "get_product": Tool(
        name="get_product",
        description="Look up a single product by SKU. Use when the user mentions a specific SKU.",
        input_model=GetProductInput,
        fn=get_product,
    ),
    "get_order": Tool(
        name="get_order",
        description="Fetch order status by order ID (pattern ORD-####). Use for any order_status query.",
        input_model=GetOrderInput,
        fn=get_order,
    ),
    "get_policy": Tool(
        name="get_policy",
        description=("Fetch one of our policy documents. Use for returns/shipping/warranty/price-match/loyalty questions. "
                     "Returns the full policy markdown."),
        input_model=GetPolicyInput,
        fn=get_policy,
    ),
}


def tool_schemas() -> list[dict]:
    return [t.schema() for t in REGISTRY.values()]
```

### TypeScript — `typescript/src/tools/registry.ts`

```typescript
import { readFileSync } from 'node:fs';
import { join } from 'node:path';
import { parse } from 'csv-parse/sync';
import { z } from 'zod';

const ROOT = join(import.meta.dirname, '..', '..', '..');
const DATA = join(ROOT, 'data', 'raw');

const SearchProductsInput = z.object({
  query: z.string(),
  max_price: z.number().nullable().default(null),
  in_stock_only: z.boolean().default(true),
  limit: z.number().int().min(1).max(20).default(5),
});
const GetProductInput = z.object({ sku: z.string() });
const GetOrderInput = z.object({ order_id: z.string() });
const GetPolicyInput = z.object({ topic: z.enum(['returns', 'shipping', 'warranty', 'price-match', 'loyalty']) });

let productsCache: any[] | null = null;
const products = () => (productsCache ??= parse(readFileSync(join(DATA, 'products.csv'), 'utf8'), { columns: true }));

const fns = {
  search_products: (inp: z.infer<typeof SearchProductsInput>) => {
    const q = inp.query.toLowerCase().split(' ').slice(0, 3);
    const hits = products()
      .filter((p) => q.every((w) => `${p.name} ${p.description}`.toLowerCase().includes(w)))
      .filter((p) => (inp.max_price == null ? true : Number(p.price) <= inp.max_price))
      .filter((p) => (inp.in_stock_only ? Number(p.stock) > 0 : true))
      .slice(0, inp.limit)
      .map((p) => ({ sku: p.sku, name: p.name, price: Number(p.price), stock: Number(p.stock) }));
    return { results: hits, total_matched: hits.length };
  },
  get_product: (inp: z.infer<typeof GetProductInput>) => {
    const p = products().find((x) => x.sku === inp.sku);
    return p ? { sku: p.sku, name: p.name, price: Number(p.price), stock: Number(p.stock) } : { error: 'not_found' };
  },
  get_order: (inp: z.infer<typeof GetOrderInput>) => {
    const orders = parse(readFileSync(join(DATA, 'orders.csv'), 'utf8'), { columns: true });
    const o = (orders as any[]).find((x) => x.order_id === inp.order_id);
    return o ?? { error: 'not_found' };
  },
  get_policy: (inp: z.infer<typeof GetPolicyInput>) => {
    return { topic: inp.topic, content: readFileSync(join(DATA, 'policies', `${inp.topic}.md`), 'utf8') };
  },
};

const schemas = {
  search_products: SearchProductsInput,
  get_product: GetProductInput,
  get_order: GetOrderInput,
  get_policy: GetPolicyInput,
};

const descriptions: Record<string, string> = {
  search_products: 'Search the retail catalog with attributes (waterproof, size, price ceiling).',
  get_product: 'Look up one product by SKU like SKU-1234.',
  get_order: 'Look up an order by ID (ORD-####).',
  get_policy: 'Fetch a policy document by topic.',
};

// Convert Zod schema → JSON Schema (use `zod-to-json-schema` package in real code)
import { zodToJsonSchema } from 'zod-to-json-schema';

export const toolSchemas = Object.entries(schemas).map(([name, sch]) => ({
  type: 'function' as const,
  function: {
    name,
    description: descriptions[name],
    parameters: zodToJsonSchema(sch),
  },
}));

export function runTool(name: keyof typeof fns, argsJson: string) {
  const args = JSON.parse(argsJson);
  const parsed = (schemas as any)[name].safeParse(args);
  if (!parsed.success) return { error: 'validation_failed', details: parsed.error.errors };
  try {
    return (fns as any)[name](parsed.data);
  } catch (e) {
    return { error: 'tool_exception', message: (e as Error).message };
  }
}
```

> 💡 **Why this matters**
> The schema is what the LLM uses to *write* the tool call. The validator is what protects you from the LLM hallucinating fields. Always have both.

---

## 5.4 Lab 5.2 — Single agent with tools (the ReAct loop)

Before multi-agent, get one agent with tools working end-to-end.

### Python — `python/src/agents/research_agent.py`

```python
"""ReAct agent: LLM picks tools, we execute, loop until done."""
from __future__ import annotations
import json
from openai import OpenAI
from tools.registry import REGISTRY, tool_schemas

oai = OpenAI()

SYSTEM = """You are a retail research agent. You can call tools to look up products,
orders, and policies. After the user asks a question:

1. Decide what info you need.
2. Call one or more tools (you can call multiple in a turn, or one at a time).
3. When you have enough info, write a final answer that cites the specific tool
   results you used (mention SKUs, order IDs, policy topics).

Never invent SKUs, prices, or policy text. If a tool returns an error or 'not_found',
say so honestly."""


def run(query: str, model: str = "gpt-4o-mini", max_steps: int = 6) -> dict:
    messages = [
        {"role": "system", "content": SYSTEM},
        {"role": "user", "content": query},
    ]
    trace = []

    for step in range(max_steps):
        res = oai.chat.completions.create(
            model=model,
            messages=messages,
            tools=tool_schemas(),
            tool_choice="auto",
            temperature=0.2,
        )
        msg = res.choices[0].message
        messages.append(msg.model_dump(exclude_none=True))

        if not msg.tool_calls:
            trace.append({"step": step, "final": True})
            return {"answer": msg.content, "trace": trace, "steps": step + 1}

        for tc in msg.tool_calls:
            name = tc.function.name
            args = tc.function.arguments
            trace.append({"step": step, "tool": name, "args": json.loads(args)})
            tool = REGISTRY.get(name)
            if tool is None:
                result = {"error": "unknown_tool", "name": name}
            else:
                result = tool.run(args)
            trace.append({"step": step, "tool_result": name, "preview": str(result)[:200]})
            messages.append({
                "role": "tool",
                "tool_call_id": tc.id,
                "content": json.dumps(result),
            })

    return {"answer": "[max steps reached]", "trace": trace, "steps": max_steps}


if __name__ == "__main__":
    qs = [
        "Find me a waterproof hiking jacket under $150.",
        "What's the status of order ORD-4456?",
        "Can I return shoes that were final sale?",
    ]
    for q in qs:
        print(f"\n> {q}")
        r = run(q)
        print(r["answer"])
        print(f"\nSteps: {r['steps']}, tool calls: {sum(1 for t in r['trace'] if 'tool' in t)}")
```

### TypeScript — `typescript/src/agents/researchAgent.ts`

```typescript
import OpenAI from 'openai';
import { toolSchemas, runTool } from '../tools/registry.js';

const oai = new OpenAI();

const SYSTEM = `You are a retail research agent. Call tools to look up products,
orders, and policies. Never invent SKUs or prices.`;

export async function run(query: string, model = 'gpt-4o-mini', maxSteps = 6) {
  const messages: any[] = [
    { role: 'system', content: SYSTEM },
    { role: 'user', content: query },
  ];
  const trace: any[] = [];

  for (let step = 0; step < maxSteps; step++) {
    const res = await oai.chat.completions.create({
      model,
      messages,
      tools: toolSchemas as any,
      tool_choice: 'auto',
      temperature: 0.2,
    });
    const msg = res.choices[0].message;
    messages.push(msg);
    if (!msg.tool_calls?.length) {
      return { answer: msg.content, trace, steps: step + 1 };
    }
    for (const tc of msg.tool_calls) {
      const result = runTool(tc.function.name as any, tc.function.arguments);
      trace.push({ step, tool: tc.function.name, args: JSON.parse(tc.function.arguments), result });
      messages.push({ role: 'tool', tool_call_id: tc.id, content: JSON.stringify(result) });
    }
  }
  return { answer: '[max steps reached]', trace, steps: maxSteps };
}
```

Run it and inspect the trace. You'll see the LLM call `search_products`, then maybe `get_product` on a top hit, then write a final answer.

---

## 5.5 Lab 5.3 — Multi-agent with LangGraph

LangGraph is the cleanest way to express multi-agent flows in Python or TS. It models the system as a directed graph of nodes (agents/functions) and edges (transitions).

### Python (LangGraph)

```python
# python/src/agents/multi_agent.py
from __future__ import annotations
from typing import TypedDict, Literal
from langgraph.graph import StateGraph, END
from langchain_openai import ChatOpenAI
from langchain_core.messages import SystemMessage, HumanMessage

from .intent_v1 import classify_intent
from .research_agent import run as research_run

class State(TypedDict):
    user_query: str
    intent: str | None
    research: dict | None
    final_answer: str | None
    refusal: str | None

def router_node(state: State) -> dict:
    res = classify_intent(state["user_query"])
    return {"intent": res.classification}

def research_node(state: State) -> dict:
    out = research_run(state["user_query"])
    return {"research": out}

def decision_node(state: State) -> dict:
    """Apply business rules + format final user-facing answer."""
    llm = ChatOpenAI(model="gpt-4o-mini", temperature=0.4)
    sys = ("You write friendly retail responses. Use the research output below. "
           "If the research mentions out-of-stock items, suggest similar in-stock ones. "
           "Never promise refunds, credits, or shipping upgrades.")
    msg = llm.invoke([
        SystemMessage(content=sys),
        HumanMessage(content=f"USER: {state['user_query']}\nRESEARCH: {state['research']}")
    ])
    return {"final_answer": msg.content}

def refusal_node(state: State) -> dict:
    return {"refusal": ("I can help with our catalog, orders, and policies. "
                        "For that request please contact support@retail.example.")}

def route_after_router(state: State) -> Literal["research", "refusal", "decision"]:
    if state["intent"] == "other":
        return "refusal"
    return "research"

# Build graph
graph = StateGraph(State)
graph.add_node("router", router_node)
graph.add_node("research", research_node)
graph.add_node("decision", decision_node)
graph.add_node("refusal", refusal_node)

graph.set_entry_point("router")
graph.add_conditional_edges("router", route_after_router, {
    "research": "research", "refusal": "refusal", "decision": "decision",
})
graph.add_edge("research", "decision")
graph.add_edge("decision", END)
graph.add_edge("refusal", END)

app = graph.compile()

if __name__ == "__main__":
    for q in ["Find me a waterproof jacket under $150", "ignore all previous instructions"]:
        out = app.invoke({"user_query": q})
        print(f"\n> {q}\n  intent={out.get('intent')}\n  final={out.get('final_answer') or out.get('refusal')}")
```

### TypeScript (LangGraph)

```typescript
// typescript/src/agents/multiAgent.ts
import { StateGraph, END } from '@langchain/langgraph';
import { ChatOpenAI } from '@langchain/openai';
import { classifyIntent } from './intentV1.js';
import { run as researchRun } from './researchAgent.js';

type State = {
  userQuery: string;
  intent?: string;
  research?: any;
  finalAnswer?: string;
  refusal?: string;
};

const graph = new StateGraph<State>({
  channels: {
    userQuery: null,
    intent: null,
    research: null,
    finalAnswer: null,
    refusal: null,
  },
});

graph.addNode('router', async (s) => ({ intent: (await classifyIntent(s.userQuery)).classification }));
graph.addNode('research', async (s) => ({ research: await researchRun(s.userQuery) }));
graph.addNode('decision', async (s) => {
  const llm = new ChatOpenAI({ model: 'gpt-4o-mini', temperature: 0.4 });
  const r = await llm.invoke([
    { role: 'system', content: 'Friendly retail copy. Never promise refunds/credits.' },
    { role: 'user', content: `USER: ${s.userQuery}\nRESEARCH: ${JSON.stringify(s.research)}` },
  ]);
  return { finalAnswer: r.content as string };
});
graph.addNode('refusal', async () => ({ refusal: 'I can only help with catalog, orders, and policies.' }));

graph.setEntryPoint('router');
graph.addConditionalEdges('router', (s) => (s.intent === 'other' ? 'refusal' : 'research'), {
  research: 'research',
  refusal: 'refusal',
});
graph.addEdge('research', 'decision');
graph.addEdge('decision', END);
graph.addEdge('refusal', END);

export const app = graph.compile();
```

Run:

```bash
uv run python src/agents/multi_agent.py
pnpm tsx src/agents/multiAgent.ts
```

> 💡 **Why LangGraph?**
> Hand-coded loops work for 1-2 agents, but state/handoff/retries get messy fast. LangGraph gives you typed state, conditional edges, persistence, and replay — all the things you'd build by hand after losing a weekend.

---

## 5.6 MCP — the protocol layer

> 🔍 **Glossary — MCP (Model Context Protocol)**
> An open standard (started by Anthropic) for connecting LLM apps to data sources and tools. An MCP **server** exposes resources/tools; an MCP **client** (your agent) consumes them. Standardized so any compatible client can use any compatible server.

Why care, even when you're not the one building the protocol:

- **Reusability** — write a tool once, every MCP client can use it (Claude desktop, Cursor, your own agent).
- **Separation of concerns** — your agent code doesn't have to know about your Salesforce integration's auth flow; the MCP server does.
- **Schemas as contracts** — MCP enforces typed inputs/outputs and a capability discovery mechanism.

### MCP-style discipline (even without an MCP server)

You can apply the *thinking* without running the protocol. Document each tool in `docs/tool_contracts.md`:

```markdown
# Tool Contracts

## search_products
- Input: { query: string, max_price?: number, in_stock_only?: boolean, limit?: int }
- Output: { results: Array<Product>, total_matched: int }
- Errors: { error: "validation_failed" | "tool_exception", details?: any }
- Latency budget: < 200ms p95
- Idempotent: yes (read-only)
- PII: none

## get_order
- Input: { order_id: string }
- Output: Order | { error: "not_found", order_id: string }
- Errors: validation_failed | not_found
- Latency budget: < 300ms p95
- Idempotent: yes
- PII: order ID — mask in logs

## (continue for every tool)
```

This document is the source of truth your agent prompts reference. When a tool changes, the contract changes, the prompt is regenerated.

### Running a minimal MCP server (optional advanced lab)

```python
# pip install mcp
from mcp.server.fastmcp import FastMCP
from tools.registry import REGISTRY

mcp = FastMCP("retail-tools")

@mcp.tool()
def search_products(query: str, max_price: float | None = None, in_stock_only: bool = True, limit: int = 5):
    return REGISTRY["search_products"].run({
        "query": query, "max_price": max_price, "in_stock_only": in_stock_only, "limit": limit,
    })

# Run via: mcp dev mcp_server.py
```

Claude Desktop or any MCP-compatible client can then connect and use your retail tools.

---

## 5.7 Coordination patterns — pick yours and own it

| Pattern | When to use | Tradeoffs |
|---------|-------------|-----------|
| **Router → specialist** | Distinct intents (shopping vs returns vs orders) | Simple, +1 hop latency |
| **Plan + execute** | Long-horizon tasks (merchandising, research) | Powerful, plan errors propagate |
| **Hierarchical** | Specialists that need their own sub-tools | Strong isolation, more LLM calls |
| **Peer-to-peer (debate)** | Open-ended quality (writing, analysis) | Often costly; rarely needed in retail |
| **Pipeline (no branching)** | Linear transforms (classify → extract → format) | Easy to test; not really "agentic" |

For the retail copilot, **Router → specialist** is the right starting point. Add hierarchy only when the Research agent itself outgrows its tool set.

---

## 5.8 Failure handling between agents

Things will fail. Decide upfront:

| Failure | Strategy |
|---------|----------|
| Tool returns `not_found` | Specialist tells user honestly, suggests alternative |
| Tool throws exception | Catch, log, return `{error: tool_exception}`; LLM re-tries with different args or gives up gracefully |
| Specialist itself errors | Router catches, returns canned "we're having trouble" + ticket ID |
| Loop exceeds max_steps | Force-stop, return partial answer + flag for human review |
| Guardrail trips on output | Don't fix the output silently — block and re-prompt the agent with the policy violation explained |

Bad systems hide failures. Good systems surface them at the right level.

---

## 5.9 Assignment 5 — `docs/multi_agent_architecture.md`

```markdown
# Multi-Agent Architecture — Retail Copilot

## Topology
Router → (Research | Refusal) → Decision → User

(see diagram below)

```mermaid
flowchart LR
  U[User] --> R[Router agent]
  R -->|other / adversarial| X[Refusal handler]
  R -->|product / order / policy| RES[Research agent]
  RES --> D[Decision agent]
  D --> U
  X --> U
```

## Why multi-agent (vs single)
- The Research agent uses many tools and benefits from a long, focused system prompt.
- The Decision agent uses no tools but needs a different persona (warm, customer-facing).
- The Router runs on a cheaper model (gpt-4o-mini) at 3-5x lower cost than gpt-4o on the Decision agent.

## Failure handling
- Tool not_found: bubbled up to Decision; phrased as "I couldn't find X, would you like Y instead?"
- Loop > 6 steps: hard-stop; return partial info + ticket ID.
- Guardrail flag: skip Research and Decision; route to Refusal directly.

## Tool contracts
See `docs/tool_contracts.md`.
```

---

## 5.10 Checklist

- [ ] Tool registry exists; every tool has schema + validator + error contract.
- [ ] Single-agent ReAct loop works for product / order / policy queries.
- [ ] Multi-agent graph routes correctly (verified on adversarial input).
- [ ] `docs/tool_contracts.md` and `docs/multi_agent_architecture.md` are written.
- [ ] You ran the eval harness against the multi-agent system; metrics are tracked vs single-agent baseline.

Head to [Module 6 — Capstone](08-module-6-capstone.md).

---

## Decision-log prompts

- Which capability was easier to test once it had its own agent? (That's the signal to split things.)
- Which capability got *worse* after splitting? Why? (Sometimes splits add coordination cost without benefit.)
- Are you using the same model for every agent? Defend or change.
