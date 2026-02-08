# 01 — Orchestration Layer Architecture

> **Status:** Draft v1 — 2026-02-08  
> **Author:** Atlas  
> **Depends on:** Research 01 (Agent SDKs), 03 (Task Management), 07 (Factory Patterns), 12 (Agent Communication)

---

## TL;DR

The orchestration layer is a **two-tier system**: Kev (OpenClaw) as the strategic brain making decisions about *what* to do, and a **custom lightweight orchestrator** built on top of the filesystem + SQLite for task state, with Inngest-style step-level durability bolted on when we need it. No Temporal — it's overengineered for our scale. No pure filesystem queues — they lack atomicity. The sweet spot is a custom DAG executor backed by SQLite, with the option to migrate to Inngest/Temporal later if scale demands it.

---

## 1. Why Not Temporal? Why Not Inngest? Why Custom?

### Decision Matrix

| Factor | Temporal | Inngest | Custom (SQLite + Workers) |
|--------|----------|---------|--------------------------|
| **Operational complexity** | High (cluster, 4+ services) | Low (managed) or Medium (self-hosted) | Minimal (single binary + SQLite) |
| **Learning curve** | Steep (determinism rules, replay) | Moderate | Low (we define it) |
| **Cost** | Free (self-hosted) but ops-heavy | Free tier → paid | Free |
| **Fit for OpenClaw agents** | Poor — agents aren't Temporal workers | Moderate — event-driven fits | Best — we shape it to our runtime |
| **Durable execution** | Excellent | Excellent | Good enough (checkpoint + retry) |
| **Time to first pipeline** | Weeks | Days | Days |
| **Migration path** | N/A | Can adopt later | Can wrap with Inngest/Temporal later |

### The Decision: **Custom, with escape hatches**

**Rationale:**
1. Our "workers" are OpenClaw agents and Pi SDK sessions — not traditional Temporal workers polling task queues. Temporal's worker model assumes you control the execution runtime. We don't — OpenClaw does.
2. Kev already runs 24/7 via OpenClaw heartbeats and cron. We don't need another always-on orchestration service.
3. SQLite gives us ACID transactions, atomic task claiming, and zero operational overhead. It runs on dreamteam alongside everything else.
4. If we outgrow this (>1000 concurrent tasks, multi-machine), we migrate the state layer to Inngest or Temporal. The task schema and API surface are designed to be portable.

---

## 2. Architecture Overview

```
┌──────────────────────────────────────────────────────────────┐
│                     TASK INTAKE LAYER                         │
│                                                              │
│  WhatsApp ──→ ┐                                              │
│  Cron jobs ──→ ├──→  Kev (Strategic Brain)                   │
│  Agent ideas → ┤     - Decides WHAT to build                 │
│  Webhooks ───→ ┘     - Decomposes into task DAGs             │
│                      - Sets priorities and budgets            │
└──────────────────────────┬───────────────────────────────────┘
                           │ Creates tasks via Orchestrator API
                           ▼
┌──────────────────────────────────────────────────────────────┐
│                   ORCHESTRATOR CORE                           │
│                                                              │
│  ┌────────────┐  ┌──────────────┐  ┌─────────────────────┐  │
│  │ Task Store │  │  DAG Engine  │  │   Scheduler/Router  │  │
│  │ (SQLite)   │  │  (resolver)  │  │   (capability match)│  │
│  └─────┬──────┘  └──────┬───────┘  └──────────┬──────────┘  │
│        │                │                      │             │
│        └────────────────┼──────────────────────┘             │
│                         │                                    │
│              ┌──────────▼──────────┐                         │
│              │   Dispatch Engine   │                         │
│              │  (assigns to agents)│                         │
│              └──────────┬──────────┘                         │
└─────────────────────────┼────────────────────────────────────┘
                          │
            ┌─────────────┼─────────────┐
            ▼             ▼             ▼
     ┌───────────┐ ┌───────────┐ ┌───────────┐
     │  OpenClaw  │ │  Pi SDK   │ │  Direct   │
     │  Subagent  │ │  Session  │ │  API Call │
     │ (Rex,Scout)│ │(Claude,   │ │(Cerebras, │
     │           │ │ Codex)    │ │ Gemini)   │
     └─────┬─────┘ └─────┬─────┘ └─────┬─────┘
           │              │              │
           └──────────────┼──────────────┘
                          ▼
                ┌──────────────────┐
                │  Result Handler  │
                │  - Validate      │
                │  - Store artifact│
                │  - Trigger next  │
                └──────────────────┘
```

