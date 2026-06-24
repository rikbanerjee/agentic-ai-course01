# PM Addon — Weekly Module 1: GenAI & Agentic Apps

> **Pairs with**: [Weekly Module 1](../course/03-module-1-genai-foundations.md)
> **Read time**: 12 minutes
> **You'll be able to**: write a system memo, frame why traditional QA doesn't work for LLM apps, and recognize the five retail-specific failure modes.

---

## The 60-second summary

The five mental shifts you need:

1. **LLM output is probabilistic.** Same input can return different outputs. Unit tests don't catch quality regressions — evals do.
2. **Bugs may not reproduce on your machine.** A 3% failure rate looks like "works fine" to a single tester.
3. **Quality, cost, and latency must be measured together.** Optimizing one in isolation often degrades another.
4. **Stack traces don't help.** You debug by inspecting prompts, retrieved chunks, and outputs.
5. **The model can return malformed JSON.** Always validate. Always have a fallback.

The five failure modes in retail to assume:
- **Hallucination** (inventing SKUs/prices)
- **Retrieval miss** ("I don't have that info" when you do)
- **Wrong tool / no tool** (answering from memory instead of looking up)
- **Policy violation** (promising refunds)
- **Format break** (returning bad JSON that breaks the UI)

---

## Why this matters for the product

This module is where "let's just ship a prompt" gets killed. The deliverable — a system memo — is the artifact PMs will refer to every week for the rest of the project. It's the most important doc in this whole course for PMs.

If you don't write the system memo now, you'll have a different conversation about scope every week for 6 weeks.

---

## What PMs decide vs what engineers decide

| PM owns | Engineering owns |
|---------|------------------|
| The whole system memo | The structured-output schema (Pydantic/Zod) |
| What's in scope vs out of bounds | Retry/fallback logic when output is malformed |
| Hard constraints (latency, cost, safety floors) | How to instrument tracing |
| Success metrics | How to compute them |
| Whether to accept LLM probabilistic behavior | How to mitigate via prompts + guardrails |

---

## Stakeholder talking points

When eng leadership asks **"why is this so much more work than a regular feature?"**:

> "Every traditional software feature has deterministic acceptance criteria — given X, return Y. LLM features have probabilistic ones — given X, return Y at least 90% of the time, with a defined failure mode for the 10%. That requires an eval suite, guardrails, and tracing. Those are upfront investments that pay back the first time you ship a change."

When a stakeholder asks **"can the AI just be more accurate?"**:

> "Yes, but not by tweaking the AI — by tightening the system around it. We constrain the model to use only retrieved sources (grounding), we validate every output (schema), and we filter outputs that violate policy (guardrails). That gets us from ~80% accuracy to ~95% on our eval set."

When CX asks **"what happens if the AI says something we can't honor?"**:

> "Three layers prevent that. First, prompts forbid making promises. Second, an output filter blocks promise patterns. Third, every output is logged with the input — so if we miss something, we can recreate it and add it to our test suite within hours."

---

## The system memo — your most-leveraged artifact

It is one page. Seven sections. Approval from PM + Eng + CX + Legal (where relevant). See the working example at [system-memo.md](../pm-artifacts/system-memo.md).

Sections, in order of importance:

| § | Section | Why it's there |
|---|---------|----------------|
| 5 | Where AI is out of bounds | Stops scope creep |
| 6 | Constraints | Stops "let's just make it bigger" |
| 4 | Where AI adds value | Stops "let's AI everything" |
| 7 | Success metrics | Defines "shipped" |
| 1-3 | User/job/baseline | Frames everything |
| 8 | Open questions | Lists what's deferred, not forgotten |

Spend 60% of your authoring time on §5. It's the section that will save the most pain.

---

## Risk lenses

| Risk | Smell | Mitigation |
|------|-------|------------|
| **Scope creep** | "While we're at it, can the agent also…" | Point to system memo §5. Be polite, be firm. |
| **Hidden constraints** | A constraint surfaces 3 weeks in ("we need to be HIPAA compliant") | Re-check §6 every Friday with eng + legal until pilot |
| **Metric vagueness** | "We want it to be helpful" with no number | Specify both LLM-judge target + a human spot-check target |
| **Implicit no-AI areas** | Nobody wrote down that refunds are out of bounds | Get the list signed off before kickoff |

---

## The five failure modes — what PMs say and do

| Failure | What PM does pre-launch | What PM says post-launch when it happens |
|---------|--------------------------|-------------------------------------------|
| Hallucination | Insist on grounding + eval coverage | "We're at 0.4% rate; here's the case in our suite tracking it." |
| Retrieval miss | Make sure §6 of system memo has a recall target | "It's a coverage gap; we've added the case and are extending the corpus." |
| Wrong tool | Insist on tool contracts (see [tool-contracts.md](../pm-artifacts/tool-contracts.md)) | "The Router misclassified; we've added it to the intent eval set." |
| Policy violation | Insist on guardrails layer | "The output filter caught it; user saw the templated message. Logging the near-miss." |
| Format break | Insist on schema validation + retries | "Structured output failed; we retried and succeeded. P95 latency took a 200ms hit but no user impact." |

---

## Reading order in the technical module

Worth reading in full: §1.1 (mental shift), §1.2 (failure surfaces), §1.7 (System Memo assignment with template).

Skim: §1.3 (intent classifier code — read the prompt + output, not the Python details).

Skip: lab implementations unless you want to dig in.

---

## Worked example: writing the system memo for *your* product

Suppose you're building a support copilot for an insurance company. Plug in:

- **User**: policyholders calling about claims, billing, coverage questions.
- **Job**: get definitive answer in <60s without phone tag.
- **Baseline**: average wait 12 min, 30% repeat-callers.
- **AI value**: parse the question, retrieve policy, surface coverage rules.
- **NOT AI**: claim approval, payment processing, anything binding.
- **Constraints**: 0% hallucinated coverage claims (legal nuclear), <5s response, HIPAA-compliant data handling.
- **Metrics**: deflection rate ≥ 25%, hallucination = 0 on coverage queries, CSAT no regression.

That's your insurance system memo v0. Take it to legal Friday.

---

## After this module

Move to [PM Addon 04 — Context Engineering](pm-addon-04-context.md). Context is where most ongoing PM tuning happens.
