# 09 — QA & Testing Framework

> The testing agent MUST be adversarial to the coding agent.
> Same model writing code and tests = same blind spots. Hawk exists to break what Rex builds.

---

## 1. Core Principle: Adversarial Separation

Rex (coding agent) and Hawk (QA agent) are architecturally separated with opposing incentives:

- **Rex** is rewarded for shipping features that pass quality gates
- **Hawk** is rewarded for finding defects, surviving mutants, and uncovered edge cases
- They never share prompts, system instructions, or chain-of-thought
- Different models or different personas with different temperature/reasoning settings
- Hawk never sees Rex's reasoning — only the diff, the spec, and the running code

This isn't optional separation. It's the entire point. When one agent writes both code and tests, it encodes identical assumptions in both. The tests pass because they share the same misunderstanding.

---

## 2. Test Generation Pipeline

```
Rex opens PR
    │
    ▼
┌─────────────────────────────────────────────┐
│  STAGE 0: TRIAGE (< 5 seconds)             │
│                                             │
│  Hawk classifies the change:                │
│  • Files touched, complexity delta          │
│  • Security-sensitive paths (auth, payments,│
│    crypto, permissions, DB schemas)          │
│  • Risk score 0-100                         │
│  • Determines depth of analysis             │
└──────────────────┬──────────────────────────┘
                   ▼
┌─────────────────────────────────────────────┐
│  STAGE 1: FAST GATES (< 30 seconds)        │
│                                             │
│  • tsc --strict / mypy --strict             │
│  • ESLint / Ruff (zero warnings)            │
│  • Prettier / Black (formatting)            │
│  • Semgrep (SAST — custom + default rules)  │
│  • Gitleaks (secrets detection)             │
│  • npm audit / pip-audit (dep vulnerabilities)│
│  • dependency-cruiser / import-linter       │
│    (architectural boundary enforcement)     │
│                                             │
│  FAIL → reject immediately, feed errors     │
│  back to Rex for self-correction            │
└──────────────────┬──────────────────────────┘
                   ▼
┌─────────────────────────────────────────────┐
│  STAGE 2: ADVERSARIAL TEST GENERATION       │
│  (1-5 minutes)                              │
│                                             │
│  Hawk reads: PR diff + linked issue/spec +  │
│  existing test suite + production code       │
│                                             │
│  Generates:                                 │
│  a) Boundary tests — off-by-one, empty,     │
│     null, max-size, negative, unicode,      │
│     concurrent access                       │
│  b) Property-based tests — invariants,      │
│     roundtrips, idempotency, commutativity  │
│     (Hypothesis / fast-check)               │
│  c) Regression tests — for bug fixes,       │
│     verify the original bug is caught       │
│  d) Specification compliance — does the     │
│     implementation match the issue/ticket?  │
│  e) Negative tests — inputs that SHOULD     │
│     fail, error paths, malformed data       │
│                                             │
│  Key: tests derived from the SPEC, not      │
│  the implementation. Hawk tests intent,     │
│  not what Rex happened to write.            │
└──────────────────┬──────────────────────────┘
                   ▼
┌─────────────────────────────────────────────┐
│  STAGE 3: TEST EXECUTION (2-10 minutes)     │
│                                             │
│  • Rex's tests (must all pass)              │
│  • Hawk's adversarial tests (must all pass) │
│  • Existing test suite (no regressions)     │
│  • Property-based suite (100+ random inputs)│
│  • Snapshot tests (flag output changes)     │
│                                             │
│  If Hawk's tests FAIL: that's a real defect.│
│  Send back to Rex with failing test + repro.│
│  Rex must fix AND Hawk re-verifies.         │
└──────────────────┬──────────────────────────┘
                   ▼
┌─────────────────────────────────────────────┐
│  STAGE 4: MUTATION TESTING (5-20 minutes)   │
│  (Changed files only — not full repo)       │
│                                             │
│  Tools: Stryker (JS/TS), mutmut (Python)    │
│                                             │
│  • Flip operators, remove lines, change     │
│    constants, negate conditions              │
│  • Run combined test suite against mutants  │
│  • Surviving mutants = test gaps            │
│  • Required mutation score: ≥ 80%           │
│                                             │
│  If score < threshold:                      │
│  • Hawk generates targeted tests to kill    │
│    surviving mutants                        │
│  • OR sends surviving mutants to Rex with   │
│    "your tests don't catch this change"     │
└──────────────────┬──────────────────────────┘
                   ▼
┌─────────────────────────────────────────────┐
│  STAGE 5: VISUAL & BEHAVIORAL (if UI)       │
│  (2-10 minutes)                             │
│                                             │
│  Three-layer visual verification:           │
│                                             │
│  1. Playwright pixel-diff                   │
│     toHaveScreenshot() against baselines    │
│     maxDiffPixelRatio: 0.01                 │
│                                             │
│  2. Stagehand exploratory testing           │
│     AI navigates key flows, checks for      │
│     broken layouts, dead links, error states│
│     Uses act/extract/observe primitives     │
│     Cached selectors for speed; self-heals  │
│     when DOM changes                        │
│                                             │
│  3. LLM visual judgment (high-risk only)    │
│     Screenshot → vision model → semantic    │
│     analysis of "does this look right?"     │
│     Catches issues pixel-diff can't:        │
│     wrong data, confusing UX, misalignment  │
│                                             │
│  Visual changes NEVER auto-approved.        │
│  Hawk flags; human decides.                 │
└──────────────────┬──────────────────────────┘
                   ▼
┌─────────────────────────────────────────────┐
│  STAGE 6: VERDICT                           │
│                                             │
│  Hawk produces:                             │
│  • PASS / FAIL / NEEDS_HUMAN_REVIEW         │
│  • Risk score (0-100)                       │
│  • Defects found with reproduction steps    │
│  • Test coverage delta                      │
│  • Mutation score                           │
│  • Visual regression report (if UI)         │
│  • Specification compliance assessment      │
│  • Confidence level in verdict              │
└─────────────────────────────────────────────┘
```

