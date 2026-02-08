# Cross-Reference Conflict Report

> **Auditor:** Hawk (subagent)
> **Date:** 2026-02-08
> **Scope:** All 20 architecture docs (01–20, including 10)
> **Severity:** 🔴 Critical (blocks implementation) · 🟡 Major (causes confusion) · 🟢 Minor (cosmetic/naming)

---

## 🔴 CRITICAL CONFLICTS

### C1. Task Queue Database: SQLite vs PostgreSQL

The single most important architectural decision — what backs the task queue — is contradicted across three docs.

| Doc | Choice | Key Detail |
|-----|--------|------------|
| **01 (Orchestration)** | **SQLite** | "SQLite gives us ACID transactions, atomic task claiming, and zero operational overhead." |
| **04 (Task Queue)** | **PostgreSQL** | Uses `FOR UPDATE SKIP LOCKED`, `LISTEN/NOTIFY`, `pg_cron`. Entire claiming algorithm depends on Postgres-specific features. |
| **20 (Integration)** | **SQLite + JSON files** | "Task Queue (SQLite) — Simple, no dependencies, filesystem-native" |

**Impact:** Doc 04 is the most detailed task queue spec and is architecturally incompatible with docs 01 and 20. `FOR UPDATE SKIP LOCKED` and `LISTEN/NOTIFY` do not exist in SQLite. The pull-based claiming model in doc 04 cannot work with SQLite as written.

**Resolution needed:** Pick one. If SQLite, doc 04 needs a complete rewrite of the claiming mechanism. If PostgreSQL, docs 01 and 20 need updating.

---

### C2. Vector Database: Qdrant vs ChromaDB

| Doc | Choice | Rationale Given |
|-----|--------|-----------------|
| **05 (Memory)** | **Qdrant** | "Rust performance, hybrid search, self-hosted, production-ready" — explicitly chosen over ChromaDB |
| **20 (Integration)** | **ChromaDB** | "Local, embedded, pip install, sufficient for single-machine" |

**Impact:** Doc 05 provides a detailed architecture built on Qdrant's API and features (payload filtering, named collections, hybrid search). ChromaDB has a different API surface and fewer features. The memory MCP server in doc 05 would need rewriting for ChromaDB.

**Resolution needed:** Pick one. Doc 05's decision table explicitly rejected ChromaDB — if ChromaDB is the real choice, doc 05's rationale needs revisiting.

---

### C3. Knowledge Graph: Neo4j Present vs Absent

| Doc | Position |
|-----|----------|
| **05 (Memory)** | Neo4j is a core component — entity schema, Cypher queries, relationship types, 4GB RAM allocation |
| **20 (Integration)** | Neo4j not mentioned anywhere. Memory = "ChromaDB + filesystem" |

**Impact:** Doc 05's entire knowledge graph architecture (entity extraction, GraphRAG, conflict resolution via graph relationships) depends on Neo4j. Doc 20's deployment topology has no Neo4j instance. If Neo4j is cut, ~40% of doc 05 is dead.

**Resolution needed:** Decide if the knowledge graph is in scope for Phase 1 or deferred. Update both docs to agree.

---

### C4. Rex Role: Orchestrator Who Never Codes vs Primary Coding Agent

| Doc | Rex's Role |
|-----|------------|
| **03 (Agent Identity)** | **Orchestrator.** "Decomposes goals into tasks, dispatches to specialists. **Does NOT: Write code**" |
| **09 (QA Testing)** | **Coding agent.** "Rex (coding agent) and Hawk (QA agent) are architecturally separated" / "Rex builds" |
| **15 (Code Gen)** | **Coding agent.** Title: "How Rex (coding agent) generates, scaffolds, and maintains projects" |
| **20 (Integration)** | **Builder.** Pipeline: "Scout → Atlas → **Rex (build)** → Hawk (test)" |

**Impact:** This is a fundamental identity conflict. Doc 03 explicitly says Rex never writes code — it delegates to Forge, Blaze, Pixel. But docs 09, 15, and 20 treat Rex as the primary code-writing agent. The entire QA architecture in doc 09 is built around Rex-writes/Hawk-reviews.

**Resolution needed:** Either Rex codes or it doesn't. If Rex orchestrates, then docs 09, 15, 20 need to reference Forge/Blaze as the coding agents. If Rex codes, doc 03 needs fundamental revision.

---

### C5. Orchestrator Identity: Kev vs Rex

