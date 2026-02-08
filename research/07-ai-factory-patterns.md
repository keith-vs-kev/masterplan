# 07 — AI Factory Architecture Patterns

> Research compiled 2026-02-08. Sources: company websites, blog posts, papers, public architecture docs.

---

## Executive Summary

The "AI factory" — a system where AI agents autonomously produce software — is being built by at least eight notable companies, each with distinct architectural bets. The dominant patterns are converging around three models: **sandboxed single-agent** (Devin, Bolt, Lovable), **multi-agent orchestration** (OpenHands, Factory), and **foundation-model-first** (Poolside, Magic). The key differentiators are not the agent logic itself but how companies handle **context management**, **long-running task persistence**, **human-in-the-loop integration**, and **self-improvement feedback loops**.

---

## Company-by-Company Analysis

### 1. Cognition (Devin)

**What they are:** The original "AI software engineer" — a fully autonomous agent that gets its own machine, shell, browser, and editor.

**Architecture pattern:** **Sandboxed autonomous agent with fleet parallelism**

- Each Devin instance runs in an isolated VM with full dev environment (shell, browser, editor, terminal)
- Hub-and-spoke model: human assigns task → Devin works autonomously → delivers PR
- Fleet parallelism: "a fleet of Devins" can execute across hundreds of repos simultaneously (e.g., migration tasks)
- Integration surface: Slack, Teams, Jira, Linear — Devin is treated like a team member you @-mention
- Custom model: SWE-1.5 (frontier-size, hundreds of billions of params), partnered with Cerebras for 950 tok/s inference
- Specialized subagents: SWE-grep/SWE-grep-mini for parallel context retrieval

**Key insights:**
- 67% PR merge rate (up from 34% in year 1) — improving rapidly
- Best at "junior execution at infinite scale" — clear requirements, verifiable outcomes
- Struggles with ambiguity, mid-task scope changes, iterative collaboration
- DeepWiki generates documentation across 400K+ repos for codebase understanding
- AskDevin for on-demand senior-level codebase Q&A

**Context handling:** Each Devin session has its own context. Playbooks provide cross-session knowledge (human writes playbook → fleet of Devins executes). No persistent memory across sessions by default — relies on codebase + docs as external memory.

**Long-running tasks:** Each Devin runs asynchronously. Human checks in via Slack/PR. No real-time streaming required.

---

### 2. Factory AI (Droids)

**What they are:** "Agent-native software development" — Droids that embed into existing workflows (IDE, CLI, Slack, Linear, CI/CD).

**Architecture pattern:** **Workflow-embedded agents with self-improvement loop**

- Interface-agnostic: same agent works across IDE, Web, CLI, Slack, Linear
- Model-agnostic: works with any model provider, extensible
- Enterprise-focused: security boundary, vendor-neutral
- **Signals system** — the most interesting architectural innovation:
  - LLM-as-judge analyzes sessions at scale for friction/delight
  - Facet schema evolves via semantic clustering (embeddings → cluster → LLM proposes new dimensions)
  - Friction detection: error events, repeated rephrasing, escalation in tone
  - **Recursive self-improvement**: when friction crosses a threshold, "Droid fixes itself"
  - Categories evolve emergently (e.g., "context churn" and "branch switches" discovered by the system)

**Key insights:**
- Closed-loop self-improvement is architecturally significant — agent observes its own failures and patches itself
- "Agent Readiness" framework: measures how well a codebase supports autonomous development across 8 pillars and 5 maturity levels
- Session-based model (like Devin) but with deeper workflow integration

**Context handling:** Deep integration with dev tooling means context is pulled from the environment (CI/CD, Linear tickets, codebase) rather than maintained internally.

**Long-running tasks:** Sessions persist across tool boundaries (start in Slack, continue in IDE). Enterprise case studies show multi-week sustained engagement (Chainguard: "Two Weeks, One Session").

---

### 3. Poolside

**What they are:** Foundation model company building code-specific models trained on proprietary synthetic data. Not an agent product — they're the model layer.

**Architecture pattern:** **Model-as-platform / Foundation layer**

