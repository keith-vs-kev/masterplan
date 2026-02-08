# MongoDB for AI Agent Orchestration

> Research by Scout | 2026-02-08
> Context: Agent factory on local RTX 3090 + cloud APIs

## TL;DR

MongoDB can consolidate 3-4 separate tools (task queue, vector DB, metrics store, document store) into one system. It won't replace a dedicated message bus for sub-millisecond pub/sub, but for a small-to-medium agent orchestration system it's a surprisingly capable "good enough at everything" choice. The free tier (M0) is viable for development; self-hosted is the move for production on local hardware.

---

## 1. Change Streams — Can They Replace a Message Bus?

### How They Work
Change streams use MongoDB's oplog (replication log) to push real-time notifications when documents are inserted, updated, deleted, or replaced. You open a cursor that tails the oplog with server-side filtering via aggregation pipeline stages.

```javascript
const pipeline = [
  { $match: { 'fullDocument.agentId': 'scout', operationType: 'insert' } }
];

const changeStream = db.collection('tasks').watch(pipeline, {
  fullDocument: 'updateLookup' // include full doc on updates
});

changeStream.on('change', (event) => {
  console.log('New task:', event.fullDocument);
});
```

### Key Properties
- **Latency**: Typically 10-50ms for local replica set, 50-200ms on Atlas (network dependent)
- **Resumable**: Token-based resume after disconnection — no lost events
- **Filtering**: Server-side via aggregation pipeline (efficient, doesn't stream everything)
- **Scalability**: Works on sharded clusters, one stream per watcher
- **Ordering**: Guaranteed total order on a single collection

### Can It Replace NATS/Redis Pub-Sub?

**For agent coordination: Mostly yes, with caveats.**

| Feature | Change Streams | NATS/Redis |
|---------|---------------|------------|
| Latency | 10-50ms local | <1ms |
| Fan-out (many subscribers) | Each opens own cursor (resource cost) | Native pub/sub, cheap fan-out |
| Durability | Built-in (oplog) | NATS JetStream / Redis Streams |
| Filtering | Rich (aggregation pipeline) | Subject-based (simpler) |
| Back-pressure | Natural (cursor-based) | Varies |
| Ordering | Total order guaranteed | Per-subject |

**Verdict**: If your agents don't need sub-millisecond reactions (most don't — LLM calls take seconds), change streams work fine. You avoid adding NATS/Redis as another dependency. For high-frequency fan-out to 50+ subscribers, a dedicated bus is better.

**Recommended pattern for agents:**
- Use change streams for task assignment notifications
- Use change streams for agent status updates
- Keep a dedicated bus (NATS) only if you need request-reply patterns or >100 msg/sec fan-out

---

## 2. Atlas Vector Search — Semantic Memory for Agents

### Capabilities
- **Dimensions**: Up to **4096 dimensions** (covers OpenAI ada-002 at 1536, text-embedding-3-large at 3072, and most open-source models)
- **Algorithms**: Approximate Nearest Neighbor (HNSW) and Exact Nearest Neighbor (ENN)
- **Similarity**: cosine, euclidean, dotProduct
- **Pre-filtering**: Filter on any field BEFORE vector search (critical for multi-agent systems — filter by agent, namespace, time window)
- **Hybrid search**: Combine vector search + full-text search + standard filters in one query

```javascript
// Vector search with pre-filtering
db.memories.aggregate([
  {
    $vectorSearch: {
      index: "memory_index",
      path: "embedding",
      queryVector: queryEmbedding,  // float[]
      numCandidates: 100,
      limit: 10,
      filter: {
        agentId: "scout",
        timestamp: { $gte: ISODate("2026-02-01") }
      }
    }
  },
  {
    $project: {
      content: 1,
      score: { $meta: "vectorSearchScore" },
      agentId: 1
    }
  }
]);
```

### Index Definition
```json
{
  "fields": [
    {
      "type": "vector",
      "path": "embedding",
      "numDimensions": 1536,
      "similarity": "cosine"
    },
    {
      "type": "filter",
      "path": "agentId"
    },
    {
      "type": "filter",
      "path": "timestamp"
    }
  ]
}
```

