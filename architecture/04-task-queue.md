# 04 — Task Queue & Work Distribution

> The nervous system of the factory. How work gets created, claimed, executed, and completed — autonomously, 24/7.

---

## 1. Design Philosophy

Three principles drive every decision:

1. **Durable execution over ephemeral jobs.** Agent work is expensive and long-running. Crashing mid-task and losing 20 minutes of LLM calls is unacceptable. Every state transition persists.
2. **Pull-based claiming over push-based assignment.** Agents know their own capacity. They claim work when ready, rather than having a central scheduler guess.
3. **DAGs over flat queues.** Real work has dependencies. A flat FIFO queue is a toy. The task graph is the source of truth.

---

## 2. Task Schema

```yaml
task:
  # Identity
  id: "task_01J8K2M..."          # ULID (time-sortable, globally unique)
  idempotency_key: "pr-123-lint"  # Prevents duplicate creation
  parent_id: null                 # If spawned by another task
  dag_id: "dag_01J8K..."         # Which DAG this belongs to

  # What
  type: "code" | "research" | "review" | "test" | "deploy" | "synthesis"
  title: "Fix auth middleware CORS handling"
  spec:                           # The actual work description
    prompt: "..."
    context_refs: ["artifact:abc", "task:xyz:output"]  # Inputs from other tasks/artifacts
    constraints:
      max_cost_usd: 0.50
      max_duration_sec: 300
      required_capabilities: ["code-execution", "web-search"]
      model_preferences: ["claude-sonnet-4", "gpt-4o"]
      fallback_models: ["claude-haiku"]

  # Priority
  priority: 50                    # 0 (critical) → 100 (background)
  priority_boost_per_minute: 0.1  # Anti-starvation: priority improves with age
  deadline: "2026-02-09T12:00:00Z"  # Optional hard deadline

  # Dependencies (DAG edges)
  depends_on: ["task_01J8K1...", "task_01J8K0..."]  # Must complete before this runs
  blocked_by: []                  # System-managed: unresolved dependencies

  # Lifecycle
  status: "pending"               # See state machine below
  claim:
    agent_id: null
    claimed_at: null
    heartbeat_at: null
    heartbeat_interval_sec: 30
  attempts: 0
  max_attempts: 3

  # Retry policy
  retry:
    strategy: "exponential"       # exponential | fixed | immediate
    initial_delay_sec: 10
    backoff_multiplier: 2.0
    max_delay_sec: 300
    jitter: true
    retry_on: ["timeout", "crash", "rate_limit", "invalid_output"]
    no_retry_on: ["auth_failure", "budget_exceeded", "cancelled"]

  # Output
  output:
    type: "code" | "markdown" | "json" | "artifact"
    validation: "json-schema" | "test-pass" | "llm-judge" | "none"
    validation_schema: null       # JSON Schema if applicable
    data: null                    # Populated on completion
    artifacts: []                 # File references in artifact store

  # Metadata
  created_at: "2026-02-08T09:00:00Z"
  created_by: "agent:orchestrator" | "human:adam" | "cron:nightly-audit"
  started_at: null
  completed_at: null
  cost_usd: 0.00
  tokens_used: { input: 0, output: 0 }
  tags: ["repo:myapp", "sprint:12"]
```

---

## 3. Task Lifecycle (State Machine)

```
                    ┌─────────────────────────────────────┐
                    │                                     │
  CREATED ──→ PENDING ──→ READY ──→ CLAIMED ──→ RUNNING ──→ VALIDATING ──→ COMPLETED
                │                     │           │              │
                │                     │           │              └──→ FAILED ──→ RETRYING ──→ READY
                │                     │           │                      │
                │                     │           └── (heartbeat miss) ──┘
                │                     │                                  │
                │                     └── (claim expired) ──────────────┘
                │                                                        │
                └── CANCELLED                              DEAD_LETTERED ←┘ (max attempts)
```

**State definitions:**

| State | Meaning |
|-------|---------|
| `CREATED` | Task record exists but DAG membership not yet resolved |
| `PENDING` | In a DAG, waiting for dependencies to complete |
| `READY` | All dependencies satisfied; available for claiming |
| `CLAIMED` | An agent has claimed it but hasn't started execution |
| `RUNNING` | Agent is actively executing; must heartbeat |
| `VALIDATING` | Output produced, being validated (schema check, tests, LLM judge) |
| `COMPLETED` | Done. Output stored. Downstream tasks unblocked. |
| `FAILED` | Attempt failed. Will retry or dead-letter. |
| `RETRYING` | Waiting for retry delay to elapse before returning to READY |
| `DEAD_LETTERED` | Exhausted all retries. Requires human attention. |
| `CANCELLED` | Manually or programmatically cancelled. |

