# 00 — Final Technology Stack Decisions

> **Date:** 2026-02-09 (revised)  
> **Author:** Atlas (Strategy Agent)  
> **Status:** FINAL — These are the decisions. Stop debating, start building.

---

## Guiding Principles

1. **MongoDB for everything persistent.** One database. No Postgres, no SQLite, no Redis for state. MongoDB change streams replace message buses. Atlas Vector Search replaces Qdrant. Jake works at MongoDB — we have free expertise.
2. **Minimize moving parts.** Every service is a thing that breaks at 3am.
3. **Free tiers first, paid when forced.** We're pre-revenue.
4. **Local-first compute.** dreamteam (RTX 3090, 64GB RAM) handles what it can. Cloud for frontier models only.
5. **Ship products, not infrastructure.** If it doesn't directly help ship a SaaS product this week, it waits.

---

## The Stack — Every Decision

### 1. Database: MongoDB ✅

**Choice:** MongoDB Community Edition (self-hosted on dreamteam), migrate to MongoDB Atlas (cloud) when needed.

**Rationale:**
- **Change streams** replace NATS/Redis pub-sub for event-driven coordination. Agent writes task → change stream triggers next agent. Zero additional infrastructure.
- **NoSQL flexibility** — agent state, task documents, memory entries, analytics events all have different shapes. No migrations, no ALTER TABLE.
- **Atlas Vector Search** — vector embeddings stored alongside the documents they describe. No separate Qdrant/ChromaDB to sync. One query hits both structured filters and semantic similarity.
- **Document model fits agent work** — a task is a document. An agent's memory is a document. A product's config is a document. Everything is already JSON.
- **Jake works at MongoDB.** Free expert support. Free Atlas credits likely. This alone is worth the decision.
- **TTL indexes** for automatic cleanup (logs, temp state). **Capped collections** for fixed-size event logs.

**What MongoDB replaces:**
| Was | Now |
|-----|-----|
| PostgreSQL (task queue) | MongoDB |
| SQLite (orchestrator state) | MongoDB |
| Redis (caching, counters, pub-sub) | MongoDB (change streams + in-memory caching) |
| Qdrant/ChromaDB (vector search) | MongoDB Atlas Vector Search |
| Neo4j (knowledge graph) | MongoDB (graph-like queries with `$graphLookup`) |
| TimescaleDB (time-series analytics) | MongoDB time-series collections |
| NATS JetStream (message bus) | MongoDB change streams |

**Collections architecture (initial):**
```
factory.tasks          — task queue (change stream for dispatch)
factory.agent_state    — agent configs, status, heartbeats
factory.memory         — agent memory with vector embeddings
factory.llm_calls      — LLM usage tracking (time-series collection)
factory.events         — system events (capped collection, change stream)
factory.products       — product registry and config
factory.revenue        — revenue tracking per product
factory.approvals      — human approval queue
```

**Task queue pattern:**
```javascript
// Atomic claim — equivalent to Postgres FOR UPDATE SKIP LOCKED
db.tasks.findOneAndUpdate(
  { status: "pending", assigned_to: null },
  { $set: { status: "claimed", assigned_to: "rex", claimed_at: new Date() } },
  { sort: { priority: -1, created_at: 1 }, returnDocument: "after" }
)
```

**Event-driven dispatch (replaces NATS):**
```javascript
const pipeline = [{ $match: { operationType: "insert", "fullDocument.type": "task.created" } }];
const changeStream = db.events.watch(pipeline, { fullDocument: "updateLookup" });
changeStream.on("change", (event) => {
  // Route to appropriate agent
});
```

**Start:** MongoDB Community 8.x in Docker on dreamteam.  
**Migrate to Atlas when:** Need multi-region, managed backups, or Atlas Vector Search features not in Community. Jake can likely get us Atlas credits.

---

### 2. Message Bus / Event System: MongoDB Change Streams ✅ (NATS deleted)

