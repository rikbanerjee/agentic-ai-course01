# Observability — Tracing, Logging, and Eval Dashboards

> **When to read this**: alongside Module 3, then again before the capstone.
> **Goal**: You can answer "what happened in that request?" in under 60 seconds for any past query.

You cannot improve an agentic system you can't see. This track covers the three layers of visibility you need: structured logs, distributed traces, and eval dashboards.

---

## 1. The three layers

| Layer | What it captures | When to look at it |
|-------|------------------|---------------------|
| **Structured logs** | Every event with structured fields | Quick "did it happen?" + post-hoc analysis |
| **Traces** | Tree of nested spans (per request) | Debugging a single slow/broken request |
| **Eval dashboards** | Aggregate metrics across runs | Quality trends; release decisions |

You built layer 1 in Module 1 (JSON logs). This track wires up layers 2 and 3.

---

## 2. Picking a tracing backend

Three popular options for LLM apps:

| Tool | Best for | Hosted / OSS | Free tier |
|------|----------|--------------|-----------|
| **Langfuse** | LLM-specific tracing, evals, prompt management, OSS | Both | Generous |
| **LangSmith** | Tight LangChain/LangGraph integration | Hosted only | Limited |
| **OpenTelemetry + Grafana** | Generic, self-hosted, vendor-neutral | OSS | Self-hosted |

For this course we use **Langfuse**. Reasons: it's OSS (you can self-host), framework-agnostic (works with raw OpenAI SDK *and* LangChain), and the free hosted tier is enough for a course.

---

## 3. Lab — Wire up Langfuse

### Step 1: get keys

Sign up at https://cloud.langfuse.com, create a project, copy public + secret keys into your `.env`:

```bash
LANGFUSE_PUBLIC_KEY=pk-lf-...
LANGFUSE_SECRET_KEY=sk-lf-...
LANGFUSE_HOST=https://cloud.langfuse.com
```

### Step 2: Python — minimal integration

```bash
uv add langfuse
```

```python
# python/src/observability.py
from langfuse import Langfuse
from langfuse.decorators import observe, langfuse_context

langfuse = Langfuse()  # reads env vars

@observe(name="retail_ask")
def traced_ask(query: str, user_id: str | None = None) -> dict:
    langfuse_context.update_current_trace(user_id=user_id, tags=["v2"])
    from agents.multi_agent import app as agent_graph
    state = agent_graph.invoke({"user_query": query})
    return state
```

For OpenAI calls, use Langfuse's OpenAI integration to auto-trace every call:

```python
from langfuse.openai import OpenAI  # drop-in replacement
client = OpenAI()
# Every chat.completions call is now traced automatically.
```

### Step 3: TypeScript — minimal integration

```bash
pnpm add langfuse langfuse-langchain
```

```typescript
// typescript/src/observability.ts
import { Langfuse } from 'langfuse';
export const langfuse = new Langfuse({
  publicKey: process.env.LANGFUSE_PUBLIC_KEY,
  secretKey: process.env.LANGFUSE_SECRET_KEY,
  baseUrl: process.env.LANGFUSE_HOST,
});

export async function tracedAsk(query: string, userId?: string) {
  const trace = langfuse.trace({ name: 'retail_ask', userId, tags: ['v2'] });
  const gen = trace.generation({ name: 'multi_agent.run' });
  // ... call your agent, then:
  gen.end({ output: { /* final state */ } });
  await langfuse.flushAsync();
}
```

For LangChain in TS, attach the callback handler:

```typescript
import { CallbackHandler } from 'langfuse-langchain';
const handler = new CallbackHandler({ /* env-driven defaults */ });
const result = await app.invoke({ userQuery: q }, { callbacks: [handler] });
```

### Step 4: open the dashboard

Make a few calls, then open https://cloud.langfuse.com → Traces. You'll see:

- One trace per request.
- Nested spans for router → research → decision.
- Each LLM call with tokens in/out, latency, cost.
- The exact prompt and completion text.

> 💡 **Why this matters**
> When a user reports "the bot said something weird Tuesday at 3pm," you click their user_id, find the trace, see the exact prompt, retrieval, and completion. Time-to-diagnosis goes from hours to minutes.

---

## 4. What to log (the field cheat sheet)

For every LLM call, log:

| Field | Why |
|-------|-----|
| `request_id` | Stitch events across a request |
| `user_id` (or hash) | Slice metrics per user, debug reports |
| `agent_name` | Router vs Research vs Decision |
| `model` | gpt-4o vs gpt-4o-mini vs gemini-2.0-flash |
| `prompt_tokens`, `completion_tokens` | Cost analysis |
| `latency_ms` | Per-call performance |
| `temperature` | Reproducibility |
| `system_prompt_version` | Track A/B tests |
| `tool_calls[]` | Which tools fired |
| `retrieval_chunk_ids[]` | Which chunks influenced answer |
| `guardrail_events[]` | Refusals, masks, blocks |
| `outcome` | success / refused / errored |

