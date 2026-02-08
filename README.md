# 🏭 The Agentic Factory — Masterplan

**Date:** 2026-02-08  
**Author:** Atlas  
**Status:** Draft v1 — Ready for Adam's review

---

## TL;DR

Wire Pi SDK as the orchestration brain, use Cerebras for free fan-out work, Claude Code/Codex for heavy building, Gemini for long-context analysis. Run 24/7 via task queues + cron cycles. First autonomous pipeline: Scout→Atlas→Rex build loop running this week.

---

## 1. Architecture

### The Stack

```
┌─────────────────────────────────────────────────┐
│                  REEF (UI Layer)                 │
│          Electron app — dashboards, control      │
├─────────────────────────────────────────────────┤
│              KEV (Orchestrator)                  │
│     OpenClaw agent — routes tasks, tracks state  │
├─────────────────────────────────────────────────┤
│              PI SDK (Execution Layer)            │
│    Unified LLM API + pi-subagents + pi-shell    │
│    ┌──────┐ ┌──────┐ ┌────────┐ ┌───────────┐  │
│    │Claude│ │ Codex│ │Cerebras│ │  Gemini    │  │
│    │ Code │ │      │ │gpt-oss │ │ 2.5 Pro   │  │
│    │(build)│ │(build)│ │(fan-out)│ │(analysis) │  │
│    └──────┘ └──────┘ └────────┘ └───────────┘  │
├─────────────────────────────────────────────────┤
│              SHARED STATE                        │
│   /home/adam/agents/shared/ (filesystem)         │
│   Task queue: JSON files or SQLite               │
│   Git repos for all project code                 │
├─────────────────────────────────────────────────┤
│              LOCAL COMPUTE                       │
│   dreamteam RTX 3090 — llama.cpp, Qwen3-TTS     │
└─────────────────────────────────────────────────┘
```

### Key Decision: Two-Layer Orchestration

**Kev (OpenClaw)** = Strategic orchestrator. Decides WHAT to build, assigns work, reviews output.  
**Pi SDK** = Execution engine. Spawns Claude Code / Codex sessions, manages concurrent builds.

Why both? OpenClaw gives Kev persistent memory, WhatsApp comms, heartbeats, and the existing agent personalities. Pi gives raw execution power — spawning 5 Claude Code sessions in parallel to build different modules.

### Communication Flow

```
Adam (WhatsApp) → Kev → decides task routing
                    ↓
              Pi SDK spawns workers:
              ├── Claude Code session (Rex work)
              ├── Codex session (Forge work)  
              ├── Cerebras API call (Scout research)
              └── Gemini API call (Atlas analysis)
                    ↓
              Results → /agents/shared/ filesystem
                    ↓
              Kev reviews → reports to Adam
```

### Model Routing Rules

| Task Type | Model | Why | Cost |
|-----------|-------|-----|------|
| Research, summarization, triage | Cerebras gpt-oss-120b | Free, 1500+ tok/s | $0 |
| Code generation (new features) | Claude Code (Opus/Sonnet) | Best code quality | ~$5-20/task |
| Code generation (parallel) | Codex | Async, good for batch | ~$2-10/task |
| Long document analysis | Gemini 2.5 Pro | 1M+ context window | ~$1-5/task |
| Quick code fixes, refactors | Cerebras qwen-3-32b | Free, fast enough | $0 |
| UI/UX review | Claude (via OpenClaw Pixel) | Visual understanding | ~$1-3/task |
| Legal/compliance review | Claude (via OpenClaw Law) | Reasoning quality | ~$2-5/task |
| Local inference, TTS | RTX 3090 llama.cpp | Zero marginal cost | $0 (electricity) |

---

## 2. Agent Roles & Routing

### Current 14 Agents → Factory Roles

| Agent | Role | Primary Model | Backup | Runs On |
|-------|------|---------------|--------|---------|
| **Kev** | Orchestrator | Claude Opus | Sonnet | OpenClaw (always on) |
| **Atlas** | Strategy & planning | Claude Opus | Gemini 2.5 Pro | OpenClaw |
| **Rex** | Core builder | Claude Code | Codex | Pi SDK spawned |
| **Forge** | DevOps & infra | Codex | Claude Code | Pi SDK spawned |
| **Scout** | Research | Cerebras gpt-oss-120b | Gemini | Pi SDK / OpenClaw |
| **Hawk** | QA & security | Claude Sonnet | Cerebras | OpenClaw |
| **Pixel** | UI/UX | Claude Sonnet | — | OpenClaw |
| **Blaze** | Marketing | Cerebras → Claude | — | OpenClaw |
| **Echo** | Content/brand | Cerebras → Claude | — | OpenClaw |
| **Chase** | Sales | Claude Sonnet | — | OpenClaw |
| **Finn** | Finance | Claude Sonnet | — | OpenClaw |
| **Dash** | Analytics | Cerebras + Gemini | — | OpenClaw |
| **Dot** | Ops & admin | Claude Haiku | — | OpenClaw |
| **Law** | Legal | Claude Opus | — | OpenClaw |
| **Glim** | Local compute | RTX 3090 | — | dreamteam |

### The Cost Pyramid

```
        ╱╲
       ╱  ╲  Claude Opus — critical decisions, complex code
      ╱ $$ ╲  (~10% of tasks)
     ╱──────╲
    ╱        ╲  Claude Sonnet / Codex — standard building
   ╱   $ $    ╲  (~30% of tasks)
  ╱────────────╲
 ╱              ╲  Cerebras / Gemini / Local — research, triage, drafts
╱   FREE / CHEAP ╲  (~60% of tasks)
╱──────────────────╲
```

**Rule:** Every task starts at the bottom. Only escalate up when quality demands it.

---

## 3. 24/7 Autonomous Operation

### Task Queue System

```
/home/adam/agents/shared/queue/
├── backlog/          # Ideas and future work
├── ready/            # Approved, ready to start
├── in-progress/      # Currently being worked
├── review/           # Done, needs human/Kev review
├── done/             # Completed
└── failed/           # Failed, needs attention
```

### Cron-Driven Work Cycles

- Every 30 min: Kev checks queue, assigns work
- Every 2 hours: Scout research cycle
- Every 4 hours: Atlas strategy review
- Daily 6am: Kev morning brief
- Daily 10pm: Hawk security scan
- Weekly Monday 9am: Atlas weekly strategy

---

## 4. Tooling Factory Pipeline

```
IDEATION → RESEARCH → PRD → BUILD → TEST → DEPLOY → MARKET
 Scout      Scout     Atlas   Rex     Hawk   Forge    Blaze
 Cerebras   Cerebras  Claude  Claude  Claude Codex    Cerebras
                      Code    Code    Sonnet          →Claude
```

**Total cost per tool: ~$40-70 estimated**  
**Timeline: 2-5 days with 24/7 operation**

---

## 5. The Endgame Vision

```
Month 1: Factory operational. 1 tool/week shipping.
Month 2: Pipeline tuned. 2-3 tools/week. First revenue.
Month 3: Factory self-funding from tool revenue.
Month 6: 50+ tools in portfolio. $5K-10K MRR.
Year 1:  Factory is a product itself. License the system.
```

---

See the full masterplan document for detailed cost projections, risk mitigations, and implementation details.
