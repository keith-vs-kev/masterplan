# 14 — MCP Integration Layer

> Architecture doc — 2026-02-08

## Overview

MCP (Model Context Protocol) is the standard interface for connecting agents to external tools and data. This document defines how the factory discovers, connects to, authorises, and manages MCP servers — both community/third-party servers for external services and custom servers for internal factory operations.

**Key principle:** MCP handles agent-to-tool. A2A handles agent-to-agent. They're complementary layers. This doc covers the MCP side.

---

## 1. Architecture

```
┌──────────────────────────────────────────────────────┐
│                    Factory Agents                      │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐              │
│  │ Agent A   │ │ Agent B   │ │ Agent C   │  ...        │
│  └────┬─────┘ └────┬─────┘ └────┬─────┘              │
│       │             │            │                     │
│  ┌────▼─────────────▼────────────▼─────┐              │
│  │         MCP Client Gateway          │              │
│  │  ┌───────────┐ ┌─────────────────┐  │              │
│  │  │ Registry  │ │ Auth / Policy   │  │              │
│  │  │ Resolver  │ │ Enforcer        │  │              │
│  │  └───────────┘ └─────────────────┘  │              │
│  └────┬──────┬──────┬──────┬──────┬───┘              │
│       │      │      │      │      │                   │
└───────┼──────┼──────┼──────┼──────┼───────────────────┘
        │      │      │      │      │
   ┌────▼──┐ ┌▼────┐ ┌▼────┐ ┌▼───┐ ┌▼──────────┐
   │GitHub │ │Slack│ │ DB  │ │Stripe│ │ Internal  │
   │Server │ │Srvr │ │Srvr │ │Srvr │ │ Factory   │
   └───────┘ └─────┘ └─────┘ └─────┘ │ Servers   │
                                       └───────────┘
```

### MCP Client Gateway

A centralised process that manages connections to all MCP servers on behalf of factory agents. Agents don't connect to MCP servers directly — they request tools through the gateway, which handles:

- **Connection lifecycle** — Spawning stdio servers, maintaining HTTP connections, health checks
- **Tool discovery** — Agents query "what tools are available?" and get a unified catalogue
- **Authorization** — Per-agent, per-tool permission checks before forwarding calls
- **Credential injection** — Secrets injected at the gateway level, never exposed to agents
- **Rate limiting & quotas** — Prevent runaway agents from hammering external APIs
- **Audit logging** — Every tool invocation logged with agent ID, parameters, result, timestamp

### Why a Gateway (Not Direct Connections)?

1. **Security** — Agents never see raw credentials. Gateway injects tokens per-connection.
2. **Governance** — Central point to enforce policies, quotas, and audit trails.
3. **Efficiency** — One Slack connection shared by all agents, not N separate ones.
4. **Manageability** — Add/remove/update MCP servers without touching agent configs.

---

## 2. MCP Server Registry

The registry is the factory's catalogue of available MCP servers — what's installed, how to connect, what tools each provides, and who's allowed to use them.

### Registry Schema

```yaml
servers:
  github:
    name: "GitHub"
    package: "@modelcontextprotocol/server-github"
    transport: stdio
    command: "npx"
    args: ["-y", "@modelcontextprotocol/server-github"]
    credentials:
      GITHUB_PERSONAL_ACCESS_TOKEN:
        source: vault
        key: "factory/github/pat"
    tier: essential
    category: dev-platform
    tools_exposed:
      - create_issue
      - create_pull_request
      - search_code
      - get_file_contents
      - list_commits
      # ... (auto-discovered on connection)
    authorization:
      default: deny
      allow:
        - role: engineer-agent
          tools: ["*"]
        - role: qa-agent
          tools: ["search_code", "get_file_contents", "list_commits"]
        - role: pm-agent
          tools: ["create_issue", "search_issues"]

  slack:
    name: "Slack"
    package: "@anthropic/slack-mcp-server"
    transport: stdio
    credentials:
      SLACK_BOT_TOKEN:
        source: vault
        key: "factory/slack/bot-token"
    tier: essential
    category: communication
    authorization:
      default: deny
      allow:
        - role: any
          tools: ["send_message"]
          constraints:
            channels: ["#agent-updates", "#alerts"]
        - role: orchestrator
          tools: ["*"]

  # ... more servers
```

### Registry Storage

- **Source of truth:** `factory-config` git repo, `mcp-servers.yaml`
- **Runtime cache:** Redis hash for fast lookups
- **Updates:** Git push → webhook → gateway reloads config, connects/disconnects servers as needed

