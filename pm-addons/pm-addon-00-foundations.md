# PM Addon — Module 0: Foundations

> **Pairs with**: [Module 0 — Foundations & Orientation](../course/00-foundations.md)
> **Read time**: 12 minutes
> **You'll be able to**: explain to a non-engineer what an "agent" is, when to use one (and when not to), and frame a problem in a way engineering can build against.

---

## The 60-second summary

Three concepts you need from this module:

1. **Agents are LLMs in a loop with tools.** A chatbot answers from training memory. An agent can *act* — call a function, look something up, decide whether to act again. Almost all production "AI features" are some flavor of agent.
2. **There are four agent patterns.** Reflexive (one tool call), ReAct (loop with reasoning), Plan-and-execute (plan upfront), Multi-agent (specialists coordinating). Beginners reach for multi-agent too early; intermediate teams default to ReAct.
3. **Problem-first beats tech-first.** Start with the user, the constraints, and the eval — not "let's use LangGraph and Pinecone."

If you understand those three, you can hold your end of a scoping conversation with engineering.

---

## Why this matters for the product

A surprising amount of AI feature failure happens at this layer — before any code. Common patterns:

- **Building a multi-agent system when one prompt would do** → 4× the cost, 2× the latency, no quality lift.
- **Picking a pattern based on a blog post** → architecture doesn't match the actual user journey.
- **Skipping the "where AI doesn't belong" question** → you ship something that can promise refunds it has no authority to make.

The module gives you the vocabulary to push back when an enthusiastic engineer (or vendor) proposes the wrong pattern.

---

## What PMs decide vs what engineers decide

| PM owns | Engineering owns |
|---------|------------------|
| Which user journeys are in scope | Which pattern fits each journey |
| Where AI is explicitly NOT allowed to act | Tool schemas, error contracts |
| Hard constraints (latency, cost, safety thresholds) | How to meet those constraints |
| Success metrics (business + quality) | Eval methodology to measure them |
| Deferral decisions ("v1 vs v2") | Effort estimates |

If you're a Technical PM, you'll co-own the right column too. Either way, **never let "where AI doesn't belong" be decided in a Slack thread**. Write it down.

---

## Stakeholder talking points

When someone asks **"can we just use AI for this?"**:

> "Maybe. Let's frame this as a problem first. Who's the user, what's the job-to-be-done, what's the current baseline, and what's the constraint that's not negotiable? Then we'll pick a pattern."

When someone asks **"why aren't we using a multi-agent system like Company X?"**:

> "We will, if a single agent's prompt becomes unmanageable. Multi-agent isn't a marker of sophistication — it's a coordination tax we pay when separation of concerns earns its keep. Today our concerns aren't separated enough to justify it."

When someone asks **"can the agent just do anything the user asks?"**:

> "It can do anything we've given it tools for. Tools are scoped by us. The agent can never act outside that scope — that's the design. Things like refunds, account changes, or pricing decisions stay with deterministic systems."

When someone asks **"how do we know it won't hallucinate?"**:

> "We don't, fully. We minimize it three ways: every claim has to come from a tool result or a cited document (grounding), we measure hallucination rate as a metric (evals), and we have a guardrail layer that blocks invented promises (filters). I'll show you the eval table."

---

## Risk lenses

Risks to watch coming out of foundations:

| Risk | Smell | Watch for |
|------|-------|-----------|
| **Pattern mismatch** | Engineers spec multi-agent for a single-tool problem | Slack threads with "should we add another agent for…" |
| **Scope creep** | "While we're at it, can the agent also…" | Stakeholders pitching capabilities not in §5 of the system memo |
| **AI bypass** | Eng or PM bypasses the "AI doesn't belong here" list | Look for tool definitions that mutate state |
| **Pattern thrash** | Switching from ReAct to multi-agent mid-build | Architecture changes without an entry in the decision log |

---

## The 4-pattern decision framework

Use this in scoping conversations:

| Question | If "yes" → consider |
|----------|--------------------|
| Is the task a single lookup with predictable structure? | Reflexive |
| Does the task need 2-10 steps with branching? | ReAct |
| Is the task long-horizon with predictable sub-structure? | Plan-and-execute |
| Do responsibilities cleanly split (and engineering pain confirms it)? | Multi-agent |

If you're not sure, **start with ReAct**. Split into multi-agent only when one prompt becomes a tangled mess of conditionals (a real felt pain, not a hypothetical).

---

## Reading order in the technical module

Pages worth reading in full: §0.1, §0.2, §0.4 (problem-first habit).

Skim: §0.3 (Tools/RAG/MCP — you'll meet these again in deeper modules).

Skip the table-of-mappings if you trust the verbal explanation.

The deliverable (`docs/agentic_ai_map.md`) is something PMs should personally draft — it's not engineering work.

---

## Worked example: framing a real scope conversation

You're in a kickoff for a "shopping assistant." Apply the framework:

1. **User journey**: shopper wants to find a product matching fuzzy constraints. ✅ This is a real translation gap.
2. **Pattern**: 2-10 steps (extract constraints → search → filter → maybe re-search) → **ReAct** fits. Multi-agent would be premature.
3. **Where AI doesn't belong**: pricing, discounts, refunds, account modification, image search. (Write all five down. Defend them weekly.)
4. **Constraints**: p95 < 4s, cost < $0.03/query, hallucination < 1%.
5. **Eval**: 30-50 cases including ≥ 20% adversarial. (You don't ship without this.)

That five-step list is the conversation. The artifacts ([system memo](../pm-artifacts/system-memo.md), [business case](../pm-artifacts/business-case.md)) are how you make it durable.

---

## Pitfalls to avoid

- **Letting "AI vibes" replace metrics.** "Feels smarter" is not a metric.
- **Approving scope based on demos.** Demos always work; production is the test.
- **Skipping the deliverable.** The `agentic_ai_map.md` doc forces clarity. Don't skip it.
- **Treating multi-agent as a status symbol.** It's a coordination tax.

---

## After this module

Read [PM Addon 03 — GenAI & Agentic Apps](pm-addon-03-genai.md) and then start drafting your own [system memo](../pm-artifacts/system-memo.md).
