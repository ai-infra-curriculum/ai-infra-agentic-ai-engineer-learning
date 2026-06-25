# mod-203-rag-and-memory: RAG & Memory Implementation

**Estimated effort:** 12 hours

An agent that can't retrieve and can't remember is a stateless function with a system prompt. This module gives it both. You'll build retrieval-augmented generation from the embedding up — chunk, embed, store in a vector database, retrieve, and ground a generation in what you found — then layer on the memory systems that turn a single-shot RAG call into an agent that accumulates state: working memory inside a task, episodic memory across turns, and long-term memory across sessions. From there you'll handle the two problems every production memory system hits — *when* to retrieve (just-in-time, not eagerly) and *what to believe* when retrieved facts contradict each other. You finish by building an advanced retrieval pipeline (sentence-window and auto-merging) and measuring it with the RAG triad so "it feels better" becomes a number.

> **Retrieval is grounding; memory is state.** They share machinery — embeddings and a vector store — but answer different questions. RAG asks *"what does my corpus say about this?"* Memory asks *"what do I already know about this user, this task, this session?"* Keep the two mental models distinct even when they share a database.

## Learning objectives

- Implement **retrieval-augmented generation** end to end: chunking, embeddings, a vector database, top-k retrieval, and a grounded generation step.
- Build the three **memory tiers** an agent needs — working (in-task), episodic (across turns), and long-term (across sessions) — and know what belongs in each.
- Implement **just-in-time retrieval** and **conflict resolution**: retrieve only when the task needs it, and resolve contradictions between stored facts by recency, source, and confidence.
- Build an **advanced RAG pipeline** — sentence-window and auto-merging retrieval — and **evaluate** it with the RAG triad (context relevance, groundedness, answer relevance).

## Lecture chapters

1. [Retrieval-Augmented Generation from the Embedding Up](01-rag-fundamentals.md) — chunk → embed → store → retrieve → ground, and the failure modes at each stage.
2. [Memory Systems for Agents](02-agent-memory.md) — working, episodic, and long-term memory; what each holds and how they hand off.
3. [Just-in-Time Retrieval and Conflict Resolution](03-jit-retrieval-conflict.md) — retrieving on demand, and resolving contradictions by recency, source authority, and confidence.
4. [Advanced Retrieval and Evaluation](04-advanced-rag-eval.md) — sentence-window and auto-merging retrieval, re-ranking, and the RAG triad as your scoreboard.

## Exercises

Hands-on practice. Reference solutions live in the paired [solutions repo](https://github.com/ai-engineering-curriculum/agentic-ai-engineer-solutions).

- [exercise-01: Build a RAG pipeline on a vector DB](exercises/exercise-01-rag-pipeline-vector-db.md) — chunk, embed, store, retrieve, and ground a generation.
- [exercise-02: Agent long-term memory](exercises/exercise-02-agent-long-term-memory.md) — working, episodic, and long-term tiers with write and recall.
- [exercise-03: Advanced RAG retrieval](exercises/exercise-03-advanced-rag-retrieval.md) — sentence-window and auto-merging retrieval with re-ranking.
- [exercise-04: RAG triad evaluation](exercises/exercise-04-rag-triad-evaluation.md) — score context relevance, groundedness, and answer relevance.

## Prerequisites

- [mod-201: Agent Fundamentals](../mod-201-agent-fundamentals/README.md) — the reason-act loop and tool calling; retrieval is "just a tool" the agent calls.
- A vector database you can run locally (pgvector, Chroma, or Qdrant) and an embedding model (API or local).
- Comfort with Python and `async` for the retrieval-as-a-tool examples.

See [resources.md](resources.md) for primary references.
