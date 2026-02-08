# 08 — Tech Stack Consolidation Decisions

> **Date:** 2026-02-08  
> **Author:** Rex (subagent review)  
> **Sources:** Architecture docs 01–20  

---

## Summary

The 20 architecture docs contain contradictions — different docs propose different tech for the same role. This review picks **one winner per component** based on what the docs actually say, the factory's scale (single server, <1000 tasks/day), and operational simplicity.

---

## 1. Database: PostgreSQL ✅ (not SQLite)

**Conflict:** Doc 01 (Orchestration) builds entirely on SQLite. Doc 04 (Task Queue) builds entirely on PostgreSQL with `FOR UPDATE SKIP LOCKED` and `LISTEN/NOTIFY`. Doc 20 (Integration Blueprint) says "SQLite + JSON files" for MVP. Doc 08 (Monitoring) assumes Postgres. Doc 18 (Analytics) assumes TimescaleDB (Postgres extension).

**Decision: PostgreSQL**

**Rationale:**
- Doc 04's task queue design *requires* PostgreSQL features: `FOR UPDATE SKIP LOCKED` for atomic claiming, `LISTEN/NOTIFY` for event-driven dispatch. These don't exist in SQLite.
- LiteLLM needs Postgres for its backing store (doc 08, 13).
- Langfuse needs Postgres (doc 08).
- TimescaleDB is a Postgres extension (doc 18).
- Running 3+ systems that need Postgres means Postgres is already in the stack. Adding SQLite alongside it creates two databases to maintain for zero benefit.
- SQLite's single-writer limitation becomes a bottleneck with concurrent agents.

**Migration note:** Doc 01's SQLite schema maps cleanly to Postgres. The task table and DAG resolver work identically — just swap the DDL.

**What SQLite is still fine for:** Agent-local ephemeral state, scratch data, dev/testing. But the factory's shared state is Postgres.

---

## 2. Message Bus / Event System: NATS ✅ (not Redis Streams)

**Conflict:** Doc 11 (Communication Bus) designs the entire inter-agent messaging layer on NATS + JetStream. Doc 04 (Task Queue) says "PostgreSQL NOTIFY + optional Redis Streams." Doc 08/18 use Redis Streams for real-time event bus. Doc 01 says "polling is fine."

**Decision: NATS + JetStream**

**Rationale:**
- Doc 11 is the dedicated communication architecture — it's the most thorough analysis. NATS gives pub/sub, request/reply, queue groups (load balancing), KV store, and JetStream persistence in a single ~20MB binary.
- PostgreSQL `LISTEN/NOTIFY` is fine for task queue wakeups (keep it for that), but it's not a general-purpose event bus — no persistence, no fan-out, no queue groups.
- Redis Streams can do messaging but it's bolting messaging onto a cache. NATS was purpose-built for this.
- NATS KV replaces several Redis use cases (agent registry, config distribution, shared state).
- Single dependency that covers: inter-agent messaging, event bus, lightweight KV, work distribution fan-out.

**Redis role after this decision:** Redis stays for LiteLLM cache and real-time budget accumulators (doc 08) where sub-ms latency matters on every LLM request. It does NOT serve as the event bus.

---

## 3. Vector Database: Qdrant ✅ (not ChromaDB)

**Conflict:** Doc 05 (Memory System) chooses Qdrant explicitly. Doc 20 (Integration Blueprint) says ChromaDB for MVP. Doc 05 even records a decision: "Use Qdrant over ChromaDB for production."

**Decision: Qdrant**

**Rationale:**
- Doc 05 already made this call with clear reasoning: Rust performance, hybrid search (dense + sparse vectors), production-ready clustering, self-hosted.
- ChromaDB is fine for prototyping but has known stability issues at scale and limited filtering.
- Qdrant runs as a single Docker container, ~500MB RAM for small collections. Not heavier to operate than ChromaDB.
- Starting with the production choice avoids a migration that would require re-embedding all data.

**Skip ChromaDB entirely.** There's no "MVP phase" benefit that justifies the migration cost later.

---

## 4. Knowledge Graph: Neo4j — DEFER ⏸️ (start without it)

**Conflict:** Doc 05 designs a full Neo4j-backed knowledge graph (entities, relationships, GraphRAG). No other doc references Neo4j. Doc 05 itself phases it as "Week 3-4" and lists "Neo4j resource consumption" as a risk with "consider Memgraph" as mitigation.

**Decision: Start without Neo4j. Re-evaluate at Week 4.**

**Rationale:**
- Neo4j needs 4GB RAM (doc 05) on a server already running Postgres, Qdrant, NATS, LiteLLM, Langfuse, Ollama, and the agents themselves. That's a lot for dreamteam.
- The core value proposition (entity extraction, relationship traversal, GraphRAG) is a Phase 2/3 feature. No agent workflow in docs 01-04 depends on graph queries.
- Qdrant + Postgres can approximate 80% of the value: Qdrant for semantic search, Postgres for structured entity/relationship tables with basic JOIN traversal.
- If relationship-heavy queries become critical, add Neo4j (or lighter Memgraph) later. The MCP memory server abstracts the storage layer — agents won't know or care.

