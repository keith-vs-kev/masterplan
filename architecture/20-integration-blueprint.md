# 20 — System Integration Blueprint

> **The Capstone Document** — How every component of the Agentic Factory fits together.
>
> Date: 2026-02-08 | Author: Atlas | Status: v1.0

---

## 1. Full System Architecture

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                            HUMAN INTERFACE LAYER                                │
│                                                                                 │
│   Adam (WhatsApp/CLI)          REEF (Electron Dashboard)         GitHub PRs     │
│         │                            │                               │          │
│         ▼                            ▼                               ▼          │
│   ┌──────────────────────────────────────────────────────────────────────┐      │
│   │                    OPENCLAW GATEWAY (always-on)                      │      │
│   │              Message routing · Agent sessions · Heartbeats           │      │
│   └──────────┬───────────────────────────────────────┬──────────────────┘      │
│              │                                       │                          │
│              ▼                                       ▼                          │
│   ┌─────────────────────┐              ┌──────────────────────────┐             │
│   │   KEV (Orchestrator)│◄────────────►│   AGENT ROSTER (13+)     │             │
│   │   Strategic brain   │  delegates   │   Atlas · Rex · Forge    │             │
│   │   Task routing      │              │   Scout · Hawk · Pixel   │             │
│   │   Review & QA gate  │              │   Blaze · Echo · Chase   │             │
│   │   Cost governance   │              │   Finn · Dash · Dot · Law│             │
│   └────────┬────────────┘              └──────────┬───────────────┘             │
│            │                                      │                             │
├────────────┼──────────────────────────────────────┼─────────────────────────────┤
│            ▼          EXECUTION LAYER              ▼                             │
│   ┌──────────────────────────────────────────────────────────────────────┐      │
│   │                        PI SDK (Execution Engine)                     │      │
│   │         Spawns parallel LLM sessions · Manages concurrency          │      │
│   │                                                                      │      │
│   │   ┌────────────┐ ┌────────────┐ ┌────────────┐ ┌────────────┐      │      │
│   │   │ Claude Code │ │   Codex    │ │  Cerebras  │ │  Gemini    │      │      │
│   │   │ (complex    │ │ (batch     │ │ (free      │ │ (long-ctx  │      │      │
│   │   │  builds)    │ │  parallel) │ │  fan-out)  │ │  analysis) │      │      │
│   │   └────────────┘ └────────────┘ └────────────┘ └────────────┘      │      │
│   └──────────────────────────────┬───────────────────────────────────────┘      │
│                                  │                                              │
├──────────────────────────────────┼──────────────────────────────────────────────┤
│                                  ▼                                              │
│   ┌──────────────────────────────────────────────────────────────────────┐      │
│   │                      SHARED STATE LAYER                              │      │
│   │                                                                      │      │
│   │   Task Queue         Git Repos          Memory/Knowledge             │      │
│   │   (SQLite/JSON)      (all projects)     (ChromaDB + filesystem)      │      │
│   │                                                                      │      │
│   │   /agents/shared/    /home/adam/         /agents/shared/memory/       │      │
│   │   queue/{backlog,    projects/           embeddings/ + MEMORY.md      │      │
│   │   ready,in-progress, masterplan/         per-agent daily notes        │      │
│   │   review,done,fail}  [product repos]                                 │      │
│   └──────────────────────────────┬───────────────────────────────────────┘      │
│                                  │                                              │
├──────────────────────────────────┼──────────────────────────────────────────────┤
│                                  ▼                                              │
│   ┌──────────────────────────────────────────────────────────────────────┐      │
│   │                      LOCAL COMPUTE LAYER                             │      │
│   │                                                                      │      │
│   │   dreamteam (RTX 3090)                                               │      │
│   │   ├── llama.cpp / Ollama (local inference)                           │      │
│   │   ├── Qwen3-TTS (voice generation)                                   │      │
│   │   ├── Embedding models (local ChromaDB ingestion)                    │      │
│   │   └── Glim agent (local compute coordinator)                         │      │
│   └──────────────────────────────────────────────────────────────────────┘      │
│                                                                                 │
├─────────────────────────────────────────────────────────────────────────────────┤
│                          DEPLOYMENT TARGETS                                     │
│                                                                                 │
│   Cloudflare Workers/Pages    Railway        Vercel        GitHub Actions        │
│   (APIs, webhooks, edge)    (backends,DB)  (frontends)   (CI/CD pipelines)      │
└─────────────────────────────────────────────────────────────────────────────────┘
```

---

## 2. Component Interfaces

Every boundary in the system has a defined interface. Here's the contract between each pair.

### 2.1 Interface Map

```
Adam ──WhatsApp/CLI──► OpenClaw Gateway ──sessions──► Kev
                                         ──sessions──► Other Agents

