# Code Generation Quality & Guardrails

> How to ensure AI-generated code is production-quality and build guardrails that prevent agents from shipping garbage.

**Research Date:** 2026-02-08

---

## The Problem

AI coding agents (Copilot, Claude, Cursor, Codex, Devin, etc.) generate code at unprecedented speed. But speed without quality is just fast garbage. Studies consistently show AI-generated code contains more bugs, more security vulnerabilities, and more style inconsistencies than carefully-written human code. The challenge isn't generating code—it's ensuring that generated code meets production standards before it reaches users.

**Key failure modes of AI-generated code:**
- Security vulnerabilities (SQL injection, XSS, insecure defaults, hardcoded secrets)
- Hallucinated APIs or deprecated function calls
- Subtle logic errors that pass tests but fail in edge cases
- Style/convention drift from existing codebase
- Unnecessary dependencies or bloated implementations
- Architectural violations (wrong layer, wrong pattern, wrong abstraction)
- Missing error handling, logging, observability
- License-incompatible code snippets

---

## The Guardrail Stack

Think of guardrails as concentric rings of defense. Each layer catches what the previous one missed.

```
┌─────────────────────────────────────────┐
│  1. PROMPT-LEVEL GUARDRAILS             │  ← Before generation
│     (system prompts, rules, context)    │
├─────────────────────────────────────────┤
│  2. INLINE / IDE GUARDRAILS             │  ← During generation
│     (LSP, real-time linting, copilot)   │
├─────────────────────────────────────────┤
│  3. PRE-COMMIT GUARDRAILS               │  ← Before commit
│     (linting, formatting, type checks)  │
├─────────────────────────────────────────┤
│  4. CI/CD GUARDRAILS                    │  ← Before merge
│     (SAST, tests, review agents)        │
├─────────────────────────────────────────┤
│  5. POST-MERGE GUARDRAILS               │  ← Before deploy
│     (DAST, staging tests, canary)       │
└─────────────────────────────────────────┘
```

---

## Layer 1: Prompt-Level Guardrails (Before Generation)

The cheapest and most effective guardrail is preventing bad code from being generated in the first place.

### System Prompts & Rules Files

- **AGENTS.md / .cursorrules / CLAUDE.md** — Project-level instructions that tell the AI about conventions, architecture, patterns
- **Explicit constraints**: "Never use `any` in TypeScript", "Always use parameterized queries", "Follow the repository pattern for data access"
- **Architectural context**: Describe the system's layers, which modules can import what, naming conventions

### Context Engineering

- Feed the agent relevant existing code as examples (few-shot)
- Provide the project's `.editorconfig`, `tsconfig.json`, `eslint.config.js` etc. as context
- Reference specific style guides or ADRs (Architecture Decision Records)
- Include test examples so the agent generates testable code

### Key Insight

> The single highest-ROI guardrail is a well-written rules file. An agent with good instructions generates dramatically better code than one flying blind. Invest heavily here.

---

## Layer 2: Inline / IDE Guardrails (During Generation)

### Language Server Protocol (LSP)

Real-time type checking and error detection as code is generated:
- **TypeScript** (`tsc --watch`) — catches type errors immediately
- **Pyright / mypy** — Python type checking
- **rust-analyzer** — Rust's borrow checker catches memory issues
- **gopls** — Go's LSP

### Real-Time Linting

- **ESLint** (JS/TS) — style + logical errors, highly configurable
- **Ruff** (Python) — extremely fast Python linter, replaces flake8/isort/black
- **Clippy** (Rust) — idiomatic Rust enforcement
- **golangci-lint** (Go) — meta-linter aggregator

### IDE Integration

Modern AI coding tools (Cursor, Windsurf, Copilot) can be configured to run linters and type checkers on generated code before presenting it to the user. This is an underutilized capability.

---

## Layer 3: Pre-Commit Guardrails (Before Commit)

### Pre-commit Hooks

