# Framework Cheatsheet

> Quick reference for picking — and switching between — the major agent frameworks.

You don't need to learn all of these. Most production stacks pick one or two. This cheatsheet is to help you choose, and to give you a starter snippet in each so you can sanity-check what feels right.

---

## The five candidates

| Framework | Language | Best for | Maturity | Course usage |
|-----------|----------|----------|----------|--------------|
| **OpenAI SDK** (direct) | Python + TS | Maximum control, simple agents | ⭐⭐⭐⭐⭐ | Modules 1, 3, 5 |
| **Anthropic SDK** (direct) | Python + TS | Claude users, prompt caching, MCP | ⭐⭐⭐⭐⭐ | Optional alternate |
| **LangChain** | Python + TS | RAG + chains, lots of integrations | ⭐⭐⭐⭐ | Module 4 |
| **LangGraph** | Python + TS | Multi-agent state machines | ⭐⭐⭐⭐ | Module 5 |
| **Vercel AI SDK** | TS only | Streaming chat UIs, Next.js | ⭐⭐⭐⭐⭐ | Module 11 (UI) |
| **LlamaIndex** | Python + TS | Heavy retrieval / data workflows | ⭐⭐⭐⭐ | Optional alternate |
| **OpenAI Agents SDK** | Python + TS | Officially-supported agent loops | ⭐⭐⭐ (newer) | Optional |
| **Smolagents** (HF) | Python | Minimal, hackable | ⭐⭐⭐ | Optional |

---

## Decision tree

```
Start here:
│
├─ Do you need to ship a streaming chat UI in TS?
│    → Vercel AI SDK for the UI layer + any backend
│
├─ Is your system primarily RAG over many sources?
│    → LangChain (retrievers, document loaders, chains) OR LlamaIndex
│
├─ Multi-agent with branching / cycles / human-in-loop?
│    → LangGraph
│
├─ One agent, a few tools, want minimum magic?
│    → OpenAI/Anthropic SDK directly
│
├─ Heavy reliance on Claude features (long context, caching, MCP)?
│    → Anthropic SDK directly
│
└─ All of the above
     → Mix: Vercel AI SDK at the edge + LangGraph for orchestration +
       OpenAI SDK for direct calls when you need the control
```

---

## Side-by-side: "ask the user query with one tool"

### OpenAI SDK (Python)

```python
from openai import OpenAI
client = OpenAI()

tools = [{
    "type": "function",
    "function": {
        "name": "get_order",
        "description": "Get order status by ID",
        "parameters": {"type": "object", "properties": {"order_id": {"type":"string"}}, "required": ["order_id"]},
    },
}]

def get_order(order_id): return {"order_id": order_id, "status": "shipped"}

messages = [{"role":"user","content":"where is ORD-4456?"}]
while True:
    r = client.chat.completions.create(model="gpt-4o-mini", messages=messages, tools=tools)
    msg = r.choices[0].message
    messages.append(msg.model_dump(exclude_none=True))
    if not msg.tool_calls: break
    for tc in msg.tool_calls:
        result = get_order(**json.loads(tc.function.arguments))
        messages.append({"role":"tool","tool_call_id":tc.id,"content":json.dumps(result)})
print(messages[-1]["content"])
```

### Anthropic SDK (Python)

```python
from anthropic import Anthropic
client = Anthropic()

tools = [{
    "name": "get_order",
    "description": "Get order status by ID",
    "input_schema": {"type":"object","properties":{"order_id":{"type":"string"}},"required":["order_id"]},
}]

messages = [{"role":"user","content":"where is ORD-4456?"}]
while True:
    r = client.messages.create(model="claude-sonnet-4-6", max_tokens=1024, tools=tools, messages=messages)
    messages.append({"role":"assistant","content": r.content})
    if r.stop_reason != "tool_use": break
    tool_results = []
    for block in r.content:
        if block.type == "tool_use":
            result = {"order_id": block.input["order_id"], "status": "shipped"}
            tool_results.append({"type":"tool_result","tool_use_id":block.id,"content":json.dumps(result)})
    messages.append({"role":"user","content": tool_results})
print(r.content[-1].text)
```

### LangChain (Python)

```python
from langchain_openai import ChatOpenAI
from langchain_core.tools import tool

@tool
def get_order(order_id: str) -> dict:
    """Get order status by ID."""
    return {"order_id": order_id, "status": "shipped"}

llm = ChatOpenAI(model="gpt-4o-mini").bind_tools([get_order])
# Simple loop helper:
from langgraph.prebuilt import create_react_agent
agent = create_react_agent(llm, [get_order])
result = agent.invoke({"messages":[{"role":"user","content":"where is ORD-4456?"}]})
print(result["messages"][-1].content)
```

### LangGraph (Python — multi-step state)