---

## 3. Quality Gates

Non-negotiable gates that block merge. These are architectural, not advisory — Rex cannot bypass them.

### Gate 1: Static Analysis (MUST PASS)
- Zero type errors (`tsc --strict` / `mypy --strict`)
- Zero lint warnings (ESLint / Ruff with strict config)
- Zero secrets detected (Gitleaks)
- Zero high/critical vulnerabilities in deps
- Zero SAST findings at ERROR severity (Semgrep)
- Architectural boundaries intact (dependency-cruiser)

### Gate 2: Test Suite (MUST PASS)
- All existing tests pass (zero regressions)
- All Rex-authored tests pass
- All Hawk-authored adversarial tests pass
- Property-based tests find no counterexamples

### Gate 3: Coverage (MUST MEET THRESHOLD)
- New code: ≥ 90% line coverage, ≥ 80% branch coverage
- Overall repo coverage: never decreases
- Mutation score on changed files: ≥ 80%

### Gate 4: Visual Regression (IF UI CHANGED)
- Pixel-diff within threshold OR human-approved
- No broken layouts detected by Stagehand exploration
- No error states on key user flows

### Gate 5: Specification Compliance
- Implementation addresses the linked issue/ticket
- No scope creep (unrequested features flagged)
- Error handling matches spec requirements

---

## 4. Graduated Trust Levels

Trust is earned through track record, not declared.

| Level | Name | Criteria to Reach | Hawk Behavior | Merge Policy |
|-------|------|-------------------|---------------|-------------|
| **L0** | Untrusted | Default for new agents | Full pipeline, maximum depth, all stages | Human approval required on everything |
| **L1** | Probationary | 20 PRs merged, 0 post-merge incidents, mutation score avg > 85% | Full pipeline | Auto-merge trivial (docs, config, formatting). Human review for logic changes |
| **L2** | Trusted | 100 PRs merged, < 2% incident rate, 30-day track record | Full pipeline, relaxed depth on low-risk | Auto-merge low + medium risk. Human review on high-risk (auth, payments, data, infra) |
| **L3** | Veteran | 500 PRs merged, < 0.5% incident rate, 90-day track record | Adaptive depth based on risk score | Auto-merge most PRs. Human review only on critical paths + architectural changes |