**Choice:** No separate message bus. MongoDB change streams handle all event-driven coordination.

**Rationale:**
- We have ONE server. All agents run on ONE machine. A distributed message bus is cosplaying as a microservices architecture.
- Change streams give us: watch for new tasks, watch for state changes, watch for approval responses, fan-out to multiple watchers.
- For the rare request/reply pattern: write a request document, watch for the response document. Simple.
- **What we lose vs NATS:** Queue groups (load balancing), request/reply primitives, KV store. **What we gain:** One less service to run, monitor, debug, and restart.

**When to add a real message bus:** Multiple machines. Not before. And even then, consider MongoDB Atlas change streams first.

---

### 3. Vector Search: MongoDB Atlas Vector Search ✅ (Qdrant deleted)

**Choice:** Store embeddings directly in MongoDB documents. Use Atlas Vector Search (or Community equivalent) for similarity queries.

**Rationale:**
- Embeddings live WITH the documents they describe. No sync layer between "the data" and "the vectors."
- One query can filter by metadata (product, agent, date range) AND do semantic similarity. Try that with a separate vector DB — you end up doing two queries and intersecting results.
- MongoDB Atlas Vector Search supports HNSW indexes, cosine/euclidean/dot-product similarity, pre-filtering. It's production-grade.
- Fewer moving parts. Qdrant is excellent software, but it's another Docker container, another port, another thing to back up.

**Embedding storage pattern:**
```javascript
{
  _id: ObjectId("..."),
  content: "Agent Rex completed the Cloudflare Workers deployment...",
  type: "memory",
  agent: "rex",
  product: "quickform",
  embedding: [0.023, -0.041, ...],  // 768 or 1536 dims
  created_at: ISODate("2026-02-08T10:30:00Z"),
  metadata: { task_id: "...", confidence: 0.92 }
}
```

**Vector search index:**
```javascript
{
  "type": "vectorSearch",
  "definition": {
    "fields": [{
      "type": "vector",
      "path": "embedding",
      "numDimensions": 768,
      "similarity": "cosine"
    }]
  }
}
```

**Note:** Community Edition has limited vector search. If we need full Atlas Vector Search before migrating to Atlas cloud, use a free M0 Atlas cluster just for vector-heavy collections. Jake can advise.

---

### 4. LLM Layer: Pi SDK ✅ (No proxy, no middleware)

**Choice:** **Pi SDK (@mariozechner/pi-ai)** as the single LLM layer. No LiteLLM. No proxy. No middleware.

**Rationale:**
- Pi SDK already supports multiple providers natively via `getModel()` with provider prefixes: `"cerebras/"`, `"anthropic/"`, `"google/"`, and any OpenAI-compatible endpoint (local llama.cpp, Ollama, Groq).
- **No proxy layer needed on a single machine.** LiteLLM adds a container, a config file, a port, and a point of failure — all to solve a problem Pi SDK already solves in-process.
- **Fallback routing** is trivial in application code: try provider A, catch error, try provider B. A few lines of TypeScript vs. an entire proxy container.
- **Cost tracking** via Pi SDK's built-in usage reporting — token counts, model used, cost per call. Log to MongoDB's `factory.llm_calls` time-series collection.
- **One less container, one less config, one less thing that breaks at 3am.**

**Provider routing in Pi SDK:**
```typescript
import { getModel } from "@mariozechner/pi-ai";

// Direct provider access — no proxy needed
const frontier = getModel("anthropic/claude-sonnet-4-5-20250514");
const mid = getModel("google/gemini-2.5-flash");
const fast = getModel("cerebras/llama-3.3-70b");
const local = getModel("openai-compatible/llama3.2", {
  baseUrl: "http://localhost:11434/v1"  // Ollama
});
```

