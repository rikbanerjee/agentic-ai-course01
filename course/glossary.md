# Glossary

Every term used in the course, defined plainly. Skim it now; refer back as you read.

---

**Agent** — A program that uses an LLM to decide what to do next given a goal, a set of tools, and prior observations. The LLM is the "brain"; the loop around it is the "body."

**Agentic RAG** — RAG with a loop: the agent judges retrieval quality and can reformulate the query, retrieve again, or escalate.

**Agentic system** — Any system where one or more LLMs make decisions about which actions to take, in what order, with the ability to act on the environment via tools.

**API key** — A string that authenticates calls to a service (OpenAI, Gemini). Treat like a password; never commit to git.

**Batch API** — A discounted (usually 50%) async API for processing many requests at once. Returns within 24h; perfect for evals and offline jobs.

**Capstone** — The end-to-end project in this course: the retail agentic search copilot in V0/V1/V2 form.

**Chain-of-thought (CoT)** — A prompting pattern where the model is instructed to reason step-by-step before answering. Improves accuracy on hard tasks; costs more tokens.

**Chroma** — An embedded vector database. Stores embeddings on local disk; no server required for Python.

**Chunking** — Splitting documents into smaller pieces for indexing. Strategies range from fixed-size to header-aware to AST-based.

**Constraints (problem-first)** — Latency, cost, accuracy, safety targets that shape architectural choices from day one.

**Context engineering** — The 2026 term for prompt engineering, expanded to mean: designing the whole pipeline of information the model sees on every call.

**Context window** — The max number of tokens the model accepts as input + output in one call.

**Cosine similarity** — A distance metric between two vectors; what most vector DBs use to find "similar" items.

**Cross-encoder** — A model that takes two inputs (query, document) and outputs a relevance score. Used for reranking.

**Decision agent** — In a multi-agent system, the agent that applies business rules and writes the user-facing answer.

**Decomposition** — Breaking a complex task into smaller, independently testable steps.

**Embedding** — A fixed-length vector representing the meaning of a piece of text.

**Embedding model** — The model that produces embeddings (e.g., `text-embedding-3-small`).

**Eval (evaluation)** — A labeled test set + scoring procedure that measures system quality.

**Eval flywheel** — The habit of converting every reported failure into a new eval case, so the same failure can never silently regress.

**Few-shot prompting** — Including a few input→output examples in the system prompt to teach the model the format/style.

**FastAPI** — A Python web framework, ergonomic for building agent APIs.

**Function calling / Tool use** — The LLM API feature where the model can choose to "call" a function (returning the name and args); your code executes it; you feed the result back.

**Gemini** — Google's family of LLMs (`gemini-2.0-flash`, `gemini-2.0-pro`, etc).

**Generation** — In RAG, the step where the LLM produces the final answer from the retrieved context.

**Groundedness** — Whether every claim in an answer can be traced to a real source (retrieved chunk or tool result).

**Guardrails** — Pre- and post-processing rules that filter unsafe inputs and outputs.

**Hallucination** — When the model invents facts that aren't in its inputs or training (or are contradicted by them).

**Helpfulness** — A quality metric (often LLM-judged) measuring how useful an answer would be to the user.

**Indexing** — The one-time/periodic process of chunking, embedding, and storing documents in a vector DB.

**Intent classification** — Mapping a user query to a label (product_search, order_status, etc.) that drives routing.

**JSON mode** — An LLM feature where the response is guaranteed to be valid JSON. Different from structured outputs (which guarantee a *specific* schema).

**LangChain** — A popular Python/TS framework with chains, retrievers, and integrations.

**LangGraph** — A LangChain library for building stateful, multi-actor agent workflows as graphs.

**Langfuse** — An open-source LLM observability tool: traces, evals, prompt management.

**LangSmith** — LangChain's hosted observability/eval platform.

**LLM (Large Language Model)** — A neural network trained on lots of text, capable of generating coherent text given a prompt.

**LLM-as-judge** — Using one LLM to score the outputs of another, against a rubric.

**MCP (Model Context Protocol)** — An open protocol for connecting LLM apps to external data sources and tools.

