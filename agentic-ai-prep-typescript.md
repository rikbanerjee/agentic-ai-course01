# Agentic AI System Design – Course Prep (TypeScript Edition)

This markdown document is a structured prep curriculum designed to mirror the focus and style of the Maven course **“Building Agentic AI Applications with a Problem‑First Approach”**.

It is organized into modules you can run over 5–6 weeks, each with clear objectives, concepts, labs, and deliverables.

---

## 0. Overview & Learning Goals

### 0.1 Course-aligned capabilities

Based on the public syllabus and associated materials, this prep focuses on building the same core capabilities as the course:

- **Problem-first thinking**: framing business problems and designing AI systems around constraints (latency, cost, accuracy, risk) instead of starting from models or tools.
- **Context engineering**: using decomposition, meta‑prompts, and structured prompting as a design tool, not a bag of tricks.
- **Evaluation as backbone**: building product-level evals, LLM judges, and eval flywheels to iteratively improve systems.
- **Agentic retrieval & RAG**: integrating retrieval, memory, and self‑reflection into agents rather than treating RAG as a bolt‑on feature.
- **Multi-agent systems & MCP/A2A**: coordinating multiple agents and tools via explicit protocols and contracts, including MCP-style interfaces.
- **End‑to‑end build & iteration**: repeatedly refining a multi‑agent agentic search system with RAG, MCP, and coordination patterns.

### 0.2 Recommended background

The original course is accessible to builders who have basic terminal familiarity and are comfortable setting up tools and running scripts; it provides both low‑code and code‑heavy tracks.

This prep assumes you:

- Have basic familiarity with TypeScript/JavaScript (or are willing to learn enough to wire APIs and scripts).
- Can use the terminal for installing packages (npm/yarn/pnpm), running scripts, and using Git.
- Have at least one LLM provider account (e.g., OpenAI, Anthropic, Gemini) and understand API keys at a basic level.

---

## 1. Environment & Tooling Setup

### 1.1 Hardware & OS

- Recent Mac, Linux, or Windows machine with **16 GB RAM or more**.
- Stable broadband connection for downloading models, embeddings, and interacting with cloud APIs.

### 1.2 Core development stack

- **Editor**: VS Code or Cursor with:
  - ESLint and Prettier extensions.
  - REST client (Thunder Client, REST Client, or Postman externally).
- **Runtime**:
  - Node.js **18+** (or 20+).
  - Package manager: `npm`, `yarn`, or `pnpm`.
  - TypeScript execution: `tsx` or `ts-node` for running `.ts` files directly.
- **Version control**:
  - Git installed.
  - GitHub / GitLab / Bitbucket account.
  - Create a mono-repo, e.g. `agentic-ai-prep`.

Suggested repo structure:

```text
agentic-ai-prep/
  src/
  data/
    raw/
    processed/
  configs/
    prompts/
    evals/
  scripts/
  package.json
  tsconfig.json
  README.md
```

### 1.3 AI & data stack

- **LLM providers** (at least one):
  - OpenAI, Anthropic, or Gemini (plus an optional lower-cost backup provider).
  - Use official SDKs (e.g., `openai`, `@anthropic-ai/sdk`, `@google/genai`).
- **Vector store** (choose one):
  - Hosted: Pinecone, Qdrant Cloud, Weaviate Cloud.
  - Local: Chroma, local PostgreSQL + `pgvector`.
- **Data processing**:
  - Libraries: `pdf-parse` (for PDFs), `cheerio` (for HTML parsing), `tiktoken` (for token counting). 

### 1.4 Observability & evaluation helpers

Not strictly required, but recommended to mirror course emphasis on evaluation and observability.

- **Logging**:
  - Configure structured logs (JSON) using `pino` or `winston` with fields like `timestamp`, `user_id`, `request_id`, `prompt`, `model`, `latency_ms`, `tokens_in`, `tokens_out`.
- **Tracing / monitoring**:
  - Optional: Langfuse, Helicone.
