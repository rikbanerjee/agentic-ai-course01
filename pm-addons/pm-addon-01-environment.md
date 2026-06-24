# PM Addon — Module 1: Environment & Tooling Setup

> **Pairs with**: [Module 1 — Environment & Tooling Setup](../course/01-environment-setup.md)
> **Read time**: 8 minutes
> **You'll be able to**: speak credibly about the stack, judge tooling proposals, and know what "table-stakes" looks like.

---

## The 60-second summary

Setting up an agentic AI project doesn't require esoteric tooling. The defensible defaults are:

- **Python or TypeScript** (or both — the team uses both in this course).
- **Two LLM provider accounts** (OpenAI + Gemini) so you can compare and fail over.
- **A vector database** (Chroma for dev, something hosted later).
- **Secrets in environment variables**, never in code.
- **Structured (JSON) logs from day one** — you cannot debug agents from print statements.

If a vendor or contractor proposes anything wildly different (e.g., custom embedding infrastructure on day 1, or "let's use a brand-new framework no one has heard of"), ask why.

---

## Why this matters for the product

This is the cheapest module to underrate and the most expensive to skip.

- **Logging discipline** decides how fast you can diagnose user complaints. Teams that skip it often spend 5× longer on debugging.
- **Provider abstraction** (a thin wrapper instead of OpenAI calls everywhere) decides how fast you can swap models when prices change or one provider degrades.
- **Repo structure** decides how easy it is to A/B test prompts, share code with contractors, and onboard new engineers.

The cost of getting this right is one engineer-day. The cost of skipping it is months of friction.

---

## What PMs decide vs what engineers decide

| PM owns | Engineering owns |
|---------|------------------|
| Which providers we use (vendor relationships) | The wrapper code |
| Where keys are stored (compliance) | How keys load at runtime |
| Repo location, name, access list | Repo structure, build/test setup |
| Budget cap per dev env | How spend alerts trigger |
| Whether to invest in Python AND TypeScript | Which framework versions |

---

## Stakeholder talking points

When finance asks **"why do we need two LLM providers in dev?"**:

> "Resilience and price-shopping. If our primary provider degrades — and they all do, occasionally — having a tested fallback prevents an outage. Also, model prices change quarterly; having both lets us route specific tasks to the cheaper one."

When security asks **"are API keys safely managed?"**:

> "Yes — in .env files locally (never committed), in the platform's secret manager (CI, prod). Rotation procedure is documented. We've got an audit trail in the secret manager."

When an engineer asks **"can we skip provider abstraction for now?"**:

> "Short term yes, long term no. Cost of adding a wrapper now is half a day. Cost of refactoring 40 OpenAI calls into a wrapper later is a week. Let's pay it now."

---

## Risk lenses

| Risk | Smell | Mitigation |
|------|-------|------------|
| **API key leak** | Commits to public repos; keys in Slack | Pre-commit hook scanning for secret patterns; rotate immediately if leaked |
| **Single-provider lock-in** | All code imports `openai` directly | Wrapper layer (course Module 1 lab) |
| **Unbudgeted spend** | No monthly cap set in providers | $20/dev/month cap; alerts at 50%/80% |
| **Inconsistent logs** | Some calls log structured JSON, others print to stdout | One logger module imported everywhere |

---

## Provider account checklist

Before you build:

- [ ] OpenAI account with payment method + monthly spend limit
- [ ] Google AI Studio account (Gemini API key)
- [ ] DPAs signed if you handle EU/CA users
- [ ] Per-developer keys (not one shared key — makes attribution impossible)
- [ ] Cost dashboard URL bookmarked in your team wiki

---

## Stack picking framework

| Question | If "yes" → |
|----------|-----------|
| Backend team is mostly Python? | Python primary, TS for UI |
| Backend team is mostly Node/Next.js? | TS primary, Python for ML/eval |
| Team is bilingual or you want options? | Both, parallel directories (course default) |
| One person, one weekend? | Whichever you're faster in |

For most teams, the "both" option only makes sense if it's a learning exercise. In production, pick one for the agent layer; you can still use Vercel AI SDK on the frontend regardless.

---

## Reading order in the technical module

Pages worth reading in full: §1.3 (API keys), §1.7 (Hello LLM lab — even if you don't write the code, read it).

Skim: §1.5/§1.6 (env setup details — let engineering handle).

Watch out for: §1.10 (Hello Agent) — this is the smallest possible agent. Worth understanding the shape even if you don't write it.

---

## Common pitfall: the "we'll abstract later" trap

The single most common mistake at this stage is shipping direct provider calls everywhere with "we'll wrap it later." Later never comes; the codebase calcifies. Cost: a week of refactoring you didn't budget for.

Insist on the wrapper from day 1. The course's `LLMClient` is the recipe.

---

## After this module

Move to [PM Addon 02 — Data](pm-addon-02-data.md). Data quality decisions shape your evals and your RAG quality more than any model choice.