---

## 3. Task Lifecycle & State Machine

### States

```
                        ┌──────────────────┐
                        │     CREATED       │
                        │ (task registered) │
                        └────────┬─────────┘
                                 │ dependencies resolved?
                        ┌────────▼─────────┐
                  no ←──│     BLOCKED       │──→ yes
                  │     │ (waiting on deps) │      │
                  │     └──────────────────┘      │
                  │              ▲                  │
                  └──────────────┘                  │
                        ┌────────▼─────────┐
                        │      READY        │
                        │ (queued, routable)│
                        └────────┬─────────┘
                                 │ agent assigned
                        ┌────────▼─────────┐
                        │    ASSIGNED       │
                        │ (claimed by agent)│
                        └────────┬─────────┘
                                 │ execution started
                        ┌────────▼─────────┐
                        │     RUNNING       │◄──── retry
                        │ (agent executing) │        │
                        └───┬─────────┬────┘        │
                            │         │              │
                   success  │         │ failure      │
                            │         │              │
                   ┌────────▼──┐  ┌───▼────────┐    │
                   │ COMPLETED │  │   FAILED    │────┘
                   │           │  │ (retryable?)│  max retries
                   └─────┬─────┘  └──────┬─────┘  exceeded
                         │               │            │
                   ┌─────▼─────┐   ┌─────▼──────┐   │
                   │ VALIDATED  │   │DEAD_LETTER │◄──┘
                   │(output OK) │   │(needs human)│
                   └─────┬─────┘   └────────────┘
                         │
                   ┌─────▼─────┐
                   │ DELIVERED  │
                   │(downstream │
                   │ notified)  │
                   └───────────┘
```

### State Transitions

| From | To | Trigger | Side Effects |
|------|----|---------|--------------|
| CREATED | BLOCKED | Has unresolved dependencies | Subscribe to dependency completion events |
| CREATED | READY | No dependencies (or all resolved) | Enters priority queue |
| BLOCKED | READY | All dependencies completed | Enters priority queue |
| READY | ASSIGNED | Router selects agent | Lease timer starts, agent notified |
| ASSIGNED | RUNNING | Agent confirms execution start | Heartbeat monitoring begins |
| RUNNING | COMPLETED | Agent returns result | Artifacts stored, dependents notified |
| RUNNING | FAILED | Error / timeout / heartbeat miss | Retry policy evaluated |
| FAILED | RUNNING | Retry (within policy) | Attempt counter incremented, possible model fallback |
| FAILED | DEAD_LETTER | Max retries exceeded | Human notification sent |
| COMPLETED | VALIDATED | Output passes validation | — |
| COMPLETED | FAILED | Output fails validation | Treated as retryable failure |
| VALIDATED | DELIVERED | Downstream tasks unblocked | Trigger dependent task resolution |

### Timeouts

| Timeout | Default | Purpose |
|---------|---------|---------|
| **Queue timeout** | 30 min | Max time in READY before escalation |
| **Claim timeout** | 5 min | Max time in ASSIGNED before reassignment |
| **Execution timeout** | Varies by type (5min–4hr) | Max time in RUNNING |
| **Heartbeat interval** | 60 sec | Agent must report alive |
| **Total task timeout** | 24 hr | Max wall-clock time including retries |

---

## 4. Task Schema