- **Evaluation scripts**:
  - Dedicated TypeScript scripts for running batch evals and generating metrics.

---

## 2. Domain & Data Preparation

To mirror the problem-first orientation, you should fix a domain and work on a realistic dataset throughout the prep.

### 2.1 Choose a domain

Pick one domain where you could realistically deploy agentic AI:

- **Retail**: agentic search and discovery across product catalogs, policies, and merchandising guidelines.
- **Support**: triage and resolution assistant over tickets, FAQs, and internal runbooks.
- **Consulting / strategy**: research copilot that structures findings from long reports and decks.
- **Internal tools**: operations copilot for internal SOPs, HR policies, or IT runbooks.

### 2.2 Build a small enterprise corpus

Create a mixed-type corpus of **50–200 documents**:

- PDFs: internal decks, long reports, policy docs.
- Markdown / HTML: knowledge base articles, FAQs, SOPs.
- CSV / JSON: product catalogs, ticket logs, datasets.

Place these under `data/raw/` and keep track of data sources and basic metadata.

### 2.3 Mono-repo scaffold

Fill the repo structure with minimal placeholder files:

- `src/app.ts` – entry point for experiments (CLI or API).
- `src/llm_client.ts` – wrapper around your chosen LLM provider.
- `src/retrieval/` – RAG-related utilities.
- `configs/prompts/` – YAML/JSON prompt templates.
- `configs/evals/` – evaluation datasets.
- `scripts/00_exploration.ts` – quick exploration of your corpus.

Add an initial `README.md` describing your domain, data sources, and high-level goal.

---

## 3. Module 0 – Foundations & Orientation (2–3 days)

This module aligns you with the instructors’ vocabulary and conceptual framing using their free **Agentic AI Crash Course**.

### 3.1 Objectives

- Understand the difference between vanilla GenAI and agentic systems.
- Learn the **four types of agentic systems** and when to use each.
- Get a high-level view of tools, RAG, and MCP from the course creators’ perspective.

### 3.2 Materials

Work through the crash course sections:

1. *What Are AI Agents Anyway?*
2. *The 4 Types of Agentic Systems (and When to Use What)*.
3. *What Are Tools in AI?*
4. *What Is RAG, and What Does It Mean to Make It Agentic?*
5. *What Is MCP and Why Should You Care?*

### 3.3 Activities

- For each agent type in the crash course, map it to **one concrete scenario** in your chosen domain.
- Sketch a quick text diagram showing:
  - User → Agent(s) → Tools / Data → Output.

### 3.4 Deliverable

Create a 1–2 page **“Agentic AI Map”**:

- Key user journeys.
- For each journey: agent type, tools needed, RAG vs no‑RAG, primary constraints (latency, cost, safety).

Store this as `docs/agentic_ai_map.md` in your repo.

---

## 4. Module 1 – Intro to GenAI & Agentic Apps (Week 1)

This mirrors the course’s first week: introduction, setup, and building foundational applications plus the first assignment.

### 4.1 Objectives

- Explain why traditional software assumptions break down in AI-driven systems.
- Build a simple, traceable GenAI micro‑application in your domain.

### 4.2 Concept work

Study and reflect on:

- Deterministic vs probabilistic behavior: why AI systems require **eval loops**.
- Failure surfaces in your domain:
  - Hallucinations.
  - Missing context or retrieval failures.
  - Unsafe or non-compliant outputs.
- Why “shipping a prompt” is the start, not the end, of building a GenAI system.

### 4.3 Labs

**Lab 1: Hello GenAI API**

- Implement a CLI or minimal HTTP API (e.g., using Express or Fastify) in `src/app.ts` that:
  - Accepts a domain-specific user request (e.g., “Help me compare product X vs Y”).
  - Calls your LLM via `src/llm_client.ts` with a **fixed base prompt**.
  - Returns structured JSON such as:

```json
{
  "classification": "comparison",
  "reasoning": "...",
  "action_suggestion": "..."
}
```

**Lab 2: Basic tracing**

