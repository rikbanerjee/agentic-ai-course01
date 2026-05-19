# Resources & Further Reading

Curated, high-signal references. The goal is depth, not exhaustiveness — fewer links, all worth reading.

---

## Foundational reading

- **Anthropic — "Building effective agents"** — the clearest field guide to when to use an agent, when not to, and what patterns work. https://www.anthropic.com/research/building-effective-agents
- **Lilian Weng — "LLM Powered Autonomous Agents"** — the canonical taxonomy of agent components. https://lilianweng.github.io/posts/2023-06-23-agent/
- **Simon Willison's blog** — running commentary on every meaningful LLM development. https://simonwillison.net/

## Evals

- **Hamel Husain — "Your AI product needs evals"** — the practitioner's argument for eval-first development. https://hamel.dev/blog/posts/evals/
- **Eugene Yan — "Building blocks for LLM systems"** — patterns for evals, guardrails, and observability. https://eugeneyan.com/writing/llm-patterns/

## RAG

- **LlamaIndex docs — "Building RAG from Scratch"** — pedagogically excellent. https://docs.llamaindex.ai/
- **LangChain docs — "Q&A with RAG"** — pragmatic and current. https://python.langchain.com/docs/tutorials/rag/
- **Pinecone "Vector Database" guide** — vector search internals explained well. https://www.pinecone.io/learn/

## Multi-agent / orchestration

- **LangGraph docs** — multi-agent patterns, persistence, human-in-loop. https://langchain-ai.github.io/langgraph/
- **OpenAI Agents SDK docs** — handoffs, guardrails, tracing. https://platform.openai.com/docs/guides/agents
- **AutoGen (Microsoft)** — alternative multi-agent framework worth studying for its patterns. https://microsoft.github.io/autogen/

## Tools & MCP

- **MCP specification** — the protocol itself. https://modelcontextprotocol.io/
- **Anthropic — "Introducing the Model Context Protocol"** — the why. https://www.anthropic.com/news/model-context-protocol

## Observability

- **Langfuse docs** — tracing, datasets, evals, prompt management. https://langfuse.com/docs
- **LangSmith docs** — closed-source alternative, deep LangChain integration. https://docs.smith.langchain.com/
- **Helicone** — proxy-based LLM observability. https://www.helicone.ai/

## Provider docs you should bookmark

- **OpenAI API reference** — https://platform.openai.com/docs/api-reference
- **Anthropic API reference** — https://docs.anthropic.com/
- **Google AI Studio / Gemini docs** — https://ai.google.dev/

## Frameworks

- **LangChain** (Py + TS) — https://python.langchain.com/, https://js.langchain.com/
- **LangGraph** (Py + TS) — https://langchain-ai.github.io/langgraph/
- **Vercel AI SDK** (TS) — https://sdk.vercel.ai/
- **LlamaIndex** (Py + TS) — https://docs.llamaindex.ai/, https://ts.llamaindex.ai/

## Books

- *Designing Data-Intensive Applications* — Martin Kleppmann. Not LLM-specific, but the data engineering instincts here are vital for production agents.
- *Prompt Engineering for LLMs* — John Berryman & Albert Ziegler (O'Reilly). Practical, code-forward.
- *Building Generative AI Services with FastAPI* — Ali Parandeh (O'Reilly). End-to-end production patterns.

## Talks worth watching

- **Andrej Karpathy — "Intro to Large Language Models"** — the best 1-hour primer ever filmed. https://www.youtube.com/watch?v=zjkBMFhNj_g
- **Hamel Husain — "AI Eng for Production"** (various Maven cohorts) — the spiritual sibling of this course.
- **Greg Brockman — OpenAI dev day keynotes** — for what's shipping next.

## Communities

- **Latent Space** (Discord + Substack) — the gathering place for AI engineers. https://www.latent.space/
- **r/LocalLLaMA** — for self-hosted, OSS-leaning discussion.
- **MLOps Community Slack** — production-focused, vendor-mixed crowd.
- **Anthropic Discord / OpenAI Forum** — provider-specific Q&A.

## Datasets to graduate to (when you've outgrown the synthetic retail corpus)

- **MSMarco** — passage retrieval benchmark.
- **HotpotQA** — multi-hop QA, great for testing query rewriting.
- **MTEB** — embedding benchmark with retail-like subsets.
- **Amazon Reviews 2023** (Hugging Face) — real product reviews at scale.

## Practice problems

When you finish the capstone, try one of these to stretch:

1. Add **personalization**: persist user prefs (size, brand, gender pref) and inject into every retrieval as a filter or rerank signal.
2. Add **a second domain** (support tickets) to the multi-agent setup. See what the router needs to learn.
3. Replace the synthetic data with a real public catalog (e.g., Open Food Facts, Amazon Reviews). Notice what breaks.
4. Build a **voice version** with the OpenAI Realtime API; reuse all your tools and prompts.
5. Self-host **everything**: Chroma, Langfuse, Ollama (for local LLMs). Measure quality drop and feel the cost cliff in reverse.

---

## Course meta — when to revisit each module

| Symptom | Re-read |
|---------|---------|
| "My prompt feels stuck" | Module 2 (context engineering) |
| "I can't tell if changes are helping" | Module 3 (evals) |
| "RAG retrieves the wrong stuff" | Module 4 (chunking + agentic RAG) |
| "My system prompt is a mess of conditionals" | Module 5 (multi-agent) |
| "Production is too slow / expensive" | Cost & Latency track |
| "I can't debug a user complaint" | Observability track |
| "Need to put this in front of real users" | Deployment Patterns |
