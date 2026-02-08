# Implementation Sprint Plan — Weeks 1–4

> Realistic build order for the Agentic Factory MVP.
> Based on all 20 architecture documents.
> Date: 2026-02-08

---

## Executive Summary

**The critical path is:** Task Queue → Kev Routing → Pi SDK integration → First Scout→Rex pipeline → First product shipped → Revenue.

Everything else is a force multiplier. The plan front-loads the minimum infrastructure needed to run one end-to-end pipeline, then layers on quality, memory, and marketing.

**What's NOT in weeks 1–4:** Self-improvement system (doc 19), full NATS bus (doc 11), A2A protocol, browser automation (doc 17), data analytics platform (doc 18), marketing automation (doc 16). These are month 2+ work.

---

## Dependency Graph

```
Week 1                    Week 2                    Week 3                    Week 4
───────                   ───────                   ───────                   ───────
Task Queue (SQLite)  ──►  Kev orchestration    ──►  QA pipeline (Hawk)   ──►  Deploy pipeline (Forge)
                          logic                      CI/CD integration         First product ships
                               │                          │
Agent Identity       ──►  Pi SDK integration   ──►  Cost tracking        ──►  REEF dashboard v1
(agent .md files)         (model routing)            (SQLite metrics)
                               │                          │
Smart Router config  ──►  First pipeline run   ──►  Approval gates       ──►  Spending controls
(LiteLLM setup)           (Scout→Rex)               (WhatsApp flow)           (budget envelopes)
                               │
                          Memory foundation ────►  MCP servers (basic)  ──►  Memory MCP server
                          (filesystem + daily)       (GitHub, filesystem)
```

---

## Week 1: Foundation (Infrastructure That Everything Depends On)

### Goal: The factory can accept, store, route, and track work items.

| # | Task | Owner | Effort | Depends On | Blocks |
|---|------|-------|--------|------------|--------|
| 1.1 | **Task Queue schema + SQLite setup** — Create the task table (doc 04 schema), state machine (pending→claimed→running→review→done→failed), ULID generation, idempotency keys. Filesystem dirs for artifacts. | Rex | 1 day | Nothing | Everything |
| 1.2 | **Agent identity files** — Create all 14 agent `.md` files with frontmatter config (doc 03 format). Name, role, default model, fallback model, skills, temperature. Start with Kev, Rex, Scout, Hawk, Forge. | Atlas/Rex | 0.5 day | Nothing | 1.4, 2.1 |
| 1.3 | **LiteLLM proxy setup** — Install LiteLLM on dreamteam, configure model aliases (doc 13 config): `local/fast`, `local/quality`, `cloud/fast`, `cloud/mid`, `cloud/frontier`. Connect Anthropic, OpenAI, Groq, Gemini API keys. Health checks. Systemd service. | Rex | 1 day | Nothing | 1.5, 2.2 |
| 1.4 | **Smart Router rules** — Implement the routing decision matrix (doc 02) in LiteLLM config: task-type → model mapping, fallback chains, per-model budget caps. Cost pyramid: Cerebras/local for simple → Gemini Flash for medium → Claude for complex. | Rex | 0.5 day | 1.3 | 2.2 |
| 1.5 | **Local inference setup** — Install Ollama/llama.cpp on dreamteam. Download Llama 3.1 8B (Q6_K), nomic-embed-text. Configure llama-server on :8080 (GPU) and :8081 (embeddings, CPU). Systemd services per doc 13. | Rex | 1 day | Nothing | 1.3 |
| 1.6 | **Shared filesystem structure** — Create `/agents/shared/queue/`, `/agents/shared/memory/`, `/agents/shared/artifacts/`, `/agents/shared/config/`. Unix permissions per agent. | Rex | 0.5 day | Nothing | 1.1 |
| 1.7 | **Basic security scaffolding** — Scoped API tokens per agent (doc 07), secrets in env files (never in code), outbound-only networking verified. Agent workspace isolation. | Rex | 0.5 day | 1.2 | 2.1 |

**Parallelism:** 1.1, 1.2, 1.3, 1.5, 1.6 can all run in parallel. 1.4 needs 1.3. 1.7 needs 1.2.

**End of Week 1 checkpoint:** LiteLLM proxying requests to local + cloud models. Task queue accepts/stores/retrieves tasks. Agent files defined. Filesystem structure in place.

---