### Trust Degradation
- Any production incident caused by an agent PR: **drop 1 level immediately**
- 2 incidents in 30 days: **drop to L0**
- Security vulnerability shipped: **drop to L0, require security-focused review for all subsequent PRs for 60 days**

### Risk Score → Review Depth Mapping

| Risk Score | Classification | L0 | L1 | L2 | L3 |
|-----------|---------------|-----|-----|-----|-----|
| 0-20 | Trivial (docs, typos, formatting) | Human | Auto | Auto | Auto |
| 21-50 | Low (tests, simple logic, config) | Human | Human | Auto | Auto |
| 51-75 | Medium (business logic, API changes) | Human | Human | Human | Auto |
| 76-90 | High (auth, data, payments) | Human | Human | Human | Human |
| 91-100 | Critical (security, infra, DB schema) | Human | Human | Human | Human |

---

## 5. Adversarial Test Strategies

### 5.1 Specification-First Testing

Hawk derives tests from the issue/spec BEFORE reading Rex's implementation:

```
Issue: "Add rate limiting to /api/login — max 5 attempts per IP per 15 minutes"

Hawk generates (from spec alone):
- 6th attempt within 15 min returns 429
- 5th attempt succeeds
- 6th attempt from different IP succeeds
- After 15 minutes, attempts reset
- Rate limit survives server restart (if persistent store)
- Concurrent requests from same IP handled correctly
- IPv6 and IPv4 handled equivalently
- X-Forwarded-For / proxy headers respected (or not — per spec)
```

Only THEN does Hawk read the diff to check for additional issues the spec didn't anticipate.

### 5.2 Fault Injection Tests

Hawk specifically tests error handling:
- What happens when the database is unavailable?
- What happens when an external API returns 500?
- What happens with malformed JSON input?
- What happens under memory pressure?
- What happens with concurrent modifications?

### 5.3 Semantic Regression Detection

Beyond "does the test pass", Hawk checks behavioral equivalence:
- Run the same API calls before/after the change
- Compare response shapes, status codes, headers
- Detect unintended side effects (new env vars, changed configs, altered DB schemas)
- Flag behavioral changes that aren't covered by the spec

### 5.4 AI-Specific Vulnerability Patterns

Hawk maintains a checklist of common AI-generated code mistakes:
- String interpolation in SQL (→ injection)
- Missing input sanitization (→ XSS)
- Hardcoded credentials that look plausible
- `any` types / type assertions that bypass safety
- Missing `await` on async calls
- Error swallowing (`catch {}` with no handling)
- Race conditions from naive async patterns
- Deprecated API usage / hallucinated functions

---

## 6. Visual Regression with Stagehand

### Architecture

```
UI change detected in PR
    │
    ▼
Build Storybook / dev server from PR branch
    │
    ├── Playwright: capture screenshots of all affected components
    │   Compare against golden baselines (pixel-diff)
    │
    ├── Stagehand: exploratory navigation
    │   act("navigate to dashboard")
    │   act("click each nav item")
    │   extract("are there any error messages or broken elements?", schema)
    │   observe("what interactive elements are available?")
    │
    └── Vision LLM (high-risk only):
        Screenshot → "Does this look correct? Check for:
        broken layouts, overlapping elements, missing images,
        error messages, empty states, accessibility issues"
```

### Stagehand Caching Strategy

First run per component: AI discovers selectors (costs tokens, ~2-5s per action).
Subsequent runs: cached selectors replayed deterministically (free, ~100ms per action).
Page redesign: cache miss → AI re-discovers → re-caches. Self-healing.

This means visual testing is expensive only once per UI state, then free until the UI changes — which is exactly when you want re-verification.

### What Hawk Checks Visually

1. **Layout integrity** — elements not overlapping, proper spacing, responsive breakpoints
2. **Content correctness** — right data shown, no placeholder text, no "undefined"
3. **Interactive elements** — buttons clickable, forms submittable, links working
4. **Error states** — graceful degradation, proper error messages
5. **Accessibility** — sufficient contrast, focusable elements, screen reader compatibility
6. **Cross-browser** — Chromium + Firefox + WebKit via Playwright

---

## 7. Feedback Loop: Hawk → Rex