Kev ──task assignment──► Pi SDK ──spawns──► Claude Code / Codex / Cerebras / Gemini
                                 ──returns──► results to /agents/shared/

Kev ──delegates──► Agent Roster ──via OpenClaw subagent sessions──► work products

All Agents ──read/write──► Shared Filesystem (/agents/shared/)
All Agents ──read/write──► Git Repos (commit, push, PR)
All Agents ──read/write──► Task Queue (JSON/SQLite state machine)

Pi SDK ──HTTP API──► LLM Providers (Anthropic, OpenAI, Google, Cerebras)
Local Compute ──OpenAI-compat API──► Ollama/llama.cpp (localhost:11434)

Forge/Rex ──CLI──► Deployment Platforms (wrangler, vercel, railway)
Hawk ──CI hooks──► GitHub Actions ──test results──► Kev

REEF Dashboard ──reads──► Task Queue + Agent Status + Cost Metrics
```

### 2.2 Interface Contracts

| Interface | Protocol | Format | Auth | Ref Doc |
|-----------|----------|--------|------|---------|
| Adam → OpenClaw | WhatsApp API / CLI | Natural language | OpenClaw session | — |
| Kev → Pi SDK | Shell (pi-subagent / pi-shell) | CLI args + stdout | API keys in env | [01], [06] |
| Kev → Agents | OpenClaw subagent spawn | Session context JSON | OpenClaw internal | [12] |
| Agents → LLM Providers | HTTPS REST | JSON (OpenAI-compat) | Bearer token | [02] |
| Agents → Shared State | Filesystem I/O | JSON files / Markdown | Unix permissions | [03], [08] |
| Agents → Git | git CLI | Commits / PRs | SSH keys | [13] |
| Agents → MCP Servers | JSON-RPC 2.0 (stdio / HTTP) | MCP protocol | Per-server | [09] |
| Forge → Deploy Platforms | Platform CLIs + REST APIs | Platform-specific | API tokens (scoped) | [10] |
| Hawk → CI/CD | GitHub Actions YAML + webhooks | JSON payloads | GitHub token | [04], [05] |
| REEF → Backend | HTTP / WebSocket | JSON | Local auth | [14] |
| Local Compute → Agents | HTTP (OpenAI-compat) | JSON | None (localhost) | [15] |

---

## 3. What Runs Where

### 3.1 Local (dreamteam server)

| Component | Why Local | Resource Needs |
|-----------|-----------|----------------|
| OpenClaw Gateway | Always-on orchestration, low-latency agent sessions | CPU, ~2GB RAM |
| All 14 OpenClaw agents | Persistent memory, heartbeats, WhatsApp bridge | CPU, ~4GB RAM total |
| Pi SDK runtime | Spawns LLM sessions, manages concurrency | CPU, ~1GB RAM |
| Task Queue (SQLite) | Low-latency reads/writes, no cloud dependency | Disk |
| Shared filesystem | Agent coordination, working state | Disk (SSD) |
| Git repos | All project code, version control | Disk |
| ChromaDB | Local vector store for agent memory/RAG | CPU + ~2GB RAM |
| Ollama / llama.cpp | Free local inference (drafts, triage, embeddings) | RTX 3090 (24GB VRAM) |
| Qwen3-TTS | Voice generation | RTX 3090 |
| REEF dashboard | Electron app for monitoring | CPU, display |

### 3.2 Cloud (API calls, no infrastructure to manage)

| Service | Purpose | Cost Model |
|---------|---------|------------|
| Anthropic API | Claude Opus/Sonnet for complex work | Per-token |
| OpenAI API / Codex | Parallel builds, batch work | Per-token / subscription |
| Cerebras API | Free fan-out (research, triage, drafts) | Free tier |
| Google Gemini API | Long-context analysis (1M+ tokens) | Per-token (free tier generous) |
| GitHub | Code hosting, Actions CI/CD | Free for public repos |
| Cloudflare | Deploy edge APIs and static sites | Free tier + usage |
| Railway | Deploy backend services + DBs | ~$5/mo + usage |
| Vercel | Deploy frontends (Next.js) | Free tier + usage |
| Stripe | Payment processing for products | 2.9% + $0.30 |

### 3.3 Decision Rule

> **Default to local.** Only use cloud for: (a) LLM inference beyond local GPU capacity, (b) production deployment of shipped products, (c) services that must be internet-accessible.

---

## 4. Tech Stack Decisions

| Layer | Choice | Why | Alternatives Considered | Ref |
|-------|--------|-----|------------------------|-----|
| **Orchestration** | OpenClaw (Kev) | Already built, persistent memory, WhatsApp, heartbeats, subagent spawning | LangGraph, CrewAI | [01], [07] |
| **Execution Engine** | Pi SDK | Unified LLM API, spawns parallel Claude/Codex sessions | Direct API calls, LangChain | [01], [06] |
| **Primary Build Model** | Claude Code (Opus/Sonnet) | Best agentic coding quality | Codex, Cursor | [02], [06] |
| **Batch Build Model** | Codex (async) | Good for parallel, cheaper | Claude Code cloud sessions | [02], [06] |
| **Free Inference** | Cerebras (gpt-oss-120b) | Free, fast (1500 tok/s), good enough for research/triage | Groq, local Llama | [02], [15] |
| **Long-Context** | Gemini 2.5 Pro | 1M token context, generous free tier | Claude (200K), GPT (128K) | [02] |
| **Local Inference** | Ollama + llama.cpp | Easy model management, OpenAI-compat API, zero marginal cost | vLLM, SGLang | [15] |
| **Task Queue** | SQLite + JSON files | Simple, no dependencies, filesystem-native, good enough to start | Redis/BullMQ, PostgreSQL | [03] |
| **Memory/Knowledge** | ChromaDB + filesystem | Local, embedded, pip install, sufficient for single-machine | Qdrant, Weaviate | [08] |
| **Vector Embeddings** | Local model (via Ollama) | Free, private, fast enough | OpenAI embeddings API | [08], [15] |
| **Tool Integration** | MCP (Model Context Protocol) | Standard protocol, growing ecosystem, supported by Claude Code | Custom APIs, LangChain tools | [09] |
| **Agent Comms** | OpenClaw subagents + filesystem | Already works, simple | A2A protocol, custom RPC | [12] |
| **CI/CD** | GitHub Actions | Free for public repos, well-understood | GitLab CI, self-hosted | [04], [10] |
| **Testing Framework** | Playwright (E2E) + Vitest/pytest (unit) | Best cross-browser, AI-integration ready | Cypress, Jest | [04], [05] |
| **Static Analysis** | ESLint + Biome + TypeScript strict | Multi-layer guardrails | SonarQube | [13] |
| **Deployment (edge)** | Cloudflare Workers/Pages | Free tier, global edge, wrangler CLI | AWS Lambda | [10] |
| **Deployment (backend)** | Railway | One-command deploy, built-in DBs, usage-based | Fly.io, Render | [10] |
| **Deployment (frontend)** | Vercel | Zero-config Next.js, preview URLs | Netlify, Cloudflare Pages | [10] |
| **Monitoring** | Custom (SQLite metrics + REEF dashboard) | Start simple, add LangSmith/Langfuse later | Datadog, Langfuse | [14] |
| **Security Model** | Scoped API tokens + filesystem permissions + egress filtering | Least-privilege, no shared secrets | Full container isolation | [17] |
| **Human-in-Loop** | WhatsApp (via OpenClaw) + GitHub PR reviews | Already works, low friction | Slack, custom approval UI | [18] |
| **Marketing** | Blaze agent + Cerebras drafts → Claude polish | Cost pyramid: free draft, paid polish | Jasper, Copy.ai | [16] |
| **Revenue** | Micro-SaaS + API services + browser extensions | Highest autonomy, lowest human involvement | Content farms, agencies | [11], [20] |

---

## 5. Deployment Topology

```
┌─────────────────────────── dreamteam (local server) ────────────────────────┐
│                                                                              │
│  ┌────────────────┐  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  │
│  │ OpenClaw       │  │ Pi SDK       │  │ Ollama       │  │ ChromaDB     │  │
│  │ Gateway        │  │ Runtime      │  │ (llama.cpp)  │  │ (embeddings) │  │
│  │ :3000          │  │              │  │ :11434       │  │ :8000        │  │
│  │                │  │ Spawns:      │  │              │  │              │  │
│  │ 14 agent       │  │ - claude -p  │  │ Models:      │  │ Collections: │  │
│  │ sessions       │  │ - codex      │  │ - qwen3-32b  │  │ - agent-mem  │  │
│  │ - heartbeats   │  │ - API calls  │  │ - llama-70b  │  │ - codebase   │  │
│  │ - cron jobs    │  │              │  │ - embed model│  │ - research   │  │
│  │ - subagents    │  │              │  │ - qwen3-tts  │  │              │  │
│  └───────┬────────┘  └──────┬───────┘  └──────────────┘  └──────────────┘  │
│          │                  │                                                │
│          ▼                  ▼                                                │
│  ┌──────────────────────────────────────────────────────────────────┐       │
│  │                    Shared Filesystem                              │       │
│  │  /home/adam/agents/shared/                                        │       │
│  │  ├── queue/{backlog,ready,in-progress,review,done,failed}/       │       │
│  │  ├── memory/                                                      │       │
│  │  ├── config/                                                      │       │
│  │  └── artifacts/                                                   │       │
│  │                                                                    │       │
│  │  /home/adam/projects/        (all git repos)                       │       │
│  │  /home/adam/.openclaw/       (agent workspaces)                    │       │
│  └──────────────────────────────────────────────────────────────────┘       │
│                                                                              │
│  ┌──────────────┐                                                            │
│  │ REEF         │  Electron dashboard — reads queue, metrics, agent status   │
│  │ (UI)         │                                                            │
│  └──────────────┘                                                            │
│                                                                              │
└──────────────────────────────────────────────────────────────────────────────┘
         │
         │  HTTPS (outbound only)
         ▼
