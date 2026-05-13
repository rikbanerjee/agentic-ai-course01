# Module 0 — Foundations & Orientation

> **Time**: 2–3 days (4–6 hours total)
> **Goal**: Build the vocabulary and mental model you need before writing a single line of agent code.
> **Deliverable**: `docs/agentic_ai_map.md` — a 1–2 page map of where agentic AI fits in retail.

---

## 0.1 Why "agentic" is different from "GenAI"

If you've used ChatGPT or built a prompt-based feature, you've used **GenAI** (generative AI): you send text in, you get text out. The system is essentially a stateless function: `prompt → completion`.

**Agentic AI** is different. An agent has three properties a plain prompt doesn't:

1. **It can call tools.** When the user asks "What's the return policy on order #1234?", an agent can call a `get_order(order_id)` function, get real data, and answer with facts — not guesses.
2. **It can loop.** It decides whether one tool call is enough, or whether it needs to retrieve more, search again, or ask a clarifying question.
3. **It can plan.** It can break a goal ("find me a hiking jacket under $150 that ships in 2 days") into sub-goals and execute them.

> 🔍 **Glossary — Agent**
> A program that uses an LLM to decide *what to do next* given a goal, a set of tools, and prior observations. The LLM is the agent's "brain"; the loop around it is the agent's "body."

The simplest mental model:

```
┌─────────────────────────────────────────────────┐
│              The Agent Loop                     │
│                                                 │
│  goal ─▶ [LLM reasons] ─▶ pick a tool          │
│            ▲                  │                 │
│            │                  ▼                 │
│         observation ◀── [run the tool]          │
│                                                 │
│  ↻ repeat until LLM says "I'm done, here's      │
│    the final answer"                            │
└─────────────────────────────────────────────────┘
```

This loop is called the **ReAct loop** (Reason + Act). Almost every agent framework — OpenAI Agents SDK, LangGraph, Anthropic's tool-use API — implements some version of it.

> 💡 **Why this matters**
> Every architectural choice you make later (when to add a tool, when to loop, when to stop) flows from this loop. If you can draw this diagram from memory, you're 30% of the way to building a real agent.

---

## 0.2 The four agentic patterns

Most production agentic systems fall into one of four patterns. You'll meet all four in this course. Knowing which pattern fits a problem is the single most important "problem-first" skill.

### Pattern 1 — Reflexive Agent (single-step tool use)

The LLM is given tools, picks **one**, runs it, and returns. No loop.

- **Example**: "What's the weather in NYC?" → calls `get_weather('NYC')` → returns.
- **Use when**: the task is a single lookup with predictable structure.
- **Don't use when**: the task needs multiple steps, judgment, or recovery from errors.

### Pattern 2 — ReAct Agent (reason + act loop)

The LLM reasons, picks a tool, observes the result, then decides whether to act again or finish.

- **Example**: "Find me a jacket under $150 that ships in 2 days" → searches catalog → filters → checks shipping → answers.
- **Use when**: the task needs 2–10 steps with branching decisions.
- **Don't use when**: the workflow is fully deterministic (use a normal pipeline) or needs >20 steps (use a planner).

### Pattern 3 — Plan-and-Execute Agent

A "planner" LLM produces a multi-step plan upfront; an "executor" runs each step. Often a separate LLM (or the same one with a different prompt) reviews the plan first.

- **Example**: "Compile a competitive analysis of 5 jacket brands and recommend our top 3 SKUs to feature." Step 1: identify brands. Step 2: pull product data for each. Step 3: score. Step 4: write summary.
- **Use when**: tasks are long-horizon, have predictable sub-structure, or you want to inspect the plan before execution.
- **Don't use when**: speed matters and the task fits in a ReAct loop.

### Pattern 4 — Multi-Agent System

Multiple specialized agents collaborate. Typically one **router** or **orchestrator** delegates to **specialist** agents, each with its own toolset and prompt.

- **Example**: A retail copilot with a Router (decides "is this a product search, a returns question, or an order status query?"), a Research agent (does the RAG), and a Decision agent (applies business rules).
- **Use when**: responsibilities cleanly split, you need different prompts/tools per role, or you want to scale parts independently.
- **Don't use when**: a single ReAct agent could do the job — multi-agent adds latency and complexity.

> ⚠️ **Common mistake**
> Beginners reach for multi-agent too early because it sounds sophisticated. It almost always makes things slower and harder to debug. Start with ReAct. Split into multi-agent **only** when a single agent's prompt becomes a tangled mess of conditionals.

---

## 0.3 Tools, RAG, and MCP — the supporting cast

Three concepts you'll keep meeting:

### Tool

> 🔍 **Glossary — Tool**
> A function the agent can call. It has a name, a description (which the LLM reads to decide when to use it), typed inputs, and typed outputs.

Example tool definition (Python, OpenAI SDK style):

```python
{
    "type": "function",
    "function": {
        "name": "search_products",
        "description": "Search the retail catalog for products matching a natural-language query. Returns up to 10 products with SKU, name, price, and stock.",
        "parameters": {
            "type": "object",
            "properties": {
                "query": {"type": "string", "description": "Free-text product description"},
                "max_price": {"type": "number", "description": "Optional price ceiling"},
                "in_stock_only": {"type": "boolean", "default": True}
            },
            "required": ["query"]
        }
    }
}
```

