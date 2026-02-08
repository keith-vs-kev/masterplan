# 03 — Simplification Review

> **Reviewer:** Atlas
> **Date:** 2026-02-08
> **Verdict:** This system is designed for a 50-person engineering org. It's one person and some LLMs. Cut 70% of it.

---

## The Core Problem

20 agents designed this architecture. Each one justified their own existence and added infrastructure to support it. The result: a system that needs NATS + PostgreSQL/TimescaleDB + Redis + ChromaDB/Qdrant + Neo4j + Grafana + LiteLLM + Browserbase + S3/MinIO + Vault + Langfuse — to help one guy ship micro-SaaS products.

**This is a distributed systems architecture for a single-machine operation.**

---

## 1. Database Consolidation: 5 → 1 (maybe 2)

### What's currently proposed

| Store | Used By | Purpose |
|-------|---------|---------|
| PostgreSQL/TimescaleDB | Analytics (doc 18) | Metrics, revenue, time-series |
| Redis | Real-time accumulators (doc 18), MCP registry cache (doc 14), session state | Fast counters, caching |
| ChromaDB/Qdrant | Memory system (doc 05, 19) | Vector embeddings, RAG |
| Neo4j | Self-improvement (doc 19) | Pattern library, knowledge graph |
| SQLite | Task queue (doc 20) | Task state |

### What you actually need

**SQLite. That's it.** For the first 6 months, probably the first year.

- **Task queue**: SQLite (already in the plan, good)
- **Metrics/analytics**: SQLite with date-partitioned tables. You're not doing 100K writes/sec. You're doing maybe 1K LLM calls/day
- **Revenue tracking**: SQLite. Stripe webhooks write to a table. Query with SQL
- **Vector search**: SQLite with `sqlite-vec` extension, OR just use the filesystem + grep for the first 3 months. Your agents already use markdown files for memory — this works
- **Knowledge graph (Neo4j)**: Just... don't. A YAML file of anti-patterns is fine. You don't need a graph database for "pattern A causes problem B"
- **Redis for real-time counters**: A variable in memory. Or a SQLite row you UPDATE. You have one server with one process

**Graduate to PostgreSQL when:** You have multiple products with real users hitting a real database, and SQLite's write contention becomes a problem. That's probably at $2K+ MRR.

**Graduate to a vector DB when:** You have >100K embeddings and sqlite-vec is too slow. That's months away.

**Never need Neo4j for this use case.**

---

## 2. Message Bus: NATS → Delete

Doc 11 (Communication Bus) is the most over-engineered document in the set. It describes:
- NATS JetStream with at-least-once delivery
- Dead-letter queues
- Circuit breakers
- Message envelope schemas with correlation IDs, causation IDs, trace IDs
- Deadlock detectors
- Saga patterns
- 3-node NATS cluster for HA

**This is a message bus for a system where all the agents run on one machine and coordinate via the filesystem.**

### What you actually have

OpenClaw already handles agent-to-agent communication via subagent spawning. Agents already read/write to a shared filesystem. The task queue is already JSON/SQLite files.

### What to do instead

- **Agent coordination**: OpenClaw subagents (already works)
- **Event notification**: Write a JSON event to a file. Other agents read it on their next heartbeat. Done
- **Task routing**: Kev reads the queue, assigns tasks. This is already the plan in doc 20
- **"Pub/sub"**: You don't need pub/sub. You have one orchestrator (Kev) that delegates to agents. It's a hub-and-spoke, not a mesh

**Graduate to a message bus when:** You have multiple machines, or agents that genuinely need async event-driven coordination at >100 events/sec. Not before.

---

## 3. Agent Count: 14 → 3-5

### Jake's insight is correct

> As context windows grow and generation speeds increase, multi-agent coordination cost may exceed benefit. Fewer bigger agents might beat many small specialists.

With 200K-1M context windows, a single Claude Opus call can hold an entire project's codebase + requirements + test results + deployment config. The overhead of:
1. Agent A writes output to filesystem
2. Kev reads output, decides next agent
3. Agent B loads context, reads Agent A's output
4. Agent B does work, writes output
5. Repeat

...is **massive** compared to one agent doing steps 1-5 in a single session with all context loaded.

### Current roster (14 agents)

Kev (orchestrator), Atlas (architect), Rex (coder), Forge (deployer), Scout (researcher), Hawk (QA), Pixel (design), Blaze (marketing), Echo (content), Chase (sales), Finn (finance), Dash (analytics), Dot (ops), Law (legal)

### Proposed roster (4 agents)

| Agent | Absorbs | Rationale |
|-------|---------|-----------|
| **Kev** (orchestrator) | Kev + Dash + Finn + Dot | One brain for strategy, cost tracking, operations, analytics. These are all "look at data, make decisions" tasks — perfect for one smart agent with the right tools |
| **Rex** (builder) | Rex + Forge + Hawk + Pixel | One agent that codes, tests, deploys, and handles basic UI. Claude Code already does all of this in a single session. Splitting "write code" from "test code" from "deploy code" creates handoff overhead for no gain |
| **Scout** (researcher + content) | Scout + Echo + Atlas | Research, write content, write docs, write architecture. These are all "read stuff, synthesize, write stuff" tasks |
| **Blaze** (growth) | Blaze + Chase + Law | Marketing, sales, and legal review. Low-volume, can share a context |

