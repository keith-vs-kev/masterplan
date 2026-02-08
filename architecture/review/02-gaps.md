# Architecture Gap Analysis

> **Date:** 2026-02-08
> **Author:** Hawk (Gap Analyzer)
> **Scope:** All 20 architecture docs + 20 research docs
> **Status:** Review v1

---

## Executive Summary

The architecture is impressively comprehensive for a v1 — 20 documents covering orchestration through self-improvement. But there are structural gaps that will bite hard in production. The biggest meta-gap: **the architecture assumes today's constraints are permanent**. Small contexts, slow generation, expensive tokens — these drive the multi-agent design. But if those constraints evaporate (10M context windows, 10K tok/s generation), the coordination overhead of 14 agents becomes the bottleneck, not the solution.

Below: every gap found, organized by severity.

---

## 1. CRITICAL: Constraint-Awareness (Jake's Insight)

### The Problem

The entire architecture is **constraint-assuming, not constraint-aware**. It's designed around:
- Small context windows (200K tokens max, practically 8-32K for local)
- Slow generation (50-100 tok/s cloud, 20-50 tok/s local)
- Expensive tokens ($5-25/MTok for frontier)

These assumptions are baked into fundamental design choices:
- **14 specialist agents** — because one model can't hold enough context to be a generalist
- **DAG-based task decomposition** — because tasks must be small enough for context windows
- **Pull-based work claiming** — because agents can only do one thing at a time effectively
- **Model tiering (budget/mid/frontier)** — because frontier is expensive
- **Memory system with retrieval** — because you can't fit everything in context

### What Happens When Constraints Change

**Scenario: 10M token context, 10K tok/s generation, $0.10/MTok frontier**

| Current Design | Becomes | Better Approach |
|---|---|---|
| 14 specialist agents | Coordination overhead > specialization benefit | 1-3 agents with full codebase in context |
| Task decomposition DAGs | Unnecessary fragmentation | Single agent handles entire feature end-to-end |
| Inter-agent communication bus (NATS) | Most messages are self-to-self | Direct in-context reasoning |
| Memory retrieval (Qdrant + Neo4j) | Irrelevant — everything fits in context | Load full project history into context |
| Model routing (LiteLLM tiers) | Only one tier matters | Direct calls, no proxy overhead |
| Heartbeat/liveness monitoring | Fewer agents = less to monitor | Simpler health checks |

### What's Missing: Adaptive Agent Topology

The architecture needs a **dial** — not a fixed agent count:

```
Model Capability Score = f(context_window, generation_speed, cost_per_token, quality)

If score < LOW_THRESHOLD:
    topology = "swarm"        # 14 specialists, heavy coordination
    decomposition = "fine"    # Small tasks, deep DAGs
    
If score > HIGH_THRESHOLD:
    topology = "monolith"     # 1-2 generalist agents
    decomposition = "coarse"  # Whole features, minimal DAGs
    
Else:
    topology = "hybrid"       # 3-5 agents with broader roles
    decomposition = "adaptive" # Decompose only when context overflows
```

**No document addresses this.** The architecture should be parameterized by model capabilities, not hardcoded to 2026-Q1 constraints.

### Specific Recommendations

1. **Add a Capability Profile** to the orchestrator that re-evaluates monthly:
   - Context window size available
   - Generation speed achievable
   - Cost per token at each quality tier
   - Quality delta between tiers
2. **Make agent count a configuration, not an architecture.** The 14-agent roster (doc 03) should be a deployment choice, not a structural requirement.
3. **Design the task decomposition algorithm to be depth-adaptive.** If one agent can handle the full task in context, don't decompose.
4. **Build the communication bus (doc 11) with a bypass mode.** If there are only 2 agents, NATS is overhead — direct function calls suffice.
5. **Memory system (doc 05) should have a "full context" mode** where everything is loaded in-context and the retrieval pipeline is skipped.

---

## 2. CRITICAL: Missing Components

### 2.1 Error Recovery & State Reconstruction

**Gap:** No document describes how to recover from a system-wide failure (dreamteam crashes, power outage, OOM kill).