┌──────────────────────────── Cloud Services ─────────────────────────────────┐
│                                                                              │
│  LLM APIs                    Deployment Targets         Services             │
│  ├── api.anthropic.com       ├── Cloudflare Workers     ├── GitHub           │
│  ├── api.openai.com          ├── Railway                ├── Stripe           │
│  ├── api.cerebras.ai         ├── Vercel                 └── WhatsApp API     │
│  └── generativelanguage.     └── GitHub Pages                                │
│      googleapis.com                                                          │
│                                                                              │
└──────────────────────────────────────────────────────────────────────────────┘
```

---

## 6. Data Flow: End-to-End Task Lifecycle

```
1. INPUT                2. ROUTING              3. EXECUTION
─────────────          ──────────────          ──────────────
Adam sends msg    →    Kev receives       →    Kev creates task
via WhatsApp           via OpenClaw             in queue/ready/
                       session
                                           →    Kev selects agent
                                                & model (cost pyramid)

                                           →    Pi SDK spawns session
                                                (Claude Code / Codex /
                                                 Cerebras / Gemini)

4. WORK                 5. REVIEW               6. DELIVERY
─────────────          ──────────────          ──────────────
Agent works in    →    Output lands in    →    Kev reviews output
git worktree           /agents/shared/          (or Hawk for QA)
                       + git commit
                                           →    If good: merge PR,
                                                deploy, notify Adam

                                           →    If bad: re-route to
                                                different agent/model,
                                                or escalate to Adam