| Doc | Who Orchestrates |
|-----|------------------|
| **01 (Orchestration)** | **Kev** — "Kev (OpenClaw) as the strategic brain making decisions about what to do" |
| **03 (Agent Identity)** | **Rex** — "Rex — The Orchestrator. Decomposes goals into tasks, dispatches to specialists" |
| **20 (Integration)** | **Kev** — "KEV (Orchestrator) — Strategic brain, Task routing, Review & QA gate" |

**Impact:** Two different agents claim the orchestrator role. Doc 01 treats Rex as just another specialist agent that Kev dispatches to. Doc 03 treats Rex as the top-level orchestrator that all work flows through. This creates ambiguity in every inter-agent flow.

**Resolution needed:** Define the hierarchy clearly. Is it Adam → Kev → Rex → Specialists? Or Adam → Kev → Specialists (Rex being one)?

---

### C6. Communication Bus: NATS vs PostgreSQL NOTIFY vs Filesystem

| Doc | Choice | Detail |
|-----|--------|--------|
| **11 (Comms Bus)** | **NATS + JetStream** | Full pub/sub, queue groups, KV buckets, message persistence. Entire doc built on NATS. |
| **04 (Task Queue)** | **PostgreSQL LISTEN/NOTIFY** | "No external message broker needed — PostgreSQL does both" |
| **20 (Integration)** | **Filesystem + OpenClaw subagents** | "Agent Comms: OpenClaw subagents + filesystem — Already works, simple" |

**Impact:** Three completely different communication architectures. Doc 11 specifies NATS with detailed topic hierarchies, queue groups, and JetStream streams. Doc 04 explicitly chose Postgres NOTIFY over external brokers. Doc 20 bypasses both in favor of filesystem coordination. These are mutually exclusive architectures with different operational requirements.

**Resolution needed:** Pick one for Phase 1. The others can be migration targets.

---

## 🟡 MAJOR CONFLICTS

### M1. Budget/Spend Limits Disagree

| Doc | Limit | Value |
|-----|-------|-------|
| **02 (Router)** | Global daily cap | **$100/day** |
| **08 (Monitoring)** | Global daily cap | **$200/day** |
| **08 (Monitoring)** | Global monthly cap | **$3,000/month** |
| **12 (Approval)** | Daily auto-approve across agents | **$200/day** |
| **20 (Integration)** | Daily spend alert threshold | **$50/day** |
| **13 (Compute)** | Monthly cloud spend cap (LiteLLM) | **$100/month** |

**Impact:** An agent checking its budget would get different answers depending on which doc was implemented. The range is $50–$200/day, a 4× difference.

**Resolution needed:** Single source of truth for budget numbers — probably in doc 08 (Monitoring) since that's the cost control doc.

---

### M2. Per-Agent Daily Budgets Disagree

| Doc | Agent | Budget |
|-----|-------|--------|
| **02 (Router)** | code-agent | $20/day |
| **02 (Router)** | research-agent | $15/day |
| **02 (Router)** | task-runner | $5/day |
| **02 (Router)** | qa-agent | $30/day |
| **12 (Approval)** | cloud_infrastructure | $500/month (~$17/day) |
| **12 (Approval)** | marketing_spend | $300/month (~$10/day) |
| **12 (Approval)** | tools_and_apis | $400/month (~$13/day) |

**Impact:** Docs 02 and 12 use different agent names AND different budget structures (daily vs monthly). No clear mapping between them.

---

### M3. LLM Gateway: LiteLLM Assumed vs Absent

| Doc | Position |
|-----|----------|
| **02 (Router)** | LiteLLM is the routing foundation. Detailed config provided. |
| **08 (Monitoring)** | LiteLLM is "LLM Gateway." Central to all cost control. |
| **13 (Compute)** | LiteLLM proxy at :4000, detailed YAML config. |
| **20 (Integration)** | **LiteLLM not mentioned.** Pi SDK talks directly to providers. |

**Impact:** Three docs build their entire architecture on LiteLLM as the single gateway. Doc 20 (the integration blueprint!) doesn't mention it at all, instead routing through "Pi SDK" which calls providers directly. This means docs 02, 08, 13 assume infrastructure that doc 20 doesn't deploy.

**Resolution needed:** Is LiteLLM in the stack or not? If yes, doc 20 needs it in the deployment topology. If no, docs 02, 08, 13 need major revision.

---

### M4. Real-Time State: Redis Required vs Not in Stack

