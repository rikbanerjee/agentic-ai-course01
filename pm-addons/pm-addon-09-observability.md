# PM Addon — Observability Track

> **Pairs with**: [Observability track](../course/09-observability.md)
> **Read time**: 10 minutes
> **You'll be able to**: scope observability investment, answer "what happened in that request" for users, and build the dashboards leadership will ask for.

---

## The 60-second summary

You can't improve a system you can't see. Three layers of visibility:

1. **Structured logs** — every event with consistent fields. Cheap; query in seconds.
2. **Traces** — tree of nested spans per request. Best for diagnosing a single weird request.
3. **Eval dashboards** — aggregate metrics over time. Best for release decisions.

For LLM apps, the practical default stack is **Langfuse** (or LangSmith if you're already on LangChain). It captures prompts, completions, tool calls, latencies, costs, traces — all in one place. Free tier covers most pilots.

The PM job here: make sure observability ships *before* the feature. Retrofitting is 5× the work.

---

## Why this matters for the product

Three concrete moments where observability earns its keep:

1. **A VP forwards you a screenshot of weird agent output.** With observability: find the trace in 60 seconds, send a precise answer same hour. Without: spend a day reproducing, possibly never.
2. **Quality slowly degrades over weeks.** With observability: trend chart shows the drop, you trace it to a model version change. Without: you discover via customer support tickets.
3. **You ship a prompt change.** With observability: A/B test in production with confidence. Without: ship and hope.

Teams that skip observability ship 2-3× slower in steady state because every change is high-risk.

---

## What PMs decide vs what engineers decide

| PM owns | Engineering owns |
|---------|------------------|
| Observability vendor + budget | Implementation + SDK choice |
| Retention period for traces | Storage costs at scale |
| Dashboard structure (what charts go where) | Building them |
| Alerting thresholds | PagerDuty / Slack routing |
| Privacy: what gets logged at what fidelity | Masking and encryption |

---

## Stakeholder talking points

When eng asks **"can we ship without observability and add it later?"**:

> "Hard no. Every customer report we get post-launch is unfixable without traces. Building observability now is two days; building it after the first user complaint is two weeks of pain plus an angry user who didn't get a real answer."

When finance asks **"why does Langfuse cost money?"**:

> "Free tier covers our pilot. Hosted plan scales with traces; at GA volume that's ~$1K/month. Self-hosting is free but needs ops time. The ROI is one prevented P0 every six months."

When legal asks **"what's in the trace data?"**:

> "Full prompts (with PII masked the same way they're masked in the model call) and full completions. Retention 90 days. Access restricted to on-call + eng leadership. No third-party access beyond Langfuse infrastructure with their DPA."

When CX asks **"can support agents see traces?"**:

> "Yes — we surface request IDs in customer-facing responses. When a user reports an issue, support can paste the ID into Langfuse and see exactly what happened. Big workflow upgrade vs current 'can you describe what you saw?'."

---

## The field cheat sheet — what every LLM call logs

PMs don't write the code, but you should know what's being captured. Demand all of these:

| Field | Why |
|-------|-----|
| `request_id` | Stitch events across a request |
| `user_id` (hashed) | Slice metrics per user, debug reports |
| `agent_name` | Router vs Research vs Decision |
| `model` | Track per-model performance |
| `prompt_tokens`, `completion_tokens` | Cost analysis |
| `latency_ms` | Per-call performance |
| `temperature`, `prompt_version` | Reproducibility |
| `tool_calls[]` | Which tools fired |
| `retrieval_chunk_ids[]` | Which chunks influenced answer |
| `guardrail_events[]` | Refusals, masks, blocks |
| `outcome` | success / refused / errored |

If any one of these is missing, you're partially blind.

---

## Risk lenses

| Risk | Smell | Mitigation |
|------|-------|------------|
| **PII in logs** | Prompts logged verbatim with user emails/addresses | Mask before log; encrypt audit DB; restricted access |
| **Trace gaps** | Some agents emit spans, others don't | Code review checklist; CI test that requires span emission |
| **Dashboard rot** | Charts that no one looks at | Quarterly review; delete unloved charts |
| **Alert fatigue** | Pages for non-incidents | Tune thresholds; warn vs page split |
| **Vendor lock-in** | Custom Langfuse-specific code everywhere | Wrap in your own thin tracer interface |

---

## The three dashboards every team needs

1. **System health (live)**: p50/p95 latency by agent, error rate, current cost burn, current QPS.
2. **Quality trend (weekly)**: eval metrics over time; new failures; resolved failures.
3. **Cost detail (monthly)**: spend by model, by agent, by feature flag; projected vs actual.

Build the first one before pilot. Second before GA. Third before quarterly review.

---

## Online vs offline metrics

| Type | Source | Use for |
|------|--------|---------|
| **Offline** | Eval suite in CI | Catch regressions before ship |
| **Online** | Real user traffic + thumbs/feedback | Catch the things you forgot to write evals for |

You need both. Offline is your gate; online is your discovery mechanism.

A lightweight pattern: add a thumbs up/down + free-text feedback in the UI. Every thumbs-down becomes a candidate eval case.

---

## Reading order in the technical module

Worth reading: §1 (three layers), §3 (Langfuse setup), §4 (field cheat sheet), §7 (diagnosis checklist).

Skim: §5 (dashboards), §6 (online vs offline).

Skip: code implementation if not co-owning.

---

## Worked example: the user complaint runbook

User sends: *"The agent recommended a jacket that's out of stock. This happens too often."*

PM playbook:

1. **Get the request ID.** Usually surfaced in the agent's response footer.
2. **Open Langfuse, paste ID.** See the trace.
3. **Inspect.** Did `search_products` return out-of-stock? Did the agent ignore the `in_stock` field? Did the filter not apply?
4. **Classify.** Is this a one-off (data freshness lag) or a pattern (filter bug)?
5. **Add eval case.** Whatever the root cause, the case goes into the suite before the fix.
6. **Communicate.** Reply to user with what happened (in plain language) and what's being done.

Total time, with observability: 20 minutes. Without: a day.

---

## After this module

Read [PM Addon 10 — Cost & Latency](pm-addon-10-cost-latency.md). Observability tells you what's happening; cost/latency optimization tells you what to do about it.
