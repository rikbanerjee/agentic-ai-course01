# Deployment Patterns — FastAPI, Next.js, Docker, CI

> **When to read this**: when your V2 capstone is working locally and you want it accessible to someone other than you.
> **Goal**: A repeatable, secure, observable deployment for both the Python and TypeScript versions.

---

## 1. The deployment shape

```
┌──────────────┐   HTTPS    ┌─────────────────┐   tools  ┌──────────────┐
│  Web client  │ ─────────▶ │   Agent API     │ ───────▶ │  Catalog DB  │
│ (Next.js,    │            │ (FastAPI /      │          │  Order DB    │
│  Streamlit)  │            │  Express)       │          │  Chroma      │
└──────────────┘            └────────┬────────┘          └──────────────┘
                                     │
                                     ▼
                            ┌──────────────────┐
                            │ LLM providers    │
                            │ (OpenAI, Gemini) │
                            └──────────────────┘
                                     │
                                     ▼
                            ┌──────────────────┐
                            │ Langfuse traces  │
                            └──────────────────┘
```

For a course project, "deploy" can mean anything from `uvicorn --host 0.0.0.0` on a laptop to a full Kubernetes cluster. We'll cover three rungs:

1. **Local + ngrok** — share with one person for 30 min.
2. **Single VM (Fly.io / Render / Railway)** — long-running, low-cost, no orchestration.
3. **Container in a managed runtime (Cloud Run / ECS / Vercel)** — production-flavored.

---

## 2. Rung 1 — Local + ngrok

```bash
# Terminal 1
cd python && uv run uvicorn src.app_v2:app --host 0.0.0.0 --port 8000

# Terminal 2
ngrok http 8000
```

Ngrok prints a `https://abc123.ngrok-free.app` URL. Share it.

When to use: demos, friends-and-family testing. Don't leave keys exposed; rotate after.

---

## 3. Rung 2 — Single-VM deploy with Docker

### Python — `python/Dockerfile`

```dockerfile
FROM python:3.11-slim
WORKDIR /app

# Install uv
RUN pip install --no-cache-dir uv

# Copy lockfile + project files
COPY pyproject.toml uv.lock ./
RUN uv sync --frozen --no-dev

COPY src/ ./src/
COPY ../data ./data/
COPY ../configs ./configs/

ENV PYTHONUNBUFFERED=1
EXPOSE 8000

CMD ["uv", "run", "uvicorn", "src.app_v2:app", "--host", "0.0.0.0", "--port", "8000"]
```

### TypeScript — `typescript/Dockerfile`

```dockerfile
FROM node:20-slim AS builder
WORKDIR /app
COPY package.json pnpm-lock.yaml ./
RUN corepack enable && pnpm install --frozen-lockfile
COPY tsconfig.json ./
COPY src ./src
RUN pnpm build

FROM node:20-slim
WORKDIR /app
COPY --from=builder /app/node_modules ./node_modules
COPY --from=builder /app/dist ./dist
COPY ../data ./data
COPY ../configs ./configs
EXPOSE 8000
CMD ["node", "dist/appV2.js"]
```

### `docker-compose.yml` (run both + Chroma + Langfuse self-hosted)

```yaml
version: "3.9"
services:
  chroma:
    image: chromadb/chroma:latest
    ports: ["8001:8000"]
    volumes: ["./chroma_db:/chroma/chroma"]

  agent-py:
    build: ./python
    env_file: .env
    environment:
      - CHROMA_HOST=chroma
      - CHROMA_PORT=8000
    ports: ["8000:8000"]
    depends_on: [chroma]

  agent-ts:
    build: ./typescript
    env_file: .env
    ports: ["8002:8000"]
    depends_on: [chroma]
```

Deploy to Fly.io:

```bash
fly launch --no-deploy           # detects Dockerfile
fly secrets set OPENAI_API_KEY=sk-...
fly deploy
```

To Render: create a new Web Service → point at the repo → it auto-detects the Dockerfile. To Railway: similar UX.

Cost: <$10/month for low-traffic.

---

## 4. Rung 3 — Managed container runtime

### Google Cloud Run (Python)

```bash
# build & push
gcloud builds submit --tag gcr.io/<project>/retail-copilot-py

# deploy
gcloud run deploy retail-copilot-py \
  --image gcr.io/<project>/retail-copilot-py \
  --platform managed \
  --region us-central1 \
  --memory 1Gi \
  --set-env-vars LANGFUSE_HOST=https://cloud.langfuse.com \
  --set-secrets OPENAI_API_KEY=openai-key:latest,GOOGLE_API_KEY=gemini-key:latest \
  --allow-unauthenticated  # remove this for private services
```

Pros: scale-to-zero, pay-per-request, integrated logging. Cons: cold starts (mitigate with min-instances=1 on the hot path).

### Vercel (Next.js + TS agent)

Vercel is the natural home for a TypeScript agent paired with a Next.js UI.

```bash
# next-app/pages/api/ask.ts
import { agenticAsk } from '@/lib/agents/multiAgent';
export default async function handler(req, res) {
  const result = await agenticAsk(req.body.query);
  res.json(result);
}
```

Deploy:

```bash
vercel
vercel env add OPENAI_API_KEY
vercel --prod
```

