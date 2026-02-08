# 11 — Agent Communication Bus

> Architecture spec for inter-agent communication in the autonomous agentic factory.

## 1. Design Philosophy

Three principles drive every decision:

1. **Async-first** — Agents are non-deterministic and variable-latency. Synchronous RPC is the exception, not the rule.
2. **Loose coupling** — Agents know message schemas, not each other's internals. Any agent can be replaced without rewiring the system.
3. **Deterministic routing, non-deterministic content** — The bus routes messages using hard logic (topic matching, capability lookup). LLMs never decide routing.

---

## 2. Architecture Overview

```
┌─────────────────────────────────────────────────────────────┐
│                     COMMUNICATION BUS                       │
│                                                             │
│  ┌──────────┐   ┌──────────┐   ┌──────────────────────┐   │
│  │  Router   │   │  Event   │   │   Agent Registry     │   │
│  │ (NATS)    │   │  Store   │   │   (Agent Cards/A2A)  │   │
│  └──────────┘   └──────────┘   └──────────────────────┘   │
│       │              │                    │                 │
│  ┌────┴──────────────┴────────────────────┴──────────┐     │
│  │              Message Envelope Layer               │     │
│  └───────────────────────────────────────────────────┘     │
└─────────────────────────────────────────────────────────────┘
        │              │              │              │
   ┌────┴───┐    ┌────┴───┐    ┌────┴───┐    ┌────┴───┐
   │Agent A │    │Agent B │    │Agent C │    │Agent D │
   │(MCP    │    │(MCP    │    │(MCP    │    │(MCP    │
   │ tools) │    │ tools) │    │ tools) │    │ tools) │
   └────────┘    └────────┘    └────────┘    └────────┘
```

**Two protocols, two purposes:**
- **A2A** — Agent-to-agent task delegation and results. Used for structured work handoffs.
- **MCP** — Agent-to-tool access. Each agent connects to MCP servers for external capabilities.

**One broker:**
- **NATS** — Lightweight, embeddable, supports pub/sub, request/reply, and JetStream for persistence. Chosen over Kafka (too heavy) and Redis Streams (less routing flexibility).

---

## 3. Message Schema

Every message on the bus uses a single envelope format:

```typescript
interface BusMessage {
  // Identity
  id: string;                    // UUIDv7 (time-sortable)
  correlationId: string;         // Groups related messages across a workflow
  causationId: string;           // ID of the message that caused this one

  // Routing
  source: string;                // Agent ID of sender (e.g. "agent:researcher")
  topic: string;                 // NATS subject (e.g. "task.research.complete")
  replyTo?: string;              // For request/reply pattern

  // Payload
  type: MessageType;             // Enum: see below
  payload: Record<string, any>;  // Type-specific structured data
  artifacts?: Artifact[];        // Output files, data, references

  // Metadata
  timestamp: string;             // ISO 8601
  ttl: number;                   // Seconds until message expires (0 = never)
  priority: 0 | 1 | 2 | 3;      // 0=background, 1=normal, 2=high, 3=critical
  attempt: number;               // Retry count (starts at 1)
  maxAttempts: number;           // Max retries before dead-letter
  traceId: string;               // OpenTelemetry trace ID
}

enum MessageType {
  // Task lifecycle
  TASK_REQUEST    = "task.request",
  TASK_ACCEPTED   = "task.accepted",
  TASK_PROGRESS   = "task.progress",
  TASK_COMPLETE   = "task.complete",
  TASK_FAILED     = "task.failed",
  TASK_CANCELLED  = "task.cancelled",

  // Events (fire-and-forget)
  EVENT           = "event",

  // Coordination
  HANDOFF         = "handoff",
  ESCALATION      = "escalation",
  HEARTBEAT       = "heartbeat",

  // Queries
  QUERY           = "query",
  QUERY_RESPONSE  = "query.response",
}
```

### Artifact Schema

```typescript
interface Artifact {
  name: string;
  mimeType: string;
  uri?: string;          // Reference to stored artifact (e.g. file path, S3 key)
  inlineData?: string;   // Base64 for small payloads (<64KB)
  metadata?: Record<string, string>;
}
```

---

## 4. Topic Hierarchy & Routing Rules

NATS subjects use dot-delimited hierarchical topics:

```
task.<domain>.<action>
event.<domain>.<event-name>
query.<domain>.<query-name>
agent.<agent-id>.inbox          # Direct messages
system.heartbeat
system.registry.<action>
escalation.<level>              # escalation.team-lead, escalation.supervisor
```

### Examples

| Subject | Purpose |
|---------|---------|
| `task.research.request` | Request research work |
| `task.research.complete` | Research finished |
| `task.code.request` | Request code generation |
| `event.git.push` | Git push happened |
| `event.deploy.success` | Deployment succeeded |
| `agent.writer.inbox` | Direct message to writer agent |
| `escalation.supervisor` | Escalate to supervisor level |
| `system.heartbeat` | Agent liveness signals |

### Subscription Patterns

