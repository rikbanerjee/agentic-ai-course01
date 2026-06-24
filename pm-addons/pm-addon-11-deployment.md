# PM Addon — Deployment Patterns

> **Pairs with**: [Deployment Patterns track](../course/11-deployment-patterns.md)
> **Read time**: 10 minutes
> **You'll be able to**: pick the right deployment rung, scope a rollout, and demand the production hygiene that prevents 3am pages.

---

## The 60-second summary

"Deployment" is a spectrum, not a binary. Three rungs:

1. **Local + ngrok** — share with one person for 30 minutes. Demos only.
2. **Single VM in a managed runtime** (Fly / Render / Railway / Vercel) — long-running, cheap, no orchestration. Good for pilot.
3. **Managed container with auto-scaling** (Cloud Run / ECS / Vercel Pro) — production. Cold starts to consider.

Production-ready isn't just "uptime." It means someone other than you can fix a problem within an hour. That requires logs + traces + evals + runbooks + rollbacks + kill switches.

PMs don't run deployments, but they own the deployment **plan**: the rollout phases, the gates between them, the runbook expectations.

---

## Why this matters for the product

Two failure patterns to prevent:

1. **"Big bang" launch** — 0% → 100% traffic on launch day. Maximum surprise surface. Inevitable rollback.
2. **"It works in dev"** — engineering ships without rate limits, secrets management, kill switches, or alerting. First production incident is also the first incident response.

Phased rollouts and production hygiene aren't optional polish. They're the difference between a launch and a launch postmortem.

---

## What PMs decide vs what engineers decide

| PM owns | Engineering owns |
|---------|------------------|
| Rollout phases and gates | Deployment mechanics (Docker, CI, etc.) |
| Kill-switch acceptance criteria (how fast, who can flip) | Implementation |
| When to advance traffic % | Pre-advance checks |
| Communication plan for incidents | Incident response runbook |
| Compliance gates (security review, DPA sign-off) | Implementation of controls |

---

## Stakeholder talking points

When eng asks **"can we ship to 100% on launch day?"**:

> "No. Phased rollout: 1% soft pilot → 10% pilot → 25% → 50% → 100%. Each step is gated on metrics, not dates. Worst case at 1% is a tiny user-base incident; worst case at 100% is a brand event."

When CX asks **"if the AI says something bad, can we turn it off fast?"**:

> "Yes — feature flag flips the copilot off in under 60 seconds, and the UI falls back to faceted search. On-call has the flip authority; no engineer needs to be paged."

When finance asks **"what's the deployment infra cost?"**:

> "At pilot, ~$50/month. At GA scale, ~$300-500/month for managed runtime + observability. Dwarfed by LLM API spend."

When security asks **"where do API keys live?"**:

> "Platform secret manager (Fly Secrets / Vercel Env / AWS Secrets Manager — depending on rung). Never in code, never in images, never in logs. Rotation procedure documented; default rotation every 90 days."

---

## Choosing the deployment rung

| Question | If yes → |
|----------|----------|
| Friends-and-family demo for an hour? | Rung 1 (Local + ngrok) |
| Pilot with low traffic, want low ops burden? | Rung 2 (single VM + Docker) |
| GA with auto-scale + multiple regions? | Rung 3 (managed container) |
| Already on Vercel for the frontend? | Vercel for everything; saves a vendor |
| Already on AWS for everything else? | ECS or Cloud Run depending on cloud |

Default for the course: start Rung 1 for dev, jump to Rung 2 for pilot, plan Rung 3 before GA.

---

## Risk lenses

| Risk | Smell | Mitigation |
|------|-------|------------|
| **No kill switch** | "We can roll back the code" but no flag | Feature flag at the API layer with <60s flip |
| **No rate limits** | First abuse spike costs $5K | Per-IP + per-user limits; default 30 RPM/user |
| **No spend cap** | LLM bill spikes 5× overnight | Daily spend cap with auto-disable + page |
| **No rollback plan** | "Last known good" image tag undocumented | Image tags + rollback runbook |
| **Manual deploys** | "Bob is the only one who deploys" | CI-driven deploys; anyone can ship/rollback |
| **No staging** | Code goes from dev laptop to prod | At minimum dogfood env mirrors prod config |
| **Secrets in env vars in logs** | grep for 'sk-' returns hits | Mask secrets in log formatter; CI scanner |

---

## Production hygiene checklist (the PM "are we ready" list)

Walk through this before any pilot launch. Demand a "✅" against each:

- [ ] Containerized; image < 500 MB
- [ ] CI runs lints + tests + **evals** on every PR
- [ ] Secrets in platform secret manager (not in code)
- [ ] Per-user + per-IP rate limits configured
- [ ] Streaming enabled on user-facing endpoints
- [ ] Observability (traces) flowing from prod
- [ ] Health check endpoint + graceful shutdown
- [ ] Documented rollback path (image tag + flip)
- [ ] Runbooks for: provider outage, vector DB outage, eval threshold breach, abuse spike
- [ ] On-call rotation + escalation path documented
- [ ] Kill switch (feature flag) verified end-to-end
- [ ] Daily LLM spend cap with auto-disable

Missing any one of these = you're not ready for pilot. Missing more than two = you're not ready for dogfood.

---

## Phased rollout — gates over dates

| Phase | Traffic | Gate to advance | Notes |
|-------|---------|------------------|-------|
| Dogfood | 0% live (internal employees) | Eval suite ≥ thresholds | 1 week minimum |
| Soft pilot | 1% of traffic | No P0 in dogfood, no provider outages | 1 week |
| Pilot | 10% | Helpfulness ≥ 4.0, no P0 in soft pilot | 4 weeks |
| GA 25% | 25% | All pilot success criteria | 1 week |
| GA 50% | 50% | Same + cost holds | 1 week |
| GA 100% | 100% | Same + retro on near-misses | — |

Every advance is a meeting (PM + Eng + CX). Document the decision in the [decision log](../pm-artifacts/decision-log.md).

---

## Reading order in the technical module

Worth reading: §1 (deployment shape), §3-4 (rungs 2 and 3), §7 (security), §9 (production checklist).

Skim: §5 (Vercel AI SDK streaming UI), §6 (CI/CD).

Skip: code-level details unless co-owning.

The §9 checklist is the PM's most reusable artifact in this module.

---

## Worked example: a launch-day timeline

Let's say GA is Tuesday. Backward plan:

- **T-7 days**: dogfood + soft pilot complete; eval thresholds green for 7 consecutive days; runbooks finalized.
- **T-3 days**: dry-run rollback (engineer flips kill switch, measures time, restores). Documented.
- **T-1 day**: 25% rollout (Tuesday morning) prep. Comms drafted. On-call paged on-call.
- **T-0 (Tuesday 9am)**: flip to 25%. Active monitoring for 4 hours. Decision at 1pm: hold or advance.
- **T+1 day**: 50%. Same monitoring cadence.
- **T+3 days**: 100%. Retro scheduled for T+10.

If anything red at any gate: don't advance. Document why. Decide next move.

---

## After this module

You've completed the PM addon series. Recommended next reads:

1. The [PM artifacts pack](../pm-artifacts/README.md) — filled-out templates for memos, RFCs, design reviews.
2. The [glossary](../course/glossary.md) — for any term still unclear.
3. The [resources](../course/resources.md) — further reading.

If you've worked through all 12 PM addons, you can confidently scope, design, ship, and defend an agentic AI feature. That's a competence very few PMs have today. Use it.