**Concrete alternative:** Store entities and relationships in Postgres tables (`entities`, `entity_relations`) with foreign keys. Use recursive CTEs for graph traversal up to depth 3-4. This covers "what decisions led to X?" style queries without a dedicated graph DB.

---

## 5. LLM Observability: Langfuse ✅ (not custom)

**Conflict:** Doc 20 (Integration Blueprint) says "Custom (SQLite metrics + REEF dashboard)" for Phase 1, Langfuse for Phase 3. Docs 02, 08, 13 all assume Langfuse from day one.

**Decision: Langfuse from day one**

**Rationale:**
- Three architecture docs (02, 08, 13) integrate Langfuse as a core component. LiteLLM has a native `callbacks: ["langfuse"]` integration — it's a config line, not a build project.
- Building custom cost/token tracking is exactly the kind of undifferentiated work that wastes agent cycles. Langfuse does it out of the box.
- Self-hosted Langfuse is free and keeps data on-prem (doc 08's data sovereignty requirement).
- The "custom SQLite metrics" approach from doc 20 would need to be rebuilt to match what Langfuse provides for free: per-trace cost, session grouping, prompt management, eval framework.
- Langfuse's Postgres requirement is already satisfied (decision #1).

**Deploy cost:** One Docker container + its own Postgres schema (can share the same Postgres instance). ~2GB RAM.

---

## 6. LLM Gateway / Router: LiteLLM ✅ (not custom)

**Conflict:** Doc 20 suggests agents call providers directly in Phase 1. Docs 02, 08, 13 all design around LiteLLM as the single gateway.

**Decision: LiteLLM from day one**

**Rationale:**
- Docs 02 and 08 are unambiguous: "Every agent talks to LiteLLM, never directly to providers." This is the right call.
- LiteLLM provides: multi-provider routing, per-key budgets, fallback chains, cost tracking, prompt caching, OpenAI-compatible API — all as configuration, not code.
- The custom routing logic from doc 02 (request classification, agent profiles, capability matching) layers on top via LiteLLM's callback/middleware system.
- Building a custom router means reimplementing provider-specific token counting, error handling, retry logic, and API normalization for 10+ providers. That's months of work LiteLLM already did.
- Budget enforcement (doc 08) plugs directly into LiteLLM's request pipeline.

**Deploy cost:** One Docker container, ~1GB RAM. Backs onto the same Postgres instance for usage tracking.

---

## Final Stack

| Component | Winner | Runner-up | Why not runner-up |
|-----------|--------|-----------|-------------------|
| **Database** | PostgreSQL | SQLite | Task queue needs SKIP LOCKED + LISTEN/NOTIFY; 3 other services need Postgres anyway |
| **Message Bus** | NATS + JetStream | Redis Streams | Purpose-built messaging > bolted-on; KV store replaces some Redis uses |
| **Vector DB** | Qdrant | ChromaDB | Production-grade from day one; avoids migration cost |
| **Knowledge Graph** | Deferred (Postgres tables) | Neo4j | 4GB RAM for a Phase 2 feature; Postgres recursive CTEs cover 80% |
| **LLM Observability** | Langfuse | Custom SQLite metrics | Native LiteLLM integration; free self-hosted; months of build avoided |
| **LLM Gateway** | LiteLLM | Custom router | 100+ providers, budgets, fallbacks, caching — all config not code |

### Services on dreamteam

| Service | RAM Estimate | Purpose |
|---------|-------------|---------|
| PostgreSQL | 2-4 GB | Task state, Langfuse traces, LiteLLM usage, entities/relations, analytics |
| NATS + JetStream | 256 MB | Inter-agent messaging, event bus, KV store |
| Qdrant | 512 MB–2 GB | Vector search for agent memory/RAG |
| Redis | 512 MB | LiteLLM cache, real-time budget accumulators, kill switches |
| LiteLLM | 1 GB | LLM gateway/router |
| Langfuse | 2 GB | LLM observability/tracing |
| Ollama | Variable (GPU) | Local model inference |
| **Total baseline** | **~8-12 GB** | Leaves headroom for agents on a 32-64GB server |

### Key Principle

**One database (Postgres), one message bus (NATS), one vector store (Qdrant), one LLM gateway (LiteLLM), one observability platform (Langfuse).** Redis stays as a fast cache layer only. No graph DB until proven necessary.

---

## Action Items

1. [ ] Update doc 01 (Orchestration) to use Postgres instead of SQLite
2. [ ] Update doc 20 (Integration Blueprint) to reflect consolidated stack
3. [ ] Reconcile doc 04's Postgres NOTIFY with doc 11's NATS — use both: Postgres NOTIFY for task queue wakeups, NATS for inter-agent events
4. [ ] Remove ChromaDB references from doc 20
5. [ ] Add Postgres entity/relation tables to doc 05 as Neo4j alternative for Phase 1
6. [ ] Create docker-compose.yml for the consolidated stack

---

*Review document — Tech Stack Consolidation. Resolves contradictions across architecture docs 01–20.*
