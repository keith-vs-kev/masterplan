# 03 — Agent Identity & Specialization

> How agents are defined, configured, discovered, and extended in the factory.

---

## 1. Design Principles

1. **Agents are markdown files.** An agent's identity lives in a single `.md` file — human-readable, version-controlled, diffable. No JSON schemas, no config sprawl.
2. **Convention over configuration.** Sensible defaults everywhere. An agent file with just a name and role sentence is valid.
3. **Composable skills, not monolithic prompts.** Capabilities are injected via skill modules, not baked into 2000-line system prompts.
4. **Model-flexible, opinion-default.** Every agent has a default model, but any can be overridden per-task or per-session.
5. **Progressive disclosure.** Simple agents are simple to define. Complex agents have more knobs, but you only touch what you need.

---

## 2. Agent File Format

Agents live in `agents/<name>.md`. The file IS the agent definition — frontmatter for machine-readable config, body for personality/instructions.

```markdown
---
name: Rex
role: orchestrator
tagline: "The conductor. Decomposes goals into agent tasks."
model: anthropic/claude-opus-4
fallback_model: anthropic/claude-sonnet-4.5
thinking: extended
max_tokens: 16384
temperature: 0.3

tools:
  - task-manager
  - agent-dispatch
  - git
  - shell

skills:
  - planning
  - code-review
  - project-management

channels:
  - slack
  - discord
  - internal

limits:
  max_parallel_tasks: 10
  max_cost_per_task_usd: 5.00
  requires_approval:
    - deploy
    - external-message
    - payment

escalation: human
---

# Rex — The Orchestrator

You are Rex, the lead orchestrator of the agent factory. Your job is to take
high-level goals from humans, decompose them into concrete tasks, and dispatch
them to specialist agents. You never write code yourself — you delegate.

## Personality

Direct, efficient, slightly military. You think in plans and dependencies.
You confirm understanding before dispatching. You track progress relentlessly.

## Decision Framework

1. Understand the goal completely (ask if ambiguous)
2. Break into subtasks with clear acceptance criteria
3. Assign each subtask to the best specialist agent
4. Monitor progress, unblock, escalate failures
5. Integrate results and report back

## What You Don't Do

- Write code (delegate to Forge, Blaze, or Dash)
- Make design decisions (delegate to Pixel or Scout)
- Deploy without human approval
```

### Frontmatter Schema

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `name` | string | ✅ | Agent identifier (unique, lowercase allowed in references) |
| `role` | string | ✅ | Primary role category |
| `tagline` | string | | One-liner description |
| `model` | string | ✅ | Default model (`provider/model-name`) |
| `fallback_model` | string | | Used when primary is unavailable or rate-limited |
| `thinking` | enum | | `off`, `low`, `extended` — reasoning mode |
| `max_tokens` | int | | Max output tokens per turn |
| `temperature` | float | | Default temperature (0.0–1.0) |
| `tools` | list | | Tools this agent can access |
| `skills` | list | | Skill modules injected into context |
| `channels` | list | | Communication channels agent can use |
| `limits` | object | | Cost, parallelism, and approval constraints |
| `limits.requires_approval` | list | | Actions requiring human sign-off |
| `escalation` | string | | Where to escalate: `human`, agent name, or `none` |
| `sandbox` | string | | Execution environment: `docker`, `vm`, `none` |
| `memory` | object | | Memory config (see §5) |

---

## 3. System Prompt Assembly

The runtime assembles the final system prompt from layers. This is NOT stored — it's computed at session start.

```
┌─────────────────────────────────┐
│  1. Factory Preamble            │  ← Global rules, safety, identity framework
│     (shared across all agents)  │
├─────────────────────────────────┤
│  2. Agent Identity (body of     │  ← From agents/<name>.md body
│     the agent .md file)         │
├─────────────────────────────────┤
│  3. Skill Injections            │  ← From skills/<skill>/SKILL.md for each
│     (capabilities & tools docs) │     listed skill
├─────────────────────────────────┤
│  4. Context Injections          │  ← Current task brief, relevant memory,
│     (dynamic, per-session)      │     project context, AGENTS.md from target repo
├─────────────────────────────────┤
│  5. Channel Adapter             │  ← Platform-specific formatting rules
│     (optional)                  │     (Discord: no tables; WhatsApp: no headers)
└─────────────────────────────────┘
```

### Layer 1: Factory Preamble (shared)

```markdown
You are an autonomous agent in the Factory. You have a specific role and
capabilities defined below. You collaborate with other specialist agents
via the task system. You do not act outside your defined scope — if a task
falls outside your expertise, delegate or escalate.

Safety rules:
- Never exfiltrate private data
- Never execute destructive actions without approval
- Always respect cost limits
- Log decisions for auditability
```

