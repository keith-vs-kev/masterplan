# 05 — Memory & Knowledge Graph System

**Date:** 2026-02-08  
**Author:** Atlas  
**Status:** Draft v1  
**Sources:** research/08-memory-knowledge.md, research/09-mcp-ecosystem.md

---

## TL;DR

Team memory is a hybrid system: **Qdrant** for semantic search, **Neo4j** for relationship graphs, unified behind a **custom MCP memory server**. Agents write freely, background workers consolidate, and a decay process keeps the graph healthy. Start with Tier 2 (vector + files), grow into the full stack.

---

## 1. Design Principles

1. **Write freely, curate later** — agents should never hesitate to store a memory. Consolidation is a background job.
2. **MCP is the interface** — no agent touches a database directly. All memory flows through the MCP memory server.
3. **Namespaced isolation** — each agent has private memory; team knowledge is explicitly promoted.
4. **Temporal by default** — every fact has a timestamp, source, and confidence. Nothing is eternally true.
5. **Human-readable fallback** — the graph is canonical, but markdown files remain the human-accessible layer.

---

## 2. Architecture

```
┌──────────────────────────────────────────────────────┐
│                  MEMORY SYSTEM                        │
│                                                      │
│  ┌────────────────────────────────────────────────┐  │
│  │          Team Memory MCP Server                │  │
│  │  (TypeScript, Streamable HTTP transport)       │  │
│  │                                                │  │
│  │  Tools:                                        │  │
│  │    store / recall / relate / decide / forget   │  │
│  │  Resources:                                    │  │
│  │    memory://{scope}/{entity}                   │  │
│  └──────────────┬─────────────────────────────────┘  │
│                 │                                    │
│       ┌─────────┴─────────┐                          │
│       │                   │                          │
│  ┌────▼─────┐      ┌─────▼──────┐                   │
│  │  Qdrant  │      │   Neo4j    │                   │
│  │ (vector) │      │  (graph)   │                   │
│  │          │      │            │                   │
│  │ semantic │      │ entities   │                   │
│  │ search   │      │ relations  │                   │
│  │ RAG      │      │ decisions  │                   │
│  │ chunks   │      │ provenance │                   │
│  └──────────┘      └────────────┘                   │
│                                                      │
│  ┌────────────────────────────────────────────────┐  │
│  │         Background Workers                     │  │
│  │  • Consolidator  (episodic → semantic)         │  │
│  │  • Extractor     (text → entities/relations)   │  │
│  │  • Decay         (reduce stale relevance)      │  │
│  │  • Conflict Resolver (flag contradictions)     │  │
│  └────────────────────────────────────────────────┘  │
│                                                      │
│  ┌────────────────────────────────────────────────┐  │
│  │         Markdown Layer (human-readable)        │  │
│  │  shared/decisions.md  shared/entities.md       │  │
│  │  agent/{name}/memory/  (daily notes)           │  │
│  └────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────┘
```

---

## 3. Memory Types & Schema

### 3.1 Entity Schema (Neo4j)

```cypher
// Core node types
(:Entity {
  id: UUID,
  name: String,
  type: "person" | "project" | "tool" | "concept" | "decision" | "event",
  properties: Map,
  created_at: DateTime,
  updated_at: DateTime,
  created_by: String,       // agent ID
  confidence: Float,        // 0.0–1.0
  source: String,           // provenance URI
  embedding: [Float]        // 1536d vector for hybrid search
})

// Relationship types
[:RELATES_TO   { since: DateTime, context: String }]
[:DEPENDS_ON   { critical: Boolean }]
[:DECIDED      { rationale: String, participants: [String] }]
[:SUPERSEDES   { reason: String }]
[:AUTHORED_BY  { }]
[:PART_OF      { }]
[:KNOWS_ABOUT  { depth: "shallow" | "deep", since: DateTime }]
```

### 3.2 Memory Record (Qdrant)

```json
{
  "id": "uuid",
  "vector": [1536 floats],
  "payload": {
    "content": "The API rate limit was increased to 1000 req/min",
    "memory_type": "semantic | episodic | procedural | decision",
    "scope": "team | agent:{name} | project:{name}",
    "agent_id": "atlas",
    "source": "conversation:2026-02-08T14:30:00Z",
    "confidence": 0.9,
    "tags": ["api", "infrastructure"],
    "created_at": "2026-02-08T14:30:00Z",
    "accessed_at": "2026-02-08T16:00:00Z",
    "access_count": 3,
    "decay_score": 1.0
  }
}
```

### 3.3 Decision Record

Decisions get first-class treatment — stored in both graph and vector:

```json
{
  "decision": "Use Qdrant over ChromaDB for production",
  "rationale": "Rust performance, hybrid search, production-ready clustering",
  "alternatives_considered": ["ChromaDB", "Pinecone", "Weaviate"],
  "participants": ["atlas", "adam"],
  "context": "architecture/05-memory-system design session",
  "status": "active | superseded | reverted",
  "made_at": "2026-02-08T22:00:00Z",
  "superseded_by": null
}
```

---

## 4. MCP Memory Server — Tool Interface

### Write Tools

| Tool | Purpose | Parameters |
|------|---------|------------|
| `store` | Store a memory | `content`, `memory_type`, `scope`, `tags`, `confidence` |
| `store_decision` | Record a decision | `decision`, `rationale`, `alternatives`, `participants` |
| `relate` | Create entity relationship | `from_entity`, `relation`, `to_entity`, `properties` |
| `store_entity` | Create/update an entity | `name`, `type`, `properties`, `relations[]` |

### Read Tools

| Tool | Purpose | Parameters |
|------|---------|------------|
| `recall` | Semantic search over memories | `query`, `scope`, `memory_type`, `limit`, `min_confidence` |
| `get_entity` | Retrieve entity + relationships | `name`, `depth` (graph traversal hops) |
| `get_decisions` | Retrieve decisions by topic | `topic`, `since`, `status` |
| `trace` | Path between two entities | `from`, `to`, `max_depth` |

### Lifecycle Tools

| Tool | Purpose | Parameters |
|------|---------|------------|
| `forget` | Mark memory for removal | `memory_id`, `reason` |
| `supersede` | Replace a decision/fact | `old_id`, `new_content`, `reason` |
| `promote` | Move from agent-private to team scope | `memory_id` |
| `consolidate` | Trigger manual consolidation | `scope`, `since` |

---

## 5. Namespace & Access Model

```
memory://team/                  ← shared team knowledge (all agents read/write)
memory://team/decisions/        ← decision log
memory://agent/{name}/          ← agent-private memory
memory://project/{name}/        ← project-scoped knowledge
```

**Access rules:**
- Agents can always read `team/` and their own `agent/{name}/`
- Agents can read `project/` namespaces they're assigned to
- Writing to `team/` from agent-private requires explicit `promote`
- Human can read everything

**Retrieval hierarchy** (L1/L2/L3 cache pattern):
1. **Working memory** — current session context (in-prompt)
2. **Agent-private** — personal observations, preferences
3. **Project scope** — project-specific knowledge
4. **Team scope** — shared facts, decisions, conventions

---

## 6. Memory Lifecycle

### 6.1 Write Path

```
Agent calls store() via MCP
    → Embed content (text-embedding-3-small, 1536d)
    → Write vector + payload to Qdrant (namespaced collection)
    → If entities detected: extract and write to Neo4j
    → If decision: write decision node + relations to Neo4j
    → Ack to agent
```

### 6.2 Consolidation (Background Worker)

Runs periodically (every 6 hours or on-demand):

1. Scan recent episodic memories in a scope
2. Cluster semantically similar memories
3. LLM summarises each cluster → new semantic memory
4. Link semantic memory to source episodic memories in graph
5. Reduce decay_score on consumed episodic memories

### 6.3 Decay & Forgetting

Every memory has a `decay_score` (starts at 1.0):

```
decay_score = base_score × recency_factor × access_factor × confidence

recency_factor = exp(-λ × days_since_created)    # λ = 0.01 (slow decay)
access_factor  = 1 + log(1 + access_count)       # frequent access preserves
```

- Memories with `decay_score < 0.1` are candidates for archival
- Archived memories move to cold storage (still queryable, lower priority in search)
- Memories explicitly marked `forget` are soft-deleted (kept 30 days for undo, then purged)

### 6.4 Conflict Resolution

When contradictory facts are detected (via embedding similarity + opposite sentiment, or explicit `supersede`):

1. **Timestamp wins** — most recent fact takes precedence by default
2. **Source authority hierarchy**: `human > verified_agent > unverified_agent > inferred`
3. **Confidence comparison** — higher confidence wins if timestamps are close
4. **Flag for review** — if no clear winner, create a `CONFLICT` relationship in graph and notify team

```cypher
(fact_a)-[:CONFLICTS_WITH { detected_at, resolution: "pending" }]->(fact_b)
```

---

## 7. Entity Extraction Pipeline

Conversations and documents flow through an extraction pipeline:

```
Raw text
    → LLM extraction prompt (structured output)
    → Entities: [{name, type, properties}]
    → Relations: [{from, relation, to, context}]
    → Deduplicate against existing graph (fuzzy name match + embedding similarity)
    → Merge or create nodes/edges in Neo4j
```

**Extraction prompt** (run via Cerebras for cost-efficiency on high volume):
```
Extract entities and relationships from this text.
Return JSON: { entities: [{name, type, properties}], relations: [{from, rel, to}] }
Entity types: person, project, tool, concept, decision, event
Relation types: RELATES_TO, DEPENDS_ON, DECIDED, AUTHORED_BY, PART_OF, KNOWS_ABOUT
```

