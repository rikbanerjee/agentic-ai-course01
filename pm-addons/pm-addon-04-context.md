# PM Addon — Weekly Module 2: Context Engineering

> **Pairs with**: [Weekly Module 2](../course/04-module-2-context-engineering.md)
> **Read time**: 12 minutes
> **You'll be able to**: write a context policy, judge whether prompts are over- or under-engineered, and frame the privacy story for legal.

---

## The 60-second summary

"Context engineering" = designing the entire pipeline of information the model sees per call. Four pillars in every agentic system:

1. **Instructions** — system prompt, rules, format.
2. **Context** — retrieved docs, conversation history, examples.
3. **Reasoning** — decomposition, chain-of-thought, reflection.
4. **Tools** — function calls.

Three patterns that recur:

- **Structured intermediates** — pass JSON between steps, not raw text.
- **Selective recall** — summarize old history, keep latest verbatim, persist durable facts separately.
- **Retrieval + reranking** — fetch broadly, narrow with a small model.

The PM artifact: a **context policy** ([example](../pm-artifacts/context-policy.md)).

---

## Why this matters for the product

Context is where the most tuning happens after launch. A new policy doc gets added → context policy updates. A new privacy rule from legal → context policy updates. Every PM-level "can we change how the agent talks about X?" lands here.

It's also the source of the longest, most repetitive arguments with legal and security. The context policy is your peace treaty.

---

## What PMs decide vs what engineers decide

| PM owns | Engineering owns |
|---------|------------------|
| What user data can enter context | The PII detection regex set |
| Retention rules for prompts and logs | Where logs are stored |
| Whether to summarize or truncate history | The summarization prompt |
| Versioned prompt approvals | YAML structure and loader |
| Tradeoffs between context size and cost | How to count tokens |

---

## Stakeholder talking points

When legal asks **"what user data goes to OpenAI / Google?"**:

> "Only what's listed in §1 of our context policy. Order IDs are masked. Emails, phones, payment info, addresses are masked. We've signed DPAs with both providers; they don't train on our API traffic. We have the policy versioned and any change requires legal sign-off."

When eng asks **"can we just stuff more context to improve quality?"**:

> "Sometimes — but every additional 1K tokens adds 200-400ms and ~$0.005 per call. Let's eval whether the lift is real before we expand. Most quality lifts come from better retrieval, not more context."

When CX asks **"can the agent remember the user across sessions?"**:

> "Today, no — session-only memory. Cross-session memory needs a privacy review (open). For now we keep size, gender, location preferences in-session, and that's it."

When a designer asks **"can the agent reference the user's name?"**:

> "Marketing context: yes. Trust context: no — we never call out PII in the response itself, even if we have it. Felt creepiness is a real risk."

---

## Risk lenses

| Risk | Smell | Mitigation |
|------|-------|------------|
| **PII leakage** | Sensitive data in prompts or logs | Pre-prompt mask + audit log review weekly |
| **Context bloat** | p95 latency creeping up; cost rising | Token budget alerts; truncation thresholds |
| **Prompt versioning chaos** | "Which version of the prompt is in prod?" silence | YAML versioned in git; every change is a PR |
| **Cross-session leakage** | Users seeing other users' info | Strict per-session state; tests for state isolation |
| **Implicit memory** | Engineers add "remember" capabilities ad-hoc | Memory layers documented; legal approval per layer |

---

## The context policy — what's in it and why

The [context policy](../pm-artifacts/context-policy.md) covers:

| Section | What it does |
|---------|--------------|
| §1 Allowed sources | The whitelist — everything else is excluded by default |
| §2 Excluded sources | Explicit "we don't pass these" — defense for legal review |
| §3 User profile attrs | The narrow set of personal data we use |
| §4 Truncation rules | What happens when context exceeds budget |
| §5 PII handling | The detection + masking layer |
| §6 Retrieval filters | What's dropped before reaching the model |
| §7 Data residency | Where data lives |
| §8 Logging policy | What's logged where, with what retention |
| §9 Change control | How updates get approved |

Section 5 (PII) is the section legal will read most carefully. Section 1+2 are the sections that catch scope creep.

---

## The structured intermediate pattern

Recurring PM win: insist that data flowing between agent steps is structured, not text.

| Step | Bad (raw text) | Good (structured) |
|------|----------------|-------------------|
| After classification | "The user wants a jacket, maybe waterproof, budget around 150" | `{intent:"product_search", attrs:["waterproof"], max_price:150}` |
| After retrieval | "Here are some chunks: ..." | `[{id, text, score, source}]` |
| After tool call | "I called search and got some results" | `{tool:"search_products", args:{...}, results:[...]}` |

Structured intermediates make each step independently testable. Demand them.

---

## Memory layers

Three layers, PMs decide which exist:

| Layer | Lifespan | Privacy implication |
|-------|----------|---------------------|
| Short-term | Current session | Low — same conversation |
| Mid-term | Days (rolling summary) | Med — depends on what's kept |
| Long-term | Weeks–months | High — durable user model |

Default for retail v1: short-term only. Mid- and long-term wait for explicit privacy clearance.

---

## Reading order in the technical module

Worth reading: §2.1 (the four pillars), §2.4 (meta-prompts/configuration), §2.7 (context policy assignment).

Skim: §2.3 (decomposition lab), §2.5 (adversarial test set).

Skip: code implementation details unless co-owning.

---

## Worked example: handling a "we want personalization" ask

VP Marketing asks: *"Can the copilot recommend based on the user's purchase history?"*

PM playbook:

1. **Acknowledge the value** — personalization can lift conversion.
2. **Surface the policy gap** — current context policy excludes purchase history.
3. **Frame the work** — needs privacy review (open question in policy doc), needs opt-in UX, needs new RAG source.
4. **Estimate** — 4-6 weeks behind v1 launch.
5. **Counter-proposal** — pilot v1 with session-only personalization; gather data; bring case back at GA review.

That conversation, repeated cleanly, prevents 3 months of misaligned expectations.

---

## After this module

Move to [PM Addon 05 — Evaluation & Guardrails](pm-addon-05-evals.md). Evals are the single most important PM-engineering interface.
