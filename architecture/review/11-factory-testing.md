# 11 — Factory Self-Tests

> The factory builds products. But who tests the factory?
> If the orchestrator silently drops tasks, no amount of product-level testing saves you.

---

## The Problem

Documents 06 and 09 define rigorous testing for **agent-generated code** — the products of the factory. But the factory infrastructure itself (orchestrator, task queue, router, memory layer, agent lifecycle) is equally critical and has zero coverage in the current architecture.

A bug in the rate limiter middleware is caught by Hawk. A bug in the task router that sends work to the wrong agent? Nobody's watching.

---

## Testing Pyramid for Factory Infrastructure

```
                    ╱╲
                   ╱  ╲
                  ╱ E2E╲          Factory smoke tests
                 ╱ (few)╲         "Submit task → agent picks up → result returned"
                ╱────────╲
               ╱Contract  ╲      Component boundaries
              ╱  Tests     ╲     Queue ↔ Orchestrator, Router ↔ Agent, Memory ↔ API
             ╱──────────────╲
            ╱  Integration   ╲   Multi-component flows
           ╱   Tests          ╲  "Task queued → routed → assigned → executed → stored"
          ╱────────────────────╲
         ╱     Unit Tests       ╲  Pure logic in isolation
        ╱  (routing rules, queue ╲ Priority calc, retry logic, trust scoring,
       ╱   ordering, memory TTL)  ╲ capacity checks, dedup, rate limiting
      ╱────────────────────────────╲
```

**Ratio target:** ~60% unit, ~25% integration, ~10% contract, ~5% E2E.

---

## 1. Unit Tests — Component Internals

Each factory component has pure logic that's testable without infrastructure.

### 1.1 Orchestrator

The orchestrator assigns tasks to agents, manages lifecycle, and handles failures.

| What to Test | Example Cases |
|---|---|
| **Task assignment logic** | Assigns to least-loaded agent; respects agent capabilities; skips unhealthy agents |
| **Priority ordering** | Critical tasks preempt low-priority; equal-priority uses FIFO; starvation prevention |
| **Retry policy** | Retries on transient failure; no retry on permanent failure; exponential backoff; max-retries respected |
| **Timeout handling** | Marks task failed after timeout; reclaims agent capacity; doesn't double-assign |
| **Concurrency limits** | Rejects when at capacity; queues overflow correctly; drains gracefully on shutdown |
| **Idempotency** | Duplicate task submission detected; same task doesn't run twice concurrently |

```typescript
// Example: orchestrator unit tests
describe('TaskAssigner', () => {
  it('assigns to agent with matching capability and lowest load', () => {
    const agents = [
      { id: 'rex-1', capabilities: ['code'], load: 3 },
      { id: 'rex-2', capabilities: ['code'], load: 1 },
      { id: 'hawk-1', capabilities: ['review'], load: 0 },
    ];
    const task = { type: 'code', priority: 'normal' };
    expect(assign(task, agents)).toBe('rex-2');
  });

  it('returns null when no agent has matching capability', () => {
    const agents = [{ id: 'hawk-1', capabilities: ['review'], load: 0 }];
    const task = { type: 'code', priority: 'normal' };
    expect(assign(task, agents)).toBeNull();
  });

  it('skips unhealthy agents even if they have lowest load', () => {
    const agents = [
      { id: 'rex-1', capabilities: ['code'], load: 0, healthy: false },
      { id: 'rex-2', capabilities: ['code'], load: 5, healthy: true },
    ];
    expect(assign({ type: 'code' }, agents)).toBe('rex-2');
  });
});
```

### 1.2 Task Queue

| What to Test | Example Cases |
|---|---|
| **Enqueue/dequeue ordering** | Priority queue maintains order; FIFO within same priority |
| **Deduplication** | Same idempotency key within window → rejected |
| **TTL / expiry** | Expired tasks removed on dequeue; never dispatched |
| **Dead letter** | Failed tasks moved to DLQ after max retries; DLQ is inspectable |
| **Visibility timeout** | In-flight task invisible to other consumers; re-appears after timeout if unacked |
| **Backpressure** | Queue rejects when full; returns appropriate signal to caller |

