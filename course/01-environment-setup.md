# Module 1 — Environment & Tooling Setup

> **Time**: 2–4 hours
> **Goal**: Have a working Python *and* TypeScript scaffold that can call OpenAI and Gemini, write structured logs, and run a "hello agent" example.
> **Deliverable**: A repo with `hello_agent.py` and `hello_agent.ts` both running successfully.

This module is hands-on. Skip nothing — environment quirks here cause 80% of beginner frustration later.

---

## 1.1 Hardware & OS

- Mac, Linux, or Windows 10/11 with **16 GB RAM** minimum (8 GB works for most labs; 16 GB is comfortable).
- Stable broadband — you'll be pulling docs, embeddings, and model responses constantly.
- ~10 GB free disk for models, embeddings, and the Chroma local DB.

---

## 1.2 Install the core toolchain

### Step 1: Install Python 3.11+

```bash
# macOS (Homebrew)
brew install python@3.11

# Ubuntu/Debian
sudo apt update && sudo apt install -y python3.11 python3.11-venv

# Windows
# Install from https://www.python.org/downloads/ — make sure "Add to PATH" is checked
```

Verify:

```bash
python3.11 --version  # should print 3.11.x or higher
```

We'll use `uv` as the package manager (faster than pip, simpler than poetry). Install it:

```bash
curl -LsSf https://astral.sh/uv/install.sh | sh  # macOS/Linux
# Windows PowerShell:
powershell -c "irm https://astral.sh/uv/install.ps1 | iex"
```

> 💡 **Why `uv`?**
> `pip` is fine, but `uv` is 10–100× faster and handles virtual environments and lockfiles in one tool. You'll save real time as the course progresses.

### Step 2: Install Node.js 20+

```bash
# macOS
brew install node@20

# Ubuntu/Debian
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
sudo apt install -y nodejs

# Windows: download from https://nodejs.org
```

Verify:

```bash
node --version  # v20.x.x
npm --version
```

We'll use `pnpm` for TypeScript dependencies (faster, disk-efficient):

```bash
npm install -g pnpm
```

### Step 3: Install Git

If `git --version` doesn't work, install from https://git-scm.com.

### Step 4: VS Code (or Cursor) + extensions

Install:

- **Python** (Microsoft)
- **Pylance**
- **ESLint**
- **Prettier**
- **REST Client** (or use Postman) — for testing your API endpoints
- **GitLens** (optional, very helpful)

If you use Cursor, you already get an AI pair-programmer; if you use VS Code, install **GitHub Copilot** or **Continue** if you want one.

---

## 1.3 Get your API keys

You need **two** providers (we run side-by-side throughout the course).

### OpenAI

1. Go to https://platform.openai.com.
2. Sign up, add a payment method.
3. Go to *API Keys* → create a key, save it.
4. (Optional but smart) Set a monthly spend limit at *Billing → Usage limits*. Start at $20/month.

### Google Gemini

1. Go to https://aistudio.google.com.
2. Sign in with a Google account.
3. Click *Get API key* → create a new key in a project, save it.
4. The free tier covers most labs; you can stay on it for the whole course.

> ⚠️ **Common mistake**
> Never commit API keys to git. We'll set up `.env` files and `.gitignore` next. If you ever paste a key into a public place, **revoke it immediately** at the provider's dashboard.

---

## 1.4 Create the project repo

```bash
mkdir agentic-retail-copilot && cd agentic-retail-copilot
git init
```

Create this folder structure:

```
agentic-retail-copilot/
├── .env.example
├── .gitignore
├── README.md
├── docs/                       # Your design docs, eval specs, decision log
├── data/
│   ├── raw/                    # Original retail data (we'll load Module 2)
│   └── processed/              # Cleaned/chunked versions
├── configs/
│   ├── prompts/                # YAML prompt templates
│   └── evals/                  # Eval datasets
├── python/                     # Python implementation
│   ├── pyproject.toml
│   └── src/
│       ├── __init__.py
│       ├── app.py
│       ├── llm_client.py
│       ├── tools/
│       ├── retrieval/
│       ├── agents/
│       └── guardrails/
├── typescript/                 # TypeScript implementation
│   ├── package.json
│   ├── tsconfig.json
│   └── src/
│       ├── app.ts
│       ├── llmClient.ts
│       ├── tools/
│       ├── retrieval/
│       ├── agents/
│       └── guardrails/
├── notebooks/
└── scripts/
```