---

## 3. Essential MCP Servers

### Tier 1: Core (Day 1)

| Server | Purpose | Key Tools |
|--------|---------|-----------|
| **GitHub** | Code, PRs, issues, reviews | `create_pr`, `create_issue`, `search_code`, `get_file_contents` |
| **Filesystem** | Agent workspace file access | `read_file`, `write_file`, `list_directory` |
| **Git** | Repo operations | `git_log`, `git_diff`, `git_status` |
| **PostgreSQL** | Factory database access | `query`, `list_tables`, `describe_table` |
| **Slack** | Team communication | `send_message`, `list_channels`, `search_messages` |
| **Fetch** | Web content retrieval | `fetch_url`, `extract_text` |

### Tier 2: Growth (Month 1-2)

| Server | Purpose | Key Tools |
|--------|---------|-----------|
| **Stripe** | Payment processing | `create_checkout`, `list_payments`, `create_subscription` |
| **Shopify** | E-commerce operations | `create_product`, `list_orders`, `update_inventory` |
| **Sentry** | Error tracking | `list_issues`, `get_issue_details`, `resolve_issue` |
| **Linear** | Project management | `create_issue`, `update_status`, `list_projects` |
| **Brave Search** | Web search | `search_web`, `search_news` |
| **Playwright** | Browser automation | `navigate`, `click`, `screenshot`, `evaluate` |

### Tier 3: Scale (Month 3+)

| Server | Purpose | Key Tools |
|--------|---------|-----------|
| **Google Calendar** | Scheduling | `create_event`, `list_events` |
| **Google Drive** | Document storage | `upload_file`, `search_files` |
| **Redis** | Caching, queues | `get`, `set`, `lpush`, `brpop` |
| **Notion** | Knowledge base | `search`, `create_page`, `update_page` |
| **AWS** | Cloud infrastructure | `deploy`, `describe_instances`, `get_logs` |
| **Figma** | Design access | `get_file`, `get_components` |

---

## 4. Custom Internal MCP Servers

These are factory-specific servers we build ourselves, exposing internal systems as MCP tools.

### 4.1 Task Queue Server (`@factory/mcp-task-queue`)

Exposes the factory's task management system to agents.

```typescript
// Tools
task.claim       // Claim next available task matching agent's skills
task.complete    // Mark task as done with deliverables
task.fail        // Report task failure with reason
task.create      // Create a subtask
task.status      // Get task status/details
task.list        // List tasks by filter (status, assignee, priority)
task.handoff     // Transfer task to another agent/role
task.log         // Append progress log entry

// Resources
task://{id}              // Task details
task://queue/{role}      // Available tasks for a role
task://active/{agent}    // Agent's current tasks
```

### 4.2 Memory Server (`@factory/mcp-memory`)

Persistent memory and knowledge management for agents.

```typescript
// Tools
memory.store     // Store a memory (key, content, tags, ttl)
memory.recall    // Retrieve memories by query (semantic search)
memory.forget    // Delete a memory
memory.update    // Update existing memory
memory.list      // List memories by tag/namespace

// Resources
memory://{agent}/{namespace}    // Agent's memories in a namespace
memory://shared/{topic}         // Shared factory knowledge
memory://decisions/{date}       // Decision log
```

### 4.3 Deployment Server (`@factory/mcp-deploy`)

Manages deployments, environments, and infrastructure.

```typescript
// Tools
deploy.preview     // Deploy to preview environment
deploy.promote     // Promote preview → staging → production
deploy.rollback    // Rollback to previous version
deploy.status      // Check deployment status
deploy.logs        // Get deployment logs
env.list           // List environments
env.vars.get       // Get env var (non-secret)
env.vars.set       // Set env var

// Resources
deploy://status/{service}       // Current deployment state
deploy://history/{service}      // Deployment history
deploy://environments           // Available environments
```

### 4.4 Agent Registry Server (`@factory/mcp-agents`)

Agents discovering and communicating with other agents.

```typescript
// Tools
agents.list        // List available agents and their capabilities
agents.status      // Get agent's current status (busy/idle/offline)
agents.request     // Request work from another agent (creates A2A task)
agents.notify      // Send notification to an agent

// Resources
agents://{id}/card     // Agent card (A2A compatible)
agents://capabilities  // Full capability matrix
```

### 4.5 Metrics Server (`@factory/mcp-metrics`)

Factory observability exposed to agents.

