# 🏭 The Agentic Factory — Executive Summary

**Date:** 2026-02-08 | **Author:** Atlas | **Read time:** 5 min

---

## The Vision

Build an **autonomous AI factory** that discovers market opportunities, builds products, deploys them, and generates revenue — 24/7, with minimal human input. Adam is the portfolio manager, not the operator.

**12-month target:** $30K–$70K/month from a portfolio of 8–15 live products.

---

## Architecture (30 Seconds)

```
 Adam (WhatsApp/CLI) → Kev (Orchestrator) → Pi SDK (Execution Engine)
                                               ├── Claude Code / Codex (building)
                                               ├── Cerebras (free research/triage)
                                               ├── Gemini (long-context analysis)
                                               └── RTX 3090 (local inference/TTS)
```

**Two-layer orchestration:**
- **Kev (OpenClaw)** — strategic brain. Decides *what* to build, assigns work, reviews output, talks to Adam.
- **Pi SDK** — execution muscle. Spawns parallel Claude Code / Codex sessions for actual building.

**14 agents**, each specialised: Rex (code), Forge (DevOps), Scout (research), Hawk (QA), Pixel (UI), Blaze (marketing), Echo (content), Chase (sales), Finn (finance), Dash (analytics), Dot (ops), Law (legal), Glim (local compute), Atlas (strategy).

→ [Full architecture](architecture/20-integration-blueprint.md) | [Orchestration](architecture/01-orchestration-layer.md)

---

## The Cost Pyramid

60% of tasks hit **free/cheap** models (Cerebras, local). 30% use mid-tier (Sonnet, Codex). Only ~10% need Opus.

| Item | Monthly Est. |
|------|-------------|
| LLM API spend (all providers) | $500–$1,500 |
| Infrastructure (hosting, DBs) | $100–$300 |
| Global hard budget cap | $3,000/day max |
| **Cost per product built** | **~$40–$70** |

Smart router auto-selects cheapest capable model per task. Hard budget limits at agent, team, and global levels with kill switches. → [Cost control](architecture/08-monitoring-costs.md) | [LLM economics](research/02-llm-provider-economics.md)

---

## The Revenue Engine

```
SCAN → VALIDATE → BUILD → LAUNCH → GROW
Scout   Scout     Rex     Forge    Blaze/Echo
```

**Pipeline:** Scout finds opportunities → Atlas writes PRD → Rex builds → Hawk tests → Forge deploys → Blaze markets. **2–5 days per product** at 24/7 operation.

**Target portfolio mix:** micro-SaaS, browser extensions, API services, digital products, niche content sites.

**Conservative model:** Launch 2 businesses/month. 30% survive. Survivors avg $2K/month by month 6. 1–2 breakouts at $10–20K each.

→ [Revenue engine](architecture/10-revenue-engine.md) | [Endgame vision](research/20-endgame-vision.md)

---

## Key Infrastructure Decisions

| Decision | Choice | Why |
|----------|--------|-----|
| Orchestration | Custom SQLite DAG executor | Temporal overkill; filesystem too fragile |
| LLM gateway | LiteLLM (self-hosted) | 100+ providers, per-key budgets, routing |
| Tracing | Langfuse (self-hosted) | Cost tracking, eval framework, no data leaves |
| Memory | Qdrant + Neo4j behind MCP | Semantic search + relationship graph |
| Comms bus | NATS | Async-first, lightweight |
| CI/CD | GitOps + progressive delivery | Preview → staging → canary → prod, auto-rollback |
| Local compute | dreamteam RTX 3090 / llama.cpp | Zero marginal cost for embeddings, classification, TTS |

→ [Task queue](architecture/04-task-queue.md) | [Memory](architecture/05-memory-system.md) | [CI/CD](architecture/06-cicd-pipeline.md) | [Security](architecture/07-security-permissions.md)

---

## Risks — Eyes Open

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|------------|
| API cost blowout | Medium | Medium | Hard budget caps, kill switches, cost velocity alerts |
| Agent reliability plateau | Medium | High | Hybrid 80/20 — agents handle bulk, humans handle edge cases |
| Platform bans (Stripe/hosting) | Medium | High | Multiple processors, multi-cloud, genuine products |
| Market saturation of AI products | Medium | Medium | Underserved niches, not tech-forward markets |
| Regulatory crackdown | Low (12mo) | High | Human-in-loop docs, compliance-first stance |
| Quality failures | Low | High | Adversarial QA (Hawk ≠ Rex), automated security scanning |