- SQLite WAL mode mentioned but no explicit crash recovery procedure
- No description of how to reconstruct agent state after restart
- Task queue (doc 04) describes state machines but not how to recover mid-flight tasks
- What happens to a Pi SDK session that was running when the system crashes?
- ChromaDB, Neo4j, Qdrant — all running on one box with no backup strategy

**What's needed:**
- Startup recovery procedure: scan for RUNNING tasks, mark as FAILED, re-queue
- Write-ahead logging for critical state transitions
- Session replay capability for interrupted agent work
- Explicit "what happens when dreamteam reboots" runbook

### 2.2 Data Backup & Disaster Recovery

**Gap:** Zero mention across all 20 documents.

| Data Store | Backup Strategy | Currently Defined |
|---|---|---|
| SQLite (task queue, metrics) | ? | ❌ None |
| ChromaDB (embeddings) | ? | ❌ None |
| Neo4j (knowledge graph) | ? | ❌ None |
| Qdrant (vector search) | ? | ❌ None |
| Git repos | GitHub remote | ✅ Implicit |
| Agent memory files | ? | ❌ None |
| Shared filesystem artifacts | ? | ❌ None |
| LiteLLM config/state | ? | ❌ None |
| Grafana dashboards | ? | ❌ None |
| TimescaleDB (analytics) | ? | ❌ None |

**Everything runs on one machine with no documented backup.** A disk failure loses:
- All task history and state
- All agent memories
- All knowledge graph data
- All analytics
- All embeddings (expensive to regenerate)

**What's needed:**
- Automated daily backup to offsite (B2, S3, or even a second local disk)
- Point-in-time recovery capability for databases
- Backup verification (test restores monthly)
- RPO/RTO targets (e.g., RPO: 24h, RTO: 2h)
- Document what's reconstructible vs. permanently lost

### 2.3 Configuration Management

**Gap:** No centralized config management strategy.

Configuration is scattered across:
- `litellm_config.yaml` (model routing)
- `mcp-servers.yaml` (MCP registry)
- Agent `.md` files (agent identity)
- `scaffold.json` (project templates)
- `policies/*.yaml` (approval policies)
- Environment variables (API keys)
- Systemd unit files (services)
- Docker Compose files (databases)
- GitHub Actions YAML (CI/CD)
- AGENTS.md per repo (coding rules)

**No document describes:**
- How config changes propagate across the system
- Config versioning and rollback
- Environment-specific config (dev vs prod)
- Config validation before deployment
- Secret rotation procedures
- What happens when a config change breaks something

**What's needed:**
- Single config management approach (even if just "everything in one git repo with a deploy script")
- Config validation on change
- Automated secret rotation schedule
- Config change audit trail

### 2.4 Versioning & Database Migrations

**Gap:** Multiple databases (SQLite, PostgreSQL, TimescaleDB, Neo4j, Qdrant, ChromaDB) with zero migration strategy.

- Doc 01 defines a SQLite schema. What happens when you need to add a column?
- Doc 04 defines a PostgreSQL schema. Migration tool? Rollback capability?
- Doc 05 defines Neo4j schemas. How do you evolve the graph schema?
- Doc 18 defines TimescaleDB schemas. Continuous aggregates need careful migration.
- No version tracking on any schema
- No migration tooling mentioned (Drizzle migrate, Flyway, Liquibase, etc.)

**What's needed:**
- Migration tool per database (e.g., Drizzle for PostgreSQL/SQLite, custom scripts for Neo4j)
- Schema version tracking
- Rollback capability for failed migrations
- Migration testing in CI before production apply
- Data migration procedures (not just schema)

---

## 3. HIGH: Undefined Interfaces

### 3.1 Orchestrator ↔ Task Queue Inconsistency

Doc 01 (Orchestration) defines SQLite-based task queue with a specific schema.
Doc 04 (Task Queue) defines PostgreSQL-based task queue with a *different* schema and `FOR UPDATE SKIP LOCKED`.

**These are two different systems.** Which one is canonical?