If you log these consistently, every report ("p95 latency went up yesterday") is a SQL query away.

---

## 5. Eval dashboards

Beyond per-trace inspection, you need aggregate views.

### Option A — Langfuse Datasets + Scores

Langfuse has native support: upload your eval cases as a "Dataset," run them, attach scores to each run. You get out-of-the-box charts for accuracy/groundedness/latency over time.

```python
from langfuse import Langfuse
lf = Langfuse()
ds = lf.create_dataset(name="retail-base-eval-v1", description="47 labeled retail cases")
for case in CASES:
    lf.create_dataset_item(dataset_name=ds.name, input=case, expected_output=case.get("gold_answer_keywords"))
```

Run evals → push scores → Langfuse dashboard shows trend.

### Option B — Roll your own with Streamlit

If you want full control or are offline:

```python
# scripts/eval_dashboard.py
import json, pandas as pd, streamlit as st
from pathlib import Path

reports = sorted(Path("data/eval_reports").glob("report_*.json"))
dfs = []
for p in reports:
    df = pd.DataFrame(json.loads(p.read_text()))
    df["ts"] = int(p.stem.split("_")[1])
    dfs.append(df)
full = pd.concat(dfs)

st.title("Retail Copilot Eval Dashboard")
agg = full.groupby("ts").agg(
    intent_acc=("intent_correct", "mean"),
    safety_acc=("safety_correct", "mean"),
    helpfulness=("helpfulness", "mean"),
).reset_index()
st.line_chart(agg.set_index("ts"))
st.dataframe(full.sort_values(["ts", "intent_correct"]))
```

Run: `streamlit run scripts/eval_dashboard.py`.

---

## 6. Online vs offline metrics

| Type | Source | Example |
|------|--------|---------|
| **Offline** | Eval suite, run in CI | "Recall@5 = 0.85 on base set" |
| **Online** | Real user traffic | "30% of users gave thumbs-down on policy_question intent today" |

Offline catches regressions before they ship. Online catches things you didn't think to write evals for. You need both.

Lightweight online metric collection:

```python
@app.post("/feedback")
def feedback(request_id: str, rating: int, comment: str | None = None):
    log_event(logger, "feedback", request_id=request_id, rating=rating, comment=comment)
    # also push to Langfuse score endpoint:
    langfuse.score(trace_id=request_id, name="user_rating", value=rating, comment=comment)
```

Add a thumbs-up/down to your demo UI. Every rating becomes a candidate eval case.

---

## 7. The diagnosis checklist

When something looks wrong, work in this order:

1. **Pull the trace.** What did the model actually see?
2. **Check tools.** Did the right tool fire with the right args? Did it return an error?
3. **Check retrieval.** Which chunks were retrieved? Were the right ones present?
4. **Check the prompt.** Were the system prompt variables interpolated correctly?
5. **Check the schema.** Was the response a valid JSON that matched your Pydantic/Zod schema?
6. **Check the guardrails.** Were any input/output filters triggered?
7. **Add a case to the eval set.** Even before you fix it — that pins the bug.

Beginners skip step 1 and jump to "let me tweak the prompt." That's how you end up with a prompt full of conditional patches that no one understands.

---

## 8. Cost tracking

Langfuse automatically computes cost per trace if you've set model pricing. You can also do it yourself:

```python
PRICING = {  # USD per 1M tokens
    "gpt-4o-mini": {"in": 0.15, "out": 0.60},
    "gpt-4o": {"in": 2.50, "out": 10.00},
    "gemini-2.0-flash": {"in": 0.10, "out": 0.40},
    "text-embedding-3-small": {"in": 0.02, "out": 0},
}
def cost(model: str, tokens_in: int, tokens_out: int) -> float:
    p = PRICING.get(model, {"in": 0, "out": 0})
    return (tokens_in * p["in"] + tokens_out * p["out"]) / 1_000_000
```

Log `cost_usd` per event; aggregate by model, by agent, by intent.

---

## 9. Checklist

- [ ] Langfuse keys in `.env`; first trace visible in dashboard.
- [ ] Multi-agent traces show all nested spans (router → research → decision).
- [ ] You can find a specific user's request by `user_id` in < 60s.
- [ ] Eval dashboard plots metric trend over time (Langfuse or Streamlit).
- [ ] `/feedback` endpoint accepts thumbs and writes both log + Langfuse score.

---

## What you can now answer

- "Show me every refused request from the last 24h." → log query
- "What's our p95 latency by agent?" → trace aggregate
- "How did helpfulness change between V1 and V2?" → eval dashboard
- "Why did this specific user's order_status request fail?" → trace tree