```typescript
// Tools
metrics.record     // Record a metric (counter, gauge, histogram)
metrics.query      // Query metrics (e.g., "error rate last hour")
metrics.alert      // Create/update alert rule

// Resources
metrics://dashboard/{name}    // Dashboard data
metrics://alerts/active       // Active alerts
```

### Implementation Notes

- All internal servers use **Streamable HTTP transport** (not stdio) — they run as long-lived services, shared across agents
- Built with the TypeScript MCP SDK (`@modelcontextprotocol/sdk`)
- Each server is a separate service in the factory's Docker Compose / k8s deployment
- Internal servers authenticate via the gateway — no separate auth layer needed

---

## 5. Tool Discovery & Resolution

### How Agents Find Tools

Agents don't need to know which MCP server provides a tool. They query the gateway:

```
Agent: "I need to create a GitHub issue"
Gateway: Resolves → github server → create_issue tool → checks auth → forwards
```

**Discovery flow:**

1. **Boot** — Gateway connects to all registered MCP servers, calls `tools/list` on each
2. **Catalogue** — Gateway builds unified tool catalogue: `{tool_name, server, description, schema, auth_requirements}`
3. **Agent context** — When an agent session starts, gateway provides the agent's allowed tool list based on its role
4. **Dynamic updates** — If a server is added/removed, gateway sends `notifications/tools/list_changed` to connected agents

### Tool Namespacing

To avoid collisions across servers, tools are namespaced:

```
github.create_issue       (from GitHub server)
linear.create_issue       (from Linear server)
factory.task.create       (from internal task queue)
factory.memory.store      (from internal memory)
```

Agents see the namespaced names. The gateway strips the namespace prefix when forwarding to the actual server.

### Capability-Based Discovery

Agents can also discover tools by capability rather than name:

```typescript
// Agent asks: "What tools can I use to manage project tasks?"
gateway.discoverTools({
  capability: "project-management",
  actions: ["create", "update", "list"]
})
// Returns: [linear.create_issue, github.create_issue, factory.task.create]
```

This is powered by semantic indexing of tool descriptions — the gateway embeds all tool descriptions and matches against queries.

---

## 6. Authorization & Security

### Permission Model

Three-layer authorization:

```
Layer 1: Role-Based Access     — Which roles can access which servers
Layer 2: Tool-Level Grants     — Within a server, which specific tools
Layer 3: Runtime Constraints   — Parameter-level restrictions (channels, repos, etc.)
```

### Role → Tool Mapping

```yaml
roles:
  engineer-agent:
    servers: [github, filesystem, git, postgres, sentry, deploy]
    denied_tools: [deploy.promote]  # Can deploy to preview, not prod
    
  qa-agent:
    servers: [github, playwright, sentry]
    github_constraints:
      actions: [read]  # Read-only GitHub access
      
  orchestrator:
    servers: ["*"]  # Full access
    constraints:
      deploy.promote:
        requires_approval: true
        approver: human
        
  pm-agent:
    servers: [linear, slack, github]
    github_constraints:
      tools: [create_issue, search_issues]
      
  revenue-agent:
    servers: [stripe, shopify, slack, postgres]
    stripe_constraints:
      max_amount: 10000  # Cannot process charges > $100
    postgres_constraints:
      tables: [orders, products, analytics]  # No access to users table
```

### Credential Management

- **Vault-backed** — All MCP server credentials stored in HashiCorp Vault (or 1Password Secrets Automation)
- **Gateway-injected** — Credentials injected as env vars when spawning stdio servers, or as headers for HTTP servers
- **Rotation** — Gateway handles credential rotation without agent awareness
- **Never in agent context** — Agents never see tokens, keys, or passwords. They call tools; the gateway handles auth.

### Audit Trail

Every tool invocation logged:

```json
{
  "timestamp": "2026-02-08T09:22:14Z",
  "agent_id": "engineer-agent-03",
  "session_id": "sess_abc123",
  "task_id": "task_456",
  "server": "github",
  "tool": "create_pull_request",
  "params": { "repo": "factory/web-app", "title": "Fix login bug", "branch": "fix/login" },
  "result": "success",
  "duration_ms": 1240,
  "cost_estimate": null
}
```

Logs stored in append-only store (event sourcing). Used for:
- Security audits
- Cost tracking (API calls to external services)
- Agent performance analysis
- Debugging failed workflows

---

## 7. Connection Management

### Transport Strategy