---

## 4. DAG-Based Dependency Resolution

### 4.1 DAG Structure

A DAG (Directed Acyclic Graph) groups related tasks with dependency edges. Every task belongs to exactly one DAG (standalone tasks get a single-node DAG).

```yaml
dag:
  id: "dag_01J8K..."
  title: "Build landing page for client X"
  created_by: "human:adam"
  status: "running"               # pending | running | completed | failed | cancelled
  tasks: ["task_01...", "task_02...", "task_03...", ...]
  
  # Computed at creation and on mutation
  roots: ["task_01..."]           # Tasks with no dependencies (entry points)
  leaves: ["task_05..."]          # Tasks with no dependents (exit points)
```

### 4.2 Resolution Algorithm

When any task completes:

```python
def on_task_complete(task):
    store_output(task)
    
    # Find all tasks that depend on this one
    dependents = get_dependents(task.id)
    
    for dep in dependents:
        dep.blocked_by.remove(task.id)
        if len(dep.blocked_by) == 0:
            dep.status = "READY"
            emit_event("task.ready", dep)
    
    # Check if entire DAG is done
    dag = get_dag(task.dag_id)
    if all(t.status in ["COMPLETED", "CANCELLED"] for t in dag.tasks):
        dag.status = "completed"
        emit_event("dag.completed", dag)
```

### 4.3 Dynamic DAG Expansion

Agents can spawn subtasks at runtime. This is critical — an agent doing research might discover it needs three sub-investigations it couldn't predict upfront.

```python
def spawn_subtask(parent_task, new_task_spec):
    new_task = create_task(new_task_spec)
    new_task.parent_id = parent_task.id
    new_task.dag_id = parent_task.dag_id
    
    # Validate no cycles
    if creates_cycle(parent_task.dag_id, new_task):
        raise CycleError("Would create circular dependency")
    
    # Add to DAG
    add_to_dag(parent_task.dag_id, new_task)
    
    # Parent can either:
    # a) Continue running (fire-and-forget subtask)
    # b) Pause and depend on subtask output (adds edge, parent goes PENDING)
```

### 4.4 Cycle Detection

On every edge addition, run DFS from the target to check if it can reach the source. O(V+E) per check — acceptable given DAGs are typically small (< 100 tasks).

---

## 5. Priority System

### 5.1 Priority Calculation

Effective priority is computed dynamically, not stored statically:

```python
def effective_priority(task):
    base = task.priority                              # 0-100, lower = more urgent
    age_boost = task.age_minutes * task.priority_boost_per_minute
    deadline_boost = deadline_urgency(task.deadline)   # Exponential as deadline approaches
    dependency_boost = critical_path_weight(task)       # Tasks on the longest path get boosted
    
    return base - age_boost - deadline_boost - dependency_boost  # Lower = higher priority
```

### 5.2 Priority Tiers

| Tier | Range | Use Case | Example |
|------|-------|----------|---------|
| Critical | 0-9 | Production incidents, blocking failures | "Fix crashed deployment" |
| High | 10-29 | Human-requested urgent work | "Client needs this today" |
| Normal | 30-59 | Standard work items | "Implement feature X" |
| Low | 60-79 | Background improvements | "Refactor auth module" |
| Background | 80-100 | Housekeeping, nice-to-haves | "Update dependencies" |

### 5.3 Anti-Starvation

Every task's priority improves over time via `priority_boost_per_minute`. A background task (priority 90) with a boost of 0.1/min reaches normal priority after ~5 hours. No task waits forever.

---

## 6. Claim-Based Work Distribution

### 6.1 How Agents Pick Up Work

Agents are **pull-based workers**. They poll for work matching their capabilities:

```python
def claim_next_task(agent):
    # Atomic operation — only one agent wins
    task = db.execute("""
        UPDATE tasks 
        SET status = 'CLAIMED', 
            claim_agent_id = :agent_id,
            claim_claimed_at = NOW()
        WHERE id = (
            SELECT id FROM tasks
            WHERE status = 'READY'
              AND required_capabilities <@ :agent_capabilities
              AND estimated_cost <= :agent_budget_remaining
            ORDER BY effective_priority ASC
            LIMIT 1
            FOR UPDATE SKIP LOCKED    -- PostgreSQL advisory lock; skip contested rows
        )
        RETURNING *
    """, agent_id=agent.id, 
         agent_capabilities=agent.capabilities,
         agent_budget_remaining=agent.budget_remaining)
    
    return task
```