7. DEPLOY               8. MONITOR
──────────────          ──────────────
Forge deploys      →    Dash tracks metrics
via platform CLI        Cost logged to SQLite
                        REEF dashboard updates
```

---

## 7. Cross-Cutting Concerns

### 7.1 Security (ref: [17])
- **Principle:** Agents are untrusted code with delegated authority
- **Secrets:** Environment variables, never in code or shared state. Scoped per-agent where possible
- **Network:** Outbound-only from dreamteam. No inbound ports exposed
- **Filesystem:** Each agent workspace has Unix permission boundaries. Shared dir is the coordination point
- **Prompt injection defense:** Agent outputs reviewed before external actions. MCP servers validate inputs

### 7.2 Cost Control (ref: [02], [14])
- **Cost pyramid enforced:** Every task starts at cheapest capable model. Escalate only on failure/quality need
- **Per-task budget caps:** Pi SDK wraps API calls with token counters. Hard-kill at budget limit
- **Daily spend alerts:** Kev checks daily spend in morning brief. Alert threshold: $50/day initially
- **Free tier maximization:** Cerebras for ~60% of tasks. Local inference for embeddings, triage, TTS

### 7.3 Quality Guardrails (ref: [04], [05], [13])
- **Layer 1:** AGENTS.md / rules files in every repo (prompt-level)
- **Layer 2:** TypeScript strict mode, ESLint, Biome (inline)
- **Layer 3:** Pre-commit hooks (format, lint, type-check)
- **Layer 4:** GitHub Actions CI (tests, SAST, mutation testing via Stryker)
- **Layer 5:** Hawk agent reviews PRs before merge. Different model than the builder

### 7.4 Human-in-the-Loop (ref: [18])
- **Auto-approve:** Read-only ops, drafts, research, internal working docs
- **Notify-after:** Standard builds, routine deploys to staging
- **Approve-before:** Production deploys, customer-facing comms, financial transactions, public posts
- **Never auto-approve:** Legal, security changes, data deletion

### 7.5 Memory & Knowledge (ref: [08])
- **Short-term:** Agent working memory within OpenClaw sessions
- **Medium-term:** Daily markdown files (memory/YYYY-MM-DD.md per agent)
- **Long-term:** MEMORY.md (curated), ChromaDB (semantic search)
- **Shared knowledge:** /agents/shared/knowledge/ + git repos as ground truth
- **MCP integration:** MCP servers for external knowledge (GitHub, docs, APIs) [09]

### 7.6 Monitoring & Observability (ref: [14])
- **Phase 1 (now):** SQLite metrics table — token usage, cost, task status, latency per call
- **Phase 2 (month 2):** REEF dashboard reads SQLite, shows real-time burn rate + agent status
- **Phase 3 (month 3):** Langfuse integration for full LLM tracing if needed
- **Alerts:** Cost velocity > threshold → WhatsApp alert to Adam. Agent stuck > 30 min → Kev escalates

---

## 8. Build Order (Dependency Graph)

What gets built first, second, third. Each phase unlocks the next.

```
PHASE 1: Foundation (Week 1-2)
═══════════════════════════════
 ┌─────────────────┐     ┌──────────────────┐     ┌──────────────────┐
 │ Task Queue       │     │ Pi SDK Setup     │     │ Model Routing    │
 │ (SQLite schema + │     │ (install, config │     │ (cost pyramid    │
 │  JSON file ops)  │     │  API keys)       │     │  rules in Kev)   │
 └────────┬────────┘     └────────┬─────────┘     └────────┬─────────┘
          │                       │                         │
          └───────────────────────┼─────────────────────────┘
                                  ▼
                    ┌──────────────────────────┐
                    │ Kev Orchestration Logic  │
                    │ (read queue → route →    │
                    │  spawn → collect results)│
                    └──────────────────────────┘