| Doc | Redis Usage |
|-----|-------------|
| **02 (Router)** | Cost tracking, semantic cache |
| **08 (Monitoring)** | Budget accumulators, kill switches, cost velocity — "Sub-millisecond reads on every request" |
| **11 (Comms)** | Event bus option (Redis Streams) |
| **18 (Analytics)** | Real-time accumulators |
| **20 (Integration)** | **Not mentioned anywhere in deployment topology** |

**Impact:** Docs 02, 08, 18 have Redis as a critical dependency for cost control and kill switches. Doc 20's deployment topology has no Redis instance. Kill switches and budget enforcement literally cannot work without it (as designed in doc 08).

**Resolution needed:** Either add Redis to doc 20's topology or redesign budget enforcement in docs 02/08 without Redis.

---

### M5. Monitoring Stack: Full Observability vs SQLite Metrics

| Doc | Stack |
|-----|-------|
| **08 (Monitoring)** | LiteLLM + Langfuse (self-hosted) + Prometheus + Grafana + Redis + custom anomaly detector. ~6 CPU, 9GB RAM. |
| **18 (Analytics)** | TimescaleDB + Redis + Grafana + S3/MinIO. ~5 CPU, 7GB RAM. |
| **20 (Integration)** | "Custom (SQLite metrics + REEF dashboard) — Start simple, add LangSmith/Langfuse later" |

**Impact:** Docs 08 and 18 specify substantial monitoring infrastructure (13+ GB RAM, 11+ CPU combined). Doc 20 says start with SQLite. These are completely different operational realities. Also, doc 08 says "Why Not LangSmith? Vendor lock-in" but doc 20 says "add LangSmith/Langfuse later" — treating LangSmith as a candidate when doc 08 already rejected it.

---

### M6. Analytics Database: TimescaleDB vs PostgreSQL vs SQLite

| Doc | Choice |
|-----|--------|
| **18 (Analytics)** | **TimescaleDB** — detailed schema with hypertables, continuous aggregates, compression |
| **04 (Task Queue)** | **PostgreSQL** |
| **20 (Integration)** | **SQLite** for metrics |

Three different databases for overlapping data (task events, cost tracking, metrics). Could potentially be unified but currently contradict.

---

### M7. Task Distribution Model: Pull vs Push

| Doc | Model |
|-----|-------|
| **04 (Task Queue)** | **Pull-based.** "Agents know their own capacity. They claim work when ready, rather than having a central scheduler guess." Explicitly chosen as a design principle. |
| **01 (Orchestration)** | **Push-based.** Router scores agents and assigns tasks. "Dispatch Engine assigns to agents." |
| **20 (Integration)** | **Push-based.** "Kev selects agent & model" then "Pi SDK spawns session." |

**Impact:** Pull vs push is a fundamental architectural choice that affects everything from agent lifecycle to failure handling. Doc 04 builds its entire design around pull-based claiming. Docs 01 and 20 assume a central dispatcher pushes work to agents.

---

### M8. Trust Level Systems: Three Incompatible Scales

| Doc | Levels | Scale |
|-----|--------|-------|
| **07 (Security)** | L0 Untrusted → L1 Supervised → L2 Assisted → L3 Autonomous → **L4 Trusted** | 5 levels |
| **09 (QA Testing)** | L0 Untrusted → L1 Probationary → L2 Trusted → **L3 Veteran** | 4 levels |
| **12 (Approval)** | SUPERVISED → ASSISTED → AUTONOMOUS → TRUSTED | 4 levels (different names) |

**Impact:** Three different trust level systems with different names, thresholds, and promotion criteria. An agent at "L2" means different things in each doc. Implementation will require a single unified trust system.

---

### M9. Execution Engine: Pi SDK vs OpenClaw Subagents vs Direct CLI

| Doc | Execution Mechanism |
|-----|---------------------|
| **20 (Integration)** | **Pi SDK** — spawns Claude Code, Codex, Cerebras sessions. Central to entire architecture. |
| **01 (Orchestration)** | **OpenClaw subagent** + **Pi SDK** + **Direct API** — three dispatch modes |
| **15 (Code Gen)** | **Direct CLI** — `claude -p`, `gemini`, `codex` invoked directly via shell |
| **Other docs (03, 04, 11)** | Don't mention Pi SDK at all |

**Impact:** "Pi SDK" is doc 20's centerpiece execution engine but isn't referenced in most other docs. Unclear if it's a real product or a planned abstraction. Doc 15 shells out to CLI tools directly.

---

### M10. Heartbeat Interval Inconsistency

| Doc | Interval |
|-----|----------|
| **01 (Orchestration)** | **60 seconds** |
| **04 (Task Queue)** | **30 seconds** (in agent registration schema) |
| **11 (Comms Bus)** | **30 seconds** |