→ [Security & permissions](architecture/07-security-permissions.md) | [QA framework](architecture/09-qa-testing.md) | [Human approval](architecture/12-human-approval.md)

---

## First Steps — This Week

1. **Wire Pi SDK** as execution layer underneath Kev
2. **Set up LiteLLM** with budget limits and Cerebras/Claude/Gemini routing
3. **Build the task queue** (SQLite-backed, DAG-aware)
4. **First autonomous pipeline:** Scout → Atlas → Rex build loop on a real micro-SaaS idea
5. **Deploy Langfuse** for cost visibility from day one

**Week 2–4:** Memory system (Qdrant), CI/CD pipeline, Hawk QA integration, first product live.

**Month 2:** Pipeline tuned. 2–3 products/week. First revenue.

**Month 3:** Factory self-funding from product revenue.

---

## The Honest Take

**Realistic at 12 months:** Semi-autonomous factory, 60–80% autonomy, $30–70K/month, system improving monthly.

**Aspirational:** $100K+/month, near-full autonomy, licensing the factory itself as a product.

**Not happening yet:** Zero human involvement, $200K+/month, agents handling complex legal/regulatory autonomously.

The factory is the real product. Individual businesses are experiments. The system that generates, tests, and scales them is what compounds.

---

## Full Document Index

| # | Architecture Doc | Research Doc |
|---|-----------------|--------------|
| 01 | [Orchestration Layer](architecture/01-orchestration-layer.md) | [Agent SDKs](research/01-agent-sdks-comparison.md) |
| 02 | [Smart Router](architecture/02-smart-router.md) | [LLM Economics](research/02-llm-provider-economics.md) |
| 03 | [Agent Identity](architecture/03-agent-identity.md) | [Task Management](research/03-task-management-systems.md) |
| 04 | [Task Queue](architecture/04-task-queue.md) | [Automated Testing](research/04-automated-testing.md) |
| 05 | [Memory System](architecture/05-memory-system.md) | [Browser Testing](research/05-browser-ui-testing.md) |
| 06 | [CI/CD Pipeline](architecture/06-cicd-pipeline.md) | [Agentic CLIs](research/06-agentic-clis.md) |
| 07 | [Security](architecture/07-security-permissions.md) | [Factory Patterns](research/07-ai-factory-patterns.md) |
| 08 | [Monitoring & Costs](architecture/08-monitoring-costs.md) | [Memory & Knowledge](research/08-memory-knowledge.md) |
| 09 | [QA & Testing](architecture/09-qa-testing.md) | [MCP Ecosystem](research/09-mcp-ecosystem.md) |
| 10 | [Revenue Engine](architecture/10-revenue-engine.md) | [Deployment Infra](research/10-deployment-infra.md) |
| 11 | [Communication Bus](architecture/11-communication-bus.md) | [Revenue Generation](research/11-revenue-generation.md) |
| 12 | [Human Approval](architecture/12-human-approval.md) | [Agent Comms](research/12-agent-communication.md) |
| 13 | [Hybrid Compute](architecture/13-hybrid-compute.md) | [Code Quality](research/13-code-quality-guardrails.md) |
| 14 | [MCP Integration](architecture/14-mcp-integration.md) | [Monitoring](research/14-monitoring-observability.md) |
| 15 | [Code Generation](architecture/15-code-generation.md) | [Local Infra](research/15-local-infrastructure.md) |
| 16 | [Marketing & Growth](architecture/16-marketing-growth.md) | [Marketing Auto](research/16-marketing-automation.md) |
| 17 | [Browser Automation](architecture/17-browser-automation.md) | [Security](research/17-security-access.md) |
| 18 | [Data & Analytics](architecture/18-data-analytics.md) | [Human-in-Loop](research/18-human-in-loop.md) |
| 19 | [Self-Improvement](architecture/19-self-improvement.md) | [Competitive Landscape](research/19-competitive-landscape.md) |
| 20 | [Integration Blueprint](architecture/20-integration-blueprint.md) | [Endgame Vision](research/20-endgame-vision.md) |