```typescript
interface Task {
  // Identity
  id: string;              // UUIDv7 (time-sortable)
  parentId?: string;        // Parent task (for decomposition)
  projectId: string;        // Grouping (e.g., "build-saas-tool-x")
  
  // Definition
  type: TaskType;           // "research" | "code" | "review" | "design" | "test" | "deploy" | "write" | "analyze"
  title: string;            // Human-readable summary
  description: string;      // Full task specification (markdown)
  
  // Dependencies
  dependsOn: string[];      // Task IDs that must complete first
  
  // Routing
  requiredCapabilities: string[];  // ["web-search", "code-execution", "browser"]
  preferredAgent?: string;         // "rex" | "scout" | "forge" | etc.
  modelPreference?: string[];      // ["claude-opus", "claude-sonnet", "cerebras"]
  
  // Constraints
  priority: number;         // 0 (critical) → 100 (backlog)
  maxCostUsd: number;       // Budget ceiling for this task
  maxDurationSec: number;   // Execution timeout
  deadline?: string;        // ISO 8601 — hard deadline
  
  // Retry Policy
  retryPolicy: {
    maxAttempts: number;       // Default: 3
    backoffMs: number;         // Initial backoff (default: 5000)
    backoffMultiplier: number; // Exponential multiplier (default: 2)
    fallbackModels?: string[]; // Try different models on retry
  };
  
  // State
  status: TaskStatus;
  assignedAgent?: string;
  assignedModel?: string;
  attempt: number;
  
  // Results
  artifacts: Artifact[];
  error?: string;
  
  // Tracking
  costUsd: number;          // Accumulated cost
  tokensUsed: number;       // Total tokens consumed
  createdAt: string;
  updatedAt: string;
  startedAt?: string;
  completedAt?: string;
  
  // Metadata
  createdBy: string;        // "kev" | "adam" | "scout" | etc.
  tags: string[];
}

interface Artifact {
  id: string;
  type: "file" | "text" | "json" | "url";
  path?: string;            // Filesystem path (for files)
  content?: string;         // Inline content (for text/json)
  mimeType?: string;
  sizeBytes?: number;
}

type TaskStatus = "created" | "blocked" | "ready" | "assigned" | "running" 
                | "completed" | "validated" | "delivered" | "failed" | "dead_letter" | "cancelled";

type TaskType = "research" | "code" | "review" | "design" | "test" | "deploy" 
              | "write" | "analyze" | "synthesize" | "decompose";
```

---

## 5. DAG Engine

### Task Decomposition

When Kev receives a high-level objective (e.g., "Build a URL shortener SaaS tool"), the decomposition follows a standard pattern:

```
"Build URL Shortener" (project)
│
├── T1: Research [Scout]
│   ├── T1.1: Analyze competitors
│   ├── T1.2: Identify tech stack
│   └── T1.3: Summarize findings
│
├── T2: PRD [Atlas] ──── depends_on: [T1]
│   └── T2.1: Write product requirements doc
│
├── T3: Build [Rex] ──── depends_on: [T2]
│   ├── T3.1: Scaffold project
│   ├── T3.2: Implement core API ── depends_on: [T3.1]
│   ├── T3.3: Build frontend ────── depends_on: [T3.1]
│   └── T3.4: Integration ────────  depends_on: [T3.2, T3.3]
│
├── T4: Test [Hawk] ──── depends_on: [T3]
│
├── T5: Deploy [Forge] ── depends_on: [T4]
│
└── T6: Market [Blaze] ── depends_on: [T5]
```

### Resolution Algorithm

Standard topological sort with dynamic expansion:

1. On task creation, compute `in_degree` (number of unresolved dependencies)
2. Tasks with `in_degree = 0` → status `READY`
3. When a task completes, decrement `in_degree` for all dependents
4. Newly zero-degree dependents → `READY`
5. **Dynamic expansion**: A running task can create child tasks. Parent moves to `BLOCKED` until children complete. This is how agents decompose work at runtime.

### Cycle Detection

Enforce at task creation time. Before adding `dependsOn` edges, run DFS from the new task's dependents back through the graph. If the new task is reachable, reject with `CYCLE_DETECTED` error.

---

## 6. Router & Dispatcher

### Routing Algorithm

When a task enters `READY`, the router scores available agents:

```
score(agent, task) = 
    capability_match(agent, task)     * 0.4   // Can the agent do this?
    + model_match(agent, task)        * 0.2   // Does it have the preferred model?
    + cost_efficiency(agent, task)    * 0.2   // Cheapest viable option?
    + load_balance(agent)             * 0.1   // How busy is the agent?
    + affinity(agent, task)           * 0.1   // Has it done similar work recently?
```

- **capability_match**: Binary (0 or 1) — agent must have ALL required capabilities
- **model_match**: 1.0 if preferred model available, 0.5 if backup, 0.0 if neither
- **cost_efficiency**: Normalized inverse of estimated cost
- **load_balance**: Inverse of current task count
- **affinity**: Bonus for agents that recently worked on the same project (context reuse)

### Agent Registry

```typescript
interface AgentRegistration {
  id: string;                    // "rex", "scout", "forge"
  capabilities: string[];       // ["code-execution", "web-search", "browser"]
  availableModels: string[];    // ["claude-opus", "claude-sonnet"]
  maxConcurrent: number;        // How many tasks can run simultaneously
  currentLoad: number;          // Current active tasks
  costPerHour?: number;         // Estimated cost rate
  executionMode: "openclaw-subagent" | "pi-sdk" | "direct-api";
  status: "online" | "busy" | "offline";
  lastHeartbeat: string;
}
```

### Dispatch Modes

| Mode | When | How |
|------|------|-----|
| **OpenClaw Subagent** | Task needs agent personality, memory, tool access | Kev spawns subagent via OpenClaw |
| **Pi SDK Session** | Task needs raw LLM execution, code generation | Pi SDK spawns Claude Code / Codex session |
| **Direct API** | Simple inference, research, summarization | Direct Cerebras/Gemini API call |

---

## 7. Failure Handling

### Retry Cascade

```
Attempt 1: Original model (e.g., Claude Sonnet)
    ↓ fail
Attempt 2: Same model, adjusted prompt (add error context)
    ↓ fail  
Attempt 3: Fallback model (e.g., Claude Opus — escalate quality)
    ↓ fail
→ DEAD_LETTER: Notify Kev → Kev notifies Adam if critical
```

### Failure Categories

| Category | Retryable? | Action |
|----------|-----------|--------|
| **Transient** (API timeout, 503, rate limit) | Yes | Exponential backoff |
| **Output invalid** (bad JSON, missing fields) | Yes | Retry with stricter prompt |
| **Quality too low** (LLM-as-judge score < threshold) | Yes | Escalate to better model |
| **Budget exceeded** | No | Dead letter, notify |
| **Permanent** (auth failure, missing capability) | No | Dead letter, notify |
| **Stuck** (heartbeat timeout) | Yes | Kill and reassign to different agent |

### Circuit Breaker

Per-model circuit breaker. If a model fails 5 times in 10 minutes:
1. Open circuit — stop sending tasks to that model
2. After 5 minutes, half-open — send one probe task
3. If probe succeeds, close circuit. If fails, reopen.

This prevents burning budget when an API is having an outage.

---

## 8. API Surface

The orchestrator exposes a simple internal API (TypeScript functions initially, HTTP later if needed):

### Task Management

```typescript
// Create a new task
createTask(task: Partial<Task>): Task

// Create a DAG of tasks (project decomposition)
createTaskGraph(project: string, tasks: Partial<Task>[], edges: [string, string][]): Task[]

// Query tasks
getTask(id: string): Task
listTasks(filter: { status?: TaskStatus, projectId?: string, assignedAgent?: string }): Task[]

// Update task state (used by agents reporting results)
completeTask(id: string, artifacts: Artifact[]): Task
failTask(id: string, error: string): Task
cancelTask(id: string, reason: string): Task

// Dynamic decomposition (agent creates subtasks at runtime)
addSubtasks(parentId: string, subtasks: Partial<Task>[], edges?: [string, string][]): Task[]
```

### Agent Management

```typescript
// Register/update agent availability
registerAgent(agent: AgentRegistration): void
heartbeat(agentId: string): void
setAgentStatus(agentId: string, status: "online" | "busy" | "offline"): void
```

### Monitoring