**Meta-prompt** — A versioned, parameterized prompt template (often YAML) loaded at runtime.

**Memory** — Information from prior turns or sessions that the agent retains. Layered into short-term, mid-term, long-term.

**Model benchmark** — Generic capability test (MMLU, HumanEval). Distinct from product evals.

**MRR (Mean Reciprocal Rank)** — A retrieval metric: 1/rank of the first correct result.

**Multi-agent system** — A system where multiple specialized agents collaborate, often via a router or orchestrator.

**OpenAI** — The provider of the GPT family (`gpt-4o`, `gpt-4o-mini`, etc.) and `text-embedding-3-*`.

**Pinecone / Qdrant / Weaviate** — Hosted vector databases.

**pgvector** — A Postgres extension for storing and querying vectors.

**Plan-and-execute agent** — An agent that produces a multi-step plan upfront, then executes each step.

**PII** — Personally Identifiable Information: emails, addresses, SSNs, payment info.

**Problem-first thinking** — Designing AI systems by starting from user, constraints, and success metrics — not from models or frameworks.

**Prompt** — The text sent to the LLM. Usually composed of system message, conversation history, retrieved context, and user input.

**Prompt caching** — Provider feature that caches static prefixes of prompts, drastically cutting cost for repeat calls.

**Prompt injection** — An attack where the user puts text in the prompt that tries to override the system instructions ("ignore previous instructions...").

**Provider** — The company/API serving the model (OpenAI, Anthropic, Google).

**Pydantic** — A Python library for typed data validation. Powers structured outputs and tool input schemas.

**RAG (Retrieval-Augmented Generation)** — Pattern: retrieve relevant docs first, then generate the answer using them.

**ReAct (Reason + Act)** — The classic agent loop: the LLM reasons about what to do, picks a tool, observes the result, and decides whether to act again or finish.

**Recall@k** — Retrieval metric: did at least one of the gold-standard documents appear in the top-k retrieved?

**Reflection** — An agent pattern where the model critiques its own answer and decides whether to retry.

**Reranking** — A second-stage scoring of retrieved results by a cross-encoder, to surface the most relevant.

**Retrieval** — The step in RAG where you find the top-k most relevant chunks for a query.

**Retrieval judge** — A small LLM call that asks "do these chunks actually answer the query?" — used to trigger retries.

**Reflexive agent** — A simple agent: pick one tool, call it, return. No loop.

**Refusal handler** — An agent / branch that returns a templated decline for off-topic or adversarial inputs.

**Router agent** — In multi-agent systems, the agent that classifies intent and dispatches to a specialist.

**Schema (JSON schema)** — A structured definition of an object's fields, types, and constraints. Used for tool inputs/outputs.

**SDK** — Software Development Kit. The official client library for a service (`openai`, `@anthropic-ai/sdk`, etc.).

**Span** — A timed unit of work in a trace (e.g., "router.classify took 612ms").

**Streaming** — Sending the response back chunk by chunk so the user sees it appear incrementally.

**Structured output** — An LLM response guaranteed to match a given schema (Pydantic, Zod, JSON Schema).

**Subagent** — An agent invoked by another agent.

**System prompt** — The instructions sent at the top of every conversation, defining the model's role, rules, and format.

**Temperature** — A parameter that controls randomness in the model's output. 0 = deterministic; higher = more varied.

**Tenacity** — A Python library for retries with backoff and jitter.

**Token** — The unit of text the model processes (roughly 4 characters or 0.75 words in English).

**Tool** — A function the agent can call. Has a name, description, typed inputs, typed outputs.

**Tool contract** — Documented agreement about a tool's inputs, outputs, error shapes, and latency budget.

**Trace** — The tree of spans for a single request. Lets you reconstruct exactly what happened.

**uv** — A fast Python package manager (Astral). Replaces pip + venv + sometimes poetry.

**Vector database** — A database optimized for nearest-neighbor search over embeddings.

**Vercel AI SDK** — A TypeScript-first SDK for building streaming AI UIs.

**Zod** — A TypeScript schema validation library. The TS counterpart to Pydantic.
