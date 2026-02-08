# 09 — Internal API Surface

> **Status:** Review Draft v1 — 2026-02-08  
> **Author:** Rex (subagent)  
> **Sources:** Architecture docs 01–20  

OpenAPI-style specifications for the five core internal APIs. All APIs are TypeScript-first (direct function calls in Phase 1, HTTP/JSON in Phase 2+). Auth is via capability tokens issued by the Capability Broker (see [07]).

---

## Table of Contents

1. [Task Queue API](#1-task-queue-api)
2. [Memory API](#2-memory-api)
3. [Router API](#3-router-api)
4. [Agent Registry API](#4-agent-registry-api)
5. [Approval API](#5-approval-api)
6. [Common Types](#6-common-types)
7. [Error Handling](#7-error-handling)
8. [Transport & Versioning](#8-transport--versioning)

---

## 1. Task Queue API

Task lifecycle management. Backs the DAG engine and dependency resolver from [01] and [04].

### `POST /tasks`

Create a task. Automatically resolves dependency state (`READY` if no deps, `PENDING` otherwise). Rejects cycles.

**Request:**
```yaml
body:
  title: string                          # required
  type: TaskType                         # required — research | code | review | test | deploy | write | analyze | synthesize | decompose
  description: string                    # required — markdown spec
  projectId: string                      # required
  parentId?: string                      # parent task ID (for subtasks)
  dependsOn?: string[]                   # task IDs
  priority?: integer                     # 0 (critical) → 100 (background). Default: 50
  maxCostUsd?: number                    # Default: 5.00
  maxDurationSec?: integer               # Default: 3600
  deadline?: string                      # ISO 8601
  requiredCapabilities?: string[]        # e.g. ["code-execution", "web-search"]
  preferredAgent?: string                # agent ID hint
  modelPreference?: string[]             # ordered model list
  retryPolicy?:
    maxAttempts?: integer                # Default: 3
    backoffMs?: integer                  # Default: 5000
    backoffMultiplier?: number           # Default: 2.0
    fallbackModels?: string[]
  tags?: string[]
  createdBy: string                      # required — "kev" | "adam" | agent ID
```

**Response:** `201 Created`
```yaml
body: Task                               # full Task object (see §6)
```

**Errors:** `400 INVALID_INPUT` · `409 CYCLE_DETECTED` · `404 DEPENDENCY_NOT_FOUND`

---

### `POST /tasks/graph`

Create a DAG of tasks atomically. All-or-nothing.

**Request:**
```yaml
body:
  projectId: string                      # required
  tasks: Partial<Task>[]                 # array of task definitions (use temp IDs)
  edges: [string, string][]             # [dependsOn, task] pairs using temp IDs
  createdBy: string
```

**Response:** `201 Created`
```yaml
body:
  dagId: string
  tasks: Task[]                          # with real IDs assigned
```

**Errors:** `400 CYCLE_DETECTED` · `400 INVALID_GRAPH`

---

### `GET /tasks/{id}`

**Response:** `200 OK` → `Task`

---

### `GET /tasks`

List/filter tasks.

**Query params:**
```yaml
status?: TaskStatus                      # filter by status
projectId?: string
dagId?: string
assignedAgent?: string
createdBy?: string
tags?: string                            # comma-separated
limit?: integer                          # default 50, max 200
offset?: integer
orderBy?: "priority" | "createdAt" | "updatedAt"   # default: priority
```

**Response:** `200 OK`
```yaml
body:
  items: Task[]
  total: integer
```

---

### `POST /tasks/{id}/claim`

Atomic claim. Returns `409` if already claimed. Starts the claim timeout (default 60s to begin execution).

**Request:**
```yaml
body:
  agentId: string                        # required
```

**Response:** `200 OK` → `Task` (status: `ASSIGNED`)

**Errors:** `409 ALREADY_CLAIMED` · `403 CAPABILITY_MISMATCH` · `403 BUDGET_EXCEEDED`

---

### `POST /tasks/{id}/start`

Agent confirms execution has started. Begins heartbeat monitoring.

**Request:**
```yaml
body:
  agentId: string
  model?: string                         # actual model being used
```

**Response:** `200 OK` → `Task` (status: `RUNNING`)

---

### `POST /tasks/{id}/heartbeat`

Must be called every `heartbeatIntervalSec` (default 30s) while running.

**Request:**
```yaml
body:
  agentId: string
  progress?: string                      # free-text progress note
  costUsd?: number                       # accumulated cost so far
  tokensUsed?: integer
```

**Response:** `200 OK`

---

### `POST /tasks/{id}/complete`

Mark task complete with output artifacts.

**Request:**
```yaml
body:
  agentId: string
  artifacts: Artifact[]                  # see §6
  costUsd?: number                       # final cost
  tokensUsed?: integer
```

**Response:** `200 OK` → `Task` (status: `COMPLETED`)

**Side effects:** Dependents with all deps satisfied transition to `READY`. If DAG fully complete, emits `dag.completed` event.

---

### `POST /tasks/{id}/fail`

Report failure. System evaluates retry policy automatically.

**Request:**
```yaml
body:
  agentId: string
  error: string                          # required
  category?: "transient" | "invalid_output" | "quality" | "budget" | "permanent" | "stuck"
  costUsd?: number
  tokensUsed?: integer
```

**Response:** `200 OK` → `Task` (status: `FAILED` or `RETRYING` or `DEAD_LETTER`)

---

### `POST /tasks/{id}/cancel`

**Request:**
```yaml
body:
  reason: string
  cancelledBy: string                    # agent or human ID
  cascade?: boolean                      # cancel dependent tasks too. Default: false
```

**Response:** `200 OK` → `Task` (status: `CANCELLED`)

---

### `POST /tasks/{id}/subtasks`

Dynamic decomposition — agent spawns child tasks at runtime.

**Request:**
```yaml
body:
  subtasks: Partial<Task>[]
  edges?: [string, string][]            # internal edges among subtasks
  blockParent?: boolean                  # Default: true — parent goes PENDING until children complete
```

**Response:** `201 Created`
```yaml
body:
  tasks: Task[]
```

---

### `GET /tasks/dead-letter`

List dead-lettered tasks awaiting human attention.

**Query params:** `projectId?`, `limit?`, `offset?`

**Response:** `200 OK` → `{ items: Task[], total: integer }`

---

### `GET /projects/{projectId}/status`

DAG-level summary.

**Response:**
```yaml
body:
  projectId: string
  dagId: string
  totalTasks: integer
  byStatus: Record<TaskStatus, integer>
  totalCostUsd: number
  estimatedRemainingCostUsd: number
  criticalPath: string[]                 # task IDs on longest path
  blockers: string[]                     # currently blocked task IDs
```

---

## 2. Memory API

Unified memory interface backed by Qdrant (vector) + Neo4j (graph). Exposed as MCP tools internally; HTTP for dashboard/external access. Based on [05].

### `POST /memory/store`

Store a memory. Embedding computed server-side.

**Request:**
```yaml
body:
  content: string                        # required
  memoryType: "semantic" | "episodic" | "procedural" | "decision"   # required
  scope: string                          # required — "team" | "agent:{name}" | "project:{name}"
  agentId: string                        # required — who stored it
  tags?: string[]
  confidence?: number                    # 0.0–1.0. Default: 0.8
  source?: string                        # provenance URI
  ttlDays?: integer                      # auto-expire. null = no expiry
```

**Response:** `201 Created`
```yaml
body:
  id: string
  embedding: null                        # not returned, stored internally
  createdAt: string
```

---

### `POST /memory/recall`

Semantic search across memories.

**Request:**
```yaml
body:
  query: string                          # required — natural language
  scope?: string                         # filter by scope. Default: agent's own + team
  memoryType?: string                    # filter
  tags?: string[]                        # filter
  minConfidence?: number                 # Default: 0.3
  limit?: integer                        # Default: 10, max 50
  maxTokens?: integer                    # token budget for results. Default: 2000
```

**Response:** `200 OK`
```yaml
body:
  items:
    - id: string
      content: string
      score: number                      # cosine similarity
      memoryType: string
      scope: string
      tags: string[]
      confidence: number
      createdAt: string
      accessCount: integer
```

**Side effect:** Increments `accessCount` and updates `accessedAt` for returned memories.

---

### `POST /memory/store-decision`

First-class decision record. Stored in both vector and graph.

**Request:**
```yaml
body:
  decision: string                       # required
  rationale: string                      # required
  alternativesConsidered?: string[]
  participants: string[]                 # required — agent/human IDs
  context?: string
  scope?: string                         # Default: "team"
```

**Response:** `201 Created` → `{ id, createdAt }`

---

### `GET /memory/decisions`

**Query params:** `topic?`, `since?` (ISO 8601), `status?` ("active" | "superseded" | "reverted"), `limit?`

**Response:** `200 OK` → `{ items: Decision[] }`

---

### `POST /memory/relate`

Create a relationship between entities in the knowledge graph.

**Request:**
```yaml
body:
  fromEntity: string                     # entity name or ID
  relation: "RELATES_TO" | "DEPENDS_ON" | "DECIDED" | "SUPERSEDES" | "AUTHORED_BY" | "PART_OF" | "KNOWS_ABOUT"
  toEntity: string
  properties?: Record<string, any>
  agentId: string
```

**Response:** `201 Created`

---

### `GET /memory/entity/{name}`

Retrieve entity and its relationships.

**Query params:** `depth?` (graph traversal hops, default 1, max 3)

**Response:**
```yaml
body:
  entity:
    id: string
    name: string
    type: string
    properties: Record<string, any>
    confidence: number
  relations:
    - relation: string
      direction: "outgoing" | "incoming"
      target:
        name: string
        type: string
      properties: Record<string, any>
```

---

### `POST /memory/forget`

Soft-delete. Kept 30 days for undo, then purged.

**Request:**
```yaml
body:
  memoryId: string
  reason: string
```

**Response:** `200 OK`

---

### `POST /memory/supersede`

Replace an existing decision or fact.

**Request:**
```yaml
body:
  oldId: string
  newContent: string
  reason: string
  agentId: string
```

**Response:** `201 Created` → new memory with `SUPERSEDES` relation to old.

---

### `POST /memory/promote`

Move a memory from agent-private scope to team scope.

**Request:**
```yaml
body:
  memoryId: string
  agentId: string
```

**Response:** `200 OK`

---

## 3. Router API

Smart LLM router. Sits in front of all provider calls. Based on [02] and [08].

### `POST /router/complete`

Route an LLM completion request to the optimal provider/model.

**Request:**
```yaml
body:
  messages: Message[]                    # required — OpenAI-format messages
  agentId: string                        # required — for budget tracking
  taskId?: string                        # for cost attribution
  intent?:                               # routing hints (optional — inferred if absent)
    taskType?: "classification" | "extraction" | "summarization" | "coding" | "reasoning" | "creative" | "general"
    quality?: "best" | "good" | "acceptable"
    speed?: "instant" | "fast" | "normal" | "batch"
    privacy?: "local_only" | "any"
    maxCostUsd?: number                  # per-call cap
  model?: string                         # explicit model override (bypasses routing)
  maxTokens?: integer
  temperature?: number
  tools?: ToolDefinition[]               # function calling
  responseFormat?: "text" | "json"
  stream?: boolean                       # Default: false
```

**Response:** `200 OK`
```yaml
body:
  id: string                             # completion ID
  model: string                          # actual model used
  provider: string                       # actual provider
  choices:
    - message: Message
      finishReason: string
  usage:
    promptTokens: integer
    completionTokens: integer
    cachedTokens: integer
  costUsd: number                        # calculated cost
  latencyMs: integer
  routingDecision:
    tier: string                         # budget | mid | premium | frontier
    reason: string                       # why this model was chosen
    fallbackUsed: boolean
```

**Errors:** `402 BUDGET_EXCEEDED` · `429 RATE_LIMITED` · `503 ALL_PROVIDERS_DOWN`

---

### `GET /router/budget/{agentId}`

Check agent's remaining budget.

**Response:**
```yaml
body:
  agentId: string
  dailyLimitUsd: number
  spentTodayUsd: number
  remainingUsd: number
  perCallLimitUsd: number
  velocityLastFiveMin: number            # $/min
  velocityAlert: boolean
```

---

### `GET /router/health`

Provider health status.

**Response:**
```yaml
body:
  providers:
    - name: string                       # "anthropic" | "openai" | "google" | "groq" | etc.
      status: "healthy" | "degraded" | "down"
      latencyP50Ms: integer
      latencyP99Ms: integer
      errorRatePct: number
      circuitState: "closed" | "open" | "half-open"
      models:
        - id: string
          available: boolean
```

---

### `GET /router/metrics`

Cost and usage metrics.

**Query params:** `since?` (ISO 8601), `agentId?`, `model?`, `groupBy?` ("agent" | "model" | "provider" | "hour")

**Response:**
```yaml
body:
  totalCostUsd: number
  totalTokens: { input: integer, output: integer, cached: integer }
  totalRequests: integer
  cacheHitRate: number                   # 0.0–1.0
  breakdown: Record<string, {
    costUsd: number
    requests: integer
    tokens: { input: integer, output: integer }
    avgLatencyMs: integer
  }>
```

---

## 4. Agent Registry API

Agent discovery, status, and capability management. Based on [03] and [11].

### `POST /agents/register`

Register or update an agent. Called on agent boot.

**Request:**
```yaml
body:
  id: string                             # required — unique agent ID (e.g. "rex", "scout")
  name: string                           # required — human-readable
  role: string                           # required — primary role
  capabilities: string[]                 # required — ["code-execution", "web-search", "browser", ...]
  availableModels: string[]              # models this agent can use
  maxConcurrent: integer                 # max simultaneous tasks. Default: 1
  executionMode: "openclaw-subagent" | "pi-sdk" | "direct-api"
  costPerHourEstimate?: number
  tools: string[]                        # tool IDs this agent has access to
  skills: string[]                       # skill module IDs
  trustLevel?: "L0" | "L1" | "L2" | "L3" | "L4"
  metadata?: Record<string, any>         # framework, version, etc.
```

**Response:** `200 OK` → `AgentRegistration`

---

### `POST /agents/{id}/heartbeat`

Agent liveness signal. Must be sent every 30s.

**Request:**
```yaml
body:
  currentLoad: integer                   # active task count
  status: "online" | "busy" | "draining" | "offline"
```

**Response:** `200 OK`

---

### `GET /agents`

List all registered agents.

**Query params:** `status?`, `capability?`, `role?`

**Response:**
```yaml
body:
  items:
    - id: string
      name: string
      role: string
      capabilities: string[]
      status: "online" | "busy" | "draining" | "offline"
      currentLoad: integer
      maxConcurrent: integer
      trustLevel: string
      lastHeartbeat: string
```

---

### `GET /agents/{id}`

Full agent profile including trust levels and stats.

**Response:**
```yaml
body:
  id: string
  name: string
  role: string
  capabilities: string[]
  availableModels: string[]
  maxConcurrent: integer
  currentLoad: integer
  executionMode: string
  status: string
  trustLevel: string
  trustByDomain: Record<string, string>  # e.g. { "code_deployment": "L3", "customer_email": "L1" }
  lastHeartbeat: string
  stats:
    tasksCompleted: integer
    tasksFailed: integer
    overrideRate: number                 # human override percentage
    avgTaskDurationSec: number
    totalCostUsd: number
```

---

### `PUT /agents/{id}/status`

Update agent status (e.g. going offline for maintenance).

**Request:**
```yaml
body:
  status: "online" | "busy" | "draining" | "offline"
  reason?: string
```

**Response:** `200 OK`

---

### `GET /agents/{id}/card`

A2A-compatible Agent Card for external interop. Auto-generated from registration data.

**Response:**
```yaml
body:
  name: string
  description: string
  url: string                            # A2A endpoint
  capabilities:
    - id: string
      name: string
      description: string
  skills:
    - id: string
      name: string
      description: string
      inputSchema: object
      outputSchema: object
  authentication:
    schemes: string[]
```

---

## 5. Approval API

Human-in-the-loop gates. Based on [07] and [12].

### `POST /approvals`

Create an approval request. Blocks the requesting agent until resolved or timed out.

**Request:**
```yaml
body:
  agentId: string                        # required
  taskId: string                         # required
  tier: 1 | 2 | 3                        # required
  actionType: string                     # required — e.g. "production_deploy", "external_email", "spending"
  summary: string                        # required — human-readable one-liner
  detail?: string                        # markdown with full context
  confidence?: number                    # agent's self-assessed confidence 0.0–1.0
  riskAssessment?:
    financial: "low" | "medium" | "high"
    reputational: "low" | "medium" | "high"
    reversibility: string                # e.g. "rollback available, <5min"
  alternativesConsidered?: string[]
  diffUrl?: string                       # link to detailed diff/preview
  amountUsd?: number                     # for spending approvals
  timeoutSec?: integer                   # Default: 14400 (4h) for T2, 86400 (24h) for T3
  timeoutAction?: "deny" | "escalate"    # Default: "deny"
  channel?: "whatsapp" | "dashboard" | "auto"   # Default: "auto"
```

**Response:** `201 Created`
```yaml
body:
  id: string                             # approval request ID
  status: "pending"
  expiresAt: string                      # ISO 8601
  notificationSentVia: string            # channel used
```

---

### `POST /approvals/{id}/resolve`

Human resolves the request.

**Request:**
```yaml
body:
  decision: "approve" | "reject" | "modify" | "escalate"
  resolvedBy: string                     # human/agent ID
  reason?: string
  modifications?: string                 # instructions if decision=modify
```

**Response:** `200 OK`
```yaml
body:
  id: string
  status: "approved" | "rejected" | "modified" | "escalated"
  resolvedAt: string
  responseTimeMs: integer
```

**Side effects:**
- `approved` → unblocks requesting agent, task continues
- `rejected` → task receives `FAILED` with `reason: "approval_rejected"`
- `modified` → task spec updated per modifications, then unblocked
- `escalated` → new approval created at tier+1 or routed to phone

---

### `GET /approvals`

List pending and recent approvals.

**Query params:** `status?` ("pending" | "approved" | "rejected" | "expired"), `agentId?`, `tier?`, `since?`, `limit?`, `offset?`

**Response:** `200 OK` → `{ items: ApprovalRequest[], total: integer }`

---

### `GET /approvals/{id}`

Full approval details with audit trail.

**Response:**
```yaml
body:
  id: string
  agentId: string
  taskId: string
  tier: integer
  actionType: string
  summary: string
  detail: string
  confidence: number
  riskAssessment: object
  status: string
  createdAt: string
  expiresAt: string
  resolvedAt: string | null
  resolvedBy: string | null
  decision: string | null
  responseTimeMs: integer | null
  notifications:                         # delivery log
    - channel: string
      sentAt: string
      acknowledged: boolean
```

---

### `GET /approvals/stats`

Gate effectiveness metrics for tuning.

**Response:**
```yaml
body:
  totalPending: integer
  approvalRate: number                   # percentage — if >95%, gate may be unnecessary
  avgResponseTimeSec: number
  byTier:
    - tier: integer
      total: integer
      approved: integer
      rejected: integer
      expired: integer
      avgResponseTimeSec: number
  byActionType:
    - actionType: string
      total: integer
      approvalRate: number
```

---

## 6. Common Types

```typescript
interface Task {
  id: string;                            // UUIDv7
  parentId: string | null;
  projectId: string;
  dagId: string;
  type: TaskType;
  title: string;
  description: string;
  status: TaskStatus;
  priority: number;
  dependsOn: string[];
  requiredCapabilities: string[];
  preferredAgent: string | null;
  modelPreference: string[];
  assignedAgent: string | null;
  assignedModel: string | null;
  maxCostUsd: number;
  maxDurationSec: number;
  deadline: string | null;
  retryPolicy: RetryPolicy;
  attempt: number;
  artifacts: Artifact[];
  error: string | null;
  costUsd: number;
  tokensUsed: number;
  createdBy: string;
  tags: string[];
  createdAt: string;
  updatedAt: string;
  startedAt: string | null;
  completedAt: string | null;
}

type TaskStatus =
  | "created" | "pending" | "ready" | "assigned" | "running"
  | "validating" | "completed" | "failed" | "retrying"
  | "dead_letter" | "cancelled";

type TaskType =
  | "research" | "code" | "review" | "design" | "test"
  | "deploy" | "write" | "analyze" | "synthesize" | "decompose";

interface RetryPolicy {
  maxAttempts: number;
  backoffMs: number;
  backoffMultiplier: number;
  fallbackModels: string[];
}

interface Artifact {
  id: string;
  type: "file" | "text" | "json" | "url";
  name: string;
  path?: string;                         // filesystem path
  content?: string;                      // inline (<64KB)
  mimeType?: string;
  sizeBytes?: number;
}

interface Message {
  role: "system" | "user" | "assistant" | "tool";
  content: string;
}
```

---

## 7. Error Handling

All errors use a consistent envelope:

```yaml
status: 4xx | 5xx
body:
  error:
    code: string                         # machine-readable — e.g. "BUDGET_EXCEEDED"
    message: string                      # human-readable
    details?: Record<string, any>        # additional context
    retryable: boolean
```

### Standard Error Codes

| Code | HTTP | Meaning |
|------|------|---------|
| `INVALID_INPUT` | 400 | Malformed request body or params |
| `CYCLE_DETECTED` | 400 | Adding dependency would create a cycle |
| `INVALID_GRAPH` | 400 | DAG structure is malformed |
| `NOT_FOUND` | 404 | Resource doesn't exist |
| `ALREADY_CLAIMED` | 409 | Task already claimed by another agent |
| `INVALID_TRANSITION` | 409 | Task status doesn't allow this action |
| `CAPABILITY_MISMATCH` | 403 | Agent lacks required capabilities |
| `BUDGET_EXCEEDED` | 402 | Agent or task budget exhausted |
| `RATE_LIMITED` | 429 | Too many requests |
| `ALL_PROVIDERS_DOWN` | 503 | No LLM provider available |
| `INTERNAL_ERROR` | 500 | Unexpected server error |

---

## 8. Transport & Versioning

### Phase 1 (MVP)
- **Transport:** Direct TypeScript function calls within the same process
- **No HTTP overhead** — orchestrator, task queue, and agents share a runtime
- **Function signatures match the HTTP specs above** (easy migration later)

### Phase 2 (Multi-process)
- **Transport:** HTTP/JSON over localhost
- **Base URL:** `http://localhost:{port}/api/v1/`
- **Auth:** Capability token in `Authorization: Bearer cap-{token}` header
- **Content-Type:** `application/json`

### Phase 3 (Distributed)
- **Transport:** HTTP/JSON over NATS (request/reply) or direct HTTPS
- **Service mesh** for inter-service auth and mTLS

### Versioning
- URL-based: `/api/v1/`, `/api/v2/`
- Breaking changes increment major version
- Additive changes (new optional fields) are non-breaking

---

*Review draft — Internal API Surface for the Agentic Factory. Covers task queue, memory, router, agent registry, and approval systems.*