The LLM reads `description` to know *when* to call this. Bad descriptions = bad tool selection. **Treat tool descriptions like product copy** — they are the most overlooked lever in agent quality.

### RAG (Retrieval-Augmented Generation)

> 🔍 **Glossary — RAG**
> A pattern where, before the LLM answers, you retrieve relevant documents from a knowledge base (usually via a vector database) and stuff them into the prompt as context.

Why it exists: LLMs don't know your private catalog, your returns policy, or your latest pricing. RAG injects that knowledge at query time.

**Traditional RAG** (one-shot):

```
user query → embed query → top-k similar docs → stuff into prompt → answer
```

**Agentic RAG** (iterative):

```
user query → agent decides if retrieval is needed
            → if yes: agent reformulates query, retrieves, evaluates results,
              maybe retrieves again with a different query, then answers
            → if low confidence: agent reflects, retries, or asks user
```

You'll build both versions in Module 4.

### MCP (Model Context Protocol)

> 🔍 **Glossary — MCP**
> An open protocol (championed by Anthropic, now widely supported) that standardizes how LLM apps connect to external data sources and tools. Think of it as "USB for AI tools."

Why it matters: instead of every app implementing custom integrations for Slack, GitHub, your CRM, etc., MCP servers expose those resources behind a standard interface. Any MCP-compatible agent can use any MCP server.

In practice, you'll build **MCP-style** services in Module 5 — meaning your tools follow MCP's discipline (typed schemas, clear error contracts, capability discovery) even if you don't run an actual MCP server.

---

## 0.4 Problem-first thinking — the most important habit

The course's title says "Problem-First Approach." Here's what that means concretely.

A bad design conversation starts with: *"Let's build an agent that uses LangGraph and Pinecone and..."*

A good one starts with:

1. **Who is the user?** (Retail merchandiser? Shopper? Support agent?)
2. **What's the job-to-be-done?** (Find products that match constraints, in under 5 seconds.)
3. **What's the current baseline?** (A keyword search that returns 200 SKUs and the user manually filters.)
4. **Where does AI add value?** (Understanding fuzzy intent: "warm but not bulky," "good for kids.")
5. **Where does AI *not* belong?** (Final price calculation, inventory deduction, payment — those must stay deterministic.)
6. **What are the hard constraints?** Latency target (p95 < 3s?), cost ceiling ($0.02/query?), accuracy floor (90% of returned products must actually match the constraint?).
7. **What's the eval that proves we shipped?**

> 💡 **Why this matters**
> Every production agentic system that fails does so because someone skipped step 6 or step 7. Models get cheaper and smarter; the discipline of constraint-driven design is what compounds.

---

## 0.5 Map the four patterns to retail scenarios

For your deliverable, sketch one concrete retail scenario for each pattern. Here are examples to anchor on; come up with your own.

| Pattern | Retail scenario |
|---------|----------------|
| Reflexive | "What's the price of SKU-9871?" → single `get_product(sku)` call |
| ReAct | "I want a waterproof jacket under $150, in stock, men's medium" → search → filter → check stock → answer |
| Plan-and-execute | "Draft a fall merchandising plan for outerwear: pick 10 hero SKUs, write headlines, suggest bundles" → multi-step plan with intermediate review |
| Multi-agent | A customer copilot that routes: returns → Returns agent (policy RAG); shopping → Research agent (catalog RAG + Decision agent for promo rules) |

---

## 0.6 Deliverable — `docs/agentic_ai_map.md`

Create a file with this structure:

```markdown
# Agentic AI Map — Retail Copilot

## User journeys
1. Shopper finds a product matching fuzzy constraints
2. Shopper checks order status / returns
3. Merchandiser explores catalog gaps

## Per-journey design notes

### Journey 1: Product discovery
- Agent pattern: ReAct
- Tools: search_products, get_product, check_shipping, check_promotions
- RAG needed? Yes (product descriptions, reviews)
- Hard constraints: p95 latency < 3s, cost < $0.03/query
- Failure modes: out-of-stock products in answer, hallucinated SKUs

### Journey 2: Order status / returns
- Agent pattern: Multi-agent (Router + Returns specialist)
- Tools: get_order, get_return_policy (RAG over policy docs)
- ...

### Journey 3: Merchandiser catalog exploration
- Agent pattern: Plan-and-execute
- ...
```

This document will evolve. You'll update it after every module.

---

## 0.7 Checklist before moving on

- [ ] You can draw the agent loop from memory.
- [ ] You can name the four agent patterns and give a retail example of each.
- [ ] You understand the difference between a Tool, RAG, and MCP.
- [ ] You wrote `docs/agentic_ai_map.md` (even if rough).
- [ ] You can explain to a non-engineer *why* you'd ever pick "no agent" for a feature.

If all checked, head to [Module 1 — Environment & Tooling Setup](01-environment-setup.md).

---

## Decision-log prompts

Add these to `docs/decision_log.md`:

- Which retail sub-domain are you targeting first (discovery, support, merchandising)? Why?
- Which agent pattern do you think your capstone will mostly use? Why?
- What's one user journey you considered and rejected, and why?
