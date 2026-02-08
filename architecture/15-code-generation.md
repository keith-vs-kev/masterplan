# 15 — Project Scaffolding & Code Generation

> Architecture document for how Rex (coding agent) generates, scaffolds, and maintains projects within the autonomous agentic factory.

**Author:** Rex (Claude)
**Date:** 2026-02-08

---

## Executive Summary

Rex doesn't just write code — it creates entire projects from scratch, maintains consistency across a growing portfolio, and orchestrates multiple coding agents in parallel. This document defines the scaffolding system, template architecture, multi-agent routing strategy, and quality gates that make autonomous code generation reliable and repeatable.

---

## 1. Project Scaffolding System

### 1.1 Template Registry

All project templates live in a central registry (monorepo: `templates/`). Each template is a cookiecutter-style scaffold with:

```
templates/
├── base/                          # Shared across ALL project types
│   ├── .editorconfig
│   ├── .gitignore
│   ├── .prettierrc
│   ├── eslint.config.js           # or ruff.toml for Python
│   ├── AGENTS.md.tmpl             # Project-level agent instructions
│   ├── CLAUDE.md.tmpl
│   ├── GEMINI.md.tmpl
│   ├── Makefile.tmpl              # Unified task runner
│   ├── .github/
│   │   ├── workflows/ci.yml.tmpl
│   │   ├── CODEOWNERS.tmpl
│   │   └── dependabot.yml
│   └── .semgrep/
│       └── custom-rules.yml
├── ts-api/                        # TypeScript REST/GraphQL API
│   ├── scaffold.json              # Metadata: name, description, variables
│   ├── tsconfig.json
│   ├── vitest.config.ts
│   ├── src/
│   │   ├── index.ts
│   │   ├── routes/
│   │   ├── services/
│   │   ├── repositories/
│   │   └── middleware/
│   └── Dockerfile
├── ts-nextjs/                     # Next.js full-stack app
├── ts-library/                    # Publishable npm package
├── py-api/                        # Python FastAPI service
├── py-cli/                        # Python CLI tool
├── openclaw-plugin/               # OpenClaw plugin/integration
└── monorepo-workspace/            # Nx/Turborepo workspace root
```

### 1.2 Scaffold Manifest (`scaffold.json`)

Each template declares its variables, dependencies, and post-scaffold hooks:

```json
{
  "name": "ts-api",
  "description": "TypeScript REST API with Express/Fastify",
  "variables": {
    "projectName": { "type": "string", "required": true },
    "framework": { "type": "enum", "options": ["express", "fastify", "hono"], "default": "hono" },
    "database": { "type": "enum", "options": ["postgres", "sqlite", "none"], "default": "postgres" },
    "orm": { "type": "enum", "options": ["drizzle", "prisma", "none"], "default": "drizzle" },
    "auth": { "type": "boolean", "default": false },
    "queue": { "type": "enum", "options": ["bullmq", "none"], "default": "none" }
  },
  "postScaffold": [
    "npm install",
    "npm run typecheck",
    "git init && git add -A && git commit -m 'initial scaffold'"
  ],
  "guardrails": {
    "requiredCI": ["typecheck", "lint", "test", "sast"],
    "minCoverage": 80,
    "architectureRules": "dependency-cruiser.config.js"
  }
}
```

### 1.3 Scaffolding Flow

```
Kev: "Create a new API for invoice processing"
         │
         ▼
┌─────────────────────────────┐
│  1. Rex selects template     │  ← Based on task description
│     (ts-api)                 │
├─────────────────────────────┤
│  2. Rex resolves variables   │  ← Infers from context or asks Kev
│     projectName: invoice-api │
│     database: postgres       │
│     orm: drizzle             │
├─────────────────────────────┤
│  3. Template engine renders  │  ← Copies base/ + ts-api/, interpolates
├─────────────────────────────┤
│  4. Post-scaffold hooks run  │  ← npm install, typecheck, git init
├─────────────────────────────┤
│  5. Rex generates domain     │  ← Business logic: routes, services,
│     code on top of scaffold  │    schemas, tests, migrations
├─────────────────────────────┤
│  6. Quality gates pass       │  ← lint + typecheck + test + build
├─────────────────────────────┤
│  7. PR created               │
└─────────────────────────────┘
```

---

## 2. Consistency Across Projects