### Comparison to Dedicated Vector DBs

| Feature | MongoDB Atlas VS | Qdrant | ChromaDB |
|---------|-----------------|--------|----------|
| Max dimensions | 4096 | Unlimited | Unlimited |
| Pre-filtering | Yes (efficient) | Yes (payload filters) | Yes (metadata) |
| Hybrid search | Vector + text + filters | Vector + filters | Vector + filters |
| Quantization | Binary, scalar | Product, scalar, binary | None |
| Self-hosted vector | Yes (since ~7.0 w/ Atlas CLI local) | Yes (Docker) | Yes (Docker) |
| Colocation with data | **Same collection** | Separate system | Separate system |
| Production maturity | Good (GA since 2023) | Good | Lightweight/dev |

### Verdict for Agent Memory

**Strong choice.** The killer feature is colocation — your agent's memories, metadata, and vectors live in the same document. No sync between a vector DB and a document store. Pre-filtering by agent/namespace is first-class.

**Limitation**: For a local RTX 3090 setup, you need either Atlas (cloud) or the local Atlas CLI deployment for vector search. Standard self-hosted `mongod` alone does NOT include vector search indexes — you need the Atlas infrastructure. This is a key consideration.

**Workaround for fully local**: Run `atlas deployments setup --type local` via Atlas CLI for a local containerized Atlas deployment with vector search support.

---

## 3. Task Queues in MongoDB

This is a well-proven pattern. MongoDB's atomic operations make it a solid task queue.

### Core Pattern: Atomic Claim with findOneAndUpdate

```javascript
// Worker claims a task atomically
const task = await db.collection('tasks').findOneAndUpdate(
  {
    status: 'pending',
    scheduledAt: { $lte: new Date() },
    // Optional: target specific agent types
    requiredCapability: { $in: workerCapabilities }
  },
  {
    $set: {
      status: 'processing',
      claimedBy: workerId,
      claimedAt: new Date()
    }
  },
  {
    sort: { priority: -1, createdAt: 1 },  // highest priority, oldest first
    returnDocument: 'after'
  }
);
```

### TTL Index for Stuck Task Recovery

```javascript
// Create TTL index — auto-expire stale claims
db.tasks.createIndex(
  { "claimedAt": 1 },
  { expireAfterSeconds: 300, partialFilterExpression: { status: "processing" } }
);

// Or better: periodic cleanup job
async function reclaimStaleTasks() {
  const staleThreshold = new Date(Date.now() - 5 * 60 * 1000);
  await db.collection('tasks').updateMany(
    { status: 'processing', claimedAt: { $lt: staleThreshold } },
    { $set: { status: 'pending' }, $unset: { claimedBy: '', claimedAt: '' } }
  );
}
```

### Change Stream for Instant Notification (No Polling!)

```javascript
// Instead of polling, watch for new tasks
const taskStream = db.collection('tasks').watch([
  { $match: { operationType: 'insert' } }
]);

taskStream.on('change', async () => {
  // Try to claim — may lose to another worker (that's fine)
  await claimNextTask();
});
```

### Task Document Schema

```javascript
{
  _id: ObjectId(),
  type: "research",           // task type
  status: "pending",          // pending | processing | completed | failed
  priority: 5,                // higher = more urgent
  payload: { query: "..." },  // task-specific data
  result: null,               // filled on completion
  requiredCapability: ["web_search", "summarize"],
  createdBy: "orchestrator",
  claimedBy: null,
  claimedAt: null,
  completedAt: null,
  attempts: 0,
  maxAttempts: 3,
  scheduledAt: ISODate(),     // for delayed tasks
  expiresAt: ISODate(),       // TTL
  createdAt: ISODate()
}
```

### Verdict

This works extremely well for agent orchestration at moderate scale (<1000 tasks/sec). The atomic claim pattern is battle-tested. Combined with change streams for notifications, you get a full task queue without Celery, Bull, or any external queue system.

---

## 4. Time Series Collections — Agent Metrics & Cost Tracking

