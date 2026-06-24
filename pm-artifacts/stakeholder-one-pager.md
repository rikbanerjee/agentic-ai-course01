# Retail Copilot — One-Pager for Stakeholders

*A single-page explainer for anyone outside the team. Print, share, paste into a Slack DM. Updated as the project changes.*

---

## What it is

A shopping copilot built into northcrest.com. Shoppers type what they want in plain English; the copilot finds matching products, answers policy questions, and looks up order status — all with cited sources.

It is **not** a general AI chatbot. It only answers questions about our catalog, policies, and orders.

---

## Why we're building it

Today's site forces shoppers to translate their needs ("warm jacket for layering") into our filter taxonomy (category + insulation type + weight class). About 38% bounce before completing the discovery. The copilot does the translation for them.

---

## How it works (no jargon)

1. The shopper types a question.
2. A small AI classifies what they're asking ("product search," "policy question," "order status," "something off-topic").
3. If it's on-topic, a research AI looks up the answer using our catalog and policy docs.
4. A writing AI composes a friendly response with citations to specific products or policies.
5. Two safety filters (one before, one after) catch anything risky.
6. The shopper sees the answer streaming back, with links to the products mentioned.

If anything goes wrong, the system falls back to "I can't help with that — let me connect you to support." Nothing is auto-promised. Nothing changes accounts or orders.

---

## What it can do

- Find products by fuzzy attributes ("warm but packable")
- Answer questions about returns, shipping, warranties, price matching, and loyalty
- Look up order status for signed-in shoppers
- Honestly say "I don't know" when the answer isn't in our documentation

## What it cannot do (by design, not by accident)

- Promise refunds, credits, or shipping upgrades
- Modify orders or accounts
- Apply discount codes
- Search competitor pricing
- Speak languages other than English
- Search by image

---

## Results so far (4-week pilot at 10% traffic)

- **+38% conversion lift** on copilot-engaged sessions vs control
- **4.3/5 helpfulness** (judged by another AI, spot-checked by humans)
- **96% safety pass rate** on adversarial test cases (prompt injections, refund-baiting)
- **0 P0 incidents**
- **$0.022 cost per query** (target was $0.03 or less)
- **3.6s p95 response time** (target was 4s or less)

---

## What's next

Approving GA rollout: 25% of traffic next week, 100% by end of month.

Then we work on the limitations we identified during pilot:
- Better answers on sizing questions
- Smarter "similar product" suggestions when something's out of stock
- Faster failover when an AI provider has a bad day

---

## Who to ask

- **Product**: J. Park (j.park@northcrest.example)
- **Engineering**: A. Reyes (a.reyes@northcrest.example)
- **CX / Trust**: R. Diaz (r.diaz@northcrest.example)

---

## Common questions

**"Is the AI making things up?"**
We measure that explicitly — currently 0.4% of responses contain unverified claims. Every claim has to come from a real document or tool result; safety filters block the rest.

**"Will it replace human support?"**
No. It deflects easy questions (policy lookups, order status) so humans can focus on harder ones (escalations, complex returns). The agents handling escalations are unchanged.

**"What if it embarrasses the brand?"**
We have kill switches (turn the whole feature off in <60s) and daily cost caps (auto-disable on breach). If a class of failure shows up, we add it to the eval suite and the system has to pass that test before we ship a fix.

**"Why isn't it personalized to me?"**
Privacy review is open. v1 stays at session-level personalization only (size, gender preference, shipping region — never full address or order history).

**"Is my data being sent to OpenAI/Google?"**
Order IDs and PII are masked before any prompt is sent. Both providers have signed data processing agreements with zero-day retention on paid plans.
