# Memory & Knowledge Management for Agent Teams

> Research document for the masterplan. How agent teams share knowledge, build persistent memory, and maintain context across sessions.

## Executive Summary

Agent teams need shared, persistent memory to function effectively. The core challenge: LLM agents are stateless by default — each session starts fresh. Building team memory requires combining **vector databases** (semantic search over unstructured knowledge), **knowledge graphs** (structured relationships and reasoning), and **shared context protocols** (standardized ways to read/write memory). The emerging stack is: vector DB for retrieval, knowledge graph for relationships, RAG for grounding responses, and MCP for interoperability.

---

## 1. The Problem Space

### Why Memory Matters for Agent Teams

- **Session amnesia**: Agents forget everything between sessions unless memory is explicitly persisted
- **Knowledge silos**: Individual agents learn things that other team members can't access
- **Context fragmentation**: Conversations, decisions, and learnings scattered across sessions
- **Redundant work**: Without shared memory, agents re-research and re-discover the same things
- **No institutional knowledge**: Teams can't build cumulative expertise over time

### What "Team Memory" Actually Means

| Memory Type | Description | Example |
|---|---|---|
| **Episodic** | What happened (events, conversations) | "User asked about X on Tuesday" |
| **Semantic** | Facts and knowledge | "The API key rotates every 90 days" |
| **Procedural** | How to do things | "Deploy by running make deploy" |
| **Working** | Current task context | "We're debugging issue #423" |
| **Shared** | Team-wide knowledge | "Team convention: use snake_case" |

---

## 2. Vector Databases

Vector databases store embeddings — numerical representations of text/data that capture semantic meaning. They enable "find me things similar to X" queries, which is the foundation of RAG.

### Comparison Matrix

| Feature | Pinecone | Weaviate | Qdrant | ChromaDB |
|---|---|---|---|---|
| **Type** | Managed cloud | Open-source + cloud | Open-source + cloud | Open-source, embedded |
| **Language** | — (SaaS) | Go | Rust | Python |
| **Self-host** | No | Yes | Yes | Yes |
| **Hybrid search** | Yes (sparse+dense) | Yes (BM25 + vector) | Yes (sparse+dense) | Limited |
| **Filtering** | Metadata filters | GraphQL-like filters | Payload filters | Metadata filters |
| **Scale** | Billions of vectors | Millions–billions | Millions–billions | Thousands–millions |
| **Pricing** | Pay per usage | Free (OSS) / Cloud plans | Free (OSS) / Cloud plans | Free (OSS) |
| **Best for** | Production SaaS, zero-ops | Full-featured, RAG pipelines | Performance-critical, self-hosted | Prototyping, local dev, embedded |
| **Unique strength** | Simplicity, scale | Native RAG/generative search, agent workflows | Speed (Rust), fine-grained control | Zero-config, pip install |

### Pinecone
- Fully managed, serverless option available
- Namespaces for multi-tenant isolation (good for per-agent or per-team memory)
- Sparse-dense hybrid vectors for combining keyword + semantic search
- Integrations with LangChain, LlamaIndex, etc.
- **Tradeoff**: No self-hosting, vendor lock-in, cost at scale

### Weaviate
- Open-source with optional cloud (WCD)
- Built-in vectorization modules (OpenAI, Cohere, HuggingFace, etc.)
- Native **generative search** — retrieve + generate in one query (built-in RAG)
- **Agent-driven workflows** explicitly supported in docs
- Multi-tenancy for team isolation
- GraphQL + REST + gRPC APIs
- **Tradeoff**: More complex to operate than ChromaDB, Java-level resource usage

### Qdrant
- Written in Rust — extremely fast, low memory footprint
- Rich filtering on payload fields (combine vector search with structured queries)
- Supports sparse vectors for hybrid search
- Snapshot & replication for production deployments
- gRPC + REST APIs
- **Tradeoff**: Fewer built-in integrations than Weaviate, less "batteries included"