Caveat: serverless function timeouts (10s Hobby, 60s Pro). For longer flows, use Vercel's edge runtime + streaming, or split the long-running step to a queue.

---

## 5. Streaming UI — Next.js + AI SDK

A real product needs a streaming chat UI. The Vercel AI SDK makes this trivial.

```tsx
// next-app/app/chat/page.tsx
'use client';
import { useChat } from 'ai/react';

export default function Chat() {
  const { messages, input, handleInputChange, handleSubmit } = useChat({ api: '/api/ask' });
  return (
    <div className="max-w-2xl mx-auto p-4">
      {messages.map((m) => (
        <div key={m.id} className={m.role === 'user' ? 'text-right' : ''}>
          <b>{m.role}: </b>{m.content}
        </div>
      ))}
      <form onSubmit={handleSubmit}>
        <input value={input} onChange={handleInputChange} className="border p-2 w-full" />
      </form>
    </div>
  );
}
```

```ts
// next-app/app/api/ask/route.ts
import { openai } from '@ai-sdk/openai';
import { streamText } from 'ai';

export async function POST(req: Request) {
  const { messages } = await req.json();
  // For multi-agent, you'd call your graph here and forward its streamed output.
  const result = await streamText({ model: openai('gpt-4o-mini'), messages });
  return result.toAIStreamResponse();
}
```

---

## 6. CI/CD pipeline

`.github/workflows/ci.yml`:

```yaml
name: CI
on: [push, pull_request]
jobs:
  python:
    runs-on: ubuntu-latest
    defaults: { run: { working-directory: python } }
    steps:
      - uses: actions/checkout@v4
      - uses: astral-sh/setup-uv@v3
      - run: uv sync
      - run: uv run ruff check src
      - run: uv run pytest -q
      - env: { OPENAI_API_KEY: ${{ secrets.OPENAI_API_KEY }} }
        run: uv run python scripts/run_evals.py --threshold-check

  typescript:
    runs-on: ubuntu-latest
    defaults: { run: { working-directory: typescript } }
    steps:
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v4
        with: { version: 9 }
      - uses: actions/setup-node@v4
        with: { node-version: 20, cache: pnpm }
      - run: pnpm install
      - run: pnpm tsc --noEmit
      - run: pnpm test

  deploy:
    needs: [python, typescript]
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: superfly/flyctl-actions/setup-flyctl@master
      - run: flyctl deploy --remote-only
        env: { FLY_API_TOKEN: ${{ secrets.FLY_API_TOKEN }} }
```

The eval-threshold step from Module 6 belongs here. A red CI on falling quality is the only thing that *really* prevents regressions.

---

## 7. Security & secrets

| Concern | Practice |
|---------|----------|
| LLM API keys | Use the platform's secret manager. Never bake into images. |
| User PII | Mask at the edge before the prompt; store raw in a separate DB. |
| Prompt injection | Module 3's guardrails + an LLM-based classifier for tricky cases. |
| Rate limiting | Per-IP and per-user. Default to 30 RPM per user; throttle hot paths. |
| Auth | At minimum, a signed bearer token. For real apps, OAuth + per-user quotas. |
| Audit log | Every "high-impact" action (refund, account change) gets a permanent log entry. |

```python
# Simple per-IP rate limit with slowapi
from slowapi import Limiter
from slowapi.util import get_remote_address
limiter = Limiter(key_func=get_remote_address)
app.state.limiter = limiter

@app.post("/ask")
@limiter.limit("30/minute")
def ask(req: AskRequest, request: Request): ...
```

---

## 8. Reliability patterns

| Failure | Pattern |
|---------|---------|
| LLM provider 429/500 | Exponential backoff + retry (e.g., `tenacity`) with jitter |
| Provider degraded | Fallback to secondary provider (Gemini ← OpenAI or vice versa) |
| Tool timeout | Wrap with `asyncio.wait_for(...)`; return `{"error":"timeout"}` instead of hanging |
| Vector DB unreachable | Cached "I'm having trouble looking that up" response |
| All retries exhausted | Open a circuit breaker for N seconds; serve degraded responses |

```python
from tenacity import retry, stop_after_attempt, wait_exponential
@retry(stop=stop_after_attempt(3), wait=wait_exponential(multiplier=1, max=10))
def call_llm(...):
    return client.chat.completions.create(...)
```

---

## 9. Production checklist

- [ ] Containerized; image < 500 MB.
- [ ] CI runs lints, tests, **and evals** on every PR.
- [ ] Secrets in the platform's secret manager.
- [ ] Per-user rate limit + per-IP rate limit.
- [ ] Streaming enabled on user-facing endpoints.
- [ ] Langfuse (or equivalent) traces flowing from prod.
- [ ] Health check endpoint (`/health`) and graceful shutdown.
- [ ] Documented rollback path (last known good image tag).
- [ ] Runbook for: provider outage, vector DB outage, eval threshold breach.

---

## 10. What "production-ready" really means

Beyond uptime: someone other than you can answer a user-reported failure within an hour. That requires logs + traces + evals + runbooks + rollbacks. Each rung above adds one more piece of that machine.

You don't need all of it on day one. You do need a plan for adding each piece before usage grows past the point where missing it becomes painful.