- Doc 01: `tasks` table in SQLite, simple polling, Kev manages
- Doc 04: `tasks` table in PostgreSQL, LISTEN/NOTIFY, pull-based agents
- Doc 20 (Integration): References "SQLite + JSON files" for task queue

The schemas share concepts but differ in field names, types, and capabilities. This must be resolved before implementation.

### 3.2 Agent Runtime: OpenClaw Subagent vs Pi SDK vs Direct

Three execution modes are described but the boundary is fuzzy:

| Mode | Docs 01, 03 | Doc 04 | Doc 20 |
|---|---|---|---|
| OpenClaw subagent | Primary dispatch method | Not mentioned | Primary method |
| Pi SDK session | Alternative for raw LLM | Not mentioned | Alternative |
| Direct API | For simple tasks | Not mentioned | Via Pi SDK |
| Pull-based agent loop | Not mentioned | Primary model | Not mentioned |

Doc 04 describes agents running persistent loops polling a database. Doc 01/20 describe Kev spawning subagents on demand. These are fundamentally different execution models.

### 3.3 Communication: NATS vs OpenClaw vs Filesystem

Three communication mechanisms exist in parallel:

- Doc 11: NATS JetStream message bus with full pub/sub, topics, queue groups
- Doc 20: OpenClaw subagent sessions + filesystem coordination
- Doc 01: Orchestrator API (TypeScript functions)

No document reconciles these. When does an agent use NATS vs filesystem vs direct API call?

### 3.4 Memory: Three Competing Architectures

- Doc 05: Qdrant + Neo4j + custom MCP server (sophisticated)
- Doc 20: ChromaDB + filesystem (simple)
- Individual agent MEMORY.md files (current reality)

The integration doc (20) explicitly chooses ChromaDB, but the memory doc (05) designs for Qdrant + Neo4j. Which is the plan?

### 3.5 Monitoring: Three Competing Stacks

- Doc 02: LiteLLM + Langfuse + Redis + Prometheus/Grafana
- Doc 08: LiteLLM + Langfuse + Prometheus/Grafana + Redis + custom anomaly detector
- Doc 18: TimescaleDB + Redis + Grafana + custom ingestion
- Doc 20: "SQLite metrics table + REEF dashboard"

Four different monitoring architectures across four docs.

---

## 4. HIGH: Unhandled Failure Modes

### 4.1 Cascading Failure

No doc addresses cascading failure scenarios:
- LiteLLM proxy goes down → all agents lose LLM access → all tasks fail → dead letter queue floods → alert system may itself depend on LLM
- NATS goes down → agents can't communicate → tasks queue indefinitely
- Git push fails (GitHub outage) → all agent work stuck locally → disk fills up

**What's needed:** Dependency graph of services with failure impact analysis. Circuit breakers between every pair of dependent services. Graceful degradation modes.

### 4.2 Split-Brain / Partial Failure

Multiple agents operating on shared filesystem with no locking beyond SQLite:
- Two agents claim the same task (possible if filesystem-based queue)
- Agent writes to a file another agent is reading
- Git merge conflicts between parallel worktrees not handled automatically
- Neo4j and Qdrant diverge if one write succeeds and the other fails

### 4.3 Resource Exhaustion

- Disk full (artifacts accumulate, logs grow, model files) — no cleanup policy
- Memory pressure (Qdrant + Neo4j + ChromaDB + TimescaleDB + Redis + NATS + Ollama all on one box = 30+ GB RAM)
- GPU contention poorly handled — doc 13 mentions a mutex but no actual implementation
- What happens when all agent sessions are busy and a critical task arrives?

### 4.4 External Service Failures

- Stripe webhook delivery fails — revenue data gap, business decisions affected
- GitHub API rate limited — CI/CD pipeline stalls
- Anthropic has a 4-hour outage — all frontier work stops
- WhatsApp bridge disconnects — human loses visibility and control

Only model provider fallbacks (doc 02) are well-designed. Other external dependencies have no fallback strategy.

---