Agents subscribe using NATS wildcards:
- `task.research.*` — All research task lifecycle events
- `event.git.>` — All git events at any depth
- `escalation.>` — All escalations

### Routing Rules (Deterministic)

1. **Capability-based routing**: Task requests go to topics; agents subscribe to topics matching their capabilities. No LLM decides who gets what.
2. **Queue groups**: Multiple agents with the same capability join a NATS queue group → only one receives each message (load balancing).
3. **Fan-out**: Events publish to topics; all subscribers get a copy (broadcast).
4. **Direct addressing**: `agent.<id>.inbox` for point-to-point when the target is known.

---

## 5. Communication Patterns

### 5.1 Task Delegation (Request/Reply)

```
Orchestrator                    Researcher
    │                               │
    │── TASK_REQUEST ──────────────►│
    │   topic: task.research.request│
    │                               │
    │◄── TASK_ACCEPTED ────────────│
    │                               │
    │◄── TASK_PROGRESS (0..n) ─────│
    │                               │
    │◄── TASK_COMPLETE ────────────│
    │    (with artifacts)           │
```

- Uses NATS request/reply with JetStream for persistence.
- Requester sets `ttl` and `maxAttempts`.
- If no `TASK_ACCEPTED` within 30s → retry or escalate.

### 5.2 Event Broadcast (Pub/Sub)

```
Git Watcher                     Bus
    │                            │
    │── EVENT ──────────────────►│ topic: event.git.push
    │                            │
    │                   ┌────────┤
    │                   ▼        ▼
    │              CI Agent   Code Review Agent
```

- Fire-and-forget from publisher's perspective.
- JetStream persists events for replay/late subscribers.
- Events are **facts about things that happened**, never commands.

### 5.3 Work Handoff

A direct handoff between agents (Swarm-style but over the bus):

```typescript
{
  type: "handoff",
  source: "agent:triage",
  topic: "agent.billing.inbox",
  payload: {
    reason: "Customer billing inquiry detected",
    conversationContext: { /* relevant context snapshot */ },
    originalTaskId: "task-uuid-123"
  }
}
```

Rules:
- Handoff includes a **context snapshot** (not full history — curated by the handing-off agent).
- Receiving agent sends `TASK_ACCEPTED` or rejects with `TASK_FAILED` (reason: "not my domain").
- If rejected, original agent escalates.

### 5.4 Escalation

When an agent can't handle something:

```typescript
{
  type: "escalation",
  source: "agent:junior-coder",
  topic: "escalation.team-lead",
  payload: {
    reason: "Tests failing after 3 attempts, need architectural guidance",
    taskId: "task-uuid-456",
    attemptsSoFar: 3,
    context: { /* what was tried */ }
  },
  priority: 2
}
```

Escalation levels:
1. **Peer** → Try another agent with the same capability (automatic via queue group retry)
2. **Team Lead** → Supervisor agent for that domain
3. **Supervisor** → Top-level orchestrator
4. **Human** → Human-in-the-loop notification

---

## 6. Delivery Guarantees

| Guarantee | Mechanism |
|-----------|-----------|
| **At-least-once delivery** | NATS JetStream with explicit ack. Default for all task messages. |
| **Ordering** | Per-subject ordering within a single stream. No global ordering (not needed). |
| **Persistence** | JetStream stores messages for configurable retention (time or count based). |
| **Dead-letter queue** | Messages exceeding `maxAttempts` route to `dead-letter.<original-topic>`. Supervisor monitors DLQ. |
| **Deduplication** | JetStream message dedup window (2 min) using message ID. Consumers also enforce idempotency via task ID. |
| **Backpressure** | NATS flow control + consumer max-ack-pending limit. |

**Explicit non-guarantee:** Exactly-once delivery is not provided at the bus level. Consumers MUST be idempotent (use task ID + state machine to reject duplicate processing).

---

## 7. Agent Registry & Discovery

Agents register on startup and deregister on shutdown. Based on A2A Agent Cards:

```typescript
interface AgentRegistration {
  id: string;                    // Unique agent ID
  name: string;                  // Human-readable name
  capabilities: string[];        // List of task domains (e.g. ["research", "summarization"])
  subscriptions: string[];       // NATS subjects this agent listens on
  endpoint?: string;             // A2A endpoint URL (for cross-process agents)
  status: "online" | "busy" | "draining" | "offline";
  maxConcurrency: number;        // Max simultaneous tasks
  currentLoad: number;           // Current active tasks
  metadata: Record<string, any>; // Framework, model, version, etc.
}
```

- Published to `system.registry.register` on startup.
- Heartbeats on `system.heartbeat` every 30s with current load.
- If heartbeat missing for 90s → marked offline, tasks reassigned.
- Registry stored in NATS KV bucket for distributed access.

---

## 8. Failure Modes & Mitigations

### 8.1 Message Storms

**Risk:** An event triggers N agents, each producing M events → exponential growth.