Time series collections use columnar compression and bucketing for efficient storage of temporal data. Perfect for tracking agent costs, latencies, and token usage.

```javascript
// Create time series collection
db.createCollection("agent_metrics", {
  timeseries: {
    timeField: "timestamp",
    metaField: "metadata",    // grouped by this (agent, model, etc.)
    granularity: "minutes"    // seconds | minutes | hours
  },
  expireAfterSeconds: 7776000 // 90 days auto-cleanup
});

// Insert a metric
db.agent_metrics.insertOne({
  timestamp: new Date(),
  metadata: {
    agentId: "scout",
    model: "claude-opus-4",
    taskType: "research"
  },
  inputTokens: 2500,
  outputTokens: 800,
  costUSD: 0.042,
  latencyMs: 3200,
  success: true
});

// Query: cost per agent per day
db.agent_metrics.aggregate([
  { $match: { "metadata.agentId": "scout" } },
  { $group: {
    _id: { $dateToString: { format: "%Y-%m-%d", date: "$timestamp" } },
    totalCost: { $sum: "$costUSD" },
    totalTokens: { $sum: { $add: ["$inputTokens", "$outputTokens"] } },
    avgLatency: { $avg: "$latencyMs" },
    taskCount: { $sum: 1 }
  }},
  { $sort: { _id: -1 } }
]);
```

### Key Benefits
- **10-20x compression** vs regular collections for metric data
- **Automatic bucketing** — MongoDB groups measurements by metadata + time window
- **Built-in expiry** via `expireAfterSeconds`
- **Aggregation pipeline** works normally — rolling averages, percentiles, etc.