Use **pre-commit** (https://pre-commit.com) or **Husky** + **lint-staged** to enforce checks before code enters the repo:

```yaml
# .pre-commit-config.yaml
repos:
  - repo: https://github.com/pre-commit/pre-commit-hooks
    hooks:
      - id: trailing-whitespace
      - id: check-merge-conflict
      - id: detect-private-key        # Catch secrets
      - id: check-added-large-files
  - repo: https://github.com/astral-sh/ruff-pre-commit
    hooks:
      - id: ruff                       # Lint
      - id: ruff-format               # Format
  - repo: https://github.com/pre-commit/mirrors-mypy
    hooks:
      - id: mypy                       # Type check
```

### Formatting

Non-negotiable. AI agents should never argue about formatting:
- **Prettier** (JS/TS/CSS/HTML/JSON/YAML)
- **Black / Ruff format** (Python)
- **gofmt** (Go — built into the language)
- **rustfmt** (Rust)

### Type Checking

| Language | Tool | Strictness |
|----------|------|------------|
| TypeScript | `tsc --strict` | Enable all strict flags |
| Python | `mypy --strict` or `pyright` | Gradually increase strictness |
| Rust | `cargo check` | Compiler enforces by default |
| Go | `go vet` + `staticcheck` | Built-in + community |
| Java/Kotlin | Compiler + NullAway | Null-safety annotations |
| C# | Nullable reference types | Enable in `.csproj` |

**Key Insight:** Type checking is the single most effective automated guardrail for correctness. Strongly-typed codebases with strict settings catch entire categories of AI-generated bugs at compile time.

---

## Layer 4: CI/CD Guardrails (Before Merge)

This is where the heavy artillery lives. Every PR from an AI agent should pass through all of these.

### 4a. Static Application Security Testing (SAST)

Scans source code for security vulnerabilities without executing it.

**Tools:**
| Tool | Type | Notes |
|------|------|-------|
| **Semgrep** | SAST | Best-in-class for custom rules. 35+ languages. Fast. OSS core. |
| **SonarQube** | SAST + Quality | Industry standard. Quality gates. Technical debt tracking. |
| **CodeQL** (GitHub) | SAST | Deep semantic analysis. Free for OSS on GitHub. |
| **Bandit** | SAST (Python) | Python-specific security linter |
| **Brakeman** | SAST (Ruby) | Rails-specific |
| **gosec** | SAST (Go) | Go security checker |
| **SpotBugs** | SAST (Java) | Bytecode analysis |

**Semgrep** deserves special mention. It lets you write custom rules in a YAML DSL that match code patterns. You can encode your entire organization's security policies as Semgrep rules:

```yaml
rules:
  - id: no-raw-sql
    patterns:
      - pattern: db.execute($QUERY)
      - pattern-not: db.execute($QUERY, $PARAMS)
    message: Use parameterized queries to prevent SQL injection
    severity: ERROR
```

### 4b. Software Composition Analysis (SCA) & Dependency Auditing

AI agents love adding dependencies. Every new dependency is an attack surface.

**Tools:**
| Tool | What It Does |
|------|-------------|
| **Snyk** | Vulnerability scanning for deps, containers, IaC. Commercial leader. |
| **Dependabot** (GitHub) | Auto-PRs for vulnerable dependencies. Free on GitHub. |
| **Renovate** | More configurable Dependabot alternative. |
| **npm audit / pip-audit / cargo audit** | Language-native vulnerability checks |
| **Trivy** (Aqua) | All-in-one scanner: deps, containers, IaC, secrets. OSS. |
| **Socket** | Detects supply-chain attacks (typosquatting, malicious packages) |
| **OSV-Scanner** (Google) | Uses the Open Source Vulnerabilities database |

**Policy recommendations for AI agents:**
- Block PRs that add new dependencies without justification
- Require license compatibility checks (e.g., no GPL in proprietary codebases)
- Set maximum dependency age/popularity thresholds
- Auto-reject packages with known vulnerabilities

### 4c. Secrets Detection

AI agents sometimes hallucinate or copy secrets into code.

**Tools:**
- **Gitleaks** — scans git history for secrets. Fast, configurable, OSS.
- **TruffleHog** — finds secrets across git, S3, etc. Deep scanning.
- **Semgrep Secrets** — integrated into Semgrep platform
- **GitHub Secret Scanning** — built into GitHub, covers 200+ token types
- **detect-secrets** (Yelp) — lightweight pre-commit hook

### 4d. Test Requirements

For AI-generated code, enforce stricter test requirements than for human code:

- **Minimum test coverage thresholds** — new code must have ≥80% line coverage
- **Mutation testing** (Stryker, mutmut, cargo-mutants) — verify tests actually catch bugs, not just cover lines
- **Property-based testing** (Hypothesis, fast-check) — generate edge cases automatically
- **Integration tests** — AI agents often write code that works in isolation but breaks integration

### 4e. AI Code Review Agents

Automated code review specifically designed for AI-generated code:

| Tool | Approach |
|------|----------|
| **Qodo (formerly CodiumAI)** | AI-powered test generation and code review |
| **CodeRabbit** | AI code review bot for GitHub/GitLab PRs |
| **Ellipsis** | AI code review with custom standards |
| **Greptile** | Codebase-aware AI review |
| **Graphite Reviewer** | AI reviewer integrated with Graphite stacking |
| **SonarQube AI Code Assurance** | Tags AI-generated code and applies extra scrutiny |
| **GitHub Copilot Code Review** | Native GitHub AI review |

**The human-in-the-loop question:** For high-stakes code (auth, payments, data handling), require human review even if AI review passes. For low-risk code (tests, config, boilerplate), AI review alone may suffice.

### 4f. Architectural Consistency

AI agents drift architecturally. They put business logic in controllers, skip the service layer, create circular dependencies.

**Tools & Approaches:**
- **ArchUnit** (Java/Kotlin) — write architecture tests: "no class in package `controller` may depend on `repository`"
- **Dependency-cruiser** (JS/TS) — validate and visualize dependency rules
- **Import-linter** (Python) — enforce import contracts between layers
- **Custom Semgrep rules** — pattern-match architectural violations
- **Module boundaries** in monorepo tools (Nx, Turborepo) — enforce which modules can import what

Example dependency-cruiser rule:
```json
{
  "forbidden": [{
    "name": "no-controller-to-repo",
    "from": { "path": "^src/controllers" },
    "to": { "path": "^src/repositories" },
    "comment": "Controllers must go through services, not directly to repos"
  }]
}
```

---

## Layer 5: Post-Merge Guardrails (Before/After Deploy)

### Dynamic Application Security Testing (DAST)

Tests the running application for vulnerabilities:

| Tool | Notes |
|------|-------|
| **OWASP ZAP** | Industry standard DAST. OSS. |
| **Burp Suite** | Commercial DAST leader |
| **Nuclei** | Fast, template-based vulnerability scanner |
| **Dastardly** (PortSwigger) | Free CI/CD DAST scanner |

### Runtime Guardrails

- **Canary deployments** — roll out to small percentage first
- **Feature flags** (LaunchDarkly, Unleash, Flagsmith) — gate AI-generated features
- **Error rate monitoring** (Sentry, Datadog) — auto-rollback if error rate spikes
- **Runtime application self-protection (RASP)** — block attacks at runtime

---

## Building the Pipeline: Practical Architecture

### For a TypeScript/Node.js Project

```
Agent generates code
    ↓
prettier --check                    # Format
eslint --max-warnings 0             # Lint (zero tolerance)
tsc --strict --noEmit               # Type check
    ↓
pre-commit hooks pass
    ↓
PR created → CI triggers:
  ├── vitest run --coverage         # Tests + coverage
  ├── semgrep scan                  # SAST
  ├── gitleaks detect               # Secrets
  ├── npm audit --audit-level=high  # Dep vulnerabilities
  ├── dependency-cruiser --validate # Architecture
  ├── CodeRabbit / AI review        # AI code review
  └── [human review if high-risk]   # Human review
    ↓
Merge → deploy to staging
  ├── Integration tests
  ├── OWASP ZAP scan               # DAST
  └── Smoke tests
    ↓
Canary deploy (5%)
  ├── Error rate monitoring
  └── Auto-rollback if >threshold
    ↓
Full deploy
```

### For a Python Project

```
Agent generates code
    ↓
ruff check + ruff format           # Lint + format
mypy --strict                      # Type check
    ↓
CI:
  ├── pytest --cov --cov-fail-under=80
  ├── semgrep + bandit              # SAST
  ├── pip-audit                     # Dep audit
  ├── gitleaks                      # Secrets
  ├── import-linter                 # Architecture
  └── AI review
```

---

## Agent-Specific Guardrails

Beyond traditional CI/CD, AI coding agents need additional guardrails:

### 1. Scope Limiting
- Restrict which files/directories the agent can modify
- Prevent agents from modifying security-critical code (auth, crypto, permissions)
- Limit the size of changes (large diffs = higher risk)

### 2. Action Approval
- Require human approval for: adding dependencies, modifying CI/CD config, changing database schemas, modifying IAM/permissions
- Auto-approve: test files, documentation, formatting-only changes

### 3. Sandboxing
- Run agent-generated code in isolated environments (containers, VMs)
- Restrict network access during code generation
- Use read-only filesystem access where possible

### 4. Audit Trail
- Log every code change with the generating model + prompt
- Tag AI-generated code in git (trailer or co-author)
- Track which guardrails caught what (measure effectiveness)

### 5. Feedback Loops
- When a guardrail catches an issue, feed the error back to the agent for self-correction
- Track repeat failures to improve system prompts
- Build a corpus of "rejected code" to improve future generation

---

## The Cost-Benefit Reality

| Guardrail | Setup Cost | Ongoing Cost | Bug-Catching Value | Recommendation |
|-----------|-----------|-------------|-------------------|----------------|
| Rules file / system prompt | Low | Low | Very High | **Must have** |
| Formatting (Prettier etc.) | Trivial | Zero | Low (style only) | **Must have** |
| Type checking (strict) | Medium | Low | Very High | **Must have** |
| Linting (ESLint/Ruff) | Low | Low | High | **Must have** |
| Pre-commit hooks | Low | Low | Medium | **Must have** |
| SAST (Semgrep) | Medium | Low | High | **Must have** |
| Secrets detection | Low | Low | Critical for security | **Must have** |
| Dependency auditing | Low | Low | High | **Must have** |
| AI code review | Low | Medium ($) | Medium-High | **Should have** |
| Architecture tests | Medium | Medium | High (long-term) | **Should have** |
| Mutation testing | Medium | High (slow) | Very High | Nice to have |
| DAST | High | Medium | Medium | For web apps |
| RASP | High | High | Medium | Enterprise |

---

## Anti-Patterns to Avoid

1. **"The agent wrote tests so it must be fine"** — AI-generated tests often test the happy path only, or worse, are tautological (testing that the code does what it does)
2. **Disabling strict mode to make AI code compile** — Fix the code, not the config
3. **Trusting AI code review of AI code without human oversight** — Especially for security-critical paths
4. **No guardrails in dev, all guardrails in CI** — Shift left. Catch issues at generation time, not 20 minutes later in CI
5. **One-size-fits-all policies** — Low-risk code (tests, docs) needs fewer guardrails than auth/payments code
6. **Ignoring architectural drift** — AI agents create technical debt faster than humans. Enforce boundaries early.

---

## Recommendations for the Masterplan

1. **Start with the "must haves"** — rules files, type checking, linting, SAST, secrets scanning. These give 80% of the value for 20% of the effort.

2. **Treat AI-generated PRs as untrusted by default** — Apply stricter CI gates to agent-created PRs than human ones (or at least equal).

3. **Invest in Semgrep custom rules** — Encode your specific security and architectural policies. This is the most flexible guardrail tool.

4. **Build the feedback loop** — When guardrails catch issues, pipe the errors back into the agent. Most agents can self-correct if told what's wrong.

5. **Tag and track AI-generated code** — You need to know what percentage of your codebase is AI-generated and what its defect rate is.

6. **Graduated trust** — Start agents with narrow scope and strict guardrails. Expand autonomy as confidence grows. Never give an agent write access to prod infra without human approval.

7. **Architecture tests are underrated** — They prevent the slow rot that happens when AI agents don't understand your system's design intent.

---

## Key Tools Summary

| Category | Top Picks |
|----------|-----------|
| **Linting** | ESLint, Ruff, Clippy, golangci-lint |
| **Type Checking** | TypeScript strict, mypy/Pyright, rust-analyzer |
| **Formatting** | Prettier, Ruff format, gofmt, rustfmt |
| **SAST** | Semgrep, SonarQube, CodeQL |
| **SCA / Deps** | Snyk, Trivy, Socket, Dependabot/Renovate |
| **Secrets** | Gitleaks, TruffleHog, GitHub Secret Scanning |
| **AI Code Review** | CodeRabbit, Qodo, Ellipsis, Copilot Review |
| **Architecture** | ArchUnit, dependency-cruiser, import-linter |
| **DAST** | OWASP ZAP, Nuclei, Dastardly |
| **Pre-commit** | pre-commit framework, Husky + lint-staged |

---

*Quality is not what you add at the end. It's what you prevent from the beginning.*