---

## 8. Hybrid Query Pattern (GraphRAG)

For complex queries, combine both stores:

```
Query: "What decisions led to our current deployment architecture?"

1. Vector search (Qdrant): find top-10 relevant memory chunks
2. Entity extraction: identify entities mentioned in query → "deployment", "architecture"  
3. Graph traversal (Neo4j): from entity nodes, traverse DECIDED, DEPENDS_ON edges (depth 3)
4. Merge context: vector results + graph traversal results
5. LLM synthesis: generate answer grounded in merged context
```

Neo4j's native vector index enables single-DB hybrid queries for simpler deployments:

```cypher
// Hybrid: vector similarity + graph traversal in one query
CALL db.index.vector.queryNodes('memory_embeddings', 10, $queryVector)
YIELD node, score
WHERE score > 0.7
MATCH path = (node)-[:DECIDED|DEPENDS_ON*1..3]->(related)
RETURN node, related, score, path
```

---

## 9. Implementation Phases

### Phase 1: Vector + Files (Week 1-2)
- Deploy Qdrant via Docker on dreamteam
- Build MCP memory server (TypeScript) with `store`, `recall`, `forget`
- Embed with `text-embedding-3-small` (switch to local model later)
- Namespace per agent + team collection
- Keep markdown files as human-readable mirror
- **Outcome:** Agents can semantically search shared memory

### Phase 2: Knowledge Graph (Week 3-4)
- Deploy Neo4j via Docker
- Add entity extraction pipeline (LLM-based)
- Add `relate`, `get_entity`, `trace`, `store_decision` tools
- Build decision log as first-class graph structure
- **Outcome:** Relationship-aware memory, decision tracking

### Phase 3: Lifecycle Workers (Week 5-6)
- Consolidation worker (episodic → semantic)
- Decay scoring and archival process
- Conflict detection and resolution pipeline
- Memory quality dashboard (access patterns, staleness, conflicts)
- **Outcome:** Self-maintaining memory that stays healthy

### Phase 4: Production Hardening (Week 7-8)
- Neo4j vector index for single-DB hybrid queries (evaluate vs separate Qdrant)
- Backup and recovery procedures
- Memory analytics (what do agents recall most? what's never accessed?)
- Human review interface for flagged conflicts
- Embedding model migration path (OpenAI → local nomic-embed-text)
- **Outcome:** Production-grade team memory

---

## 10. Infrastructure Requirements

| Component | Resource | Deployment |
|-----------|----------|------------|
| Qdrant | 2GB RAM, 10GB disk | Docker on dreamteam |
| Neo4j | 4GB RAM, 20GB disk | Docker on dreamteam |
| MCP Memory Server | 512MB RAM | Node.js process, Streamable HTTP |
| Embedding calls | OpenAI API (phase 1) → Ollama nomic-embed-text (phase 4) | API / local |
| Background workers | Minimal CPU | Cron-triggered or event-driven |

---

## 11. Key Design Decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Vector DB | Qdrant | Rust performance, hybrid search, self-hosted, production-ready |
| Graph DB | Neo4j | Dominant ecosystem, native vector index, Cypher maturity |
| Interface | MCP server | Standard protocol, any agent can connect, decouples storage from agents |
| Transport | Streamable HTTP | Multi-agent access, remote-capable, production-grade |
| Embedding model | text-embedding-3-small → nomic-embed-text | Quality first, migrate to local later |
| Conflict strategy | Timestamp + authority + confidence | Simple, deterministic, escalates ambiguity to humans |
| Decay model | Exponential with access boost | Mirrors human memory — use it or lose it |

---

## 12. Risks & Mitigations

| Risk | Mitigation |
|------|------------|
| Memory pollution (bad data) | Confidence scores, source tracking, conflict detection |
| Embedding model change breaks search | Version embeddings in metadata, re-index pipeline ready |
| Context window overflow from too much recall | Token budget per recall (default 2000 tokens), ranked truncation |
| Privacy leak across agent namespaces | Strict scope filtering at MCP server layer, not DB layer |
| Operational complexity | Start Phase 1 (vector only), add graph only when relationship queries are needed |
| Neo4j resource consumption | Monitor, consider Memgraph if Neo4j is too heavy for dreamteam |

---

## 13. Open Questions

- [ ] Should the markdown file layer be auto-generated from graph, or maintained separately?
- [ ] How to handle memory during agent forks/clones (copy-on-write semantics?)
- [ ] What's the right consolidation frequency? Too often = noisy, too rare = bloated episodic store
- [ ] Should agents have read access to other agents' private memory (opt-in)?
- [ ] Integration with task system — should task completion auto-generate memory entries?

---

*Architecture document for the Agentic Factory masterplan.*