### Layer 3: Skill Injection

Skills are modular capability packages. Each skill has a `SKILL.md` that documents how to use the associated tools. When an agent lists a skill, that SKILL.md is injected into its system prompt.

```
skills/
├── planning/
│   └── SKILL.md          # Task decomposition, dependency graphs
├── code-review/
│   └── SKILL.md          # PR review patterns, quality criteria
├── web-scraping/
│   └── SKILL.md          # Browser automation, fetch, extraction
├── testing/
│   └── SKILL.md          # Test generation, coverage analysis
├── deployment/
│   └── SKILL.md          # CI/CD, deploy workflows
└── seo/
    └── SKILL.md          # SEO audit, content optimization
```

---

## 4. The 14 Specialist Agents

### Overview Matrix

| Agent | Role | Default Model | Thinking | Key Tools | Key Skills |
|-------|------|---------------|----------|-----------|------------|
| **Rex** | Orchestrator | claude-opus-4 | extended | task-manager, agent-dispatch, git | planning, project-management |
| **Scout** | Researcher | claude-sonnet-4.5 | extended | web-fetch, browser, search | research, analysis, summarization |
| **Forge** | Backend Engineer | claude-sonnet-4.5 | low | shell, git, editor, docker | backend, databases, apis, testing |
| **Hawk** | Code Reviewer / QA | claude-sonnet-4.5 | extended | git, linters, test-runner | code-review, testing, security-audit |
| **Pixel** | Frontend / Design | claude-sonnet-4.5 | low | browser, editor, screenshot | frontend, css, design-systems, a11y |
| **Blaze** | Full-Stack Builder | claude-sonnet-4.5 | low | shell, git, editor, browser, docker | fullstack, rapid-prototyping |
| **Echo** | Writer / Content | claude-sonnet-4.5 | off | editor, web-fetch | copywriting, docs, seo, social-media |
| **Chase** | Sales / Outreach | claude-sonnet-4.5 | off | email, crm, web-fetch | outreach, lead-gen, follow-up |
| **Ally** | Support / Success | claude-haiku-3.5 | off | email, knowledge-base, ticketing | support, triage, escalation |
| **Dash** | DevOps / Infra | claude-sonnet-4.5 | low | shell, docker, k8s, terraform, ssh | deployment, monitoring, infra |
| **Finn** | Finance / Analytics | claude-sonnet-4.5 | extended | spreadsheet, database, stripe-api | accounting, forecasting, reporting |
| **Dot** | Data / ML | claude-sonnet-4.5 | extended | shell, jupyter, database | data-engineering, ml, analytics |
| **Law** | Legal / Compliance | claude-opus-4 | extended | editor, web-fetch | legal-review, compliance, contracts |
| **Atlas** | Personal Assistant | claude-opus-4 | extended | all (scoped per user) | generalist — varies per user config |

### Agent Profiles

#### Rex — The Orchestrator
- **Purpose:** Decomposes goals into tasks, dispatches to specialists, tracks progress, integrates results.
- **When to use:** Any multi-step project. Rex is the default entry point for complex work.
- **Model rationale:** Opus for superior planning and reasoning across long task chains.
- **Unique tools:** `agent-dispatch` (spawn/assign tasks to other agents), `task-manager` (create/track/close tasks).
- **Does NOT:** Write code, design UIs, send external comms directly.

#### Scout — The Researcher
- **Purpose:** Deep research, competitive analysis, technology evaluation, fact-finding.
- **When to use:** "Research X", "Compare Y vs Z", "What's the best approach for W?"
- **Model rationale:** Sonnet for speed + extended thinking for synthesis.
- **Unique tools:** `search` (web search), `browser` (full browser automation for deep dives).
- **Output:** Structured research reports in markdown.

#### Forge — The Backend Engineer
- **Purpose:** Backend systems, APIs, databases, server-side logic, system architecture.
- **When to use:** "Build an API for X", "Set up the database", "Fix this server bug".
- **Sandbox:** Docker (isolated build/test environment).
- **Model rationale:** Sonnet — best code generation speed/quality ratio.

#### Hawk — The Reviewer
- **Purpose:** Code review, quality assurance, security audits, test coverage analysis.
- **When to use:** Every PR before merge. Can also be invoked for security audits.
- **Model rationale:** Sonnet + extended thinking — needs to reason about edge cases.
- **Pattern:** Review Chain — Hawk reviews what Forge/Blaze/Pixel produce.
- **Unique tools:** `linters`, `test-runner`, `security-scanner`.

#### Pixel — The Frontend Engineer
- **Purpose:** UI/UX implementation, component libraries, responsive design, accessibility.
- **When to use:** "Build this UI", "Make it responsive", "Fix the CSS".
- **Unique tools:** `screenshot` (visual regression), `browser` (preview/test).
- **Skills:** Design system knowledge, accessibility standards, animation.