| Server Type | Transport | Lifecycle |
|-------------|-----------|-----------|
| External (community) servers | stdio | Spawned per-gateway, one instance shared by all agents |
| Internal factory servers | Streamable HTTP | Long-running services, gateway connects on boot |
| High-volume external (Stripe, etc.) | Streamable HTTP preferred | Persistent connection, connection pooling |

### Health & Resilience

- **Health checks** — Gateway pings each server every 30s (`ping` → `pong`)
- **Auto-restart** — stdio servers restarted on crash (3 retries, then alert)
- **Circuit breaker** — If a server fails 5 consecutive calls, circuit opens for 60s
- **Graceful degradation** — If GitHub MCP is down, agents get clear error "GitHub tools temporarily unavailable" rather than cryptic failures
- **Connection pooling** — For HTTP servers, maintain connection pool per server

### Scaling

- **Single gateway** handles all MCP connections initially (sufficient for <50 agents)
- **Gateway sharding** — At scale, shard by server category (dev-tools gateway, commerce gateway, internal gateway)
- **Server multiplexing** — One MCP server instance serves multiple concurrent agent requests (servers are stateless per-call)

---

## 8. Integration with Factory Architecture

### MCP + A2A Boundary

```
Agent-to-Tool (MCP):     Agent → Gateway → MCP Server → External Service
Agent-to-Agent (A2A):    Agent → A2A Protocol → Agent
Agent-to-Orchestrator:   Agent → Event Bus → Orchestrator
```

MCP is strictly for tool/data access. Agent coordination uses A2A or the event bus (see architecture doc on agent communication).

### Agent Session Lifecycle

1. Agent spawns → registers with gateway → receives allowed tool catalogue
2. Agent claims task from task queue (via `factory.task.claim`)
3. Agent works — calls tools as needed (GitHub, filesystem, etc.)
4. Agent stores learnings (`factory.memory.store`)
5. Agent completes task (`factory.task.complete`)
6. Agent idles or claims next task

### OpenClaw Integration Path

Since our agents run on OpenClaw, and OpenClaw has its own tool system:

1. **Bridge skill** — Build an OpenClaw skill that acts as MCP client, connecting to the gateway
2. **Dynamic tool exposure** — MCP tools appear as native OpenClaw tools to the agent
3. **Transparent** — Agent calls `github.create_issue` as if it were a built-in OpenClaw tool; the skill forwards to the MCP gateway
4. **Fallback** — For tools OpenClaw already provides natively (browser, filesystem, exec), use OpenClaw's built-in tools — don't duplicate via MCP

---

## 9. Implementation Plan

### Phase 1: Foundation (Week 1-2)
- [ ] Set up MCP Client Gateway service (TypeScript, Streamable HTTP)
- [ ] Implement registry loader (YAML config → runtime state)
- [ ] Connect to GitHub + Filesystem + Git MCP servers
- [ ] Basic role-based authorization
- [ ] Audit logging

### Phase 2: Internal Servers (Week 3-4)
- [ ] Build `@factory/mcp-task-queue` server
- [ ] Build `@factory/mcp-memory` server
- [ ] Build `@factory/mcp-deploy` server
- [ ] Tool namespacing and discovery API

### Phase 3: External Expansion (Week 5-6)
- [ ] Add Slack, Stripe, Shopify, Linear, Sentry servers
- [ ] Credential management via Vault
- [ ] Rate limiting and circuit breakers
- [ ] OpenClaw bridge skill

### Phase 4: Polish (Week 7-8)
- [ ] Semantic tool discovery (capability-based search)
- [ ] Agent Registry server (for A2A discovery)
- [ ] Metrics server
- [ ] Dashboard for monitoring MCP server health and usage
- [ ] Load testing and gateway sharding strategy

---

## 10. Key Decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Centralised gateway vs direct connections | **Gateway** | Security (credential isolation), governance, efficiency |
| stdio vs HTTP for internal servers | **HTTP** | Long-lived shared services, multiple concurrent clients |
| Build vs buy for internal tools | **Build** | Task queue, memory, deploy are factory-specific; no generic MCP server fits |
| Tool namespacing strategy | **server.tool_name** | Simple, avoids collisions, easy to understand |
| Auth model | **Role-based + tool-level + constraints** | Least privilege without being unmanageable |
| Registry storage | **Git (source of truth) + Redis (runtime)** | GitOps workflow, fast lookups |

---

*Architecture doc for the Autonomous Agentic Factory — MCP Integration Layer*