### 1.3 Router

| What to Test | Example Cases |
|---|---|
| **Route resolution** | Task type → correct agent pool; fallback routes work |
| **Load balancing** | Round-robin / least-connections / weighted — distributes evenly |
| **Circuit breaking** | Opens after N failures; half-open probe; closes on success |
| **Rate limiting** | Per-agent, per-task-type limits enforced; burst allowed within window |
| **Routing table updates** | Hot-reload config; new agent registered → immediately routable |

### 1.4 Memory Layer

| What to Test | Example Cases |
|---|---|
| **Read/write correctness** | Store value → retrieve same value; absent key → null/default |
| **TTL enforcement** | Expired entries not returned; cleanup runs on schedule |
| **Namespace isolation** | Agent A's memory invisible to Agent B (unless shared namespace) |
| **Concurrent access** | Optimistic locking / CAS works; no lost updates |
| **Size limits** | Per-agent quotas enforced; oldest evicted under LRU |
| **Serialization roundtrip** | Complex objects survive store → retrieve without corruption |

---

## 2. Integration Tests — Multi-Component Flows

These tests run real components wired together, but with test doubles for external dependencies (LLM APIs, GitHub, etc.).

### 2.1 Test Environments

```
┌─────────────────────────────────────────────┐
│  Integration Test Environment               │
│                                             │
│  Real: Queue (Redis) + Orchestrator + Router│
│  Real: Memory store (SQLite/Postgres)       │
│  Fake: LLM API (deterministic responses)    │
│  Fake: GitHub API (in-memory)               │
│  Fake: Deploy platform (no-op)              │
│                                             │
│  Docker Compose spins up in CI              │
│  Torn down after each test suite            │
└─────────────────────────────────────────────┘
```

### 2.2 Key Integration Scenarios

**Task Lifecycle (happy path):**
```
Submit task via API
  → Verify task appears in queue
  → Orchestrator picks up and assigns to agent
  → Agent stub completes task
  → Result stored in memory
  → Task marked complete
  → Webhook/callback fired
```

**Failure & Retry:**
```
Submit task → agent fails with transient error
  → Verify task re-queued with incremented attempt count
  → Verify exponential backoff delay applied
  → Agent succeeds on retry
  → Task marked complete (not duplicate)
```

**Agent Health & Failover:**
```
Submit task → assigned agent goes unhealthy mid-task
  → Verify timeout triggers
  → Task reassigned to different agent
  → Original agent's partial work cleaned up
  → Task completes on new agent
```

**Queue Saturation:**
```
Submit 1000 tasks rapidly
  → Verify ordering preserved per priority
  → Verify no tasks lost
  → Verify backpressure signal returned when at capacity
  → Drain all tasks → queue empty, all results stored
```

**Memory Isolation:**
```
Agent A stores context for task-1
Agent B stores context for task-2
  → Agent A cannot read Agent B's context
  → Shared namespace (if configured) is readable by both
  → After task completion, ephemeral context cleaned up
```

### 2.3 Chaos / Fault Injection

These run as a separate CI stage (not on every PR — nightly or weekly):

| Fault | Expected Behavior |
|---|---|
| Kill Redis mid-pipeline | Orchestrator retries connection; in-flight tasks not lost (persisted) |
| Partition between orchestrator and agent | Task times out; reassigned; no zombie work |
| Clock skew (5min forward) | TTL-based logic still correct; no premature expiry |
| Disk full on memory store | Graceful error; no data corruption; alerts fired |
| Agent returns garbage | Validation catches it; task marked failed, not "complete with bad data" |

---

## 3. Contract Tests — Component Boundaries

Every internal interface has a contract test ensuring producer and consumer agree on shape.

### 3.1 Contracts to Enforce