```typescript
// Dashboard data
getProjectStatus(projectId: string): ProjectSummary
getFactoryMetrics(): { 
  activeTasks: number,
  queueDepth: number, 
  completedToday: number, 
  failedToday: number,
  costToday: number,
  agentUtilization: Record<string, number>
}
getCostReport(since: string): CostBreakdown
```

---

## 9. Storage Layer

### SQLite Schema (Core Tables)

```sql
CREATE TABLE tasks (
    id TEXT PRIMARY KEY,
    parent_id TEXT REFERENCES tasks(id),
    project_id TEXT NOT NULL,
    type TEXT NOT NULL,
    title TEXT NOT NULL,
    description TEXT NOT NULL,
    status TEXT NOT NULL DEFAULT 'created',
    priority INTEGER NOT NULL DEFAULT 50,
    
    -- Routing
    required_capabilities TEXT,  -- JSON array
    preferred_agent TEXT,
    model_preference TEXT,       -- JSON array
    assigned_agent TEXT,
    assigned_model TEXT,
    
    -- Constraints
    max_cost_usd REAL DEFAULT 5.0,
    max_duration_sec INTEGER DEFAULT 3600,
    deadline TEXT,
    
    -- Retry
    retry_max_attempts INTEGER DEFAULT 3,
    retry_backoff_ms INTEGER DEFAULT 5000,
    retry_backoff_multiplier REAL DEFAULT 2.0,
    retry_fallback_models TEXT, -- JSON array
    attempt INTEGER DEFAULT 0,
    
    -- Results
    artifacts TEXT,             -- JSON array
    error TEXT,
    
    -- Tracking
    cost_usd REAL DEFAULT 0,
    tokens_used INTEGER DEFAULT 0,
    created_by TEXT NOT NULL,
    tags TEXT,                  -- JSON array
    
    -- Timestamps
    created_at TEXT DEFAULT (datetime('now')),
    updated_at TEXT DEFAULT (datetime('now')),
    started_at TEXT,
    completed_at TEXT
);

CREATE TABLE task_dependencies (
    task_id TEXT NOT NULL REFERENCES tasks(id),
    depends_on TEXT NOT NULL REFERENCES tasks(id),
    PRIMARY KEY (task_id, depends_on)
);

CREATE TABLE task_events (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    task_id TEXT NOT NULL REFERENCES tasks(id),
    event_type TEXT NOT NULL,   -- "status_change", "heartbeat", "cost_update", "log"
    old_value TEXT,
    new_value TEXT,
    metadata TEXT,              -- JSON
    created_at TEXT DEFAULT (datetime('now'))
);

CREATE TABLE agents (
    id TEXT PRIMARY KEY,
    capabilities TEXT NOT NULL,  -- JSON array
    available_models TEXT,       -- JSON array
    max_concurrent INTEGER DEFAULT 1,
    current_load INTEGER DEFAULT 0,
    execution_mode TEXT NOT NULL,
    status TEXT DEFAULT 'offline',
    last_heartbeat TEXT
);

-- Indexes
CREATE INDEX idx_tasks_status ON tasks(status);
CREATE INDEX idx_tasks_project ON tasks(project_id);
CREATE INDEX idx_tasks_priority ON tasks(status, priority) WHERE status = 'ready';
CREATE INDEX idx_events_task ON task_events(task_id);
```

### File Storage

Artifacts are stored on the filesystem:

```
/home/adam/agents/shared/
├── artifacts/
│   └── {project_id}/
│       └── {task_id}/
│           ├── output.md
│           ├── code/
│           └── metadata.json
├── orchestrator.db          # SQLite database
└── orchestrator.log         # Structured JSON log
```

---

## 10. Kev's Role: The Strategic Brain

Kev is NOT the orchestrator engine — Kev is the **decision-maker** that uses the orchestrator.

### What Kev Does

| Responsibility | How |
|---------------|-----|
| **Intake** | Receives requests from Adam (WhatsApp), other agents, or scheduled triggers |
| **Decomposition** | Breaks high-level objectives into task DAGs using `createTaskGraph()` |
| **Prioritization** | Sets priority based on business value, urgency, cost |
| **Budget allocation** | Sets `maxCostUsd` per task based on overall budget |
| **Quality review** | Reviews completed tasks, decides accept/reject/redo |
| **Escalation** | Handles dead-letter tasks, notifies Adam when needed |
| **Strategy** | Decides what projects to pursue (informed by Atlas analysis) |