When Hawk finds defects, the feedback must be actionable:

```json
{
  "verdict": "FAIL",
  "defects": [
    {
      "severity": "high",
      "type": "edge_case_failure",
      "description": "Rate limiter allows 6th request when requests arrive simultaneously",
      "test_file": "hawk-generated/rate-limit-concurrent.test.ts",
      "reproduction": "Send 6 concurrent POST /api/login from same IP — all succeed",
      "suggested_fix": "Use atomic increment in Redis rather than GET+SET",
      "line_reference": "src/middleware/rateLimiter.ts:42"
    }
  ],
  "surviving_mutants": [
    {
      "file": "src/middleware/rateLimiter.ts",
      "line": 38,
      "mutation": "Changed `>=` to `>` in limit check",
      "implication": "Off-by-one: allows maxAttempts+1 requests"
    }
  ],
  "coverage": {
    "line": 94,
    "branch": 78,
    "mutation_score": 72,
    "verdict": "BELOW_THRESHOLD (mutation_score < 80)"
  }
}
```

Rex receives this, fixes, re-submits. Hawk re-runs the full pipeline. No shortcutting.

---

## 8. Post-Merge Monitoring

Quality gates don't end at merge. Hawk watches production:

1. **Canary deployment** — agent PRs deploy to 5% traffic first
2. **Error rate monitoring** — if error rate > baseline + threshold → auto-rollback
3. **Behavioral monitoring** — response time, status code distribution, log anomalies
4. **Incident correlation** — when an incident occurs, trace back to the PR that caused it → update agent trust level

---

## 9. Implementation Priorities

### Phase 1: Foundation (Week 1-2)
- [ ] Fast gates in CI: type checking, linting, formatting, Semgrep, Gitleaks
- [ ] Coverage enforcement: no-decrease rule + 80% minimum on new code
- [ ] AGENTS.md with strict testing requirements for Rex
- [ ] Hawk as a CI reviewer that comments on PRs (read-only, advisory)

### Phase 2: Adversarial Testing (Week 3-4)
- [ ] Hawk generates adversarial tests from PR diff + spec
- [ ] Property-based test generation for business logic
- [ ] Mutation testing on changed files (Stryker / mutmut)
- [ ] Trust level tracking (start everything at L0)

### Phase 3: Visual & Behavioral (Month 2)
- [ ] Playwright visual regression in CI
- [ ] Stagehand exploratory testing for UI changes
- [ ] Before/after behavioral diffing for API changes
- [ ] Contract testing at API boundaries (Pact / Schemathesis)

### Phase 4: Full Autonomy (Month 3+)
- [ ] Graduated trust with auto-merge for passing low-risk PRs
- [ ] Post-merge production monitoring with auto-rollback
- [ ] Self-improving test corpus (Hawk learns from past incidents)
- [ ] LLM visual judgment for major releases
- [ ] Trust dashboard: agent quality scores, mutation trends, incident rates

---

## 10. Key Design Decisions

1. **Hawk and Rex use different models** (or at minimum, different system prompts with no shared context). Same model same prompt = same blind spots.

2. **Tests from spec, not implementation.** Hawk derives tests from the issue/ticket before reading code. This catches "correctly implemented the wrong thing."

3. **Mutation score > line coverage.** Line coverage is gameable. Mutation score measures whether tests actually verify behavior. It's the ground truth metric.

4. **Visual changes always require human approval.** No auto-merge for UI changes. Pixel-diff and LLM judgment flag; humans decide.

5. **Trust degrades faster than it builds.** One incident drops a level. Earning it back takes weeks of clean PRs. This is intentional — the cost of a bad merge is higher than the cost of extra review.

6. **Quality gates are architectural, not advisory.** Rex cannot skip stages, disable checks, or modify the CI pipeline. The gates are enforced by infrastructure, not by good intentions.

7. **Feedback is structured, not prose.** Hawk returns machine-parseable defect reports so Rex can programmatically address them. No ambiguous "looks off" — specific file, line, reproduction, suggested fix.

---

*The factory is only as reliable as its QA. Rex builds fast. Hawk ensures Rex builds correctly. Neither trusts the other, and that's exactly right.*