## 5. MEDIUM: Rate Limiting Between Internal Components

### 5.1 No Internal Rate Limiting

Doc 02 describes external API rate limiting. Doc 08 describes per-agent budget enforcement. But there's no rate limiting between internal components:

- No limit on how many tasks Kev can create per minute
- No limit on how many subagents can be spawned simultaneously
- No limit on filesystem I/O from agents
- No limit on memory system queries
- No limit on NATS message publishing (doc 11 mentions it but it's not implemented)
- No limit on MCP server requests

A single agent in a loop could overwhelm any internal service.

### 5.2 Back-Pressure

No back-pressure mechanism exists:
- If task queue fills up, what happens? Doc 04 mentions it but doesn't define behavior
- If all agents are busy, incoming tasks queue indefinitely with no bounds
- If ChromaDB is slow, agents block waiting for memory retrieval
- If the embedding pipeline falls behind, memory writes queue up

**What's needed:** Bounded queues with explicit overflow behavior (reject, shed load, alert).

---

## 6. MEDIUM: Operational Gaps

### 6.1 No Runbook / Operations Guide

20 architecture docs, zero operational documentation:
- How to start the system from cold
- How to stop gracefully (drain agents, complete in-flight tasks)
- How to add a new agent to production
- How to update a model without downtime
- How to investigate a stuck task
- How to recover from common failure scenarios
- Monitoring thresholds and who gets alerted
- On-call procedures (it's one person — Adam — but still needs documentation)

### 6.2 No Capacity Planning

- How many concurrent tasks can dreamteam handle?
- At what point does SQLite become the bottleneck?
- How much disk space does the system consume per day/week/month?
- What's the memory footprint with all services running?
- When does a second GPU become necessary vs nice-to-have?

Doc 13 does the cost math well but doesn't address capacity limits.

### 6.3 No Upgrade / Maintenance Window Strategy

- How do you upgrade Qdrant, Neo4j, TimescaleDB without data loss?
- How do you update agent prompts across 14 agents atomically?
- Can you do rolling updates of the orchestrator?
- Is there a maintenance mode that gracefully pauses the factory?

### 6.4 Logging Strategy

Logs mentioned everywhere, but no unified strategy:
- What format? (JSON structured? Plain text?)
- Where stored? (Each service's own logs? Central location?)
- Retention? (Some docs say 90 days, some say 1 year, some say nothing)
- Log levels? (When is something DEBUG vs INFO vs WARN vs ERROR?)
- Correlation IDs across services? (OpenTelemetry mentioned in docs 08, 11 but not implemented)

---

## 7. MEDIUM: Missing Non-Functional Requirements

### 7.1 Performance Targets

No doc defines acceptable performance:
- Max latency for task claiming?
- Max latency for memory retrieval?
- Max time from task creation to agent pickup?
- Target throughput (tasks/hour)?
- Dashboard refresh rate?
- Acceptable startup time after reboot?

### 7.2 Availability Targets

- No SLA defined for the factory itself
- No SLA for shipped products
- No availability target for the monitoring system
- What's the acceptable downtime per month?

### 7.3 Data Consistency Model

Multiple data stores with no defined consistency model:
- Is the task queue strongly consistent? (SQLite: yes. Filesystem JSON: no.)
- Is the memory system eventually consistent? What's the lag?
- What happens when the knowledge graph and vector store disagree?
- Are audit logs guaranteed to be complete, or best-effort?

---

## 8. LOW: Design Tensions & Contradictions

### 8.1 Simple vs Sophisticated

The architecture oscillates between "keep it simple" and "enterprise-grade":

- Doc 01: "SQLite, no Temporal, KISS" → Doc 04: "PostgreSQL, LISTEN/NOTIFY, SKIP LOCKED"
- Doc 20: "ChromaDB + filesystem" → Doc 05: "Qdrant + Neo4j + consolidation workers"
- Doc 20: "SQLite metrics" → Doc 18: "TimescaleDB + continuous aggregates + Grafana"

This isn't necessarily wrong (phased approach), but the docs don't clearly say "start here, migrate to there when X threshold is crossed." The phases exist but trigger conditions are vague.

### 8.2 14 Agents vs Reality

Doc 03 defines 14 specialist agents. Doc 20 plans to run all 14 on dreamteam. But:
- Each OpenClaw agent session consumes context and memory
- 14 agents × heartbeat polling = significant background load
- Most tasks only involve 2-3 agents
- Several agents (Law, Dot, Ally) may never be used in early months

No document describes which agents to activate first, or how to run a minimal viable roster.

### 8.3 Cost Pyramid vs Quality

The cost pyramid (cheapest model first) conflicts with the quality-first design:
- Doc 02: "Default cheap, escalate when needed"
- Doc 09: "Hawk and Rex use different models for adversarial testing"
- Doc 15: "Use cheapest capable agent"
- Doc 19: "Progressive delegation to cheaper models"

But: cheap models produce worse code → more QA failures → more retries → potentially MORE expensive than starting with a good model. No doc models this tradeoff quantitatively.

---

## 9. LOW: Missing Edge Cases

### 9.1 Multi-Tenant / Multi-User

The entire architecture assumes a single user (Adam). If a second human wants to use the factory:
- No user isolation in the task queue
- No per-user budget envelopes
- No per-user approval chains
- No per-user agent memory isolation
- WhatsApp approval channel assumes one recipient

### 9.2 Time Zone Handling

- Factory runs 24/7 but Adam is in NZ (GMT+13)
- Cron jobs need to be timezone-aware
- The "3am protocol" (doc 12) needs to handle daylight saving
- Scheduled tasks from different products may serve different timezones
- No doc mentions this

### 9.3 Dependency Between Products

Products in the portfolio may share:
- Common libraries
- Shared infrastructure
- Customer overlap
- API dependencies

No doc addresses cross-product dependency management.

### 9.4 Graceful Shutdown

What happens when you need to take the system down for maintenance?
- In-flight tasks?
- Scheduled cron jobs?
- Pending approval requests?
- Webhook deliveries?
- Agent sessions with accumulated working memory?

---

## 10. Summary: Top 10 Gaps by Priority

| # | Gap | Severity | Effort to Fix |
|---|-----|----------|---------------|
| 1 | **Constraint-aware architecture** (adaptive agent topology) | Critical | Large — requires rethinking core assumptions |
| 2 | **Data backup & disaster recovery** | Critical | Medium — standard tooling, needs procedure |
| 3 | **Error recovery from system crash** | Critical | Medium — startup recovery logic |
| 4 | **Resolve competing architectures** (SQLite vs PG, ChromaDB vs Qdrant) | High | Small — just make a decision and document it |
| 5 | **Database migration strategy** | High | Small — pick migration tools, add to CI |
| 6 | **Cascading failure handling** | High | Medium — dependency mapping, circuit breakers |
| 7 | **Configuration management** | Medium | Medium — centralize, validate, version |
| 8 | **Internal rate limiting & back-pressure** | Medium | Medium — bounded queues, overflow policies |
| 9 | **Operations runbook** | Medium | Small — document what you already know |
| 10 | **Capacity planning** | Medium | Small — benchmark, set thresholds, document |

---

## 11. Recommended Next Steps

1. **Write a "Constraints & Assumptions" doc** that explicitly lists every assumption about model capabilities, and defines trigger conditions for architectural changes.
2. **Pick ONE stack** for each concern (task queue, memory, monitoring) and deprecate the alternatives. The phased approach is fine, but name Phase 1 explicitly as "the plan" and Phase 2+ as "future upgrade path."
3. **Add a backup doc** (15 minutes of work to set up automated SQLite/Postgres backups to B2/S3).
4. **Write the crash recovery runbook** before the first production workload runs.
5. **Define the Minimum Viable Agent Roster** — which 3-5 agents are needed for Phase 1? Don't start all 14.

---

*Reviewed 20 architecture docs and 20 research docs. Analysis focused on structural gaps, not content quality — the individual docs are thorough and well-reasoned. The gaps are in the seams between them.*