## Week 2: First Pipeline (Scout → Rex End-to-End)

### Goal: Kev can receive a task, route it through Scout (research) and Rex (build), and produce a PR.

| # | Task | Owner | Effort | Depends On | Blocks |
|---|------|-------|--------|------------|--------|
| 2.1 | **Kev orchestration logic** — Kev reads from task queue, selects agent based on task type and agent capabilities (doc 01 routing logic), spawns OpenClaw subagent sessions, collects results. This is the brain — not fancy, just functional. | Rex | 2 days | 1.1, 1.2 | 2.4 |
| 2.2 | **Pi SDK integration** — Wire Pi SDK to spawn Claude Code / Codex / Cerebras sessions. Model selection follows Smart Router rules. Per-task token budget caps (hard kill at limit). | Rex | 1 day | 1.3, 1.4 | 2.4 |
| 2.3 | **Memory foundation** — Daily markdown files per agent (`memory/YYYY-MM-DD.md`). MEMORY.md for long-term curated knowledge. Agent read/write helpers. No vector DB yet — just filesystem. | Rex | 0.5 day | 1.6 | 3.5 |
| 2.4 | **First pipeline run: Scout → Rex** — Scout does market research (Cerebras, free), outputs research brief. Rex takes brief, scaffolds project (doc 15 templates), implements, commits to feature branch. End-to-end test with a real micro-SaaS idea. | Rex/Kev | 2 days | 2.1, 2.2 | 3.1 |
| 2.5 | **Cost tracking (basic)** — SQLite metrics table: every LLM call logged with agent_id, model, tokens, cost_usd, task_id, timestamp. LiteLLM callback hook → SQLite writer. Daily spend query. | Rex | 0.5 day | 1.1, 1.3 | 3.3 |
| 2.6 | **Git workflow automation** — Agents create feature branches, commit with structured messages, open PRs via GitHub CLI. PR template with test instructions and risk notes. | Rex | 0.5 day | Nothing | 2.4 |

**Parallelism:** 2.3, 2.5, 2.6 can run in parallel with 2.1/2.2. 2.4 needs 2.1 + 2.2.

**End of Week 2 checkpoint:** A real task flows from queue → Scout research → Rex build → PR created. Cost is tracked per-call. This is the first "factory produces code" moment.

---

## Week 3: Quality Gates + Approval System

### Goal: Code doesn't ship without QA. Humans approve what matters.

| # | Task | Owner | Effort | Depends On | Blocks |
|---|------|-------|--------|------------|--------|
| 3.1 | **Hawk QA agent** — Hawk reviews Rex's PRs with a *different* model (doc 09 adversarial separation). Runs typecheck, lint, test, build. Reports pass/fail back to Kev. Self-correction loop: fail → error piped back to Rex → retry (max 3). | Rex | 2 days | 2.4 | 4.1 |
| 3.2 | **CI/CD pipeline (GitHub Actions)** — Per doc 06: PR triggers lint → typecheck → test → SAST (Semgrep) → build. Hawk reviews after CI passes. Template workflow reusable across projects. | Rex | 1 day | 2.6 | 4.2 |
| 3.3 | **WhatsApp approval flow** — Tier classification (doc 12): auto-execute / notify / approve-before / mandatory. Approval requests sent to Adam via WhatsApp with emoji reply interface. Timeout defaults (4h deny for Tier 2). Reminder at 50%/90% of deadline. | Rex | 1.5 days | 2.5 | 4.3 |
| 3.4 | **Action classification engine** — Every agent action tagged with tier (0-3) per doc 12 classification. Production deploys = Tier 2. Spending >$20 = Tier 2. Rollbacks always auto-approved. | Rex | 0.5 day | 3.3 | 3.3 |
| 3.5 | **MCP servers (basic)** — Set up GitHub MCP server + Filesystem MCP server (doc 14 Tier 1). Gateway with role-based access control. Agents call tools through gateway, not directly. | Rex | 1 day | 2.3 | 4.4 |
| 3.6 | **Structured audit logging** — Every approval decision, tool invocation, and deployment logged immutably (doc 12 audit trail format). Append-only SQLite table. | Rex | 0.5 day | 3.3 | 4.5 |

**Parallelism:** 3.1 and 3.2 run together (QA + CI). 3.3/3.4 run in parallel. 3.5 independent. 3.6 after 3.3.