### What the Orchestrator Engine Does (Automatically)

| Responsibility | How |
|---------------|-----|
| **Dependency resolution** | Topological sort, automatic unblocking |
| **Routing** | Score agents, assign tasks to best match |
| **Dispatch** | Spawn subagents, Pi SDK sessions, or API calls |
| **Monitoring** | Track heartbeats, detect timeouts, reassign stuck tasks |
| **Retry** | Execute retry policy automatically |
| **Cost tracking** | Accumulate per-task and per-project costs |

### Interaction Pattern

```
Kev's Cron (every 30 min):
  1. Check for new requests (WhatsApp, etc.)
  2. Review completed tasks in "review" status
  3. Check dead letter queue
  4. Look at factory metrics — anything stuck?
  5. Optionally: trigger new work from backlog
```

---

## 11. Scaling Path

### Phase 1: MVP (Now)

- SQLite on dreamteam
- Kev polls via heartbeat/cron
- 1-3 concurrent agent tasks
- Manual decomposition (Kev writes the DAG)

### Phase 2: Automated (Month 2)

- Auto-decomposition (Kev uses LLM to break down tasks)
- 5-10 concurrent tasks
- Basic dashboard (Reef integration)
- Cost alerting

### Phase 3: Scale (Month 3+)

- Migrate to PostgreSQL if SQLite hits limits
- Consider Inngest for step-level durability on critical pipelines
- A2A protocol for cross-agent communication
- Self-improvement loop (Factory-style Signals)

### Migration Strategy

The API surface is stable across phases. `createTask()`, `completeTask()`, `getTask()` don't change. Only the backing store and dispatch mechanism evolve. Agents never talk to the database directly — they go through the API.

---

## 12. Open Questions

1. **LLM-as-judge for validation**: Should every task output be validated by a secondary LLM call? Cost vs quality tradeoff. Recommendation: only for tasks above a cost threshold ($1+).

2. **Inter-agent context passing**: When Scout researches and Rex builds, how much context transfers? Full artifact? Summary? Recommendation: artifacts on filesystem, summary in task description for the dependent task.

3. **Human-in-the-loop granularity**: Should Adam approve every task DAG, or only projects above a cost threshold? Recommendation: auto-approve tasks under $5, require approval for projects over $20.

4. **Concurrency limits per model**: How many simultaneous Claude Opus calls can we afford? Need to set per-model concurrency caps in the router.

5. **Event bus**: Do we need Redis/NATS for real-time events now, or is SQLite polling sufficient for Phase 1? Recommendation: polling is fine for <10 concurrent tasks.

---

## 13. Implementation Plan

| Step | What | Effort | Priority |
|------|------|--------|----------|
| 1 | SQLite schema + basic CRUD API | 2 hours | P0 |
| 2 | Task state machine (transitions + validation) | 3 hours | P0 |
| 3 | DAG dependency resolver | 2 hours | P0 |
| 4 | Simple router (capability match only) | 1 hour | P0 |
| 5 | Kev integration (create/review tasks via cron) | 3 hours | P0 |
| 6 | OpenClaw subagent dispatch | 2 hours | P1 |
| 7 | Pi SDK session dispatch | 3 hours | P1 |
| 8 | Retry engine + circuit breaker | 2 hours | P1 |
| 9 | Cost tracking + budget enforcement | 1 hour | P1 |
| 10 | Reef dashboard integration | 4 hours | P2 |
| 11 | Auto-decomposition (LLM-powered) | 4 hours | P2 |
| 12 | LLM-as-judge validation | 2 hours | P2 |

**Total MVP (Steps 1-5): ~11 hours**  
**Full Phase 1 (Steps 1-9): ~19 hours**

---

*This document defines the orchestration layer. See [02-agent-runtime.md] for how individual agents execute within this framework, and [03-state-and-memory.md] for the shared state architecture.*