### Limitation
- **No change streams** on time series collections (can't watch for new metrics in real-time)
- **Insert-heavy** — updates/deletes are limited
- Workaround: Write metrics to time series AND emit a notification to a regular collection if real-time alerts needed

---

## 5. Atlas Device Sync (formerly Realm) — Mobile/Edge Agents

### What It Is
Atlas Device Sync provides offline-first data sync between mobile/edge devices and MongoDB Atlas. Uses conflict-free resolution and a lightweight embedded database (Atlas Device SDK, formerly Realm).

### Relevance for Mobile/Edge Agents

**Potentially interesting for:**
- Mobile agents that need to work offline (e.g., a field research agent on a phone)
- Edge devices running lightweight agents (Raspberry Pi, etc.)
- Sync agent state/memories between edge and cloud

**Practical concerns:**
- Device SDK supports: Swift, Kotlin, Flutter, React Native, .NET, C++ — **no Python SDK** (Realm Python was deprecated)
- Designed for mobile apps, not server-side agents
- Adds complexity and Atlas dependency

### Verdict

**Low relevance for our use case.** Our agent factory runs on a local server with RTX 3090. If we later need mobile agents, we'd likely use a simpler sync mechanism (HTTP API + local SQLite). Device Sync solves a real problem but for a different architecture.

---

## 6. Limitations — What MongoDB Won't Do Well

### ❌ Full-Text Search Quality
- MongoDB's built-in `$text` search is basic (no BM25 ranking, no typo tolerance)
- Atlas Search (Lucene-based) is much better but requires Atlas infrastructure
- For serious full-text search, Elasticsearch/Meilisearch still wins
- **For agents**: Vector search is usually more relevant than full-text anyway

### ❌ Complex Multi-Collection Transactions
- Multi-document transactions exist but are expensive (locks, performance hit)
- MongoDB's sweet spot is document-level atomicity
- Agent orchestration rarely needs cross-collection ACID — usually fine

### ❌ High-Frequency Pub/Sub (>1000 msg/sec)
- Change streams aren't designed for high-throughput message passing
- Each watcher = own cursor = resource cost
- Fan-out to many consumers is expensive vs dedicated pub/sub

### ❌ Relational Joins
- `$lookup` exists but is slow compared to SQL joins
- Design around embedded documents instead
- Agent data is naturally document-shaped, so this rarely matters

### ❌ Vector Search Without Atlas
- Standard `mongod` does NOT support `$vectorSearch`
- Need Atlas (cloud or local via Atlas CLI) or handle vector search externally
- This is the biggest gotcha for a fully self-hosted setup

### ❌ Real-Time Streaming Analytics
- No native stream processing (Atlas Stream Processing is Atlas-only)
- Time series collections are for storage + batch queries, not real-time CEP

---

## 7. Cost Analysis

### MongoDB Atlas Free Tier (M0)

| Resource | Limit |
|----------|-------|
| Storage | 512 MB |
| RAM | Shared |
| Connections | 500 |
| Collections | 500 max |
| Change Streams | ✅ Supported |
| Vector Search | ✅ Supported (limited) |
| Time Series | ✅ Supported |
| Network Peering | ❌ |
| Backups | ❌ |

**Good for**: Development, testing, small personal agents.
**Not enough when**: You exceed 512MB (easy with vector embeddings — 1536-dim float32 = ~6KB per vector, so ~85K vectors fills it).

### When You Need to Pay

- **M2/M5 Flex clusters**: $9-25/month, 5GB storage — good for light production
- **M10+ Dedicated**: Starts ~$57/month — for real workloads
- **Vector Search nodes**: Separate pricing on dedicated clusters

### Self-Hosted (Our Best Option)

For a local RTX 3090 setup, self-hosted MongoDB is the clear winner:

```bash
# Docker Compose — local MongoDB replica set (needed for change streams)
version: '3.8'
services:
  mongo1:
    image: mongo:7
    command: mongod --replSet rs0 --bind_ip_all
    ports:
      - 27017:27017
    volumes:
      - mongo_data:/data/db

# Initialize replica set
docker exec -it mongo1 mongosh --eval "rs.initiate()"
```

**Free, unlimited storage, full change stream support.**

### The Vector Search Problem (Self-Hosted)

For vector search locally, you have two options:

1. **Atlas CLI local deployment** (containerized Atlas — includes vector search)
   ```bash
   atlas deployments setup localDev --type local --port 27017
   ```

2. **Hybrid approach** (recommended):
   - Self-hosted MongoDB for everything else
   - Qdrant in Docker for vector search (lighter, purpose-built)
   ```bash
   docker run -p 6333:6333 qdrant/qdrant
   ```

### Cost Summary

| Setup | Monthly Cost | Vector Search | Best For |
|-------|-------------|---------------|----------|
| Atlas M0 (free) | $0 | ✅ Limited | Dev/testing |
| Atlas M10 | ~$57 | ✅ | Small production |
| Self-hosted + Atlas CLI local | $0 (your hardware) | ✅ | Our use case |
| Self-hosted + Qdrant | $0 (your hardware) | Via Qdrant | Our use case (simpler) |

---

## Recommendation for Our Agent Factory

### Go with: Self-hosted MongoDB + Qdrant

```
┌─────────────────────────────────────────────┐
│              Agent Factory (RTX 3090)         │
│                                               │
│  ┌──────────┐  ┌──────────┐  ┌────────────┐ │
│  │ MongoDB  │  │  Qdrant  │  │   Agents   │ │
│  │ (Docker) │  │ (Docker) │  │            │ │
│  │          │  │          │  │ Scout      │ │
│  │ • Tasks  │  │ • Vector │  │ Kev        │ │
│  │ • State  │  │   search │  │ Workers    │ │
│  │ • Metrics│  │ • Memory │  │            │ │
│  │ • Comms  │  │   embeddings│            │ │
│  └──────────┘  └──────────┘  └────────────┘ │
│       ↕ change streams                       │
│       ↕ task queue                           │
└─────────────────────────────────────────────┘
```

**Why this split:**
- MongoDB handles: task queues, agent state, metrics (time series), coordination (change streams), document storage
- Qdrant handles: vector search for semantic memory (purpose-built, faster, simpler API)
- No NATS/Redis needed initially — change streams cover coordination
- Add NATS later only if you hit fan-out limits

**Alternative**: If you want ONE system, use Atlas CLI local deployment for everything including vector search. Trades some complexity for simplicity.