Minor but could cause false timeout detections if implementations disagree.

---

### M11. Embedding Model Conflicts

| Doc | Choice |
|-----|--------|
| **05 (Memory)** | text-embedding-3-small (OpenAI) Phase 1 → nomic-embed-text (local) Phase 4 |
| **07 (Security)** | Uses Cerebras for entity extraction (cost-efficiency) |
| **13 (Compute)** | nomic-embed-text on CPU via llama.cpp |
| **20 (Integration)** | "Local model (via Ollama)" — no specific model named |

Not a hard conflict but unclear which embedding model is canonical for Phase 1.

---

### M12. Secret Management: Vault vs Env Vars

| Doc | Choice |
|-----|--------|
| **07 (Security)** | **HashiCorp Vault** with dynamic secrets, 15-min TTLs. Infisical as fallback. |
| **14 (MCP)** | **Vault or 1Password Secrets Automation** |
| **20 (Integration)** | **"Environment variables, never in code"** — no Vault mentioned |

Doc 07 builds an elaborate capability token system around Vault's dynamic secret generation. Doc 20 uses plain env vars. These have vastly different security postures.

---

## 🟢 MINOR CONFLICTS

### N1. Agent Count: 13 vs 14

- Doc 03 lists **14** specialist agents (Rex through Atlas)
- Doc 20 says "**13+**" agents
- Some docs reference agents not consistently in the roster (e.g., "Glim" appears in doc 20 but not in doc 03's roster)

### N2. Retry Backoff Defaults

| Doc | Initial Backoff |
|-----|-----------------|
| **01** | 5,000ms |
| **04** | 10,000ms |

### N3. Coverage Thresholds

| Doc | Line Coverage | Branch Coverage |
|-----|--------------|-----------------|
| **06 (CI/CD)** | ≥80% overall (gate) / ≥90% new code | ≥80% new code |
| **09 (QA)** | ≥90% new code | ≥80% new code |

Doc 06 gates overall at 80%; doc 09 gates new code at 90%. Not contradictory but could confuse implementers.

### N4. Mutation Score Threshold

| Doc | Threshold |
|-----|-----------|
| **06 (CI/CD)** | ≥75% |
| **09 (QA)** | ≥80% |

### N5. Task ID Format

| Doc | Format |
|-----|--------|
| **01** | UUIDv7 |
| **04** | ULID |

Both are time-sortable unique IDs but they're different formats.

### N6. Doc 01 Cross-References Don't Match

Doc 01 ends with: *"See [02-agent-runtime.md] for how individual agents execute"* and *"[03-state-and-memory.md] for the shared state architecture"* — but these filenames don't match any actual docs in the architecture folder. Doc 02 is "Smart Router," doc 03 is "Agent Identity."

### N7. Default Tech Stack for Products

| Doc | Framework | Database |
|-----|-----------|----------|
| **10 (Revenue)** | Next.js + Supabase (Postgres) | Supabase |
| **15 (Code Gen)** | Hono (default API framework) + Drizzle ORM | PostgreSQL |

Doc 10's revenue engine defaults to Next.js API routes + Supabase. Doc 15 defaults to Hono + Drizzle. For API services, these are different stacks.

### N8. Cost Projection Discrepancy

| Doc | Estimate | Basis |
|-----|----------|-------|
| **02 (Router)** | $30-60/day optimized (50M tok/day) | Smart tiering + cache + batch + local |
| **08 (Monitoring)** | $200/day cap | Hard limit |
| **13 (Compute)** | $73-148/month total hybrid | 2M tok/day |
| **20 (Integration)** | <$30/day target | Alert threshold |

The token volume assumptions vary wildly (2M vs 50M tokens/day), making cost projections incomparable.

---

## Summary: Top 6 Decisions Needed

1. **Database for task queue:** SQLite or PostgreSQL? (Blocks: 01, 04, 20)
2. **Vector/graph DB:** Qdrant + Neo4j or ChromaDB? (Blocks: 05, 20)
3. **Communication mechanism:** NATS, Postgres NOTIFY, or filesystem? (Blocks: 04, 11, 20)
4. **Rex's identity:** Orchestrator or coder? (Blocks: 03, 09, 15, 20)
5. **LiteLLM + Redis:** In the Phase 1 stack or not? (Blocks: 02, 08, 13, 20)
6. **Trust level system:** Unify the three incompatible scales (Blocks: 07, 09, 12)

---

*Report generated by Hawk (Cross-Reference Auditor) — 2026-02-08*