**Key properties:**
- **Atomic claiming** via `FOR UPDATE SKIP LOCKED` — no double-claiming, no contention blocking
- **Capability matching** — agents only see work they can do
- **Budget-aware** — agents don't claim work they can't afford
- **Priority-ordered** — highest effective priority wins

### 6.2 Agent Registration

```yaml
agent:
  id: "agent_coder_01"
  type: "coder"
  capabilities: ["code-execution", "git", "testing", "typescript", "python"]
  models_available: ["claude-sonnet-4", "claude-haiku"]
  max_concurrent_tasks: 2
  current_tasks: 1
  budget_remaining_usd: 45.00
  status: "active"                # active | draining | offline
  last_heartbeat: "2026-02-08T09:15:00Z"
  poll_interval_sec: 5
  claim_ttl_sec: 60               # Must start executing within 60s of claim
```

### 6.3 Heartbeats & Liveness

While executing, agents must heartbeat:

```python
# Agent side — every heartbeat_interval_sec
def heartbeat(task_id, progress=None):
    update_task(task_id, heartbeat_at=now(), progress=progress)

# Server side — reaper process
def reap_stale_tasks():
    stale = db.query("""
        SELECT * FROM tasks 
        WHERE status IN ('CLAIMED', 'RUNNING')
          AND heartbeat_at < NOW() - INTERVAL '90 seconds'
    """)
    for task in stale:
        task.status = "FAILED"
        task.failure_reason = "heartbeat_timeout"
        task.attempts += 1
        schedule_retry_or_dead_letter(task)
```

### 6.4 Polling Strategy: Hybrid Event-Driven + Poll

Pure polling wastes resources. Pure event-driven misses edge cases. We use both:

```
Agent                                    Task Queue (PostgreSQL + NOTIFY)
  │                                              │
  │──── LISTEN task_ready ──────────────────────→│  (PostgreSQL LISTEN/NOTIFY)
  │                                              │
  │←──── NOTIFY: "new READY task" ──────────────│  (instant wake-up)
  │                                              │
  │──── claim_next_task() ─────────────────────→│  (attempt claim)
  │←──── task claimed ──────────────────────────│
  │                                              │
  │   ... also polls every 30s as fallback ...   │
  │──── claim_next_task() ─────────────────────→│  (catches missed NOTIFYs)
```

**Why hybrid:**
- `NOTIFY` gives sub-second response to new work (event-driven)
- Fallback poll catches any missed notifications (reliability)
- No external message broker needed — PostgreSQL does both

---

## 7. Retry Logic & Dead Letter Queue

### 7.1 Retry Flow

```python
def schedule_retry_or_dead_letter(task):
    if task.attempts >= task.max_attempts:
        dead_letter(task)
        return
    
    delay = calculate_retry_delay(task)
    task.status = "RETRYING"
    task.retry_at = now() + delay
    
    # Optional: switch to fallback model on retry
    if task.attempts >= 2 and task.spec.constraints.fallback_models:
        task.spec.constraints.model_preferences = task.spec.constraints.fallback_models
    
    save(task)

def calculate_retry_delay(task):
    if task.retry.strategy == "exponential":
        delay = task.retry.initial_delay_sec * (task.retry.backoff_multiplier ** (task.attempts - 1))
        delay = min(delay, task.retry.max_delay_sec)
        if task.retry.jitter:
            delay *= random.uniform(0.5, 1.5)  # ±50% jitter
        return delay
```

### 7.2 Retry Promoter

A periodic process (every 5s) promotes retrying tasks back to ready:

```python
def promote_retries():
    db.execute("""
        UPDATE tasks 
        SET status = 'READY', blocked_by = '{}'
        WHERE status = 'RETRYING' AND retry_at <= NOW()
    """)
```

### 7.3 Dead Letter Queue

Tasks that exhaust all retries land here. They never disappear — they wait for human attention.

```yaml
dead_letter_entry:
  task_id: "task_01J8K..."
  dag_id: "dag_01J8K..."
  failure_history:
    - attempt: 1
      agent: "agent_coder_01"
      error: "LLM returned invalid JSON"
      duration_sec: 45
      cost_usd: 0.12
    - attempt: 2
      agent: "agent_coder_02"
      error: "Test suite failed: 3 failures"
      duration_sec: 120
      cost_usd: 0.31
    - attempt: 3
      agent: "agent_coder_02"
      error: "Timeout after 300s"
      duration_sec: 300
      cost_usd: 0.08
  total_cost_usd: 0.51
  dead_lettered_at: "2026-02-08T10:30:00Z"
  reviewed: false
  resolution: null                # human sets: "retry" | "cancel" | "modify_and_retry"
```