### Why this works

1. **Less handoff overhead**: No "Agent A outputs → Kev routes → Agent B inputs" for every step
2. **Better context**: Rex holds the full codebase + tests + deploy config in one session instead of passing artifacts between 4 agents
3. **Cheaper**: Fewer agent sessions = fewer system prompts loaded = fewer tokens burned on coordination
4. **Simpler**: 4 agents to monitor, debug, and improve vs 14
5. **Context windows are huge now**: Opus 4.6 has 200K tokens. Gemini has 1M. A single agent can hold an entire micro-SaaS project

### When to split again

Split an agent when it consistently hits context limits or when task types have genuinely conflicting system prompts. Not before.

---

## 4. MCP Gateway: Overkill → Direct Connections

Doc 14 proposes a centralised MCP Gateway with:
- Role-based access control per tool
- Credential injection via Vault
- Tool namespacing
- Semantic tool discovery
- Rate limiting per agent per tool
- Custom internal MCP servers (5 of them)

### What you need

Agents connect directly to MCP servers. OpenClaw already supports this. Put API keys in environment variables (you already do this).

- **No gateway**: Direct MCP connections. One agent, one server
- **No Vault**: `.env` files or OpenClaw's built-in secret management
- **No custom internal MCP servers**: The task queue is a SQLite file. The memory is the filesystem. The deploy tool is `wrangler deploy`. You don't need to wrap these in MCP
- **No semantic tool discovery**: You have 4 agents. They know their tools

**Graduate to a gateway when:** You have 20+ MCP servers and genuinely need centralised credential rotation. Not at 4 agents with 5 tools each.

---

## 5. Monitoring: Grafana + TimescaleDB + Langfuse → SQLite + Terminal

Doc 18 proposes TimescaleDB, Redis, Grafana, S3/MinIO for a full analytics platform.

### What you need for months 1-6

```bash
# Cost tracking
sqlite3 metrics.db "SELECT date(time), SUM(cost_usd) FROM llm_calls GROUP BY 1 ORDER BY 1 DESC LIMIT 7;"

# Agent performance
sqlite3 metrics.db "SELECT agent_id, COUNT(*), AVG(cost_usd) FROM tasks WHERE status='done' GROUP BY 1;"

# Revenue
sqlite3 metrics.db "SELECT date(time), SUM(amount) FROM revenue GROUP BY 1;"
```

That's it. Log LLM calls to SQLite (LiteLLM can do this with its built-in SQLite support). Query with SQL. Look at the numbers.

**REEF dashboard** (doc 20): A simple HTML page that queries SQLite and renders tables. Not Grafana with 15 panels and continuous aggregates.

**Graduate to Grafana when:** You're actually staring at dashboards daily and need alerting. That's a month 3+ problem.

---

## 6. Self-Improvement System: Premature Optimization

Doc 19 is impressive but describes:
- LLM-as-judge signal extraction on every session
- Vector clustering for emergent category discovery
- A/B testing framework with Thompson Sampling
- Knowledge distillation pipeline
- Anti-pattern library in Neo4j
- Meta-learning loop

### What you need

1. **When something breaks**: Write down what happened in a markdown file
2. **Update AGENTS.md**: Add the lesson learned
3. **Review weekly**: Read the week's failures, update prompts

That's the self-improvement system. It's manual, it works, and it costs $0.

**Graduate to automated signals when:** You're running 100+ tasks/day and can't review them manually. Month 3-4 at earliest.

---

## 7. Approval System: 4 Tiers → 2 Rules

Doc 12 proposes a sophisticated 4-tier approval system with trust scores, graduated autonomy, policy-as-code YAML, domain-specific trust levels, and phone call escalation.

### What you need

1. **Auto-execute**: Reading, researching, writing code, running tests, deploying to staging
2. **Ask Adam first**: Spending money, deploying to production, sending anything to a customer, anything public

Two rules. No trust scores. No graduated autonomy engine. No phone call escalation via Twilio.

**The 3am protocol** in doc 12 is good thinking but: just queue everything non-urgent. Nothing is so urgent it can't wait 8 hours when you have zero customers.

---

## 8. The Minimum Viable Architecture