- Train models on internet-scale data + proprietary synthetic datasets designed to represent real-world engineering and agent-model interactions
- Differentiation: commercially-licensed training data with indemnity
- Deploy on-premise or VPC — "model to your data, not data to your model"
- Connected to org data: knowledge bases, CI/CD pipelines, databases, runtime environments
- Multi-surface delivery: terminal, APIs, agents, IDEs — "built for a future where editors are no longer the primary surface"
- Single agent binary extensible across an organization

**Key insights:**
- Betting that **models are the moat**, not the editor/agent wrapper
- Their architecture implies a hub model: one foundation model powers many surfaces and agents
- Focus on enterprise security (fully isolated, on-prem, full observability)

---

### 4. Magic

**What they are:** AGI-focused lab building ultra-long-context models for code generation and AI research automation.

**Architecture pattern:** **Frontier model research lab**

- 8,000 H100s, $515M raised
- Approach: frontier-scale pre-training + domain-specific RL + ultra-long context + inference-time compute
- Goal: automate AI research itself (recursive self-improvement at the model level)
- Small team, few products — research-first

**Key insights:**
- Ultra-long context is their architectural bet — if you can fit an entire codebase in context, you don't need complex RAG/retrieval pipelines
- This is the "just make the model better" approach vs. building agent scaffolding
- Contrast with Cognition (build the agent) vs Magic (build the model that doesn't need the agent)

---

### 5. All Hands AI (OpenHands)

**What they are:** Open-source platform for AI software development agents. Formerly OpenDevin.

**Architecture pattern:** **Multi-agent sandboxed platform**

- Open-source (MIT license), 2.1K+ contributions, 188+ contributors
- Agents interact via: code writing, command line, web browsing — all in sandboxed environments
- **Multi-agent coordination** built into the platform architecture
- Pluggable agent implementations — platform provides the runtime, sandboxing, and coordination
- Evaluation benchmarks integrated (SWE-BENCH, WEBARENA, etc.)
- Docker-based sandboxed execution environments

**Key insights:**
- The "Linux of AI agents" play — open platform, community-driven
- Architecture separates **agent logic** from **runtime environment** from **evaluation**
- Multi-agent coordination is first-class: agents can delegate to sub-agents
- Event-stream architecture: agents communicate through an event log (observations and actions)
- Controller pattern: a controller manages the agent loop, dispatching actions to the runtime

**Context handling:** Event stream serves as shared state. Each agent sees the accumulated history of observations and actions.

**Long-running tasks:** Sandboxed Docker environments persist for the duration of a task. State is the filesystem + event stream.

---

### 6. Replit Agent

**What they are:** AI agent embedded in Replit's cloud IDE that can build full applications from natural language.

**Architecture pattern:** **Environment-integrated autonomous builder**

- Deeply integrated into Replit's existing infrastructure (Nix environments, hosting, databases)
- Agent has access to: file system, shell, package manager, web preview, deployment
- Plan-then-execute: generates a step-by-step plan, then executes each step
- Iterative: runs the app, checks for errors, self-corrects
- Built on top of Replit's existing multiplayer/collaboration infrastructure

**Key insights:**
- Advantage of owning the entire stack: IDE + runtime + hosting + databases
- No sandboxing overhead — the execution environment IS the product
- Replit's "vertically integrated" approach means context is richer (they know your deployment, your DB schema, your packages)
- App-builder focus (full apps from prompts) rather than PR-level coding agent

**Context handling:** Full access to the project state (files, terminal output, browser preview). Context is the live environment itself.

**Long-running tasks:** Projects persist in Replit's cloud. Agent can be re-engaged on existing projects.

---

### 7. Lovable

**What they are:** AI app builder — generate full-stack web applications from natural language descriptions.

**Architecture pattern:** **Prompt-to-app pipeline**

- Chat-based interface: describe what you want → get a working app
- Real-time preview of the generated application
- Integrated hosting and deployment
- Iterative refinement through conversation
- Focus on non-technical users building production apps

**Key insights:**
- Simpler architecture than Devin/Factory — more of a sophisticated code generation pipeline than a multi-agent system
- The "app" is the artifact, not a PR or code change
- Vertical integration: generation + preview + hosting in one product
- Less about agent autonomy, more about high-quality single-shot generation with iterative refinement

**Context handling:** Conversation history + generated project state. Simpler context model than autonomous agents.

---

### 8. Bolt (by StackBlitz)

**What they are:** In-browser AI app builder using WebContainers for instant full-stack development.

**Architecture pattern:** **Browser-native sandboxed builder**

- Runs entirely in the browser via WebContainers (Node.js runtime in WebAssembly)
- No server-side execution needed — everything runs client-side
- "V2" integrates frontier coding agents from multiple AI labs
- Automatic testing, refactoring, iteration — "98% less errors"
- Built-in: databases, auth, SEO, hosting, analytics, custom domains

**Key insights:**
- Architectural innovation is the **browser-native sandbox** — zero infrastructure overhead
- Can handle "projects 1,000x larger than before" with improved context management
- Multi-model: integrates agents from different providers (not locked to one)
- Bolt Cloud provides backend infrastructure (databases, hosting) — vertical integration

**Context handling:** "Improved built-in context management" — likely project-level context window management with smart chunking.

---

## Architecture Pattern Taxonomy

### Pattern 1: Hub-and-Spoke (Devin, Factory)

```
                    Human
                      │
                  Orchestrator
                 /    |    \
            Agent   Agent   Agent
              │       │       │
            Sandbox Sandbox Sandbox
```

- Human dispatches tasks to a central system
- System spawns isolated agent instances
- Each agent works independently in its sandbox
- Results flow back (PRs, reports, messages)
- **Strengths:** Infinite parallelism, isolation, clear ownership
- **Weaknesses:** No inter-agent collaboration, context doesn't flow between agents
- **Best for:** Repetitive tasks at scale (migrations, test generation, vulnerability fixes)

### Pattern 2: Multi-Agent Mesh (OpenHands)

```
            Agent A ←→ Agent B
              ↕           ↕
            Agent C ←→ Agent D
                  ↕
              Event Stream
```

- Multiple agents collaborate through shared event streams
- Agents can delegate to sub-agents
- Shared state via event log
- **Strengths:** Complex task decomposition, specialization, emergent collaboration
- **Weaknesses:** Coordination overhead, harder to debug, state consistency challenges
- **Best for:** Complex multi-step tasks requiring different capabilities

### Pattern 3: Vertically Integrated Pipeline (Replit, Bolt, Lovable)

```
    Prompt → Plan → Generate → Execute → Preview → Deploy
                        ↑___________|
                        (iterate on errors)
```

- Single agent with deep access to the full stack
- Owns the entire lifecycle: generation through deployment
- Tight feedback loop: generate → run → observe → fix
- **Strengths:** Rich context (knows everything about the environment), fast iteration
- **Weaknesses:** Locked to one platform, less flexible for existing codebases
- **Best for:** Greenfield app building, prototyping, non-technical users

### Pattern 4: Foundation Model Layer (Poolside, Magic)

```
    Foundation Model
     /    |    \    \
   IDE  Agent  CLI  API
     \    |    /    /
    Organization Data
```

- Build the best model, let others build the agent layer
- Model connects to org data (CI/CD, databases, knowledge bases)
- Multiple surfaces consume the same intelligence
- **Strengths:** One model to rule them all, deepest code understanding, enterprise control
- **Weaknesses:** Dependent on model quality, no agent scaffolding included
- **Best for:** Enterprises wanting to build custom agent workflows on a code-native foundation

---

## Cross-Cutting Concerns

### Shared State

| Approach | Companies | Mechanism |
|----------|-----------|-----------|
| Event stream / action log | OpenHands | Append-only log of observations + actions |
| Filesystem as state | Devin, Replit, Bolt | The sandbox's filesystem IS the shared state |
| Conversation history | Lovable, Bolt | Chat context as the state backbone |
| Playbooks / templates | Devin, Factory | Human-authored instructions reused across runs |
| Org data integration | Poolside, Factory | CI/CD, databases, knowledge bases as context sources |

### Context Passing

The fundamental tension: **context windows are finite, codebases are not**.

- **Cognition's approach:** Specialized retrieval models (SWE-grep) for fast parallel context retrieval. Subagents that find context, main agent that acts.
- **Magic's approach:** Make context windows so large the problem goes away (ultra-long context).
- **Factory's approach:** Pull context from the environment (git, CI/CD, Linear) dynamically.
- **Poolside's approach:** Connect model to org data stores, bring model to data.
- **OpenHands' approach:** Event stream accumulates context; agents can read history.
- **Bolt's approach:** "Improved context management" at the project level — likely smart chunking/summarization.

### Long-Running Tasks

| Challenge | Solutions Observed |
|-----------|--------------------|
| Task exceeds context window | Checkpoint + resume (Devin), event stream (OpenHands) |
| Agent needs human input mid-task | Slack/Teams integration (Devin, Factory), chat interface (Replit, Bolt) |
| Multi-hour execution | Async sandboxed VMs (Devin), persistent cloud environments (Replit) |
| Failure recovery | Self-correction loops (all), Signals-based self-improvement (Factory) |
| Cross-session memory | Playbooks (Devin), org data integration (Poolside, Factory) |

### Self-Improvement

Factory's **Signals** system is the most architecturally mature self-improvement loop observed:

1. LLM judges analyze agent sessions for friction/delight
2. Semantic clustering discovers new failure categories emergently
3. When friction crosses threshold, agent patches itself
4. Categories evolve over time (schema is dynamic, not fixed)

Cognition takes a different approach: specialized model training (SWE-1.5) based on real-world deployment data. The agent improves because the underlying model improves.

---

## Key Takeaways

1. **The sandbox is table stakes.** Every serious agent runs in an isolated environment. The debate is how rich that environment is (VM vs Docker vs WebContainer vs cloud IDE).

2. **Context > Architecture.** The companies winning are those with the best context strategies — not the fanciest multi-agent topologies. Cognition's SWE-grep, Factory's environment integration, Magic's ultra-long context.

3. **Fleet parallelism is the killer feature for enterprises.** Devin's "fleet of Devins" doing migrations across hundreds of repos is what sells to Goldman Sachs. Not a single smart agent — a thousand adequate ones running in parallel.

4. **Self-improvement loops are emerging.** Factory's Signals is the clearest example, but all companies are building feedback loops from production deployments back into model/agent improvements.

5. **Vertical integration wins for app builders.** Replit, Bolt, and Lovable benefit enormously from owning the stack. Devin and Factory have to work with whatever environment the customer has.

6. **The model vs agent debate is real.** Magic and Poolside bet on making models so good you barely need agent scaffolding. Cognition and Factory bet on sophisticated agent systems that compensate for model limitations. Both paths are viable.

7. **Hub-and-spoke dominates today.** Most production deployments are human → agent → result. True multi-agent mesh (agents coordinating with each other) is still mostly in research/OSS (OpenHands). The coordination overhead isn't worth it yet for most use cases.

8. **Human-in-the-loop is non-negotiable.** Even the most autonomous systems (Devin) have 33% of PRs rejected. Every company is investing in better human review UX (Devin Review, Factory's Signals).

---

## Implications for Our Architecture

- **Start with hub-and-spoke.** It's battle-tested and what enterprises expect. A human dispatches, agents execute, results come back.
- **Invest in context retrieval.** SWE-grep-style specialized retrieval is more valuable than complex agent topologies.
- **Build the self-improvement loop early.** Factory's Signals pattern (LLM-as-judge → friction detection → auto-fix) is a competitive moat.
- **Own the sandbox.** Whether VM, Docker, or WebContainer — the execution environment determines what agents can do.
- **Fleet parallelism as a feature.** Design for running N agents in parallel on similar tasks from day one.
- **Playbooks > prompts.** Devin's playbook system (human writes once, fleet executes many times) is a better UX pattern than per-task prompting.

---

*Last updated: 2026-02-08*
