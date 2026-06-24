# Tool Contracts — Retail Copilot

**Owner**: A. Reyes (Eng lead)
**Status**: Active v1
**Companion to**: [RFC](rfc-agentic-search.md), [Multi-agent architecture](multi-agent-architecture.md)

> 💡 **Why this exists**
> Tools are the agent's hands. A tool with a vague description gets called wrong. A tool with no error contract crashes the loop. This is the source of truth for every tool the agent can invoke.

---

## Contract format

Every tool documents:

- **Purpose**: one sentence, the LLM reads this.
- **Inputs**: JSON schema with typed fields + descriptions.
- **Outputs**: success shape + error shapes.
- **Errors**: enumerated.
- **Idempotent**: yes/no.
- **Latency budget**: p95 SLA.
- **PII**: what's inside, what's masked.
- **Side effects**: anything beyond a read.
- **Versioning**: how breaking changes propagate.

---

## search_products

**Purpose**: Search the retail catalog with natural-language attributes (color, material, use case), price, and stock filters. Use whenever the user expresses a shopping intent.

**Inputs**:

```json
{
  "type": "object",
  "properties": {
    "query": {"type": "string", "description": "Free-text product description"},
    "max_price": {"type": ["number", "null"], "description": "Optional price ceiling in USD"},
    "in_stock_only": {"type": "boolean", "default": true},
    "limit": {"type": "integer", "minimum": 1, "maximum": 20, "default": 5}
  },
  "required": ["query"]
}
```

**Output (success)**:

```json
{
  "results": [
    {"sku": "SKU-1023", "name": "Northcrest Men's Rain Shell", "price": 129.0, "stock": 12, "category": "outerwear"}
  ],
  "total_matched": 7
}
```

**Output (error)**:

- `{"error": "validation_failed", "details": [...]}` — bad arg types
- `{"error": "tool_exception", "type": "...", "message": "..."}` — unexpected
- `{"error": "rate_limited", "retry_after_ms": 1000}` — internal QPS cap hit

**Idempotent**: yes (read-only).
**Latency**: p95 ≤ 200ms.
**PII**: none in input; no PII in output.
**Side effects**: none.
**Versioning**: schema additions are non-breaking; renames bump the tool name (`search_products_v2`).

---

## get_product

**Purpose**: Fetch a single product by SKU. Use when the user mentions a specific SKU pattern.

**Inputs**:

```json
{
  "type": "object",
  "properties": {"sku": {"type": "string", "pattern": "^SKU-\\d+$"}},
  "required": ["sku"]
}
```

**Output (success)**:

```json
{"sku": "SKU-1023", "name": "...", "price": 129.0, "stock": 12, "description": "..."}
```

**Output (error)**:

- `{"error": "not_found", "sku": "SKU-9999"}`
- `{"error": "validation_failed", "details": [...]}`

**Idempotent**: yes.
**Latency**: p95 ≤ 50ms (in-memory cache).
**PII**: none.

---

## get_order

**Purpose**: Look up an order by ID (`ORD-####`). Use only for authenticated sessions or when the order belongs to the requesting user.

**Inputs**:

```json
{
  "type": "object",
  "properties": {"order_id": {"type": "string", "pattern": "^ORD-\\d+$"}},
  "required": ["order_id"]
}
```

**Output (success)**:

```json
{
  "order_id": "ORD-4456",
  "status": "shipped",
  "ordered_at": "2026-04-30T15:32:00Z",
  "sku": "SKU-1023",
  "qty": 1
}
```

**Output (error)**:

- `{"error": "not_found", "order_id": "ORD-9999"}`
- `{"error": "forbidden", "reason": "cross_account_lookup_blocked"}`
- `{"error": "rate_limited", "retry_after_ms": 1000}`

**Idempotent**: yes.
**Latency**: p95 ≤ 300ms.
**PII**: order_id is treated as sensitive — masked in prompts, kept raw in audit DB only.
**Side effects**: none.
**Side effects we explicitly disallow**: this tool **never** modifies orders; modification requires a separate tool (not in v1) with a different auth scope.

---

## get_policy

**Purpose**: Fetch one of our policy documents in full markdown. Use for any returns/shipping/warranty/price-match/loyalty question.

**Inputs**:

```json
{
  "type": "object",
  "properties": {
    "topic": {"type": "string", "enum": ["returns", "shipping", "warranty", "price-match", "loyalty"]}
  },
  "required": ["topic"]
}
```

**Output (success)**:

```json
{"topic": "returns", "content": "# Returns Policy\n\nWe accept returns within 30 days..."}
```

**Output (error)**:

- `{"error": "unknown_topic", "topic": "tax"}`

**Idempotent**: yes.
**Latency**: p95 ≤ 50ms (file read).
**PII**: none.
**Versioning**: policy edits version the content but not the schema; the LLM is told policies were "last updated YYYY-MM-DD" so it can reflect freshness.

---

## Tools we deliberately do not expose in v1

| Tool | Reason for exclusion |
|------|----------------------|
| `apply_discount` | Authority sits with Promotions service; legal review pending |
| `modify_order` | Mutation; v1 is read-only |
| `issue_refund` | Mutation + financial; requires multi-step approval |
| `subscribe_user` | Compliance: subscription consent flow lives in checkout |
| `external_web_search` | Risk of hallucination amplification + brand control |
| `competitor_lookup` | Brand + legal |

Every "wouldn't it be cool if the agent could…" pitch lands here first.

---

## Cross-tool rules the agent must obey

These show up in the Research agent's system prompt:

1. Never claim a tool was called if it wasn't (see `tool_trace`).
2. If a tool returns `not_found`, say so plainly. Do not invent data.
3. If a tool returns `forbidden`, escalate (Decision agent surfaces a human-handoff message).
4. If a tool returns `rate_limited`, retry once after the suggested delay; if still failing, surface gracefully.
5. Tools may be called in parallel when independent; never call the same tool twice with identical args in the same turn.

---

## How a new tool gets added

```
PR proposing the tool
  ├── tool definition (Pydantic / Zod schema)
  ├── implementation
  ├── tests (success + every error case)
  ├── eval cases that exercise it (≥ 3)
  └── update to this document

  → review by Eng + PM + (Security if it touches user data or mutates state)
  → eval suite green
  → merge → deploy to dogfood
  → 1 week observation
  → promote to pilot
```

---

## How this gets out of date

- Every tool's "Latency p95" should be re-measured monthly from prod traces.
- "Tools we deliberately do not expose" gets shorter over time as authority migrates from CX to agent — keep the reasons updated, not deleted.
- Versioning policy needs a written deprecation path the first time we deprecate a tool. We have not yet.