**End of Week 3 checkpoint:** PRs get adversarial QA review. CI catches regressions. Adam approves production deploys via WhatsApp. Audit trail of all decisions.

---

## Week 4: Ship First Product + Visibility

### Goal: A real product deployed to production. Dashboard shows factory state.

| # | Task | Owner | Effort | Depends On | Blocks |
|---|------|-------|--------|------------|--------|
| 4.1 | **Full pipeline: Scout → Rex → Hawk → Forge** — Forge agent deploys to Cloudflare/Railway/Vercel (doc 06 progressive delivery). Canary → staging → production with health checks. Auto-rollback on failure. | Rex | 2 days | 3.1, 3.2 | 4.6 |
| 4.2 | **Project scaffolding templates** — `ts-api` and `ts-nextjs` templates per doc 15. Shared config packages (@factory/eslint-config, @factory/tsconfig). Quality gate script (format + lint + typecheck + test + build). | Rex | 1 day | 3.2 | 4.1 |
| 4.3 | **Spending controls** — Budget envelopes per agent (doc 12): daily caps, per-transaction limits, velocity detection (3x normal = pause + notify). Redis counters for real-time spend tracking. | Rex | 1 day | 3.3 | — |
| 4.4 | **Memory MCP server** — `@factory/mcp-memory` (doc 14): store, recall, list. Semantic search via local embeddings. Agents write memories through MCP, not raw filesystem. | Rex | 1 day | 3.5 | — |
| 4.5 | **REEF Dashboard v0.1** — Electron app reading from SQLite. Shows: active tasks, agent status, daily/weekly cost, task completion rate, recent deploys. Read-only. Not pretty, just functional. | Rex | 1 day | 3.6, 2.5 | — |
| 4.6 | **Ship first product** — Full end-to-end: Scout researches a validated micro-SaaS idea → Rex builds it using template → Hawk QA passes → Forge deploys → Live URL. Stripe integration if subscription model. | All agents | 2 days | 4.1, 4.2 | — |

**Parallelism:** 4.2 and 4.3 and 4.4 and 4.5 can all run in parallel. 4.1 needs 3.1 + 3.2. 4.6 needs 4.1 + 4.2.

**End of Week 4 checkpoint:** First product live with a real URL. Dashboard shows factory health. Spending controls prevent runaway costs. Full audit trail.

---

## Critical Path Analysis

```
Task Queue (1.1) → Kev Routing (2.1) → First Pipeline (2.4) → Hawk QA (3.1) → Full Pipeline (4.1) → Ship Product (4.6)
   Day 1              Day 6               Day 9                 Day 13            Day 19              Day 23
```

**Total critical path: ~23 working days (4.5 weeks).**

Any delay on the critical path pushes the first product ship date. The most fragile link is **2.1 (Kev orchestration)** — it's the most complex single piece and everything downstream depends on it.

### Risk mitigation for critical path:
- Start 2.1 design on Day 1 even while 1.1 is being built
- 2.4 (first pipeline) can use a simplified Kev (hardcoded routing) if full orchestration isn't ready
- 4.6 can use manual deploy if Forge automation isn't complete

---

## The Hardest Parts (Ranked)

### 1. Kev Orchestration Logic (2.1) — 🔴 Hardest
**Why:** This is the brain. It needs to: read task queue, classify tasks, select agents, select models (cost pyramid), spawn subagent sessions, handle timeouts, collect results, manage retries, and route failures. It's the most complex state machine in the system and nothing works without it.
**Mitigation:** Start simple — hardcoded task→agent mapping. Add intelligence incrementally.

### 2. WhatsApp Approval Flow (3.3) — 🟠 Hard
**Why:** Bidirectional messaging with state management. Approval requests need to track: sent, reminded, approved/rejected/timed-out. Emoji parsing. Timeout behavior varies by tier. Must work when Adam is asleep (3am protocol).
**Mitigation:** Start with simple text replies (not emoji). Hardcode Tier 2 for everything initially.

### 3. Hawk QA + Self-Correction Loop (3.1) — 🟠 Hard
**Why:** Adversarial QA needs genuinely different model/prompt from Rex. The self-correction loop (fail → pipe error → retry) is conceptually simple but tricky to get right without infinite loops or meaningless retries.
**Mitigation:** Cap retries at 3. Different model is mandatory (e.g., Rex=Opus, Hawk=Sonnet). Start with just build/test pass as the gate.