Create it in one shot:

```bash
mkdir -p docs data/raw data/processed configs/prompts configs/evals \
         python/src/{tools,retrieval,agents,guardrails} \
         typescript/src/{tools,retrieval,agents,guardrails} \
         notebooks scripts
touch python/src/__init__.py
```

### `.gitignore`

```gitignore
# Secrets
.env
.env.local
*.key

# Python
__pycache__/
*.pyc
.venv/
.uv/
*.egg-info/

# Node
node_modules/
dist/
.next/

# Data
data/processed/
data/raw/*.bin
*.db
chroma_db/

# OS
.DS_Store
Thumbs.db

# IDE
.vscode/settings.json
.idea/
```

### `.env.example`

```bash
# Copy to .env and fill in your real keys
OPENAI_API_KEY=sk-...
GOOGLE_API_KEY=AIza...

# Optional — for later modules
LANGFUSE_PUBLIC_KEY=
LANGFUSE_SECRET_KEY=
LANGFUSE_HOST=https://cloud.langfuse.com
```

Now copy to `.env` and fill in real values. **Never** commit `.env`.

---

## 1.5 Python setup

```bash
cd python
uv init --no-readme --no-pin-python
```

Edit the generated `pyproject.toml`:

```toml
[project]
name = "agentic-retail-copilot"
version = "0.1.0"
requires-python = ">=3.11"
dependencies = [
    "openai>=1.50.0",
    "google-genai>=0.3.0",
    "python-dotenv>=1.0.0",
    "pydantic>=2.8.0",
    "httpx>=0.27.0",
    "chromadb>=0.5.0",
    "tiktoken>=0.7.0",
    "pypdf>=4.3.0",
    "beautifulsoup4>=4.12.0",
    "pandas>=2.2.0",
    "fastapi>=0.115.0",
    "uvicorn>=0.30.0",
    "langchain>=0.3.0",
    "langchain-openai>=0.2.0",
    "langchain-google-genai>=2.0.0",
    "langchain-chroma>=0.1.0",
    "langgraph>=0.2.0",
    "rich>=13.7.0",
]

[tool.uv]
dev-dependencies = [
    "ruff>=0.6.0",
    "pytest>=8.3.0",
    "ipykernel>=6.29.0",
]
```

Install everything:

```bash
uv sync
```

This creates `.venv/` and a `uv.lock`. Activate the venv when you want to run scripts manually:

```bash
source .venv/bin/activate   # macOS/Linux
.venv\Scripts\activate      # Windows
```

Or just prefix commands with `uv run`:

```bash
uv run python src/app.py
```

---

## 1.6 TypeScript setup

```bash
cd ../typescript
pnpm init
```

Edit `package.json`:

```json
{
  "name": "agentic-retail-copilot-ts",
  "version": "0.1.0",
  "type": "module",
  "scripts": {
    "dev": "tsx watch src/app.ts",
    "build": "tsc",
    "start": "node dist/app.js"
  },
  "dependencies": {
    "openai": "^4.65.0",
    "@google/genai": "^0.3.0",
    "dotenv": "^16.4.0",
    "zod": "^3.23.0",
    "chromadb": "^1.9.0",
    "langchain": "^0.3.0",
    "@langchain/openai": "^0.3.0",
    "@langchain/google-genai": "^0.1.0",
    "@langchain/community": "^0.3.0",
    "@langchain/langgraph": "^0.2.0",
    "ai": "^3.4.0",
    "@ai-sdk/openai": "^0.0.66",
    "@ai-sdk/google": "^0.0.50",
    "express": "^4.21.0",
    "pino": "^9.4.0"
  },
  "devDependencies": {
    "typescript": "^5.6.0",
    "tsx": "^4.19.0",
    "@types/node": "^22.5.0",
    "@types/express": "^4.17.0"
  }
}
```

Install:

```bash
pnpm install
```

Create `tsconfig.json`:

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "ES2022",
    "moduleResolution": "bundler",
    "esModuleInterop": true,
    "strict": true,
    "skipLibCheck": true,
    "outDir": "dist",
    "rootDir": "src",
    "resolveJsonModule": true,
    "allowSyntheticDefaultImports": true
  },
  "include": ["src/**/*"]
}
```

---

## 1.7 Lab 1.1 — Hello LLM (both providers, both languages)

Goal: call OpenAI and Gemini once each, in both Python and TypeScript, and print the result.

### Python — `python/src/hello_llm.py`

```python
"""Hello-LLM: smoke-test both providers."""
import os
from dotenv import load_dotenv
from openai import OpenAI
from google import genai