```python
from langgraph.graph import StateGraph, END
from typing import TypedDict, Annotated
from langgraph.graph.message import add_messages
from langchain_openai import ChatOpenAI

class State(TypedDict):
    messages: Annotated[list, add_messages]

def llm_node(s: State) -> dict:
    msg = ChatOpenAI(model="gpt-4o-mini").bind_tools([get_order]).invoke(s["messages"])
    return {"messages": [msg]}

def tool_node(s: State) -> dict:
    last = s["messages"][-1]
    results = []
    for tc in last.tool_calls or []:
        r = get_order.invoke(tc["args"])
        results.append({"role":"tool","tool_call_id":tc["id"],"content":str(r)})
    return {"messages": results}

g = StateGraph(State)
g.add_node("llm", llm_node); g.add_node("tool", tool_node)
g.set_entry_point("llm")
g.add_conditional_edges("llm", lambda s: "tool" if s["messages"][-1].tool_calls else END, {"tool":"tool", END: END})
g.add_edge("tool", "llm")
app = g.compile()
```

### Vercel AI SDK (TypeScript)

```typescript
import { generateText, tool } from 'ai';
import { openai } from '@ai-sdk/openai';
import { z } from 'zod';

const result = await generateText({
  model: openai('gpt-4o-mini'),
  tools: {
    get_order: tool({
      description: 'Get order status by ID',
      parameters: z.object({ order_id: z.string() }),
      execute: async ({ order_id }) => ({ order_id, status: 'shipped' }),
    }),
  },
  prompt: 'Where is ORD-4456?',
  maxSteps: 5,
});
console.log(result.text);
```

### OpenAI SDK (TypeScript)

```typescript
import OpenAI from 'openai';
const client = new OpenAI();
const tools = [{
  type: 'function' as const,
  function: { name: 'get_order', description: 'Get order status', parameters: { type:'object', properties: { order_id: { type:'string' } }, required: ['order_id'] }},
}];
const messages: any[] = [{ role: 'user', content: 'where is ORD-4456?' }];
while (true) {
  const r = await client.chat.completions.create({ model: 'gpt-4o-mini', messages, tools });
  const msg = r.choices[0].message;
  messages.push(msg);
  if (!msg.tool_calls?.length) { console.log(msg.content); break; }
  for (const tc of msg.tool_calls) {
    const args = JSON.parse(tc.function.arguments);
    const result = { order_id: args.order_id, status: 'shipped' };
    messages.push({ role: 'tool', tool_call_id: tc.id, content: JSON.stringify(result) });
  }
}
```

---

## Where each framework shines (and stumbles)

### OpenAI / Anthropic SDK (direct)

- ✅ **Shines**: control, predictability, no extra abstraction to debug.
- ❌ **Stumbles**: lots of boilerplate for multi-step loops; you re-invent state management.

### LangChain

- ✅ **Shines**: huge integration library (every retriever, every doc loader), composable chains.
- ❌ **Stumbles**: abstractions change between versions; deep stack traces; some "magic" that's hard to debug.

### LangGraph

- ✅ **Shines**: typed state, conditional edges, persistence, replay, visualization, human-in-loop.
- ❌ **Stumbles**: steep learning curve for graph thinking; can be overkill for linear flows.

### Vercel AI SDK

- ✅ **Shines**: ergonomic streaming, beautiful React/Next.js integration, multi-provider abstraction.
- ❌ **Stumbles**: TS-only; less suitable for heavy backend orchestration; newer ecosystem.

### LlamaIndex

- ✅ **Shines**: best-in-class retrieval primitives, query engines, sub-questions, agents over data.
- ❌ **Stumbles**: docs span multiple paradigms (legacy + modern); duplication with LangChain.

### OpenAI Agents SDK

- ✅ **Shines**: official, opinionated, simple multi-agent handoffs.
- ❌ **Stumbles**: newer (2024+); fewer integrations than LangChain; OpenAI-flavored.

---

## How to actually pick

1. **What's your team using?** Pick what others can review.
2. **What are you optimizing?** Streaming UI → Vercel SDK. State machines → LangGraph. RAG breadth → LangChain/LlamaIndex.
3. **How much abstraction can you debug?** If you're 2 weeks into LLMs, prefer the SDK directly. Add frameworks when you feel the pain.
4. **Will you switch providers?** Frameworks make swap easy; raw SDKs lock you in (a little). If multi-provider matters, lean toward Vercel AI SDK or LangChain.

---

## A pragmatic full-stack pick

For most teams building something like the retail copilot, a balanced stack:

- **Frontend**: Next.js + Vercel AI SDK (streaming, beautiful UX).
- **Agent orchestration**: LangGraph (state, multi-agent, retries).
- **Retrieval**: LangChain's loaders + Chroma (or upgrade to Qdrant later).
- **Direct LLM calls** (intent, judges, evals): OpenAI/Anthropic SDK directly — small functions, no framework.
- **Observability**: Langfuse.

That's the stack the course's V2 capstone effectively mirrors.

---

## Quick swap guide

If you're already coded against framework A and want to try B:

| From | To | Effort |
|------|----|--------|
| OpenAI SDK → LangChain | Wrap tools with `@tool`, use `create_react_agent` | ~1 hour |
| LangChain → LangGraph | Recompose chain as nodes, add state | ~half day |
| OpenAI SDK → Vercel AI SDK | Replace `chat.completions.create` with `generateText` / `streamText` | ~1 hour |
| Python → TS rewrite | Mirror folder structure, port tool defs to Zod, port prompts as-is | ~1 week for a full app |

The data layer (Chroma, evals, prompts in YAML) stays the same.