### ChromaDB
- Python-native, `pip install chromadb` and go
- Embedded mode (runs in-process) or client-server
- Perfect for local agent memory, prototyping, single-machine setups
- Simple API: `collection.add()`, `collection.query()`
- **Tradeoff**: Not designed for massive scale or distributed deployments; limited hybrid search

### Recommendation for Agent Teams

- **Prototyping / single agent**: ChromaDB (zero setup, embedded)
- **Production team memory**: Qdrant or Weaviate self-hosted
- **Zero-ops cloud**: Pinecone (if budget allows and vendor lock-in is acceptable)
- **RAG-first architecture**: Weaviate (native generative search)

---

## 3. Knowledge Graphs

Knowledge graphs store structured relationships: entities (nodes) connected by typed edges. They answer "how is X related to Y?" rather than "what's similar to X?"

### Neo4j
- The dominant graph database, mature ecosystem
- **Cypher** query language — declarative, SQL-like for graphs
- Property graph model: nodes and relationships both have typed properties
- **GraphRAG**: Combines knowledge graph traversal with LLM generation
- Vector search index available (added 2023) — can do hybrid graph+vector
- AuraDB cloud offering
- APOC library for extended analytics
- **Best for**: Complex relationship queries, enterprise knowledge management, GraphRAG

### Memgraph
- In-memory graph database, Cypher-compatible
- Written in C++ — extremely fast for real-time queries
- **MAGE** library: 20+ graph algorithms built-in (PageRank, community detection, etc.)
- Streaming ingestion from Kafka, Pulsar
- Docker-first deployment, Memgraph Lab for visualization
- **Best for**: Real-time graph analytics, streaming data, performance-critical workloads
- **Tradeoff**: Smaller ecosystem than Neo4j, less mature tooling

### When to Use Knowledge Graphs vs Vector DBs

| Use Case | Vector DB | Knowledge Graph |
|---|---|---|
| "Find relevant context for this query" | ✅ | ❌ |
| "How are these entities related?" | ❌ | ✅ |
| "What decisions led to this outcome?" | ❌ | ✅ |
| "Find similar past experiences" | ✅ | ❌ |
| "Who knows about topic X?" | ⚠️ (approximate) | ✅ (explicit) |
| "What's the full history of project Y?" | ⚠️ (chunks) | ✅ (traversal) |

---

## 4. Hybrid Approaches

The most powerful agent memory systems combine both paradigms.

### Vector + Graph (GraphRAG)

Microsoft's GraphRAG pattern (2024) demonstrated that combining knowledge graph extraction with vector retrieval dramatically improves answer quality for complex questions.

**Architecture:**
```
Documents → LLM extracts entities/relationships → Knowledge Graph
Documents → Embedding model → Vector DB
Query → Vector search (local context) + Graph traversal (global context) → LLM generates answer
```

**Why it works:**
- Vector search finds relevant chunks (local retrieval)
- Graph traversal finds connected entities and relationships (global context)
- Together, the LLM gets both relevant snippets AND structural understanding

### Practical Hybrid Stack

```
┌─────────────────────────────────────────┐
│            Agent Team Memory            │
├─────────────────────────────────────────┤
│                                         │
│  ┌─────────────┐  ┌─────────────────┐  │
│  │  Vector DB   │  │ Knowledge Graph  │  │
│  │  (Qdrant)    │  │ (Neo4j)          │  │
│  │              │  │                  │  │
│  │ - Embeddings │  │ - Entities       │  │
│  │ - Semantic   │  │ - Relations      │  │
│  │   search     │  │ - Provenance     │  │
│  │ - RAG chunks │  │ - Decision trees │  │
│  └──────┬───────┘  └────────┬────────┘  │
│         │                   │           │
│         └─────────┬─────────┘           │
│                   │                     │
│           ┌───────▼────────┐            │
│           │  Memory Layer  │            │
│           │  (MCP Server)  │            │
│           └───────┬────────┘            │
│                   │                     │
│     ┌─────────────┼─────────────┐       │
│     │             │             │       │
│  ┌──▼──┐     ┌───▼───┐    ┌───▼───┐   │
│  │Agent│     │Agent  │    │Agent  │   │
│  │  A  │     │  B    │    │  C    │   │
│  └─────┘     └───────┘    └───────┘   │
│                                         │
└─────────────────────────────────────────┘
```