- Add logging middleware that records:
  - `request_id`, `user_query`, `prompt`, `model_name`, `latency_ms`, `tokens_in`, `tokens_out`.
- Create a simple script (`scripts/analyze_logs.ts`) to:
  - Load logs.
  - Compute and visualize average latency and token usage.

### 4.4 Assignment 1 – Problem & System Memo

Create a 1–2 page **System Memo** (`docs/system_memo.md`):

- **Problem framing**: target user, pain points, current non‑AI baseline.
- **AI role**: where AI adds value and where you explicitly **do not** allow AI to act.
- **Constraints**: latency, cost, safety/security requirements, correctness thresholds.

This mirrors the course’s emphasis on problem‑first, constraint‑aware design.

---

## 5. Module 2 – Context Engineering & Enterprise Agents (Week 2)

Week 2 of the course focuses on **context engineering in 2026** and building agents suitable for enterprise environments.

### 5.1 Objectives

- Use prompt decomposition and meta‑prompts to structure complex tasks.
- Build an agent that uses context windows intelligently rather than “stuffing more text.”

### 5.2 Concept work

- Study the “pillars” of GenAI systems described by the instructors: instructions, context, reasoning, and tools.
- Explore patterns for context management:
  - Structured inputs (JSON schemas, enumerations).
  - Summarization and selective recall.
  - Retrieval + reranking for relevance.
- Understand how enterprise constraints (PII, compliance, access control) should appear in prompts and context policies.

### 5.3 Labs

**Lab 3: Prompt decomposition**

- Take a complex domain task and decompose into steps:
  - Extract relevant facts or entities.
  - Reason over those facts.
  - Format a final answer for the user.
- Implement each step as a separate function + prompt; chain them in `src/workflows/decomposed_task.ts`.

**Lab 4: Meta-prompt & config-driven design**

- Define a meta-prompt file in `configs/prompts/meta_prompt.yaml` containing:
  - Role and persona.
  - Style and tone guidelines.
  - Tool usage guidelines.
  - Safety & compliance rules.
- Load this config at runtime and interpolate variables like `{domain}`, `{tools}`, `{constraints}`.

**Lab 5: Enterprise guardrails via prompts**

- Encode explicit **“never” rules** in your prompts (e.g., never invent confidential IDs, never claim access to systems you don’t have).
- Design a small adversarial test set where the user tries to elicit policy violations, and confirm the agent responds safely.

### 5.4 Assignment 2 – Context Policy Doc

Create `docs/context_policy.md` describing:

- What user & system information is allowed into context.
- How you **truncate, summarize, or filter** conversation histories.
- How you handle sensitive attributes (masking, exclusion).
- How retrieval results are filtered before being passed to the model.

This mirrors the course’s focus on **context engineering for enterprise use cases**.

---

## 6. Module 3 – Evaluation & Guardrails as Backbone (Week 3)

The course treats evaluation as the backbone of reliable agentic systems and emphasizes **product‑level evals** and **eval flywheels**.

### 6.1 Objectives

- Build a labeled evaluation dataset tied to your business problem.
- Implement both automated and LLM-as-judge evaluation.
- Introduce guardrails and measure their impact.

### 6.2 Concept work

- Distinguish between:
  - **Model benchmarks** (e.g., standard LLM leaderboards) and **product evals** (task success in your app).
  - **Offline evals** (test suites) and **online evals** (metrics from real usage).
- Define metrics that matter for your domain:
  - Correctness/groundedness.
  - Helpfulness.
  - Safety/compliance.
  - Coverage/recall.

### 6.3 Labs

**Lab 6: Eval dataset construction**

- Create 30–50 test cases in `configs/evals/base_eval_set.json`:
  - `input` – user query.
  - `gold_answer` – your ideal answer.
  - `tags` – category, difficulty, sensitive vs non-sensitive.

**Lab 7: LLM-as-judge harness**

- Implement `scripts/run_evals.ts` that:
  - Runs your current system on all test cases.
  - Calls an evaluator model with a rubric-based prompt to score outputs on correctness and safety.
  - Aggregates scores and prints a summary (mean, percent above threshold, worst cases).