**Mitigations:**
- **Rate limiting per agent**: Each agent has a publish rate limit (configurable, default 100 msg/s). Enforced at the bus client library level.
- **Circuit breaker**: If an agent publishes > 10x its normal rate in a 10s window, circuit trips → publishes buffered/dropped with alert.
- **Fan-out limits**: Events can set `maxFanout` in metadata. Bus enforces subscriber cap per topic.
- **No event-triggered events**: Architectural rule. Events → tasks, never events → more events. Tasks produce artifacts and completion events, but an event cannot directly trigger another event publication without going through a task.

### 8.2 Deadlocks

**Risk:** Agent A waits for B, B waits for A.

**Mitigations:**
- **Mandatory TTL**: Every `TASK_REQUEST` must have a TTL. No infinite waits. Bus client library enforces a max TTL of 1 hour.
- **DAG enforcement**: The supervisor maintains a dependency graph. Circular task chains are rejected at submission time.
- **Deadlock detector**: Background process monitors the wait-for graph (which agents are blocked on which tasks). Cycles detected → oldest waiter's task is cancelled with `TASK_FAILED(reason: "deadlock_broken")`.

### 8.3 Lost Messages

**Mitigations:**
- **JetStream persistence**: Messages survive broker restarts.
- **Explicit ack**: Consumer must ack within ack-wait window (default 60s) or message is redelivered.
- **Dead-letter queue**: After `maxAttempts`, message goes to DLQ rather than being dropped.
- **Audit log**: All messages written to append-only log for debugging/replay.

### 8.4 Poison Messages

**Risk:** A malformed message crashes every consumer that tries to process it.

**Mitigation:** After 3 nack/redeliveries of the same message ID, auto-route to DLQ. Never redeliver indefinitely.

### 8.5 Agent Crash Mid-Task

**Mitigation:**
- Heartbeat timeout → task lease expires → task returned to queue.
- Agent state machine: tasks are `claimed` → `in-progress` → `complete/failed`. If agent dies at `in-progress`, lease expiry returns task to `pending`.
- Saga pattern for multi-step workflows: each step has a compensating action logged before execution.

---

## 9. Shared State

Agents should prefer message passing over shared state. When shared state is needed:

| Use Case | Mechanism |
|----------|-----------|
| **Agent registry** | NATS KV bucket (distributed, consistent) |
| **Task state machines** | NATS KV bucket (per-task key with state + version) |
| **Workflow context** | Passed in messages (context snapshot), not shared store |
| **Large artifacts** | Local filesystem or object store; pass URI in messages |
| **Configuration** | NATS KV bucket, agents watch for changes |

**Anti-pattern:** Agents reading/writing a shared database to coordinate. This leads to coupling, race conditions, and invisible dependencies. Use messages.

---

## 10. Observability

Every message carries a `traceId` (OpenTelemetry). The bus provides:

- **Distributed tracing**: Follow a workflow from initial request through every agent hop via correlation/causation IDs.
- **Metrics** (emitted to Prometheus):
  - `bus_messages_total` (by topic, type, source)
  - `bus_message_latency_seconds` (publish to ack)
  - `bus_consumer_lag` (pending messages per consumer)
  - `bus_dlq_depth` (dead-letter queue size — alert if > 0)
  - `agent_active_tasks` (per agent)
  - `agent_task_duration_seconds` (by agent and task type)
- **Audit log**: Every message persisted to append-only store. Enables replay and forensic debugging.

---

## 11. Implementation Plan

### Phase 1: Foundation
- Deploy NATS with JetStream (single node, Docker)
- Implement bus client library (TypeScript) with envelope schema, publish/subscribe, request/reply
- Agent registry in NATS KV
- Basic supervisor that monitors heartbeats and DLQ

### Phase 2: Patterns
- Task delegation with full lifecycle (request → accepted → progress → complete/failed)
- Handoff and escalation flows
- Rate limiting and circuit breakers in client library
- Dead-letter queue processing

### Phase 3: Resilience
- Deadlock detector
- Saga orchestrator for multi-step workflows
- NATS cluster (3 nodes) for HA
- OpenTelemetry integration

### Phase 4: A2A Bridge
- A2A protocol adapter: expose internal agents as A2A endpoints for external interop
- Consume external A2A agents as bus participants
- Agent Card generation from registry

---

## 12. Decision Log

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Broker | NATS + JetStream | Lightweight, embeddable, supports all patterns (pub/sub, queue groups, KV, request/reply). Single dependency. |
| Inter-agent protocol | Custom envelope over NATS (A2A for external) | A2A is too heavy for internal high-frequency messaging. Use A2A at the boundary for interop. |
| Routing | Deterministic topic-based | LLM-driven routing is unpredictable. Agents declare capabilities; topics match capabilities. |
| Delivery | At-least-once + idempotent consumers | Exactly-once is a distributed systems myth. Idempotency is simpler and more honest. |
| Shared state | Minimized; NATS KV where needed | Message passing > shared state for agent coordination. Reduces coupling. |
| Serialization | JSON | Human-readable, debuggable, good enough perf for our scale. Protobuf if we hit serialization bottlenecks. |
| Ordering | Per-subject only | Global ordering is expensive and unnecessary. Causal ordering via causationId chains. |

---

*Architecture spec — 2026-02-08*