load_dotenv(dotenv_path="../.env")

PROMPT = "In one sentence: what's the difference between an agent and a chatbot?"


def call_openai() -> str:
    client = OpenAI(api_key=os.environ["OPENAI_API_KEY"])
    response = client.chat.completions.create(
        model="gpt-4o-mini",
        messages=[{"role": "user", "content": PROMPT}],
    )
    return response.choices[0].message.content


def call_gemini() -> str:
    client = genai.Client(api_key=os.environ["GOOGLE_API_KEY"])
    response = client.models.generate_content(
        model="gemini-2.0-flash",
        contents=PROMPT,
    )
    return response.text


if __name__ == "__main__":
    print("=== OpenAI ===")
    print(call_openai())
    print("\n=== Gemini ===")
    print(call_gemini())
```

Run:

```bash
cd python
uv run python src/hello_llm.py
```

### TypeScript — `typescript/src/helloLlm.ts`

```typescript
import 'dotenv/config';
import OpenAI from 'openai';
import { GoogleGenAI } from '@google/genai';

const PROMPT =
  "In one sentence: what's the difference between an agent and a chatbot?";

async function callOpenAI(): Promise<string> {
  const client = new OpenAI({ apiKey: process.env.OPENAI_API_KEY });
  const res = await client.chat.completions.create({
    model: 'gpt-4o-mini',
    messages: [{ role: 'user', content: PROMPT }],
  });
  return res.choices[0].message.content ?? '';
}

async function callGemini(): Promise<string> {
  const client = new GoogleGenAI({ apiKey: process.env.GOOGLE_API_KEY! });
  const res = await client.models.generateContent({
    model: 'gemini-2.0-flash',
    contents: PROMPT,
  });
  return res.text ?? '';
}

(async () => {
  console.log('=== OpenAI ===');
  console.log(await callOpenAI());
  console.log('\n=== Gemini ===');
  console.log(await callGemini());
})();
```

Run:

```bash
cd typescript
pnpm tsx src/helloLlm.ts
```

> ⚠️ **If you see "API key not found"**
> Make sure `.env` is at the **repo root** (one level above `python/` and `typescript/`). Both `python-dotenv` and `dotenv` are configured to look there.

---

## 1.8 Lab 1.2 — Provider-agnostic LLM client

Hard-coding a provider in every file is a trap — you'll want to switch later. Let's build a small wrapper.

### Python — `python/src/llm_client.py`

```python
"""Unified LLM client: same interface for OpenAI and Gemini."""
from __future__ import annotations
import os
from dataclasses import dataclass
from typing import Literal, Sequence
from dotenv import load_dotenv
from openai import OpenAI
from google import genai

load_dotenv(dotenv_path="../.env")

Provider = Literal["openai", "gemini"]


@dataclass
class Message:
    role: Literal["system", "user", "assistant"]
    content: str


@dataclass
class LLMResponse:
    text: str
    model: str
    prompt_tokens: int
    completion_tokens: int


class LLMClient:
    def __init__(self, provider: Provider = "openai", model: str | None = None):
        self.provider = provider
        if provider == "openai":
            self.client = OpenAI(api_key=os.environ["OPENAI_API_KEY"])
            self.model = model or "gpt-4o-mini"
        else:
            self.client = genai.Client(api_key=os.environ["GOOGLE_API_KEY"])
            self.model = model or "gemini-2.0-flash"

    def chat(self, messages: Sequence[Message], temperature: float = 0.2) -> LLMResponse:
        if self.provider == "openai":
            res = self.client.chat.completions.create(
                model=self.model,
                messages=[{"role": m.role, "content": m.content} for m in messages],
                temperature=temperature,
            )
            return LLMResponse(
                text=res.choices[0].message.content or "",
                model=self.model,
                prompt_tokens=res.usage.prompt_tokens,
                completion_tokens=res.usage.completion_tokens,
            )
        else:
            # Gemini collapses system messages into the user turn; do it explicitly.
            system_parts = [m.content for m in messages if m.role == "system"]
            non_system = [m for m in messages if m.role != "system"]
            contents = []
            for m in non_system:
                role = "user" if m.role == "user" else "model"
                contents.append({"role": role, "parts": [{"text": m.content}]})
            config = {"temperature": temperature}
            if system_parts:
                config["system_instruction"] = "\n".join(system_parts)
            res = self.client.models.generate_content(
                model=self.model, contents=contents, config=config
            )
            usage = res.usage_metadata
            return LLMResponse(
                text=res.text or "",
                model=self.model,
                prompt_tokens=usage.prompt_token_count or 0,
                completion_tokens=usage.candidates_token_count or 0,
            )
