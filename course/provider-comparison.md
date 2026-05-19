# Provider Comparison — OpenAI vs Gemini

> The two providers used throughout this course. This doc helps you pick which to default to for each role in your system.

---

## At a glance

| Dimension | OpenAI (GPT-4o family) | Google Gemini (2.0 family) |
|-----------|------------------------|-----------------------------|
| **Flagship model** | gpt-4o | gemini-2.0-pro |
| **Workhorse small model** | gpt-4o-mini | gemini-2.0-flash |
| **Context window** | 128K | up to 2M (gemini-2.0-pro) |
| **Free tier** | None (pay as you go) | Generous |
| **Multi-modal** | Text, image, audio (Realtime API) | Text, image, audio, video |
| **Structured output** | `response_format: json_object`, strict schemas | JSON via response schema |
| **Function calling** | Mature, parallel tool calls | Mature; slightly different schema shape |
| **Streaming** | Yes (SSE) | Yes |
| **Prompt caching** | Automatic for repeated prefixes | Explicit cached_content API |
| **Embeddings** | text-embedding-3-small/-large | text-embedding-004 |
| **SDK quality (Py/TS)** | Excellent for both | Excellent for both |
| **Latency (typical mini/flash p50)** | 600–900 ms | 400–700 ms |
| **Pricing (mini/flash, per 1M tokens)** | $0.15 in / $0.60 out | $0.10 in / $0.40 out |

(Prices and capabilities change frequently; verify on the providers' pricing pages.)

---

## Where each shines

### OpenAI

- **Strict structured output**: `response_format` with strict JSON schemas is rock-solid. Best if your downstream code parses LLM JSON.
- **Tool use ergonomics**: API design and SDK around function calling is the most polished.
- **Realtime API**: low-latency voice/text bi-directional sockets.
- **Batch API**: 50% discount for async bulk processing.
- **Mature ecosystem**: every framework integration "just works."

### Gemini

- **Huge context** (up to 2M tokens on Pro): you can stuff entire docs or codebases.
- **Cost per token**: cheapest of the major providers at the flash tier; the free tier covers most personal dev.
- **Native multimodality**: video understanding, audio, images out of the box.
- **Speed**: gemini-2.0-flash is among the fastest hosted models.
- **Citations + Google Search grounding** (in some configurations).

---

## Where each can stumble

### OpenAI

- **Cost**: gpt-4o is pricey for high-volume use. The mini/cheap tier is great for everything except final-answer prose.
- **Tool-call hallucinations**: more frequent on smaller models if descriptions are vague.

### Gemini

- **System message handling**: collapses differently than OpenAI; some prompts that work verbatim in OpenAI need restructuring.
- **JSON adherence**: strict schemas exist but are slightly less ironclad than OpenAI's strict mode. Always validate with Pydantic/Zod.
- **Tool-call schema differences**: not 1:1 with OpenAI; check the SDK before swapping.

---

## Picking per role in the agentic stack

| Role | Recommended | Why |
|------|-------------|-----|
| Intent classifier | **gemini-2.0-flash** | Cheap, fast, good enough for classification |
| Query rewriter | gpt-4o-mini or gemini-2.0-flash | Both work; flash slightly faster |
| Retrieval judge | gpt-4o-mini | Strict JSON adherence helps |
| Tool-calling research agent | **gpt-4o-mini** | More predictable tool calls |
| Final answer (user-facing) | gpt-4o or gemini-2.0-pro | Either; A/B test for your domain |
| LLM-as-judge in evals | gpt-4o-mini | Cheap, reliable JSON |
| Long-doc analysis (>50K tokens) | **gemini-2.0-pro** | The context wins |
| Voice / realtime | OpenAI Realtime API | Currently the most mature |

---

## Provider-agnostic code

The course's `LLMClient` wrapper (Module 1) is the recipe: a thin interface with two implementations. Avoid leaking provider-specific knowledge into agent code; isolate it in the client.

```python
client = LLMClient(provider="openai")    # default everywhere
client_cheap = LLMClient(provider="gemini")  # for classification/judging
```

Pattern: read the provider for each agent from a YAML config, so you can swap without code changes.

---

## When to use both at once (multi-provider routing)

Three good reasons:

1. **Cost ceiling**: route to gemini-2.0-flash up to your daily Gemini budget, then fall back to gpt-4o-mini.
2. **Resilience**: if OpenAI is degraded (rare but happens), fail over to Gemini.
3. **Quality A/B**: serve 10% of traffic via Gemini and compare eval scores week over week.

Reasons it's overkill:

- Personal projects.
- Small traffic where one provider's free/cheap tier suffices.
- Early stage where prompt iteration speed > production resilience.

---

## A note on Anthropic Claude

The course uses OpenAI + Gemini per your preference, but Claude (Anthropic) is the third major provider you should know about:

- Strongest for long, careful reasoning and writing.
- Best prompt caching ergonomics (explicit `cache_control` per content block).
- Native MCP support out of the box.
- Try Claude on the *final answer* agent — many teams find it produces more grounded, less syrupy retail copy.

Adding Claude to the wrapper is a 30-minute change; the rest of the course stays identical.

---

## Quick smoke test you can run anytime

```python
from src.llm_client import LLMClient, Message

PROMPT = "List 3 retail return-policy edge cases in one sentence each."

for p in ("openai", "gemini"):
    c = LLMClient(provider=p)
    r = c.chat([Message("system","Be terse."), Message("user", PROMPT)])
    print(f"\n[{p} | {r.model} | {r.prompt_tokens}+{r.completion_tokens}t]")
    print(r.text)
```

Run weekly to keep a feel for how outputs drift as the providers update their models.
