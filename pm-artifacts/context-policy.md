# Context Policy — Retail Copilot

**Owner**: J. Park (PM) + K. Ahmed (Security/Privacy)
**Status**: Approved 2026-05-13
**Review cadence**: Quarterly + on any new data source

> 💡 **Why this exists**
> Every byte we put into a prompt costs latency, money, and risk. This policy is the rulebook for *what* the model sees on every call, *how* old conversation gets summarized, *what* gets masked, and *who* signs off when we change any of that.

---

## 1. Sources allowed into context

| Source | Allowed | Notes |
|--------|---------|-------|
| System prompt (versioned YAML) | ✅ | Source-controlled; PR-reviewed |
| Current user turn | ✅ | Always after input guardrail mask |
| Last 6 dialog turns verbatim | ✅ | 3 user + 3 assistant |
| Rolling summary of older turns | ✅ | LLM-generated, max 200 tokens |
| Retrieved chunks (top-5 reranked) | ✅ | Citation IDs preserved |
| User profile attributes | ✅ — see §3 | Limited set only |
| Product / order / policy data via tool results | ✅ | Returned by deterministic tools |

## 2. Sources excluded from context

| Source | Why |
|--------|-----|
| Full home address | PII; not needed for shopping |
| Payment method details (card numbers, CVV, expiry) | PCI scope avoidance + obvious risk |
| Customer service notes (free-text agent comments) | May contain unredacted PII; v1.1 will add a screened version |
| Order IDs older than 90 days | Privacy minimization; archive separately |
| Internal SKU cost data | Not needed; could leak margin info |
| Other shoppers' reviews mentioning real names | Indexed reviews are first-name-only |

## 3. User profile attributes used in context

Only these may be referenced; everything else stays out:

- Logged-in user's: preferred size, preferred gender, default shipping region (city + state, never full address).
- Session-level: last 3 SKUs viewed; current cart contents (SKU + qty only, no pricing math).

We do **not** pass: name, email, phone, lifetime value, segment labels, churn risk score.

---

## 4. Truncation & summarization rules

We stack rules in this order; first that triggers applies:

1. **Always keep**: system prompt, current user turn, top-3 retrieval chunks.
2. **Soft cap**: if total context > 12K tokens, drop retrieval chunks below rank 3.
3. **Medium cap**: if > 16K tokens, re-summarize history into 100 tokens.
4. **Hard cap**: if > 20K tokens, return a templated "I can only handle one topic at a time" deflection and reset.

### Summary prompt (for rolling history)

```
Summarize the prior conversation in ≤100 tokens. Preserve:
- Any product the user is considering (SKU + name)
- Any constraint the user stated (price, size, attributes)
- Any commitment the agent made ("I'll look that up")
Drop pleasantries.
```

---

## 5. Sensitive attribute handling

### PII detection (pre-prompt)

Regex patterns fire on:

- SSN format: `\d{3}-\d{2}-\d{4}` → `[SSN]`
- Credit card 13-19 digits: → `[CC]`
- Order IDs (`ORD-####`) → kept but logged separately as `[ORDER_ID]` with mapping stored encrypted
- Email addresses → `[EMAIL]` (full email stored encrypted with request_id)
- Phone numbers (US format) → `[PHONE]`

We do **not** auto-detect: names, addresses (rely on user not pasting them; we never ask).

### Mask scope

- Masking applied to: prompt sent to LLM, logs at WARN+, traces in Langfuse.
- NOT applied to: the raw event in our audit DB (encrypted, retained 90 days, restricted access).

---

## 6. Retrieval filtering

Before any retrieved chunk enters context:

| Filter | Rule |
|--------|------|
| Similarity threshold | Drop if cosine distance > 0.55 |
| Stock filter | For `product_search` intent, drop chunks where metadata.in_stock = false |
| Source allowlist | Only `product`, `policy`, `faq` sources in v1 |
| Recency | Policy chunks older than 6 months are flagged; agent must mention "as of [date]" |
| Confidence floor | If top-1 distance > 0.45, return "I don't have that info" rather than risk hallucination |

---

## 7. Cross-region / data-residency notes

- All chunked corpus and embeddings stay in our US-East region.
- LLM providers' Data Processing Addendum (DPA) signed; both OpenAI and Google offer zero-day retention for paid API plans.
- We do not yet have an EU pilot. If we do, embeddings + LLM calls must route through EU endpoints (Gemini EU; Azure OpenAI EU).

---

## 8. Logging & audit

Every prompt that touches a model is logged with:

- `request_id`, `user_id` (hashed if anonymous), `session_id`
- `prompt_tokens`, `completion_tokens`, `model`, `latency_ms`
- `tool_calls[]`, `retrieval_chunk_ids[]`, `guardrail_events[]`
- The **rendered prompt text** is stored only in the audit DB (encrypted, 90-day retention). Langfuse spans store summaries, not full PII-bearing prompts.

---

## 9. Change control

Any change to:

- The system prompt (semantic, not formatting)
- The list of attributes in §3
- Truncation thresholds in §4
- Regex set in §5
- Filters in §6

…requires:

- PR review by PM + Security/Privacy.
- Eval rerun (must not regress safety metrics).
- Note in the [decision log](decision-log.md).

---

## 10. Open questions

| # | Question | Owner | Target |
|---|----------|-------|--------|
| 1 | Should we add an LLM-based PII redactor alongside regex? | Eng + Security | v1.1 |
| 2 | Can we let users opt into broader personalization (history)? | Legal + UX | Pilot week 3 |
| 3 | Do we redact agent-typed text to other agents (cross-agent leaks)? | Eng | Pre-GA |

---

## How this gets out of date

- Add a row to §1/§2 every time a new data source is integrated.
- Re-check §7 if we add an international market.
- Re-review §5 regex set every 6 months; abuse patterns evolve.