PHASE 2: First Pipeline (Week 2-3)
══════════════════════════════════
 Scout (research)  →  Atlas (PRD)  →  Rex (build)  →  Hawk (test)
     Cerebras          Claude Opus     Claude Code     Claude Sonnet
                                                           │
                                                           ▼
                                                    Forge (deploy)
                                                       Codex

PHASE 3: Automation (Week 3-4)
══════════════════════════════
 ┌────────────────┐  ┌──────────────┐  ┌───────────────┐
 │ Cron Cycles    │  │ Cost Tracking│  │ REEF Dashboard │
 │ (heartbeats,   │  │ (SQLite      │  │ (Electron,     │
 │  scheduled     │  │  metrics)    │  │  read-only v1) │
 │  work)         │  │              │  │                 │
 └────────────────┘  └──────────────┘  └───────────────┘

PHASE 4: Quality & Memory (Month 2)
════════════════════════════════════
 ┌────────────────┐  ┌──────────────┐  ┌───────────────┐
 │ CI/CD Pipeline │  │ ChromaDB     │  │ MCP Servers   │
 │ (GitHub Actions│  │ (vector mem, │  │ (GitHub, docs, │
 │  per product)  │  │  RAG)        │  │  filesystem)   │
 └────────────────┘  └──────────────┘  └───────────────┘