### 4. Model Swap Manager (local inference) — 🟡 Medium
**Why:** Single 3090 can't run 8B + 32B simultaneously. Swapping models takes 15-30s. During swap, requests must route to cloud seamlessly. Race conditions if multiple requests arrive during swap.
**Mitigation:** Start with 8B only. Route quality requests to cloud. Add swapping in week 3-4 only if needed.

### 5. Cost Tracking Accuracy — 🟡 Medium
**Why:** LiteLLM callback must capture every call. Cached tokens have different pricing. Provider billing may not exactly match calculated costs. Under-counting → budget overruns; over-counting → unnecessary throttling.
**Mitigation:** Conservative estimates. Reconcile with provider invoices weekly.

---

## What Can Run in Parallel

```
Parallel Track A (Infrastructure):    1.1 → 1.6 → 2.5 → 3.6 → 4.5
Parallel Track B (Models/Routing):    1.3 → 1.4 → 1.5 → 2.2 → 4.3  
Parallel Track C (Agent Pipeline):    1.2 → 2.1 → 2.4 → 3.1 → 4.1 → 4.6
Parallel Track D (Quality/Approval):  2.6 → 3.2 → 3.3 → 3.4
Parallel Track E (Memory/Tools):      2.3 → 3.5 → 4.4
```

**Maximum parallelism: 3 tracks simultaneously** (given one primary engineer + AI agents).

With AI coding agents doing the building, realistic parallelism is 2-3 streams. Kev orchestration (Track C) is the bottleneck everyone waits on.

---

## What's Explicitly Deferred (Month 2+)

| Component | Doc | Why Deferred |
|-----------|-----|-------------|
| NATS Communication Bus | 11 | OpenClaw subagents + filesystem work for <20 agents. Bus is over-engineering for Week 1-4. |
| Full Memory System (Qdrant + Neo4j) | 05 | Filesystem + basic MCP memory is enough initially. Vector DB when we have enough data to search. |
| Browser Automation (Stagehand) | 17 | Not needed until E2E testing or scraping. Visual regression is month 2. |
| Data Analytics Platform (TimescaleDB + Grafana) | 18 | SQLite metrics + REEF dashboard is enough. Full analytics when there's revenue to analyze. |
| Marketing Automation (Blaze + Echo) | 16 | No point marketing until product 1 ships. Launch marketing in week 5-6. |
| Self-Improvement System | 19 | Needs months of operational data before signals are useful. Instrument now, analyze later. |
| Revenue Engine (full pipeline) | 10 | The manual Scout→Rex→ship loop is the revenue engine for now. Full automation month 2-3. |
| Hybrid Compute (model swapping) | 13 | Start with 8B local + cloud for quality. Add swapping when traffic justifies it. |
| Graduated Autonomy / Trust Scores | 12 | All agents start SUPERVISED. Trust scoring needs data. Enable after 100+ tasks. |
| A2A Protocol / Agent Registry | 11, 14 | Internal filesystem comms work fine at this scale. A2A for external interop later. |
| Code Generation Templates (full suite) | 15 | Start with ts-api only. Add ts-nextjs, py-api etc. as needed by real products. |

---

## Weekly Success Criteria

| Week | Done When... |
|------|-------------|
| **1** | `curl localhost:4000/v1/models` returns local + cloud models. Task inserted and retrieved from SQLite. Agent .md files committed. |
| **2** | Kev receives "research X and build a prototype" → Scout outputs research → Rex outputs code in a PR. Cost per run visible in SQLite. |
| **3** | PR gets Hawk review with pass/fail. CI runs on PR. Adam gets WhatsApp approval request and can ✅/❌ reply. |
| **4** | Live URL for first product. REEF shows task status + daily cost. Budget envelope blocks spend over limit. |

---

## Resource Assumptions

- **1 human** (Adam) — approves Tier 2+, sets direction, reviews weekly
- **1 orchestrator agent** (Kev) — routes all work
- **Primary build agent** (Rex / Claude Code) — does the heavy lifting
- **dreamteam server** — RTX 3090, 64GB RAM, always on
- **Cloud API budget** — ~$30-50/day during build phase (heavy Claude Code usage)
- **Total estimated cloud cost for weeks 1-4:** $600-1000

---

*This plan is designed to be wrong in the details and right in the sequence. Adjust tasks as reality intrudes, but protect the critical path.*