### Neo4j + Vector Index

Neo4j now supports native vector indexes, allowing a single database to serve both graph queries and vector similarity search. This simplifies the stack significantly:

```cypher
-- Create vector index on nodes
CREATE VECTOR INDEX memory_embeddings
FOR (m:Memory) ON (m.embedding)
OPTIONS {indexConfig: {`vector.dimensions`: 1536, `vector.similarity_function`: 'cosine'}}

-- Hybrid query: vector similarity + graph traversal
CALL db.index.vector.queryNodes('memory_embeddings', 10, $queryEmbedding)
YIELD node, score
MATCH (node)-[:RELATED_TO]->(related)
RETURN node, related, score
```

---

## 5. RAG Patterns for Agent Teams

### Basic RAG
```
Query → Embed → Vector Search → Top-K chunks → LLM + chunks → Answer
```
Simple, effective for single-agent Q&A over documents.

### Agentic RAG
Agent decides *when* and *what* to retrieve, can do multi-step retrieval:
```
Query → Agent plans retrieval strategy → Multiple targeted searches → Agent synthesizes → Answer
```

### Corrective RAG (CRAG)
Adds a verification step — if retrieved documents seem irrelevant, the agent reformulates the query or falls back to web search.

### Multi-Agent RAG Patterns

**Shared retrieval pool**: All agents query the same vector DB. Simple but no isolation.

**Namespaced memory**: Each agent has its own namespace/collection, plus a shared "team" namespace:
```
team-knowledge/       ← shared facts, decisions, conventions
agent-scout/          ← Scout's personal observations and context
agent-kev/            ← Kev's preferences and history
project-masterplan/   ← project-specific knowledge
```

**Hierarchical retrieval**: Agent checks local memory first, then team memory, then global knowledge base. Like L1/L2/L3 cache.

**Memory consolidation**: Background process periodically:
1. Scans agent episodic memories
2. Extracts key facts and decisions
3. Deduplicates and merges
4. Writes consolidated knowledge to shared store
5. Prunes stale/outdated entries

---

## 6. Shared Context Protocols

### Model Context Protocol (MCP)

MCP (by Anthropic) is the emerging standard for giving LLMs access to external data and tools. It's JSON-RPC based with three core primitives:

- **Resources**: Data the server exposes (files, DB records, etc.)
- **Tools**: Actions the model can invoke (search, write, etc.)
- **Prompts**: Reusable prompt templates

**MCP Memory Server** (official reference implementation):
- Knowledge graph-based persistent memory
- Stores entities and relations as a local JSON graph
- Tools: `create_entities`, `create_relations`, `search_nodes`, `open_nodes`, `delete_entities`, `delete_relations`
- Simple but effective for single-agent memory
- **Limitation**: File-based, no vector search, doesn't scale to teams natively

**Why MCP matters for team memory:**
- Standard interface means any agent (Claude, GPT, local models) can read/write the same memory
- Memory server can be shared across agents via network MCP
- Composable: combine memory MCP server with other servers (filesystem, git, etc.)
- Subscribe to resource changes — agents can be notified when shared memory updates

### Building a Team Memory MCP Server

Extending the reference memory server for teams:

```typescript
// Conceptual team memory MCP server
tools: {
  // Write
  "store_memory": { agent_id, content, tags, memory_type },
  "store_decision": { decision, rationale, participants, context },
  "store_entity": { name, type, properties, relations },
  
  // Read
  "recall": { query, scope: "personal" | "team" | "all", limit },
  "get_entity": { name, depth },  // graph traversal depth
  "get_decisions": { topic, since },
  
  // Maintain
  "consolidate": { agent_id },  // merge episodic → semantic
  "forget": { memory_id, reason },
}
```

### A2A (Agent-to-Agent Protocol)

Google's A2A protocol focuses on agent communication but includes context sharing:
- **Task artifacts**: Agents can pass structured data between each other
- **Context**: Task-level context shared during collaboration
- Less focused on persistent memory, more on in-flight coordination

### Comparison: MCP vs A2A for Memory

| Aspect | MCP | A2A |
|---|---|---|
| Focus | Model ↔ Data/Tools | Agent ↔ Agent |
| Memory pattern | Agent reads/writes shared store | Agents pass context in-flight |
| Persistence | Server manages persistence | Ephemeral (task-scoped) |
| Best for | Shared knowledge base | Real-time collaboration |
| Complementary? | Yes — use together | Yes — use together |

---

## 7. Building Persistent Team Memory

### Architecture Recommendations

#### Tier 1: Simple (File-Based)
- Markdown files in shared git repo (what OpenClaw does now)
- `MEMORY.md` for curated long-term, `memory/YYYY-MM-DD.md` for daily
- Pros: Dead simple, version controlled, human readable
- Cons: No semantic search, linear scan, doesn't scale

#### Tier 2: Intermediate (Vector DB + Files)
- ChromaDB or Qdrant for semantic search over memories
- Files still used for structured/curated knowledge
- MCP server wrapping the vector DB for standard access
- Pros: Semantic recall, still simple to operate
- Cons: No relationship reasoning

#### Tier 3: Advanced (Hybrid Graph + Vector)
- Neo4j with vector indexes OR Qdrant + Neo4j
- Knowledge graph for entities, decisions, relationships
- Vector search for fuzzy/semantic recall
- MCP server as the unified interface
- Background consolidation process
- Pros: Full-featured, relationship-aware, semantic search
- Cons: Operational complexity, needs embedding pipeline

#### Tier 4: Production (Full Stack)
```
Embedding Service (OpenAI / local model)
    ↓
Qdrant (vector storage + search)
Neo4j (knowledge graph + vector index)
    ↓
Team Memory MCP Server
    ↓
MCP clients (agents)
    ↓
Background Workers:
  - Consolidation (episodic → semantic)
  - Entity extraction (conversations → graph)
  - Decay/pruning (remove stale memories)
  - Conflict resolution (contradictory memories)
```

### Key Design Decisions

**1. Embedding model choice**
- OpenAI `text-embedding-3-small` (1536d): Good quality, cheap, requires API
- `nomic-embed-text` (768d): Open-source, runs locally via Ollama
- Cohere `embed-v3`: Good multilingual support
- **Recommendation**: Start with OpenAI for quality, plan migration to local model

**2. Chunking strategy**
- Conversations: Per-message or per-turn (not whole conversations)
- Documents: ~500 token chunks with ~50 token overlap
- Decisions: Atomic — one decision per chunk with full rationale
- **Key**: Include metadata (timestamp, agent, source, tags) with every chunk

**3. Memory lifecycle**
```
Event occurs → Agent writes episodic memory
    ↓
Background job → Extracts entities/facts → Writes to knowledge graph
    ↓
Periodic consolidation → Merges related memories → Updates semantic store
    ↓
Decay process → Reduces relevance of old, unaccessed memories
    ↓
Pruning → Archives or deletes irrelevant/contradictory entries
```

**4. Conflict resolution**
When agents store contradictory information:
- Timestamp wins (most recent)
- Source authority (human > agent, verified > unverified)
- Confidence scores on memories
- Flag conflicts for human review