**Dead letter notifications:** On dead-letter, emit an event. The notification system picks this up and alerts the human (Slack, email, dashboard).

### 7.4 Poison Pill Detection

If the same task spec fails across multiple different agents, it's likely a bad task, not a bad agent:

```python
def is_poison_pill(task):
    unique_agents_failed = len(set(h.agent for h in task.failure_history))
    return unique_agents_failed >= 2 and task.attempts >= 3
```

Poison pills skip remaining retries and go straight to dead letter with a `poison_pill: true` flag.

---

## 8. Task Creation: Where Work Comes From

### 8.1 Sources

| Source | Trigger | Example |
|--------|---------|---------|
| **Human** | Chat command, dashboard, API | "Build a landing page for X" |
| **Agent** | Subtask spawning during execution | Research agent spawns 3 sub-investigations |
| **Schedule** | Cron expression | "Audit dependencies every Monday 9am" |
| **Event** | Webhook, file change, PR opened | "New PR → run code review agent" |
| **DAG completion** | Downstream DAG triggered | "When data pipeline completes → generate report" |
| **Self-improvement** | Factory observes own friction | "Auth module has 40% failure rate → spawn refactor task" |

### 8.2 Cron-Driven vs Event-Driven

**Not an either/or — use both for different purposes:**

| | Cron-Driven | Event-Driven |
|---|---|---|
| **When** | Predictable schedules | In response to something happening |
| **Examples** | Nightly audits, weekly reports, dependency updates | PR opened, deployment failed, human request |
| **Implementation** | Cron scheduler creates tasks on schedule | Event listeners create tasks on event |
| **Latency** | Minutes (next cron tick) | Sub-second (NOTIFY) |
| **Use for** | Maintenance, monitoring, batch processing | Interactive work, CI/CD, real-time response |

### 8.3 Task Decomposition

Complex work arrives as a high-level goal. An **orchestrator agent** decomposes it into a DAG:

```
Human: "Build a landing page for client X"
                    │
            Orchestrator Agent
                    │
        ┌───────────┼───────────┐
        ▼           ▼           ▼
   [Research]   [Design]    [Implement]
   competitors  wireframe    code
        │           │           │
        └─────┬─────┘           │
              ▼                 │
         [Synthesize            │
          requirements] ────────┘
                                │
                                ▼
                           [Test & Deploy]
```

The orchestrator creates the DAG as a single atomic operation. Tasks start in `PENDING`, and the resolution algorithm handles ordering.

---

## 9. Execution Model: How Agents Actually Work

### 9.1 Agent Main Loop

```python
async def agent_loop(agent):
    register(agent)
    listen_for_notifications("task_ready")
    
    while agent.status != "offline":
        if agent.current_tasks >= agent.max_concurrent_tasks:
            await sleep(agent.poll_interval_sec)
            continue
        
        task = claim_next_task(agent)
        if task:
            asyncio.create_task(execute_task(agent, task))
        else:
            await wait_for_notification_or_timeout(agent.poll_interval_sec)

async def execute_task(agent, task):
    try:
        task.status = "RUNNING"
        task.started_at = now()
        save(task)
        
        # Start heartbeat loop
        heartbeat_handle = start_heartbeat(task)
        
        # Do the actual work (LLM calls, code execution, etc.)
        output = await agent.run(task.spec)
        
        # Validate output
        task.status = "VALIDATING"
        save(task)
        validation_result = validate(output, task.output.validation, task.output.validation_schema)
        
        if validation_result.passed:
            task.output.data = output
            task.status = "COMPLETED"
            task.completed_at = now()
            save(task)
            on_task_complete(task)  # Unblock dependents
        else:
            raise ValidationError(validation_result.errors)
            
    except Exception as e:
        task.status = "FAILED"
        task.failure_reason = str(e)
        task.attempts += 1
        save(task)
        schedule_retry_or_dead_letter(task)
    finally:
        stop_heartbeat(heartbeat_handle)
        agent.current_tasks -= 1
```

### 9.2 Output Validation

Validation is a first-class step, not an afterthought. Options:

| Method | Use Case | Example |
|--------|----------|---------|
| `json-schema` | Structured data output | API response must match schema |
| `test-pass` | Code changes | Run test suite, must pass |
| `llm-judge` | Subjective quality | "Is this summary accurate and complete?" |
| `regex` | Simple format checks | Output must contain a URL |
| `none` | Trust the agent | Research notes, brainstorming |

---

## 10. Storage & Infrastructure

### 10.1 Technology Choices

| Component | Choice | Why |
|-----------|--------|-----|
| **Task state** | PostgreSQL | ACID, `FOR UPDATE SKIP LOCKED`, `LISTEN/NOTIFY`, battle-tested |
| **Artifacts** | S3-compatible (MinIO locally) | Large files, cheap, durable |
| **Event bus** | PostgreSQL NOTIFY + optional Redis Streams | Simple, no extra infra; Redis for high-throughput |
| **Scheduling** | pg_cron or application-level | Cron expressions in the database |
| **Monitoring** | Structured logs → Loki, metrics → Prometheus | Standard observability stack |

### 10.2 Key Indexes

```sql
-- The claim query must be fast
CREATE INDEX idx_tasks_claimable ON tasks (effective_priority) 
  WHERE status = 'READY';

-- Dependency resolution
CREATE INDEX idx_tasks_dag ON tasks (dag_id, status);

-- Heartbeat reaping  
CREATE INDEX idx_tasks_heartbeat ON tasks (heartbeat_at) 
  WHERE status IN ('CLAIMED', 'RUNNING');

-- Retry promotion
CREATE INDEX idx_tasks_retry ON tasks (retry_at) 
  WHERE status = 'RETRYING';
```

### 10.3 Why Not Temporal/Inngest?

The research (03) recommends Temporal or Inngest. We diverge intentionally:

1. **Simplicity.** PostgreSQL is already in our stack. Adding Temporal adds a complex distributed system to operate.
2. **Transparency.** Every task is a row in a table. `SELECT * FROM tasks WHERE status = 'RUNNING'` — no event-sourcing replay needed to understand state.
3. **Control.** We need AI-specific primitives (budget tracking, model fallback, capability matching, poison pill detection) that don't exist in Temporal/Inngest natively.
4. **Scale.** Our initial scale is hundreds of tasks/day, not millions. PostgreSQL handles this trivially.
5. **Migration path.** If we outgrow PostgreSQL, the state machine and claim protocol can move to Temporal without rewriting agent logic.

This is a **pragmatic choice, not a permanent one.** If we hit scale or durability limits, Temporal is the upgrade path.

---

## 11. Observability & Cost Tracking

### 11.1 Metrics

```
# Queue health
task_queue_depth{status="ready", priority_tier="normal"}
task_queue_wait_seconds{quantile="p50|p95|p99"}

# Throughput
tasks_completed_total{type="code|research|review"}
tasks_failed_total{reason="timeout|crash|invalid_output|budget"}
tasks_dead_lettered_total

# Cost
task_cost_usd{type, agent, model}
daily_spend_usd
budget_remaining_usd{agent}

# Agent health
agent_utilization{agent_id}           # % time executing vs idle
agent_heartbeat_misses_total{agent_id}
agent_claim_success_rate{agent_id}
```

### 11.2 Budget Controls

Every task tracks cost. Every agent has a budget. Every DAG has a budget ceiling.

```python
def check_budget(agent, task):
    if task.spec.constraints.max_cost_usd and agent.budget_remaining_usd < task.spec.constraints.max_cost_usd:
        return False  # Don't claim work you can't afford
    
    dag = get_dag(task.dag_id)
    if dag.budget_ceiling_usd and dag.spent_usd + task.spec.constraints.max_cost_usd > dag.budget_ceiling_usd:
        return False  # DAG would exceed budget
    
    return True
```

---

## 12. Summary

The system is deliberately simple: **PostgreSQL as the task store, DAGs as the dependency model, atomic claims as the distribution mechanism, heartbeats as the liveness check.** No external orchestration engine, no message broker, no distributed consensus protocol.

This buys us:
- **One database to operate**, not three services
- **SQL-queryable state** — debugging is `SELECT`, not event-log replay  
- **Millisecond claiming** via `SKIP LOCKED` — no contention, no thundering herd
- **Sub-second dispatch** via `LISTEN/NOTIFY` — event-driven without Kafka

The factory runs 24/7 because agents run in a loop: claim → execute → heartbeat → complete → claim. No human in the loop for normal operation. Humans intervene only for dead-lettered tasks and priority overrides.

---

*Architecture document — 2026-02-08*
