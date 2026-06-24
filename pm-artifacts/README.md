# PM Artifacts Pack

Filled-out, opinionated examples of the artifacts a PM (or engineer wearing a PM hat) actually has to produce when shipping an agentic AI feature. Every file is a working example based on the course's retail copilot — copy, adapt, ship.

These are **not** blank templates with `[INSERT HERE]` placeholders. They are completed v1 documents showing what "good enough to circulate" looks like. The bracketed callouts (`> 💡 Why this section exists`) explain *why* each section is in the doc, so you know what to keep, what to cut, and what to adapt for your context.

---

## What's in the pack

### Documents that ship with the product

| File | Audience | When to write it |
|------|----------|------------------|
| [Business case](business-case.md) | Exec sponsor, finance | Before any engineering kicks off |
| [System memo](system-memo.md) | Cross-functional team | Week 1, with engineering |
| [RFC: Agentic Search v1](rfc-agentic-search.md) | Engineering leadership | Before architecture lock-in |
| [Context policy](context-policy.md) | Legal, security, engineering | Week 2; review every quarter |
| [Eval & guardrails spec](eval-and-guardrails-spec.md) | QA, engineering, leadership | Week 3; lives forever |
| [RAG design](rag-design.md) | Engineering, infra | Week 4 |
| [Multi-agent architecture](multi-agent-architecture.md) | Engineering, ops | Week 5 |
| [Tool contracts](tool-contracts.md) | Engineering, integrations team | Week 5; lives forever |
| [Capstone design review](capstone-design-review.md) | Leadership, demo audience | Week 6 |
| [Demo script](demo-script.md) | You + your stakeholders | Right before the demo |
| [Decision log](decision-log.md) | Future you, new joiners | Updated continuously |
| [Stakeholder one-pager](stakeholder-one-pager.md) | Anyone outside the team | Whenever someone asks "what is this?" |

---

## How a PM should use this pack

1. **Read the business case first.** It frames the why.
2. **Read the system memo second.** It frames the what and the boundaries.
3. **Skim the eval & guardrails spec.** This is your contract with quality.
4. **Cherry-pick the rest** as your project hits the matching milestone.

For each doc you actually circulate, keep the structure but rewrite the content in your voice and for your domain. The structure is the durable asset; the content has to be yours.

---

## How an engineer should use this pack

1. **Read the RFC** — it's the closest thing to a real architectural artifact.
2. **Use the tool contracts and RAG design files** as templates when you write yours.
3. **Steal the decision log format** — it's the single most underrated engineering doc.

---

## How a non-technical PM uses this pack to look (and be) competent

These artifacts are the language that earns you credibility with engineering and leadership simultaneously. When you say:

> "I drafted a system memo with the constraints we agreed on, and I'd like engineering to red-team the eval spec before we finalize scope."

…you've named two artifacts that signal you understand the discipline. The PM artifacts in this pack are your vocabulary kit.

---

## Conventions

- All artifacts are written as if the company is a fictional outdoor retailer ("Northcrest") and the product is "Retail Copilot."
- Dates are anchored to the course's reference month (May 2026).
- Costs/latencies are realistic for the model families in use.
- Every artifact has a tiny "How this gets out of date" note at the bottom — read it.