```
┌─────────────────────────── dreamteam ───────────────────────────┐
│                                                                  │
│  OpenClaw Gateway                                                │
│  ├── Kev (orchestrator + ops + analytics)                        │
│  ├── Rex (build + test + deploy)                                 │
│  ├── Scout (research + content + architecture)                   │
│  └── Blaze (marketing + sales)                                   │
│                                                                  │
│  Shared State                                                    │
│  ├── SQLite (tasks, metrics, revenue, everything)                │
│  ├── Filesystem (agent memory, working files, artifacts)         │
│  └── Git repos (all project code)                                │
│                                                                  │
│  Local Compute                                                   │
│  ├── Ollama (embeddings, triage, drafts — free)                  │
│  └── LiteLLM proxy (route to local vs cloud)                     │
│                                                                  │
│  Cloud APIs (outbound only)                                      │
│  ├── Anthropic (Claude — complex work)                           │
│  ├── Google (Gemini — long context, free tier)                   │
│  ├── Groq (fast + cheap)                                         │
│  └── Stripe, GitHub, deployment platforms                        │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

### What's gone

| Removed | Replacement |
|---------|-------------|
| NATS + JetStream | OpenClaw subagents + filesystem |
| PostgreSQL/TimescaleDB | SQLite |
| Redis | In-memory / SQLite |
| ChromaDB/Qdrant | sqlite-vec or filesystem |
| Neo4j | YAML files |
| Grafana | SQL queries + simple HTML dashboard |
| HashiCorp Vault | Environment variables |
| S3/MinIO | Local filesystem |
| Langfuse | SQLite logging |
| MCP Gateway | Direct MCP connections |
| 5 custom MCP servers | SQLite + filesystem + CLI tools |
| Browserbase | Local Playwright (add Browserbase later for scraping) |
| 14 agents | 4 agents |
| A2A protocol | Not needed (single machine) |
| Saga pattern | Not needed (no distributed transactions) |
| NATS KV store | SQLite |
| OpenTelemetry | Console.log + SQLite |
| Phone call escalation | WhatsApp only |
| Trust score engine | Two rules: auto-execute or ask first |

### What stays (the good stuff)

- **LiteLLM proxy** — genuinely useful for multi-provider routing and cost tracking
- **Cost pyramid** (doc 02) — route cheap tasks to cheap models. This saves real money
- **OpenClaw as the orchestration layer** — already built, already works
- **Local inference with Ollama** — free tokens for simple tasks
- **Git-based workflow** — agents commit, PR, deploy. Clean
- **AGENTS.md per project** — prompt-level quality guardrails. Zero infrastructure cost
- **The human approval concept** — just simplified to 2 tiers
- **Deployment to Cloudflare/Railway/Vercel** — serverless, cheap, no infra to manage
- **Playwright for testing** — solid, but local only initially

---

## 9. Migration Path (When to Add Complexity Back)

| Trigger | Add |
|---------|-----|
| SQLite write contention (multiple agents hitting DB simultaneously) | PostgreSQL |
| >100K embeddings, search is slow | Qdrant or pgvector |
| >100 tasks/day, can't review manually | Automated signal detection (doc 19, simplified) |
| Multiple machines needed | Redis for coordination (not NATS — still overkill) |
| >10 MCP servers with credential rotation needs | MCP Gateway (simplified) |
| Need real-time dashboards daily | Grafana |
| Revenue >$2K/mo, need proper financial tracking | TimescaleDB or proper accounting integration |
| Agent context window consistently maxed | Split agents back out |
| External scraping at scale | Browserbase |

**The rule: add infrastructure when the pain of not having it exceeds the pain of maintaining it. Not before.**

---

## 10. Cost Comparison

### Proposed architecture (all 20 docs)
- Infrastructure to run: NATS, PostgreSQL, TimescaleDB, Redis, ChromaDB, Neo4j, Grafana, MinIO, LiteLLM, Vault, 5 custom MCP servers, Langfuse, Browserbase
- Setup time: 4-6 weeks just for infra
- Maintenance: Ongoing, non-trivial
- 14 agents burning system prompt tokens every heartbeat

### Simplified architecture
- Infrastructure: SQLite + LiteLLM + Ollama (already have these)
- Setup time: Most of it already works via OpenClaw
- Maintenance: Minimal — it's files on disk
- 4 agents, less coordination overhead

### Estimated savings
- **LLM costs**: ~40% less (fewer agents = fewer system prompts = fewer coordination tokens)
- **Setup time**: 4-6 weeks → 1-2 weeks to first product shipped
- **Cognitive overhead**: One person can hold 4 agents + SQLite in their head. Not 14 agents + 6 databases + a message bus

---

## 11. The Meta-Point

Jake's insight deserves repeating:

> As context windows grow and generation speeds increase, multi-agent coordination cost may exceed benefit. Fewer bigger agents might beat many small specialists.

This applies to the infrastructure too. Every component you add is:
- A thing that can break
- A thing you need to monitor
- A thing you need to upgrade
- A thing that consumes memory and CPU
- A thing that makes debugging harder
- A thing that slows down "I want to change how X works"

**The right architecture for a one-person AI factory is the one that lets you ship products fastest, not the one that would scale to 1000 agents.**

Start embarrassingly simple. Add complexity only when forced by actual pain. The architecture docs are a great *menu* of options — not a shopping list to buy all at once.

---

*Reviewed: 2026-02-08 by Atlas*
*Status: SIMPLIFY AGGRESSIVELY*
