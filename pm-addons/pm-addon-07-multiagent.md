# PM Addon — Weekly Module 5: Multi-Agent & MCP

> **Pairs with**: [Weekly Module 5](../course/07-module-5-multi-agent-mcp.md)
> **Read time**: 12 minutes
> **You'll be able to**: decide when multi-agent is worth the tax, write a multi-agent architecture doc, and define tool contracts engineering can build against.

---

## The 60-second summary

A multi-agent system has multiple specialized agents collaborating. Typical retail topology: Router → Research → Decision. Multi-agent is **not** sophistication for its own sake — it's a coordination tax you pay when separation of concerns earns its keep.

Three indicators it's time to split:

1. Your system prompt is over ~500 tokens and full of conditionals.
2. One prompt change breaks unrelated capabilities.
3. Different parts want different models (cheap classifier + expensive synthesizer).

Tools are the agent's hands. Every tool needs a typed contract: inputs (JSON schema), outputs (success + error shapes), latency budget, idempotency status.

MCP (Model Context Protocol) is the emerging standard for connecting agents to tools. Even without running an MCP server, **MCP-style discipline** (typed schemas, capability discovery) makes your tools reusable.

PM artifacts: [multi-agent architecture](../pm-artifacts/multi-agent-architecture.md), [tool contracts](../pm-artifacts/tool-contracts.md).

---

## Why this matters for the product

Multi-agent decisions get re-litigated weekly if not written down. They also balloon scope: each new agent is a new prompt, new evals, new latency budget.

The PM job here is to:

1. **Resist multi-agent until pain proves it's worth it.**
2. **Lock the topology before engineering builds it.**
3. **Demand tool contracts** — they prevent half a class of production bugs.

---

## What PMs decide vs what engineers decide

| PM owns | Engineering owns |
|---------|------------------|
| Whether to split | How to split (state passing, framework choice) |
| Topology (who calls whom) | Implementation (LangGraph, raw code) |
| Tools to expose (capability set) | Tool implementations |
| Tools NOT to expose (authority boundary) | Schema definitions |
| Latency budget per agent | How to meet it |

---

## Stakeholder talking points

When eng proposes **"let's split into 5 agents"**:

> "Walk me through what each agent does and why it has to be its own agent rather than a section of one prompt. If the answer is 'cleaner code,' that's not enough — coordination has latency and cost. If the answer is 'this agent gets a different model and a different evaluator,' we're talking."

When CX asks **"can we add an agent that handles refunds?"**:

> "An agent doesn't add authority. Refunds need authority. Even if we built the agent, the tool it would call doesn't exist — and authoring that tool means legal + finance sign-off, not just engineering. Want me to start that conversation, or are we good with escalating refunds to humans?"

When eng asks **"can I add a new tool without docs?"**:

> "No. Every tool needs the contract (inputs, outputs, errors, latency, idempotency, PII). That's a PR template requirement. Otherwise we'll have undocumented agent behavior in a month."

When marketing asks **"can the agent call our newsletter signup API?"**:

> "Two questions. One: is signing up a user the kind of action we want the AI deciding to take? Two: do we have explicit consent flow? If both no, then no. If both yes, then we add a tool with strict criteria."

---

## The "should we go multi-agent" framework

Use this in scope conversations:

| Question | If yes → |
|----------|----------|
| Does the system prompt > 500 tokens and full of conditionals? | Consider split |
| Does each capability want a different model? | Strong indicator to split |
| Do capabilities have independent SLAs? | Strong indicator |
| Does one prompt change break unrelated things? | Real pain, split |
| Do you want to evaluate capabilities independently? | Split |
| All "yes" but team is small? | Maybe defer; the coordination tax is real |

Default: **stay single agent until you have three concrete pain points**. Then split.

---

## Multi-agent topology choices

| Pattern | When |
|---------|------|
| Router → specialist | Distinct intents (shopping vs returns vs orders). Most retail starts here. |
| Plan → execute | Long-horizon tasks (research, merchandising plans). |
| Hierarchical | Specialists with their own sub-tools/agents. Save for v2+. |
| Peer-to-peer (debate) | Open-ended quality (writing). Rarely useful in retail. |
| Pipeline (linear) | classify → extract → format. Often not "agentic" at all. |

For retail copilot: Router → specialist is right for v1. Hierarchical comes when the Research agent itself outgrows its tool set.

---

## Tool contracts — the most underrated artifact

Every tool documented:

```markdown
## tool_name
Purpose: one sentence, the LLM reads this.
Inputs: JSON schema with typed fields + descriptions.
Outputs: success shape + error shapes.
Errors: enumerated.
Idempotent: yes/no.
Latency budget: p95 SLA.
PII: what's inside, what's masked.
Side effects: anything beyond a read.
```

Why it matters:

- Vague description → LLM calls tool at wrong time.
- No error contract → agent loops on retry instead of giving up gracefully.
- No latency budget → no way to tell when a tool degrades.

See [tool contracts example](../pm-artifacts/tool-contracts.md).

---

## Risk lenses

| Risk | Smell | Mitigation |
|------|-------|------------|
| **Premature multi-agent** | Architecture has 4 agents but a single ReAct could do it | Force a "what pain made you split" answer per agent |
| **Tool sprawl** | New tool added with every feature; no allowlist | Quarterly tool audit; remove unused |
| **Authority creep** | Tools that mutate state shipping without compliance review | Mutation tools require Security sign-off |
| **Cross-agent prompt injection** | Research output contains "ignore previous" and Decision follows it | Sanitize inter-agent messages |
| **Untraced handoffs** | A failure between agents can't be diagnosed | Each agent emits a Langfuse span; trace tree visible |

---

## Reading order in the technical module

Worth reading: §5.1 (when to add a second agent), §5.2 (classic topology), §5.6 (MCP-style discipline), §5.9 (architecture doc assignment).

Skim: §5.3/§5.4 (tool registry and ReAct loop code).

Skip: code internals if not implementing.

---

## Worked example: a scope conversation about merchandising

Merch lead asks: *"Can we build an agent that drafts a fall outerwear merchandising plan?"*

PM playbook:

1. **Frame the pattern.** This is plan-and-execute, not ReAct. It's a different architecture.
2. **Frame the tool set.** Needs product analytics, trend data, internal merchandising guidelines. Some don't exist as tools yet.
3. **Frame the safety surface.** Internal-only audience reduces brand risk; but plan quality matters for business decisions.
4. **Estimate.** 4 weeks for v1 with existing tools; 6-8 weeks if new tools needed.
5. **Compare to v1 capability.** This is materially different from the shopper copilot — separate roadmap entry, not a feature.

That conversation prevents "while we're at it, can the shopper copilot also do merchandising plans?" — which would 3× the scope of v1.

---

## After this module

Move to [PM Addon 08 — Capstone](pm-addon-08-capstone.md). The capstone is where you defend everything you built.