### 2.1 Shared Config Packages

Instead of copying config files, publish them as packages that projects extend:

```
@factory/eslint-config        → ESLint rules (extends in eslint.config.js)
@factory/tsconfig              → Base tsconfig.json (extends)
@factory/prettier-config       → Prettier settings
@factory/semgrep-rules         → Custom SAST rules
@factory/github-actions        → Reusable CI workflow templates
```

Projects consume them:
```json
// tsconfig.json
{ "extends": "@factory/tsconfig/api.json" }

// eslint.config.js
import factoryConfig from '@factory/eslint-config';
export default [...factoryConfig.recommended];
```

**Benefit:** Update one package → all projects get the improvement on next dependency bump. Renovate/Dependabot handles the propagation.

### 2.2 Monorepo vs Polyrepo Decision

| Factor | Monorepo | Polyrepo |
|--------|----------|----------|
| Shared code / libs | ✅ Easy imports | ❌ Publish packages |
| CI complexity | ⚠️ Needs Nx/Turborepo | ✅ Simple per-repo CI |
| Agent parallelism | ⚠️ Git conflicts | ✅ Isolated repos |
| Dependency consistency | ✅ Single lockfile | ❌ Version drift |
| New project setup | ✅ Add workspace | ⚠️ Scaffold from template |

**Recommendation: Hybrid approach.**

- **Monorepo** for tightly coupled projects (e.g., platform services that share types/utils)
- **Polyrepo** for independent products/experiments
- Use **shared config packages** (§2.1) to maintain consistency across polyrepos
- Templates produce monorepo-ready or standalone projects depending on `scaffold.json` config

### 2.3 Tech Stack Standards

Codified defaults — Rex uses these unless Kev overrides:

| Layer | Default | Alternatives (require justification) |
|-------|---------|--------------------------------------|
| **Language** | TypeScript (strict) | Python (for ML/scripting), Go (for infra) |
| **Runtime** | Node.js (LTS) | Bun (experiments only) |
| **API framework** | Hono | Express, Fastify |
| **Frontend** | Next.js (App Router) | Remix, Astro (static sites) |
| **ORM** | Drizzle | Prisma |
| **Database** | PostgreSQL | SQLite (local/embedded), Redis (cache) |
| **Testing** | Vitest + Playwright | Jest (legacy only) |
| **CI** | GitHub Actions | — |
| **Formatting** | Prettier + ESLint flat config | Biome (watching) |
| **Package manager** | pnpm | npm (simple projects) |
| **Containerisation** | Docker (multi-stage) | — |
| **IaC** | Pulumi (TypeScript) | Terraform |

### 2.4 Architecture Decision Records (ADRs)

Every non-trivial decision gets an ADR in `docs/adr/`. Rex generates these as part of scaffolding:

```markdown
# ADR-001: Use Hono as API framework

## Status: Accepted
## Context: Need a lightweight, type-safe HTTP framework
## Decision: Hono — fast, small, edge-compatible, great DX
## Consequences: Team must learn Hono patterns; middleware ecosystem smaller than Express
```

Rex references existing ADRs before making architectural decisions in a project. Prevents re-litigating settled choices.

---

## 3. Multi-Agent Routing Strategy

### 3.1 Agent Selection Matrix

Rex orchestrates multiple coding CLIs, routing tasks to the best tool:

| Task Type | Primary Agent | Why | Fallback |
|-----------|--------------|-----|----------|
| **Complex multi-file refactors** | Claude Code | Best agentic reasoning, deep codebase nav | — |
| **New feature implementation** | Claude Code | Multi-step planning + execution | Codex CLI |
| **Large codebase exploration** | Gemini CLI | 1M token context, free | Claude Code |
| **Quick targeted edits** | Codex CLI | Fast, lightweight, sandboxed | Claude Code |
| **Test generation** | Claude Code `-p` | Scriptable, batch mode | Gemini CLI |
| **Documentation generation** | Gemini CLI | Free tier, good at prose | Claude Code |
| **Code review** | Claude Code + Gemini CLI | Cross-model review catches more | — |
| **Dependency upgrades** | Gemini CLI | Web search grounding for changelog research | — |
| **Bug triage / debugging** | Claude Code | Best at reading stack traces + codebase | — |
| **Boilerplate / scaffolding** | Template engine + Claude Code | Template first, agent fills in domain logic | — |
| **Security audit** | Semgrep + Claude Code | SAST rules + LLM reasoning | — |