#### Blaze — The Full-Stack Builder
- **Purpose:** Rapid end-to-end prototyping. Gets something working fast across the entire stack.
- **When to use:** "Build a quick prototype of X", MVPs, hackathon-style sprints.
- **Model rationale:** Sonnet for speed. Blaze optimizes for velocity over perfection.
- **Relationship to Forge/Pixel:** Blaze builds fast; Forge/Pixel refine for production.

#### Echo — The Writer
- **Purpose:** Content creation, documentation, copywriting, blog posts, social media.
- **When to use:** "Write docs for X", "Draft a blog post", "Create social media content".
- **Model rationale:** Sonnet with thinking off — creative writing benefits from flow, not deliberation.
- **Skills:** SEO optimization, brand voice adherence, multi-format content.

#### Chase — The Sales Agent
- **Purpose:** Outreach, lead generation, follow-ups, proposal drafting.
- **When to use:** "Find leads for X", "Draft outreach emails", "Follow up with Y".
- **Unique tools:** `email`, `crm` (CRM integration), `linkedin` (if available).
- **Limits:** All external messages require human approval.

#### Ally — The Support Agent
- **Purpose:** Customer support, ticket triage, FAQ responses, escalation.
- **When to use:** Incoming support requests, user questions.
- **Model rationale:** Haiku for cost efficiency — support is high-volume, often routine.
- **Escalation:** Complex issues → Scout (research) or Forge (bugs).

#### Dash — The DevOps Engineer
- **Purpose:** Infrastructure, CI/CD, deployment, monitoring, incident response.
- **When to use:** "Deploy to production", "Set up CI", "Why is the server down?"
- **Unique tools:** `docker`, `k8s`, `terraform`, `ssh`, `monitoring-api`.
- **Limits:** Production deployments require human approval.

#### Finn — The Finance Agent
- **Purpose:** Financial tracking, invoicing, forecasting, analytics, Stripe/payment operations.
- **When to use:** "How much did we spend?", "Generate invoice", "Revenue forecast".
- **Model rationale:** Sonnet + extended thinking — financial calculations need precision.
- **Unique tools:** `stripe-api`, `spreadsheet`, `database`.

#### Dot — The Data Engineer
- **Purpose:** Data pipelines, ML model training, analytics dashboards, data transformation.
- **When to use:** "Analyze this dataset", "Build a data pipeline", "Train a model for X".
- **Unique tools:** `jupyter`, `database`, ML framework CLIs.
- **Sandbox:** Docker with GPU access when needed.

#### Law — The Legal Agent
- **Purpose:** Contract review, compliance checks, terms of service, privacy policies.
- **When to use:** "Review this contract", "Are we GDPR compliant?", "Draft ToS".
- **Model rationale:** Opus — legal analysis demands the strongest reasoning.
- **Limits:** All legal output includes disclaimer; never provides legal advice, only analysis.
- **Escalation:** Always recommends human legal review for binding documents.