**Lab 8: Guardrails pass**

- Implement pre- and post-processing hooks in `src/guardrails/`:
  - Input filters for disallowed topics.
  - Output filters to catch policy violations or missing citations.
- Re-run evals and compare metrics before/after guardrails.

### 6.4 Assignment 3 – Evaluation & Guardrails Spec

Create `docs/eval_and_guardrails_spec.md` covering:

- Metrics and target thresholds.
- Evaluation process (how often you run evals, how you add new cases).
- Guardrail logic and tradeoffs (risk of false positives vs false negatives).

---

## 7. Module 4 – Agentic Retrieval, RAG, Memory & Reflection (Week 4)

The course highlights integrating retrieval, memory, and **self‑reflective behavior** into agentic systems rather than treating RAG as an isolated feature.

### 7.1 Objectives

- Implement a full RAG pipeline over your corpus.
- Add “agentic“ behaviors such as query reformulation and reflection.

### 7.2 Concept work

- Compare **traditional RAG** vs **agentic RAG**:
  - Traditional: single-step query → retrieve top‑k → answer.
  - Agentic: iterative retrieval, query refinement, multiple tools, and self‑reflection.
- Understand design tradeoffs:
  - Chunk size & overlap.
  - Top‑k selection and reranking.
  - Latency vs accuracy vs cost.

### 7.3 Labs

**Lab 9: Baseline RAG**

- Build an index:
  - Chunk your `data/raw/` documents into passages.
  - Create embeddings and store in your chosen vector DB.
- Implement `src/retrieval/rag.ts` with typed functions:
  - `retrieve(query: string): Document[]`.
  - `answer_with_evidence(query: string, docs: Document[]): string`, where the prompt forces citing sources.

**Lab 10: Agentic behaviors in RAG**

Implement at least one of the following:

- **Query rewriting**: an agent that rewrites vague user queries into more specific retrieval queries.
- **Multi-step retrieval**: initial retrieval → gap analysis → targeted follow-up retrieval.
- **Reflection**: model critiques its own answer and, if low-confidence, attempts a second answer using different docs.

**Lab 11: RAG evals**

Extend your eval harness to capture RAG-specific metrics:

- Was the gold answer’s source document retrieved at all?
- How often are citations correct?
- Do agentic behaviors improve eval scores relative to baseline RAG?

### 7.4 Assignment 4 – RAG Design Doc

Create `docs/rag_design.md` describing:

- Chunking strategy and rationale.
- Embedding model and distance metric.
- Retrieval parameters (k, filters, rerankers).
- Reflection and query-rewriting strategies, including tradeoffs.

---

## 8. Module 5 – Multi-Agent Systems, Tools & MCP/A2A (Week 5)

The course explicitly covers **multi-agent systems**, coordination patterns, and protocols like **MCP/A2A** for more complex workflows.

### 8.1 Objectives

- Design a multi-agent architecture for your domain problem.
- Implement multi-agent coordination using clear tool/service contracts.

### 8.2 Concept work

- Revisit the four agentic system types from the crash course and map them to multi-agent patterns.
- Study tradeoffs in multi-agent systems:
  - Autonomy vs control.
  - Complexity vs reliability.
  - Latency cost of coordination.
- Understand MCP-style thinking:
  - Tools as typed interfaces with schemas for input/output and errors.

### 8.3 Labs

**Lab 12: Single agent with tools**

- Implement a primary agent that can:
  - Call your RAG tool.
  - Call at least one external API (e.g., product DB, analytics, search).
- Define tool signatures as JSON schemas and validate arguments at runtime using libraries like `zod` or `ajv`.

**Lab 13: Multi-agent workflow**

- Split responsibilities across 2–3 agents, e.g.:
  - **Router agent** – classifies user intent and chooses a path.
  - **Research agent** – performs retrieval and synthesis.
  - **Decision agent** – applies business rules and writes final answer.
- Implement message passing using a simple envelope:

```json
{
  "from": "router",
  "to": "researcher",
  "conversation_id": "...",
  "payload": { ... }
}
```

**Lab 14: MCP-style service contracts**

- Document your tools and services in `docs/tool_contracts.md`:
  - Inputs, outputs, error codes, latency expectations.
- Adjust your agents’ prompts to respect these contracts, making calls explicit and predictable.

### 8.4 Assignment 5 – Multi-Agent Architecture Diagram

Create `docs/multi_agent_architecture.md` containing:

- A diagram (ASCII, Mermaid, or embedded image) showing agents, tools, and data stores.
- Notes on:
  - Why you chose a multi-agent approach vs single agent.
  - Failure handling strategies between agents.

---

## 9. Module 6 – Capstone-style Agentic Search System (Week 6)

The Maven capstone centers on building an **end‑to‑end agentic AI solution**, specifically an agentic search system iterated through multiple versions and presented to a large audience.

### 9.1 Objectives

- Assemble everything into a coherent agentic search or copilot system.
- Drive iterations using evaluation and observability.

### 9.2 Build iterations

**V0 – Simple RAG search**

- Expose an `ask(query: string)` endpoint backed by your RAG implementation.
- Log all inputs, retrieved docs, and outputs.

**V1 – Agentic search**

- Insert an agent layer that:
  - Reformulates user queries for retrieval.
  - Chooses between multiple document spaces or tools as needed.
  - Uses reflection to retry weak answers.

**V2 – Multi-agent search with MCP-style services**

- Introduce dedicated agents:
  - Search orchestrator.
  - Domain specialist.
  - Safety/compliance checker.
- Treat RAG, external APIs, and internal services as MCP-style tools with explicit contracts.

### 9.3 Evaluation, observability & iteration

- Integrate your eval harness into CI or a regular job.
- Track metrics over time as you change prompts, architectures, or tools.
- Use logs and traces to:
  - Identify common failure modes.
  - Add new eval cases for those failures (your eval flywheel).

### 9.4 Capstone artifacts

Create:

- A **design review deck** (3–5 slides or a markdown equivalent) covering:
  - Problem & constraints.
  - Architecture.
  - Evaluation methodology.
  - Key metrics and limitations.
- A **demo script** (~5 minutes) walking through:
  - One “happy path” scenario.
  - One “edge case” that highlights a design tradeoff.

Store these as `docs/capstone_design_review.md` and `docs/capstone_demo_script.md`.

---

## 10. Weekly Study Rhythm

The course runs in a cohort format with live sessions, async work, and capstone time; you can mirror this pacing.

### 10.1 Suggested schedule (per week)

Aim for **10–12 hours/week**:

- 3–4 hours – “Live equivalent”:
  - Watch or re-watch system design / agentic AI talks.
  - Take active notes and immediately implement 1–2 ideas.
- 3–4 hours – Async work:
  - Reading, refining prompts, improving configs and scripts.
- 3–4 hours – Project time:
  - Advancing your capstone agentic search system.

### 10.2 Decision log

Maintain `docs/decision_log.md`:

- Every significant choice (model, embedding, prompt pattern, architecture change).
- Rationale, experiment results, and links to eval reports.

This mirrors the course’s emphasis on **decision maps and tradeoffs** rather than checklist-style topic coverage.

---

## 11. Using This Prep

### 11.1 If you are taking the Maven course

- Aim to complete **Modules 0–3** before the cohort starts so you can use live time on nuance and feedback rather than basic setup.
- Use Modules 4–6 during or after the course to deepen your capstone.

### 11.2 If you are mirroring the course internally

- Treat each module above as a **weekly block** for your team.
- Use the assignments (memos, design docs, eval specs) as templates for internal deliverables.
- Run weekly review sessions where team members “present” their work, mirroring the course’s live project reviews.

This document should give you a structured, repo‑friendly way to prepare for or mirror the **Building Agentic AI Applications with a Problem‑First Approach** course while aligning closely with its syllabus and philosophy.