### 3.2 Parallel Coding with Git Worktrees

Rex uses git worktrees to run multiple agents simultaneously on the same repo:

```bash
# Rex creates isolated worktrees per task
git worktree add /tmp/worktrees/feature-auth -b feature/auth
git worktree add /tmp/worktrees/feature-billing -b feature/billing
git worktree add /tmp/worktrees/refactor-db -b refactor/db-layer

# Dispatch agents in parallel
claude -p "implement OAuth2 flow" --cwd /tmp/worktrees/feature-auth &
gemini "write API documentation" --cwd /tmp/worktrees/feature-billing &
claude -p "refactor repository layer to use Drizzle" --cwd /tmp/worktrees/refactor-db &

# Wait, then merge results
wait
```

**Conflict resolution:** Rex runs parallel tasks on non-overlapping file sets when possible. When overlap is unavoidable, tasks are sequenced or Rex resolves merge conflicts post-hoc.

### 3.3 Pipeline Orchestration Pattern

For complex features, Rex chains agents in a pipeline:

```
Step 1: PLAN
  Claude Code → reads requirements, generates implementation plan (plan.md)

Step 2: VALIDATE PLAN
  Gemini CLI → reviews plan against best practices, web-searches for pitfalls

Step 3: IMPLEMENT (parallel)
  Claude Code (worktree A) → implements backend
  Claude Code (worktree B) → implements frontend
  Gemini CLI → generates tests from plan

Step 4: INTEGRATE
  Rex merges worktrees, resolves conflicts

Step 5: QUALITY GATES
  typecheck → lint → test → SAST → build

Step 6: CROSS-MODEL REVIEW
  Gemini CLI → reviews Claude's code (catches model-specific blind spots)

Step 7: PR
  Rex creates PR with summary, test instructions, risk notes
```

### 3.4 Cost Optimisation

| Agent | Cost Model | When to Prefer |
|-------|-----------|----------------|
| Gemini CLI | Free (1000 req/day) | Exploration, docs, reviews, low-stakes tasks |
| Codex CLI | ChatGPT subscription | Quick edits, simple tasks |
| Claude Code | API credits or Max plan | High-value work requiring best reasoning |

**Rule:** Use the cheapest agent that can do the job well. Reserve Claude Opus/Sonnet for tasks that actually need it. Use Gemini's free tier aggressively for research, docs, and secondary reviews.

---

## 4. Quality Gates for Generated Code

Every piece of code Rex generates passes through these gates (from research [13-code-quality-guardrails]):

### 4.1 Pre-Generation (Prompt Engineering)

- **AGENTS.md** in every project with architecture rules, naming conventions, forbidden patterns
- **Shared config packages** provide lint/type rules the agent sees in context
- **ADRs** fed as context so agent respects settled decisions
- **Existing code examples** provided as few-shot patterns

### 4.2 Post-Generation (Before PR)

```bash
# Rex runs this checklist on every change — non-negotiable
pnpm run format:check          # Prettier
pnpm run lint                  # ESLint (zero warnings)
pnpm run typecheck             # tsc --strict --noEmit
pnpm run test                  # Vitest
pnpm run build                 # Must compile cleanly
```

### 4.3 CI Pipeline (Before Merge)

```yaml
# Triggered on every PR
jobs:
  quality:
    steps:
      - typecheck
      - lint (zero warnings)
      - test (coverage ≥80%)
      - semgrep scan (SAST)
      - gitleaks (secrets detection)
      - dependency-cruiser (architecture rules)
      - pnpm audit (dependency vulnerabilities)
      - build

  review:
    steps:
      - AI code review (CodeRabbit or cross-model review)
      - Human review (required for: auth, payments, data models, infra)
```

### 4.4 Self-Correction Loop

When a quality gate fails:

```
Gate fails → error message piped back to agent → agent fixes → re-run gates
Max 3 retry cycles before escalating to Kev as BLOCKER
```

---

## 5. Playbook System

Inspired by Devin's playbooks (research [07-factory-patterns]), Rex maintains reusable playbooks for common operations:

```
playbooks/
├── new-api-endpoint.md         # Add a REST endpoint end-to-end
├── new-db-migration.md         # Schema change + migration + rollback
├── add-auth-to-route.md        # Secure an existing endpoint
├── create-npm-package.md       # Scaffold + publish a shared package
├── upgrade-dependency.md       # Safe major version upgrade flow
├── add-monitoring.md           # Instrument a service with metrics
└── security-fix.md             # Patch a vulnerability + verify
```

Each playbook is a step-by-step instruction set that Rex (or a fleet of Rex instances) can execute deterministically. Playbooks evolve: when Rex discovers a better approach, it updates the playbook.

---

## 6. Architecture Patterns Library

Rex draws from a catalogue of proven patterns when generating domain code:

| Pattern | When | Implementation |
|---------|------|----------------|
| **Repository + Service** | Any data-backed API | Repo handles DB, Service handles logic, Route handles HTTP |
| **Command/Query (CQRS-lite)** | Complex domains | Separate read/write paths without full event sourcing |
| **Event-driven** | Cross-service communication | BullMQ jobs, webhook dispatchers |
| **Middleware chain** | Cross-cutting concerns | Auth, logging, rate limiting as composable middleware |
| **Feature modules** | Large apps | Co-locate route+service+repo+test per feature |
| **Plugin architecture** | Extensible systems | Core + plugin interface + registry |

Rex selects patterns based on project type and `scaffold.json` configuration. The AGENTS.md in each project documents which patterns are in use, preventing architectural drift.

---

## 7. Feedback & Continuous Improvement

### 7.1 Metrics to Track

- **Scaffold-to-PR time** — how long from "create project" to first mergeable PR
- **Quality gate pass rate** — % of Rex's PRs that pass CI on first attempt
- **Rework rate** — how often Kev requests changes on Rex's PRs
- **Cross-model review catch rate** — issues found by second-model review
- **Template reuse rate** — how often new projects use existing templates vs custom setup

### 7.2 Template Evolution

```
Rex scaffolds project → ships to production → observes pain points
    → updates template → next project benefits
```

Templates are living artifacts. Every friction point becomes a template improvement.

### 7.3 Self-Improvement Signal (Factory-inspired)

Borrow from Factory AI's Signals pattern:
- After each coding session, log: what worked, what failed, what was slow
- Cluster failures to discover systemic issues (e.g., "Drizzle migrations always need manual fixing")
- Update AGENTS.md, playbooks, or templates to address recurring friction

---

## 8. Implementation Roadmap

### Phase 1: Foundation (Week 1-2)
- [ ] Create `templates/base/` with shared configs
- [ ] Build `ts-api` template with scaffold.json
- [ ] Set up shared config packages (@factory/eslint-config, @factory/tsconfig)
- [ ] Write first 3 playbooks (new-api-endpoint, new-db-migration, add-auth)
- [ ] Establish quality gate script (format + lint + typecheck + test + build)

### Phase 2: Multi-Agent Routing (Week 3-4)
- [ ] Script git worktree management for parallel agent execution
- [ ] Implement pipeline orchestration (plan → validate → implement → review)
- [ ] Set up cross-model review (Claude generates, Gemini reviews)
- [ ] Cost tracking per agent per task

### Phase 3: Scaling (Week 5+)
- [ ] Add templates: ts-nextjs, ts-library, py-api, openclaw-plugin
- [ ] Implement self-correction loop (gate failure → agent retry)
- [ ] Build metrics dashboard (scaffold-to-PR time, pass rates)
- [ ] Template evolution based on production feedback

---

## Key Design Decisions

1. **Templates over generation.** Don't generate project structure from scratch every time. Use templates for the skeleton, agents for the domain logic. Deterministic scaffolding + creative implementation.

2. **Shared packages over copied configs.** A config change should propagate to all projects via dependency updates, not manual copying.

3. **Cheapest capable agent wins.** Gemini CLI for research/docs/reviews (free). Claude Code for complex implementation (best reasoning). Don't burn Opus tokens on documentation.

4. **Cross-model review is mandatory.** No single model reviews its own output. Different models catch different classes of errors.

5. **Build before done. Always.** (Per AGENTS.md lesson learned 2026-01-19.) If it doesn't build, it's not done.

---

*The factory doesn't ship code. It ships verified, tested, architecturally-consistent code. Everything else is just text generation.*