```
Queue ←→ Orchestrator
  Schema: TaskMessage { id, type, priority, payload, metadata, attempt }
  Contract: enqueue() accepts TaskMessage; dequeue() returns TaskMessage | null

Orchestrator ←→ Router
  Schema: RouteRequest { task, constraints, context }
  Contract: route() returns AgentEndpoint | null

Router ←→ Agent
  Schema: AgentTask { id, type, payload, timeout, context }
  Contract: agent.execute(AgentTask) returns AgentResult { id, status, output, error? }

Agent ←→ Memory
  Schema: MemoryOp { namespace, key, value?, ttl? }
  Contract: get/set/delete with consistent types

API ←→ Queue
  Schema: TaskSubmission { type, payload, priority?, idempotencyKey? }
  Contract: submit() returns TaskHandle { id, status, estimatedStart }
```

### 3.2 Implementation

Use schema validation (Zod / JSON Schema) at every boundary. Contract tests verify:
- Producer output matches schema
- Consumer accepts schema
- Schema changes break tests (forcing explicit migration)

```typescript
// Contract: Queue → Orchestrator
describe('Queue-Orchestrator contract', () => {
  const schema = z.object({
    id: z.string().uuid(),
    type: z.enum(['code', 'review', 'test', 'deploy']),
    priority: z.number().int().min(0).max(100),
    payload: z.record(z.unknown()),
    metadata: z.object({
      submittedAt: z.string().datetime(),
      attempt: z.number().int().min(1),
      idempotencyKey: z.string().optional(),
    }),
  });

  it('queue produces valid messages', async () => {
    await queue.enqueue(testTask);
    const msg = await queue.dequeue();
    expect(() => schema.parse(msg)).not.toThrow();
  });

  it('orchestrator accepts valid messages', async () => {
    const msg = schema.parse(makeTestMessage());
    await expect(orchestrator.handle(msg)).resolves.not.toThrow();
  });
});
```

---

## 4. E2E Factory Smoke Tests

Minimal set of tests that verify the entire factory works end-to-end. Run on every deploy of the factory itself.

### 4.1 Smoke Suite

| Test | What It Proves |
|---|---|
| **Submit → Complete** | Full path works: API → Queue → Orchestrator → Agent → Memory → Response |
| **Submit → Fail → Retry → Complete** | Error handling and retry machinery works |
| **Submit → Timeout → Reassign** | Timeout detection and failover works |
| **Health endpoint** | All components report healthy |
| **Metrics endpoint** | Queue depth, active agents, task throughput all reporting |

### 4.2 Canary Task

A synthetic task that runs continuously in production:

```typescript
// Canary: runs every 5 minutes
const canaryTask = {
  type: 'canary',
  payload: { input: 'echo', expected: 'echo' },
  timeout: 30_000,
  priority: 0, // lowest — never preempts real work
};

// Canary agent: trivial implementation that echoes input
// If canary fails → alert: factory is broken
// Track canary latency over time → detect degradation
```

---

## 5. Testing the Testing (Meta-Tests)

Since Hawk (QA agent) is itself factory infrastructure, it needs self-tests too.

### 5.1 Hawk Verification

| What to Test | How |
|---|---|
| **Hawk catches known bugs** | Feed Hawk a PR with a planted bug → verify Hawk flags it |
| **Hawk doesn't false-positive** | Feed Hawk a correct PR → verify PASS verdict |
| **Hawk tests derive from spec** | Give Hawk a spec + a wrong implementation that "looks right" → verify Hawk catches spec violation |
| **Hawk mutation score is meaningful** | Run Hawk's generated tests through Stryker → verify they actually kill mutants |
| **Hawk feedback is parseable** | Verify output matches JSON schema; Rex can programmatically consume it |

### 5.2 Pipeline Self-Test

The CI/CD pipeline from doc 06 should have its own validation:

- **Config linting**: `actionlint` for GitHub Actions YAML
- **Pipeline dry-run**: Validate workflow syntax without executing
- **Rollback test**: Deploy known-good → deploy known-bad → verify auto-rollback triggers → verify known-good restored
- **Gate bypass impossible**: Attempt to merge with failing gates → verify rejection

---

## 6. Observability for Factory Health

Testing catches bugs before deploy. Observability catches them in production.

### 6.1 Key Factory Metrics

| Metric | Alert Threshold |
|---|---|
| Task queue depth | >100 pending for >5min (processing stalled) |
| Task completion rate | <90% of submitted tasks complete within SLA |
| Task latency (p95) | >2x baseline for task type |
| Agent error rate | >10% of tasks failing per agent |
| Memory store latency | >100ms p99 |
| Orchestrator decision time | >1s per assignment |
| Dead letter queue depth | >0 (any DLQ entry needs investigation) |
| Canary failure | Any failure → immediate alert |

### 6.2 Structured Logging

Every factory component emits structured logs with correlation IDs:

```json
{
  "timestamp": "2026-02-08T09:28:00Z",
  "component": "orchestrator",
  "event": "task_assigned",
  "taskId": "task-abc-123",
  "agentId": "rex-2",
  "correlationId": "corr-xyz-789",
  "latencyMs": 12,
  "queueDepth": 7
}
```

Correlation ID follows a task through: API → Queue → Orchestrator → Router → Agent → Memory → Response. One ID, full trace.

---

## 7. When Factory Tests Run

| Trigger | What Runs |
|---|---|
| **PR to factory code** | Unit + Integration + Contract + Smoke |
| **Merge to main** | Full suite + deploy canary |
| **Nightly** | Chaos/fault injection suite |
| **Weekly** | Hawk verification suite (planted bugs), full performance benchmarks |
| **Factory deploy** | Smoke suite against new deployment; canary must pass before traffic shift |

---

## 8. Gap Analysis vs Current Architecture

| Gap | Risk | Recommendation |
|---|---|---|
| Doc 09 tests product code only, not factory infra | Silent factory failures corrupt all output | This document — implement factory self-tests |
| Doc 06 pipeline has no self-test | Broken pipeline = no quality gates = everything ships | Add `actionlint` + pipeline dry-run + rollback test |
| No contract tests at internal boundaries | Component interface drift causes subtle bugs | Zod schemas at every boundary, contract test suite |
| No canary task in production | Factory degradation undetected until user reports | Deploy synthetic canary on 5-min interval |
| Hawk has no self-verification | QA agent could silently degrade, passing bad code | Planted-bug test suite run weekly |
| No chaos testing | Unknown failure modes in production | Nightly fault injection against staging |

---

## 9. Implementation Priority

### Phase 1: Foundation (Week 1)
- [ ] Unit tests for orchestrator assignment logic, queue ordering, router rules
- [ ] Zod schemas at all component boundaries
- [ ] Contract tests for Queue↔Orchestrator, Router↔Agent
- [ ] `actionlint` in CI for pipeline YAML

### Phase 2: Integration (Week 2)
- [ ] Docker Compose test environment with real Redis + fake LLM
- [ ] Integration tests: task lifecycle, failure/retry, queue saturation
- [ ] E2E smoke suite (submit → complete)
- [ ] Canary task deployment

### Phase 3: Resilience (Week 3-4)
- [ ] Chaos test suite (kill Redis, partition, disk full)
- [ ] Hawk self-verification with planted bugs
- [ ] Rollback self-test
- [ ] Performance benchmarks (throughput, latency baselines)

### Phase 4: Continuous (Ongoing)
- [ ] Canary running in production 24/7
- [ ] Nightly chaos runs
- [ ] Weekly Hawk verification
- [ ] Factory health dashboard (queue depth, task latency, agent error rates)

---

*The factory tests its products obsessively. It must test itself with equal rigor. An untested factory is a factory you hope works — and hope is not a strategy.*

---

*Review document 11. Reviewing architecture documents 06 and 09.*