**5. Access patterns**
- **Write-heavy**: Log everything, consolidate later (prefer this)
- **Read-heavy**: Pre-compute summaries, maintain indexes
- **Mixed**: Separate hot (working memory) from cold (archive) storage

---

## 8. Practical Implementation Path

### Phase 1: Enhanced File Memory (Now)
- Keep current markdown-based system
- Add structured metadata to memory files (YAML frontmatter)
- Create `shared/` directory for team-wide knowledge
- Add simple keyword search over memory files

### Phase 2: Add Semantic Search (Near-term)
- Deploy ChromaDB (embedded) or Qdrant (Docker)
- Index all memory files on write
- Build MCP memory server with `recall` and `store` tools
- Agents use semantic search for context retrieval

### Phase 3: Knowledge Graph (Medium-term)
- Add Neo4j for entity/relationship storage
- Extract entities from conversations automatically
- Build decision log as graph (decision → rationale → participants → context)
- Hybrid queries: semantic search + graph traversal

### Phase 4: Team Memory Platform (Long-term)
- Multi-agent MCP memory server
- Per-agent namespaces + shared team space
- Background consolidation and pruning workers
- Memory quality metrics and monitoring
- Human-in-the-loop review for important memories

---

## 9. Key Risks & Mitigations

| Risk | Impact | Mitigation |
|---|---|---|
| Memory pollution (wrong/outdated info) | Agents act on bad data | Confidence scores, source tracking, periodic review |
| Embedding drift (model changes) | Old embeddings incompatible | Version embeddings, re-index on model change |
| Context window overflow | Too much retrieved context | Smart ranking, summarization, max-token budgets |
| Privacy leakage | Private info in shared memory | Namespace isolation, access controls, redaction |
| Operational complexity | Hard to maintain | Start simple (Tier 1-2), add complexity only when needed |
| Vendor lock-in | Stuck on one provider | Use MCP as abstraction layer, prefer open-source |

---

## 10. Tools & Libraries Worth Knowing

| Tool | What it does | URL |
|---|---|---|
| **LangChain** | Framework with memory modules, RAG chains | langchain.com |
| **LlamaIndex** | Data framework for RAG, knowledge graphs | llamaindex.ai |
| **Mem0** | "Memory layer for AI" — managed memory service | mem0.ai |
| **Zep** | Long-term memory for AI assistants | getzep.com |
| **LangGraph** | Stateful agent workflows with persistence | langchain.com/langgraph |
| **MCP Memory Server** | Reference knowledge graph memory | github.com/modelcontextprotocol/servers |
| **GraphRAG** | Microsoft's graph+RAG approach | github.com/microsoft/graphrag |
| **txtai** | All-in-one embeddings DB + RAG | github.com/neuml/txtai |

---

## 11. Summary & Recommendations

### For the Masterplan

1. **Start with what works**: The current file-based memory system (MEMORY.md + daily files) is Tier 1 and already functional. Don't over-engineer.

2. **Next step**: Add a vector DB (ChromaDB for simplicity, Qdrant for production) behind an MCP server. This gives semantic recall with minimal infrastructure.

3. **Knowledge graph when needed**: Add Neo4j when agents need to reason about relationships (entity graphs, decision chains, dependency tracking). Not before.

4. **MCP is the integration layer**: Build memory as an MCP server. This decouples agents from storage backends and allows any MCP-compatible agent to use the same memory.

5. **Hybrid is the endgame**: Vector search for "what's relevant?" + knowledge graph for "how is it connected?" + file storage for human-readable artifacts. All behind a unified MCP interface.

6. **Memory lifecycle matters more than storage choice**: The hard problems are what to remember, when to forget, how to consolidate, and how to resolve conflicts — not which database to use.

---

*Research completed 2026-02-08. Sources: Pinecone docs, Weaviate docs, Qdrant docs, ChromaDB docs, Neo4j docs, Memgraph docs, MCP specification, Microsoft GraphRAG paper, LangChain/LlamaIndex documentation.*