#### Atlas — The Personal Assistant
- **Purpose:** General-purpose personal assistant. Manages calendar, email, tasks, life admin.
- **When to use:** The default agent for direct human interaction.
- **Model rationale:** Opus — needs to handle anything, reason broadly.
- **Unique:** Tool access is scoped per-user (each user's Atlas has different permissions).
- **Memory:** Full persistent memory (daily logs + long-term MEMORY.md).

---

## 5. Tool Access Control

Tools are granted per-agent in the frontmatter. The runtime enforces this — an agent cannot call a tool not in its `tools` list.

### Tool Categories

| Category | Tools | Agents With Access |
|----------|-------|--------------------|
| **Core** | shell, editor, git | Forge, Blaze, Pixel, Dash, Dot, Hawk |
| **Web** | web-fetch, browser, search | Scout, Echo, Chase, Atlas |
| **Comms** | email, slack, discord | Chase, Ally, Echo, Atlas |
| **Infra** | docker, k8s, terraform, ssh | Dash, Forge, Dot |
| **Data** | database, spreadsheet, jupyter | Finn, Dot, Forge |
| **Factory** | task-manager, agent-dispatch | Rex, Atlas |
| **Payments** | stripe-api, invoicing | Finn |
| **Review** | linters, test-runner, security-scanner | Hawk |

### Approval Gates

Some tools require human approval regardless of agent:
- `deploy-production` — always
- `email-send` (external) — always for Chase, configurable for others
- `payment-create` — always
- `delete-resource` — always
- `agent-dispatch` with cost > threshold — configurable

---

## 6. Memory Architecture

```yaml
memory:
  working: true          # Short-term, within-session scratchpad
  episodic: true         # Daily logs (memory/YYYY-MM-DD.md)
  semantic: true         # Long-term curated (MEMORY.md)
  shared: false          # Access to shared factory knowledge base
```

- **Working memory:** In-context, dies with the session.
- **Episodic memory:** Auto-logged daily files. All agents get this.
- **Semantic memory:** Curated long-term memory. Only persistent agents (Atlas, Rex) maintain this by default.
- **Shared knowledge:** Factory-wide knowledge base (project docs, architecture decisions, past research). Read-only for most agents; Scout and Rex can write.

---

## 7. Adding a New Agent

### Step 1: Create the agent file

```bash
touch agents/nova.md
```

### Step 2: Define identity and config

Write the frontmatter (model, tools, skills, limits) and the body (personality, instructions, decision framework). Use an existing agent as a template.

### Step 3: Register skills

If the agent needs a new skill, create `skills/<skill-name>/SKILL.md`. Existing skills are reused by reference.

### Step 4: Test

```bash
factory agent test nova --task "Do a simple task in your domain"
```

The test harness runs the agent against sample tasks and checks:
- Does it stay in scope?
- Does it use only its granted tools?
- Does it escalate appropriately?
- Does it respect cost limits?

### Step 5: Deploy

The agent is available immediately — the factory discovers agents by scanning `agents/*.md`. No registration step needed.

### Checklist for New Agents

- [ ] `agents/<name>.md` with valid frontmatter
- [ ] Role doesn't overlap excessively with existing agents
- [ ] Tool access is minimal (principle of least privilege)
- [ ] Cost limits are set
- [ ] Escalation path is defined
- [ ] At least one skill is assigned
- [ ] Test suite passes

---

## 8. Model Selection Strategy

### Default Assignments

| Tier | Model | Use Case | Agents |
|------|-------|----------|--------|
| **Reasoning** | claude-opus-4 | Planning, legal, complex orchestration | Rex, Law, Atlas |
| **Workhorse** | claude-sonnet-4.5 | Code gen, research, most tasks | Forge, Scout, Hawk, Pixel, Blaze, Echo, Chase, Dash, Finn, Dot |
| **Speed/Cost** | claude-haiku-3.5 | High-volume, routine tasks | Ally |

### Override Rules

1. **Per-task override:** Rex (or human) can assign a different model for a specific task.
2. **Cost-based fallback:** If a task exceeds budget with the default model, fall back to a cheaper model.
3. **Provider diversity:** For review chains, use different providers (e.g., Forge writes with Claude, Hawk reviews with GPT) to catch model-specific blind spots.
4. **Local models:** Privacy-sensitive tasks can use local models (Ollama/vLLM) via Aider-style integration.

---

## 9. Agent Discovery & Communication

Agents discover each other through the factory's agent registry (the `agents/` directory). Communication happens via the task system, not direct agent-to-agent messaging.

```
Human → Rex → Task Queue → Specialist Agent → Result → Rex → Human
```

For A2A interoperability with external systems, each agent can expose an **Agent Card** (per Google's A2A protocol) describing its capabilities, skills, and endpoint. This is auto-generated from the agent `.md` file.

---

## 10. Security Model

1. **Principle of least privilege:** Agents only get tools they need.
2. **Sandboxed execution:** Code-writing agents run in Docker/VM sandboxes.
3. **Approval gates:** Destructive/external/costly actions require human approval.
4. **Audit trail:** All agent actions are logged with agent ID, timestamp, and outcome.
5. **Cost caps:** Per-task and per-agent cost limits prevent runaway spending.
6. **No self-modification:** Agents cannot modify their own agent files or other agents' files.
7. **Escalation chains:** Every agent has a defined escalation path (typically → Rex → human).

---

## Appendix: Directory Structure

```
factory/
├── agents/
│   ├── rex.md
│   ├── scout.md
│   ├── forge.md
│   ├── hawk.md
│   ├── pixel.md
│   ├── blaze.md
│   ├── echo.md
│   ├── chase.md
│   ├── ally.md
│   ├── dash.md
│   ├── finn.md
│   ├── dot.md
│   ├── law.md
│   └── atlas.md
├── skills/
│   ├── planning/
│   │   └── SKILL.md
│   ├── code-review/
│   │   └── SKILL.md
│   ├── research/
│   │   └── SKILL.md
│   ├── testing/
│   │   └── SKILL.md
│   ├── deployment/
│   │   └── SKILL.md
│   └── .../
├── prompts/
│   └── factory-preamble.md
└── config/
    ├── tool-policies.yaml
    └── model-defaults.yaml
```

---

*Architecture document — Agent Identity & Specialization. Part of the Masterplan series.*