```

### TypeScript — `typescript/src/llmClient.ts`

```typescript
import 'dotenv/config';
import OpenAI from 'openai';
import { GoogleGenAI } from '@google/genai';

export type Provider = 'openai' | 'gemini';

export interface Message {
  role: 'system' | 'user' | 'assistant';
  content: string;
}

export interface LLMResponse {
  text: string;
  model: string;
  promptTokens: number;
  completionTokens: number;
}

export class LLMClient {
  private provider: Provider;
  private model: string;
  private openai?: OpenAI;
  private gemini?: GoogleGenAI;

  constructor(provider: Provider = 'openai', model?: string) {
    this.provider = provider;
    if (provider === 'openai') {
      this.openai = new OpenAI({ apiKey: process.env.OPENAI_API_KEY });
      this.model = model ?? 'gpt-4o-mini';
    } else {
      this.gemini = new GoogleGenAI({ apiKey: process.env.GOOGLE_API_KEY! });
      this.model = model ?? 'gemini-2.0-flash';
    }
  }

  async chat(messages: Message[], temperature = 0.2): Promise<LLMResponse> {
    if (this.provider === 'openai') {
      const res = await this.openai!.chat.completions.create({
        model: this.model,
        messages,
        temperature,
      });
      return {
        text: res.choices[0].message.content ?? '',
        model: this.model,
        promptTokens: res.usage?.prompt_tokens ?? 0,
        completionTokens: res.usage?.completion_tokens ?? 0,
      };
    }

    // Gemini path
    const systemParts = messages.filter((m) => m.role === 'system').map((m) => m.content);
    const nonSystem = messages.filter((m) => m.role !== 'system');
    const contents = nonSystem.map((m) => ({
      role: m.role === 'user' ? 'user' : 'model',
      parts: [{ text: m.content }],
    }));
    const res = await this.gemini!.models.generateContent({
      model: this.model,
      contents,
      config: {
        temperature,
        systemInstruction: systemParts.join('\n') || undefined,
      },
    });
    const usage = res.usageMetadata;
    return {
      text: res.text ?? '',
      model: this.model,
      promptTokens: usage?.promptTokenCount ?? 0,
      completionTokens: usage?.candidatesTokenCount ?? 0,
    };
  }
}
```

---

## 1.9 Lab 1.3 — Structured logging

This will pay dividends from Module 3 onward. Set it up now.

### Python — `python/src/logging_setup.py`

```python
import json
import logging
import sys
import time
import uuid
from contextvars import ContextVar

request_id_var: ContextVar[str] = ContextVar("request_id", default="-")


class JsonFormatter(logging.Formatter):
    def format(self, record: logging.LogRecord) -> str:
        base = {
            "ts": time.strftime("%Y-%m-%dT%H:%M:%S", time.gmtime(record.created)),
            "level": record.levelname,
            "logger": record.name,
            "request_id": request_id_var.get(),
            "msg": record.getMessage(),
        }
        if hasattr(record, "extra_fields"):
            base.update(record.extra_fields)  # type: ignore[attr-defined]
        return json.dumps(base, ensure_ascii=False)


def setup_logging(level: str = "INFO") -> None:
    handler = logging.StreamHandler(sys.stdout)
    handler.setFormatter(JsonFormatter())
    root = logging.getLogger()
    root.handlers.clear()
    root.addHandler(handler)
    root.setLevel(level)


def new_request_id() -> str:
    rid = uuid.uuid4().hex[:12]
    request_id_var.set(rid)
    return rid


def log_event(logger: logging.Logger, event: str, **fields):
    logger.info(event, extra={"extra_fields": fields})
```

### TypeScript — `typescript/src/logging.ts`

```typescript
import pino from 'pino';
import { randomUUID } from 'node:crypto';