PHASE 5: Revenue (Month 2-3)
════════════════════════════
 ┌────────────────┐  ┌──────────────┐  ┌───────────────┐
 │ First Product  │  │ Stripe       │  │ Marketing     │
 │ Ships          │  │ Integration  │  │ Pipeline      │
 │ (micro-SaaS)   │  │              │  │ (Blaze/Echo)  │
 └────────────────┘  └──────────────┘  └───────────────┘

PHASE 6: Scale (Month 3+)
═════════════════════════
 ┌────────────────┐  ┌──────────────┐  ┌───────────────┐
 │ Multi-product  │  │ Self-funding │  │ Factory as     │
 │ Portfolio      │  │ Operations   │  │ Product        │
 │ Management     │  │              │  │ (license it)   │
 └────────────────┘  └──────────────┘  └───────────────┘
```

### Critical Path

```
Task Queue → Kev Routing → Pi SDK → First Scout→Rex pipeline → First shipped product → Revenue
```

Everything else is a force multiplier but not a blocker. Ship the first product before perfecting the infrastructure.

---

## 9. Research Document Cross-Reference

Every architecture decision traces to research. Here's the map:

| Decision Area | Primary Research | Supporting Research |
|---------------|-----------------|-------------------|
| Framework choice (OpenClaw + Pi SDK) | [01] Agent SDKs | [07] Factory Patterns, [12] Agent Comms |
| Model selection & routing | [02] LLM Economics | [15] Local Infra, [06] Agentic CLIs |
| Task queue design | [03] Task Management | [07] Factory Patterns |
| Testing strategy | [04] Automated Testing | [05] Browser UI Testing, [13] Code Quality |
| UI/E2E testing | [05] Browser Testing | [04] Automated Testing |
| Build agent tooling | [06] Agentic CLIs | [01] Agent SDKs |
| Architecture patterns | [07] Factory Patterns | [19] Competitive Landscape |
| Memory & knowledge | [08] Memory/Knowledge | [09] MCP Ecosystem |
| Tool integration protocol | [09] MCP Ecosystem | [01] Agent SDKs |
| Deployment platforms | [10] Deployment Infra | [15] Local Infra |
| Revenue model | [11] Revenue Generation | [20] Endgame Vision |
| Agent coordination | [12] Agent Communication | [01] Agent SDKs, [03] Task Management |
| Code quality | [13] Code Guardrails | [04] Automated Testing |
| Monitoring & cost | [14] Monitoring/Observability | [02] LLM Economics |
| Local compute | [15] Local Infrastructure | [02] LLM Economics |
| Marketing automation | [16] Marketing Automation | [11] Revenue Generation |
| Security model | [17] Security/Access | [18] Human-in-Loop |
| Approval workflows | [18] Human-in-Loop | [17] Security/Access |
| Competitive positioning | [19] Competitive Landscape | [07] Factory Patterns |
| Long-term vision | [20] Endgame Vision | [11] Revenue Generation |

---

## 10. Key Risks & Mitigations

| Risk | Impact | Mitigation |
|------|--------|------------|
| API cost blowout (runaway loops) | $$$$ | Per-task budget caps, daily spend alerts, cost pyramid enforcement |
| Single point of failure (dreamteam) | Full stop | UPS, auto-restart services, cloud fallback for critical agents |
| Model quality regression (provider update) | Bad output ships | Hawk review gate, different model for review vs build, rollback capability |
| Prompt injection via external data | Security breach | Agent outputs validated before external actions, no raw user input in system prompts |
| Over-engineering before shipping | Wasted time | Phase 1-2 is MVP. Ship first product before Phase 4+ |
| Vendor lock-in (Anthropic/OpenAI) | Price/availability risk | Multi-model routing already baked in. Local inference as fallback |

---

## 11. Success Metrics

| Timeframe | Metric | Target |
|-----------|--------|--------|
| Week 2 | First Scout→Rex automated pipeline runs | ✅ Working |
| Week 4 | First product deployed to production | ✅ Live URL |
| Month 2 | 2-3 products/week shipping rate | ✅ Consistent |
| Month 2 | Daily LLM spend | < $30/day avg |
| Month 3 | First revenue | > $0 MRR |
| Month 3 | Factory self-funding | Revenue ≥ LLM costs |
| Month 6 | Portfolio | 30+ products live |
| Month 6 | MRR | $5K-$10K |
| Year 1 | Factory-as-product | Licensable system |

---

*This document is the integration layer. For deep dives on any component, see the corresponding research doc [01]-[20] in `/research/`.*