**Fallback routing in application code:**
```typescript
async function callWithFallback(prompt: string, tiers: string[]) {
  for (const modelId of tiers) {
    try {
      const model = getModel(modelId);
      const result = await model.complete(prompt);
      // Log usage to MongoDB
      await db.collection("llm_calls").insertOne({
        timestamp: new Date(),
        model: modelId,
        tokens: result.usage,
        cost: calculateCost(modelId, result.usage),
      });
      return result;
    } catch (err) {
      console.warn(`${modelId} failed, trying next...`);
    }
  }
  throw new Error("All providers failed");
}
```

**Budget enforcement:**
```typescript
// Application-level budget checks — no proxy needed
async function checkBudget(agent: string): Promise<boolean> {
  const today = new Date(); today.setHours(0,0,0,0);
  const spent = await db.collection("llm_calls").aggregate([
    { $match: { agent, timestamp: { $gte: today } } },
    { $group: { _id: null, total: { $sum: "$cost" } } }
  ]).toArray();
  return (spent[0]?.total ?? 0) < AGENT_BUDGETS[agent];
}
```

**What this replaces:**
- ~~LiteLLM proxy container~~ → Pi SDK `getModel()` calls
- ~~LiteLLM config YAML~~ → Provider prefixes in code
- ~~LiteLLM budget system~~ → MongoDB aggregation queries
- ~~LiteLLM fallback chains~~ → try/catch in TypeScript
- ~~LiteLLM cost logging~~ → Pi SDK usage + MongoDB writes

---

### 5. LLM Providers: Tiered Strategy ✅

| Tier | Provider | Models | Use Case | Est. % of Calls |
|------|----------|--------|----------|-----------------|
| **Local (free)** | Ollama / llama.cpp | Llama 3.2 8B, nomic-embed-text | Embeddings, classification, triage, drafts | 50-60% |
| **Speed (cheap)** | Cerebras | Llama 3.3 70B | Fast generation, research summaries | 15-20% |
| **Mid (balanced)** | Google | Gemini 2.5 Flash / Pro | Long-context analysis, content, general coding | 15-20% |
| **Frontier (expensive)** | Anthropic | Claude Sonnet 4.5 / Opus 4.6 | Complex architecture, critical code, agent reasoning | 5-10% |
| **Backup** | OpenAI | GPT-5 mini | Fallback when Anthropic is down | <5% |

All accessed directly via Pi SDK's `getModel()`. No proxy layer between the application and the providers.

**Cost pyramid target:** 60% free/local, 25% cheap/mid, 15% frontier = ~$30-50/day at full operation.

**Prompt caching strategy:**
- Structure system prompts as stable prefixes (cacheable) + dynamic suffixes
- Anthropic cache TTL is 5 min — batch related tasks within windows
- Realistic cache hit rate budget: 30% (not the 90% the docs fantasized about)

---

### 6. Monitoring & Observability: Langfuse + MongoDB ✅

**Choice:** Langfuse (self-hosted) for LLM tracing. MongoDB for everything else.

**Why Langfuse stays (despite the simplification review saying cut it):**
- It's ONE Docker container. The setup cost is 10 minutes.
- Pi SDK calls can be instrumented to send traces to Langfuse via its SDK — a few lines of wrapper code.
- Per-trace cost tracking, prompt management, eval framework — building this custom would take weeks.
- We need cost visibility from day one. This is non-negotiable when you're burning $30-50/day on LLM calls.

**Why NOT Grafana/Prometheus/TimescaleDB:**
- Overkill. We have 4 agents on one machine.
- MongoDB time-series collections handle metrics storage.
- Build a simple dashboard (HTML + MongoDB queries) for system metrics. Langfuse handles LLM metrics.