export const logger = pino({
  level: process.env.LOG_LEVEL ?? 'info',
  formatters: { level: (label) => ({ level: label }) },
  timestamp: () => `,"ts":"${new Date().toISOString()}"`,
});

export function newRequestId(): string {
  return randomUUID().slice(0, 12);
}
```

---

## 1.10 Lab 1.4 — Your first "hello agent"

Not a real agent yet — but a function that *thinks* like one. Given a user query, classify intent and produce a structured JSON response.

### Python — `python/src/hello_agent.py`

```python
import json
from logging_setup import setup_logging, new_request_id, log_event
import logging
from llm_client import LLMClient, Message

logger = logging.getLogger(__name__)
setup_logging()

SYSTEM = """You are a retail intent classifier. Given a shopper's query,
return JSON with these fields:
  classification: one of [product_search, order_status, return_request, policy_question, other]
  confidence: float 0-1
  rationale: one short sentence
Return only the JSON object, no markdown."""


def classify(query: str, provider: str = "openai") -> dict:
    rid = new_request_id()
    log_event(logger, "agent.request", query=query, provider=provider)
    client = LLMClient(provider=provider)  # type: ignore[arg-type]
    res = client.chat([Message("system", SYSTEM), Message("user", query)])
    log_event(
        logger,
        "agent.response",
        prompt_tokens=res.prompt_tokens,
        completion_tokens=res.completion_tokens,
        model=res.model,
    )
    return json.loads(res.text)


if __name__ == "__main__":
    samples = [
        "Do you have waterproof hiking jackets in size M under $150?",
        "Where is my order #4456?",
        "Can I return shoes that were on sale?",
    ]
    for q in samples:
        print(f"\n> {q}")
        print(classify(q))
```

### TypeScript — `typescript/src/helloAgent.ts`

```typescript
import { LLMClient } from './llmClient.js';
import { logger, newRequestId } from './logging.js';

const SYSTEM = `You are a retail intent classifier. Given a shopper's query,
return JSON with these fields:
  classification: one of [product_search, order_status, return_request, policy_question, other]
  confidence: float 0-1
  rationale: one short sentence
Return only the JSON object, no markdown.`;

export async function classify(query: string, provider: 'openai' | 'gemini' = 'openai') {
  const rid = newRequestId();
  logger.info({ rid, query, provider }, 'agent.request');
  const client = new LLMClient(provider);
  const res = await client.chat([
    { role: 'system', content: SYSTEM },
    { role: 'user', content: query },
  ]);
  logger.info(
    { rid, promptTokens: res.promptTokens, completionTokens: res.completionTokens, model: res.model },
    'agent.response'
  );
  return JSON.parse(res.text);
}

const samples = [
  'Do you have waterproof hiking jackets in size M under $150?',
  'Where is my order #4456?',
  'Can I return shoes that were on sale?',
];

(async () => {
  for (const q of samples) {
    console.log(`\n> ${q}`);
    console.log(await classify(q));
  }
})();
```

Run:

```bash
# Python
cd python && uv run python src/hello_agent.py
# TypeScript
cd typescript && pnpm tsx src/helloAgent.ts
```

You should see structured JSON output and JSON log lines.

> 💡 **Why this matters**
> This is the smallest possible step from "prompt → text" to "prompt → structured action." Every agent you build from here on follows the same shape: prompt the LLM, parse a typed response, log everything.

---

## 1.11 First commit

```bash
git add .
git commit -m "Module 1: env setup, llm client, hello agent"
git branch -M main
# Create a private repo on GitHub, then:
git remote add origin git@github.com:<you>/agentic-retail-copilot.git
git push -u origin main
```

---

## 1.12 Checklist before moving on

- [ ] `hello_llm` runs against both providers in Python and TypeScript.
- [ ] `hello_agent` returns valid JSON with classification + confidence + rationale.
- [ ] Logs are emitted as JSON with `request_id`.
- [ ] `.env` exists locally, is in `.gitignore`, and is **not** committed.
- [ ] You can switch providers with one argument (`provider="gemini"`).
- [ ] Repo is pushed to GitHub.

If all checked, head to [Module 2 — Domain & Data Preparation](02-domain-and-data.md).

---

## Decision-log prompts

- Which provider did you pick as your default? Why (cost, speed, quality)?
- Are you committing to Python, TypeScript, or both? If both, how will you keep them in sync?
- What's your monthly LLM budget? Where will you cap it?