**Langfuse backing store:** Langfuse needs Postgres (it doesn't support MongoDB natively). Run a minimal Postgres instance ONLY for Langfuse. 512MB RAM, no other use.

**Alternative if Postgres-for-Langfuse feels wrong:** Use Pi SDK's usage reporting + custom MongoDB dashboard. Lose Langfuse's trace visualization but gain zero-Postgres purity. **Decision: Keep Langfuse. The trace visualization is worth a tiny Postgres sidecar.**

**System monitoring:**
```javascript
// MongoDB time-series collection for system metrics
db.createCollection("metrics", {
  timeseries: {
    timeField: "timestamp",
    metaField: "source",
    granularity: "minutes"
  },
  expireAfterSeconds: 2592000  // 30 day retention
})
```

Simple cron job collects CPU/RAM/GPU/disk every minute → writes to MongoDB. Query with aggregation pipeline. No Prometheus needed.

---

### 7. Framework & Language: TypeScript + Node.js ✅

**Choice:** TypeScript for everything. Node.js runtime.

**Rationale:**
- Pi SDK is TypeScript. OpenClaw is Node.js. The ecosystem is already TypeScript.
- Every product the factory builds will likely be TypeScript (Next.js, Cloudflare Workers, etc.). One language for the factory AND the products it builds.
- MongoDB's Node.js driver is best-in-class. Mongoose for schema validation if wanted, but native driver is fine.
- Claude Code and Codex both excel at TypeScript generation.

**NOT Python.** Python is great for ML/data, but we're building web products and agent orchestration. TypeScript wins on: type safety, async/await maturity, deployment targets (Cloudflare Workers, Vercel, etc.), and ecosystem alignment.

**Key packages:**
```json
{
  "mongodb": "^7.0",           // Native MongoDB driver
  "@mariozechner/pi-ai": "*",  // Pi SDK — LLM orchestration AND provider routing
  "zod": "^3.23",              // Runtime validation
  "tsx": "^4.0",               // TypeScript execution
  "vitest": "^3.0",            // Testing
  "playwright": "^1.50"        // Browser automation / E2E
}
```

---

### 8. Deployment (Products the Factory Builds): Cloudflare First ✅

**Choice:** Cloudflare Workers + Pages + D1 + R2 as the default deployment target for products.

**Rationale:**
- **Generous free tier:** 100K requests/day (Workers), unlimited static (Pages), 5GB storage (R2), 5GB DB (D1). Most micro-SaaS products live entirely in free tier.
- **Zero cold starts** (Workers run on V8 isolates, not containers).
- **Global edge deployment** by default. No region selection, no load balancers.
- **`wrangler deploy`** — one command deployment. Agents can deploy without complex CI/CD.
- **Cost at scale:** Workers paid plan is $5/mo for 10M requests. Absurdly cheap.

**Fallback targets:**
| Platform | When to Use |
|----------|-------------|
| **Vercel** | Next.js apps that need SSR, or when Cloudflare Workers limitations hit |
| **Railway** | Apps that need persistent processes (WebSocket servers, background workers) |
| **Fly.io** | Docker containers, non-JS backends |

**NOT AWS/GCP/Azure.** Too complex, too expensive for micro-SaaS. The factory builds small products fast — serverless platforms match that.

---

### 9. CI/CD: GitHub Actions + Simple Pipeline ✅

**Choice:** GitHub Actions for CI. Direct deployment commands for CD.

**Rationale:**
- GitHub Actions free tier: 2,000 minutes/month for private repos. Enough for dozens of products.
- Pipeline per product: lint → type-check → test → build → deploy. No Kubernetes, no ArgoCD, no progressive delivery (yet).
- Agents push code → GitHub Actions runs checks → if green, agent deploys with `wrangler deploy` or `vercel deploy`.

**Pipeline (per product):**
```yaml
# .github/workflows/ci.yml
name: CI
on: [push, pull_request]
jobs:
  check:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
      - run: npm ci
      - run: npm run lint
      - run: npm run typecheck
      - run: npm test
      - run: npm run build
```

**Deployment:** Agent runs `wrangler deploy` directly after CI passes. No staging environment for MVP products. Add preview deployments when a product has real users.

**Auto-rollback:** Cloudflare Workers support instant rollback via `wrangler rollback`. Agent monitors error rates post-deploy → rollback if spike detected.

---

### 10. Auth (for Products): Clerk or Supabase Auth ✅

**Choice:** **Clerk** as the default auth provider for products the factory builds.

**Rationale:**
- 10,000 MAU free tier. Most micro-SaaS products won't exceed this for months.
- Drop-in React components. Agents can integrate auth in minutes, not hours.
- Handles email/password, social login, MFA, session management, webhooks — everything.
- Better DX than rolling auth with NextAuth or Lucia.

**Alternative:** Supabase Auth (free tier, pairs well if product uses Supabase). Use when a product specifically needs Supabase's database.

**For the factory itself:** No auth needed. It runs locally, accessed via WhatsApp/CLI through OpenClaw. Adam IS the auth layer.

---

### 11. Payments (for Products): Stripe ✅

**Choice:** Stripe. No debate.

**Rationale:**
- Industry standard. Best API. Best documentation. Agents can integrate it reliably.
- Stripe Checkout for quick integration. Stripe Billing for subscriptions.
- **Budget for 2.9% + $0.30/txn** as a real cost line item (the cost audit correctly flagged this).

**Consider later:** LemonSqueezy or Paddle for products selling to EU (they handle VAT/sales tax as Merchant of Record). But Stripe first — it's what the agents know how to integrate.

---

### 12. Hosting (Factory Infrastructure): dreamteam + Cloud Edge ✅

**Choice:** dreamteam for the factory. Cloudflare/Vercel/Railway for products.

**Architecture:**
```
┌─────────────────── dreamteam (local) ────────────────────┐
│                                                           │
│  Docker Compose:                                          │
│  ├── mongodb:8.0          (3-4 GB RAM)                    │
│  ├── langfuse + postgres  (2 GB RAM total)                │
│  └── ollama               (GPU: 8-16 GB VRAM)             │
│                                                           │
│  Native processes:                                        │
│  ├── openclaw-gateway     (1 GB RAM)                      │
│  └── Pi SDK agents        (in-process LLM routing)        │
│                                                           │
│  Total: ~7-8 GB RAM + GPU                                 │
│  Leaves: ~55+ GB for agents and OS                        │
│                                                           │
└───────────────────────────────────────────────────────────┘
         │
         │ Agents deploy products to:
         ▼
┌──────────────────────────────────────┐
│  Cloudflare Workers/Pages (products)  │
│  Vercel (Next.js products)            │
│  Railway (persistent process products)│
└──────────────────────────────────────┘
```

**RAM budget:**

| Service | RAM | Notes |
|---------|-----|-------|
| MongoDB | 3-4 GB | WiredTiger cache, scales with data |
| Langfuse + Postgres | 2 GB | Langfuse web + tiny Postgres |
| Ollama | GPU VRAM (8-16 GB) | Models loaded on demand |
| OpenClaw Gateway | 1 GB | Agent runtime |
| **Total** | **~7-8 GB** | vs 8-10 GB with LiteLLM |

That's 55+ GB free for agent processes on a 64GB machine. No second server needed for months.

**Docker containers on dreamteam: 3** (MongoDB, Langfuse, Langfuse-Postgres). Ollama runs natively for GPU access. OpenClaw runs natively. Pi SDK routes LLM calls directly from agent processes — no proxy container needed.

---

### 13. Knowledge Graph: MongoDB $graphLookup ✅ (Neo4j deleted)

**Choice:** Store entities and relationships in MongoDB. Use `$graphLookup` for graph traversal.

```javascript
// Entities collection
{ _id: "entity:rex:quickform", type: "product", name: "QuickForm", properties: {...} }

// Relationships collection  
{ from: "entity:rex:quickform", to: "entity:stripe", type: "integrates_with", weight: 0.9 }

// Graph traversal
db.relationships.aggregate([
  { $match: { from: "entity:rex:quickform" } },
  { $graphLookup: {
    from: "relationships",
    startWith: "$to",
    connectFromField: "to",
    connectToField: "from",
    as: "graph",
    maxDepth: 3
  }}
])
```

Not as powerful as Neo4j for complex graph algorithms, but covers "what's connected to X?" which is 90% of what we need. Zero additional infrastructure.

---

### 14. Browser Automation: Local Playwright ✅ (Browserbase deferred)

**Choice:** Playwright running locally on dreamteam for testing and basic scraping.

**Rationale:**
- Playwright is free, fast, and agents already know how to use it.
- For QA: run E2E tests against deployed products.
- For scraping: handle simple market research, competitor analysis.

**Add Browserbase when:** Anti-bot detection blocks local Playwright (stealth mode), or we need parallel browser sessions at scale. Month 3+.

---

### 15. Secrets Management: Environment Variables ✅ (Vault deleted)

**Choice:** `.env` files on dreamteam. OpenClaw's built-in secret management. GitHub Actions secrets for CI.

No HashiCorp Vault. We have ONE machine and ONE operator. `.env` + `chmod 600` is fine.

---

### 16. Agent Architecture: 4 Core Agents ✅

Per the simplification review (which was correct):

| Agent | Role | Absorbs |
|-------|------|---------|
| **Kev** | Orchestrator + ops + analytics + finance | Kev, Dash, Finn, Dot |
| **Rex** | Builder + QA + deploy | Rex, Forge, Hawk, Pixel |
| **Scout** | Research + content + strategy | Scout, Echo, Atlas |
| **Blaze** | Marketing + sales + growth | Blaze, Chase |

Expand to specialist agents ONLY when context windows consistently max out or task types genuinely conflict. Not before.

---

## What's NOT in the Stack

| Rejected | Why |
|----------|-----|
| PostgreSQL | MongoDB handles everything Postgres would (Langfuse's tiny sidecar is the sole exception) |
| SQLite | Same — MongoDB is the single database |
| Redis | MongoDB change streams replace pub-sub; in-process caching replaces Redis cache |
| NATS / RabbitMQ | Change streams. One machine. No message bus needed. |
| Qdrant / ChromaDB | Atlas Vector Search. Vectors live with documents. |
| Neo4j / Memgraph | `$graphLookup` covers 90%. Add if proven insufficient. |
| LiteLLM | Pi SDK routes directly to providers. No proxy needed on a single machine. |
| Grafana / Prometheus | MongoDB time-series + Langfuse. Custom dashboard if needed. |
| TimescaleDB | MongoDB time-series collections. |
| HashiCorp Vault | `.env` files. |
| S3 / MinIO | Local filesystem. Add R2 when artifacts need CDN distribution. |
| Kubernetes | One machine. Docker Compose. |
| Terraform | `docker compose up`. |
| ArgoCD | `wrangler deploy`. |
| 14 agents | 4 agents. |

---

## Docker Compose (Production)

```yaml
version: "3.8"
services:
  mongodb:
    image: mongo:8.0
    ports: ["27017:27017"]
    volumes:
      - mongo_data:/data/db
      - ./mongo-init:/docker-entrypoint-initdb.d
    environment:
      MONGO_INITDB_DATABASE: factory
    command: mongod --replSet rs0  # Required for change streams
    restart: unless-stopped

  mongo-init-replica:
    image: mongo:8.0
    depends_on: [mongodb]
    entrypoint: >
      mongosh --host mongodb --eval 'rs.initiate({_id:"rs0",members:[{_id:0,host:"mongodb:27017"}]})'
    restart: "no"

  langfuse-db:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: langfuse
      POSTGRES_USER: langfuse
      POSTGRES_PASSWORD: ${LANGFUSE_DB_PASS}
    volumes: ["langfuse_pg:/var/lib/postgresql/data"]
    restart: unless-stopped

  langfuse:
    image: langfuse/langfuse:2
    ports: ["3001:3000"]
    environment:
      DATABASE_URL: postgresql://langfuse:${LANGFUSE_DB_PASS}@langfuse-db:5432/langfuse
      NEXTAUTH_SECRET: ${LANGFUSE_AUTH_SECRET}
      NEXTAUTH_URL: http://localhost:3001
    depends_on: [langfuse-db]
    restart: unless-stopped

volumes:
  mongo_data:
  langfuse_pg:
```

**3 containers** (MongoDB, Langfuse, Langfuse-Postgres) + the init helper. Ollama and OpenClaw run natively. Pi SDK runs in-process with the agents — no container needed.

**Note:** MongoDB requires a replica set (even single-node) for change streams. The `mongo-init-replica` service handles this on first boot.

---

## Migration Path

| Trigger | Action |
|---------|--------|
| Need full Atlas Vector Search | Free M0 Atlas cluster, or upgrade to Atlas dedicated |
| dreamteam RAM > 80% sustained | Optimize first (model sizes, connection pools). Then consider M2 Mac Mini as second node |
| Products need global low-latency DB | MongoDB Atlas (cloud). Jake can help with credits/architecture |
| >1000 tasks/day with complex routing | Consider adding Redis for hot-path caching only |
| Langfuse too limited | Evaluate Lunary or custom dashboard on MongoDB |
| Compliance requirements | MongoDB Atlas (SOC 2, HIPAA, etc.) |

---

## Cost Estimate (Monthly)

| Item | Month 1 | Month 6 | Month 12 |
|------|---------|---------|----------|
| LLM APIs | $500-700 | $1,300-1,800 | $2,000-2,800 |
| Infrastructure (electricity) | $50 | $60 | $65 |
| Product hosting (free tiers) | $10 | $200 | $600 |
| Marketing | $200 | $1,200 | $2,500 |
| Domains | $2 | $8 | $15 |
| Stripe fees | $0 | $250 | $1,280 |
| **Total** | **~$800-1,000** | **~$3,000-3,500** | **~$6,500-7,300** |

Infrastructure cost is LOWER than ever — 3 containers instead of 5. The RAM freed up means no second server until much later (if ever).

---

## Summary: The Final Stack

```
MongoDB ──── one database for everything
  ├── Task queue (change streams for dispatch)
  ├── Agent state & memory (with vector embeddings)
  ├── LLM call logging (time-series collections)
  ├── Product configs & revenue tracking
  ├── Event bus (change streams, not NATS)
  └── Knowledge graph ($graphLookup)

Pi SDK ──── single LLM layer (routing + execution)
  ├── getModel("openai-compatible/...") → Ollama (local, free)
  ├── getModel("cerebras/...")          → Cerebras (fast, free tier)
  ├── getModel("google/...")            → Google Gemini (mid-tier)
  ├── getModel("anthropic/...")         → Anthropic Claude (frontier)
  └── Application-level fallback routing via try/catch

Langfuse ──── LLM observability (+ tiny Postgres sidecar)

OpenClaw ──── agent runtime (Kev, Rex, Scout, Blaze)

Cloudflare ──── default deployment target for products

GitHub Actions ──── CI for products

Clerk + Stripe ──── auth + payments for products
```

**Total Docker containers on dreamteam: 3** (MongoDB, Langfuse, Langfuse-Postgres)  
**Total RAM: ~7-8 GB** out of 64 GB available  
**Total agents: 4** (expandable to 14 when justified)  

This is the stack. Build on it.

---

*Final decisions by Atlas — 2026-02-08, revised 2026-02-09 (removed LiteLLM — Pi SDK handles all LLM routing directly)*
*Supersedes: architecture/review/08-tech-stack-decisions.md*
