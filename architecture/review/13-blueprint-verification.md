# 13 — Integration Blueprint Verification

> Cross-verification of `20-integration-blueprint.md` against all architecture docs (01–19).
> **Date:** 2026-02-08 | **Reviewer:** Hawk | **Status:** ISSUES FOUND

---

## Summary

The Integration Blueprint (doc 20) has **12 material discrepancies** against the detailed architecture documents. Most stem from the Blueprint describing an earlier/simpler design that was superseded by the detailed docs. The Blueprint needs a significant update to reflect actual architectural decisions.

**Severity key:** 🔴 Wrong (contradicts arch doc) | 🟡 Missing (not mentioned) | 🟢 Aligned

---

## 1. Task Queue — 🔴 WRONG

**Blueprint says:** SQLite + JSON files for task queue (§1, §3.1, §4)
**Doc 04 says:** PostgreSQL with `FOR UPDATE SKIP LOCKED`, `LISTEN/NOTIFY`, DAG-based dependencies, claim-based distribution

**Details:** This is a fundamental contradiction. Doc 04 explicitly rejects simple file-based queues in favor of PostgreSQL for ACID guarantees, atomic claiming, and real-time notification. The Blueprint's "SQLite/JSON" model appears nowhere in doc 04. Doc 04 even addresses this directly: *"PostgreSQL is already in our stack"* and uses SQL indexes, advisory locks, and NOTIFY — none of which work with SQLite/JSON files.

**Correction:** Replace all references to "SQLite/JSON" task queue with "PostgreSQL" task queue. Update the Shared State Layer diagram and §4 tech stack table.

---

## 2. Memory/Knowledge System — 🔴 WRONG

**Blueprint says:** ChromaDB + filesystem (§1, §3.1, §4)
**Doc 05 says:** Qdrant (vector store) + Neo4j (knowledge graph) + custom MCP memory server

**Details:** Doc 05 explicitly chose Qdrant over ChromaDB (*"Rust performance, hybrid search, production-ready clustering"*) and added Neo4j for relationship graphs. The Blueprint's "ChromaDB" is the wrong technology. Doc 05 also specifies a custom MCP memory server as the interface layer — agents don't touch databases directly.

**Correction:** Replace "ChromaDB" with "Qdrant" for vector store. Add Neo4j as the knowledge graph component. Add "MCP Memory Server" as the access layer. Update deployment topology (§5) with correct ports and resource allocations.

---

## 3. LLM Routing Layer — 🔴 WRONG

**Blueprint says:** Pi SDK as the execution engine that wraps API calls with token counters (§1, §2, §4, §7.2)
**Doc 02 says:** LiteLLM Proxy as the routing foundation — all calls flow through LiteLLM, never directly to providers
**Doc 08 says:** LiteLLM as the LLM Gateway with budget enforcement, kill switches, and Langfuse tracing
**Doc 13 says:** LiteLLM Proxy as the unified API surface for hybrid local/cloud routing

**Details:** Three architecture docs (02, 08, 13) consistently specify LiteLLM as the central routing/proxy layer. The Blueprint doesn't mention LiteLLM once. "Pi SDK" appears to be an OpenClaw-level execution wrapper, not the LLM routing layer. The Blueprint conflates agent session management (OpenClaw/Pi SDK) with LLM call routing (LiteLLM).

**Correction:** Add LiteLLM Proxy as a distinct component in the architecture diagrams between agents and LLM providers. Clarify that Pi SDK manages agent sessions while LiteLLM handles model routing, budget enforcement, and provider failover.

---

## 4. Monitoring & Observability — 🔴 WRONG

**Blueprint says:** "Custom (SQLite metrics + REEF dashboard)" for monitoring (§4, §7.6)
**Doc 08 says:** LiteLLM + Langfuse (self-hosted) + Prometheus + Grafana + Redis for real-time state
**Doc 18 says:** TimescaleDB as the primary analytical store, with Redis accumulators and Grafana dashboards

**Details:** The Blueprint describes a toy monitoring stack (SQLite + custom dashboard). Docs 08 and 18 specify a professional stack: Langfuse for LLM tracing, Prometheus/Grafana for metrics and alerting, Redis for real-time budget enforcement and kill switches, and TimescaleDB for business analytics. The Blueprint's phased approach (§7.6) is roughly directionally correct but the technology choices are wrong.

**Correction:** Replace "SQLite metrics" with "TimescaleDB + Prometheus + Langfuse." Replace "REEF dashboard" with "Grafana dashboards (with REEF as optional Electron overlay)." Add Redis for real-time budget state and kill switches.

---

## 5. Agent Communication — 🔴 WRONG

**Blueprint says:** "OpenClaw subagents + filesystem" for agent comms (§4)
**Doc 11 says:** NATS + JetStream as the communication bus, with structured message envelopes, queue groups, pub/sub, and dead-letter queues

**Details:** Doc 11 specifies a full event-driven communication bus using NATS. The Blueprint doesn't mention NATS at all. While OpenClaw subagent spawning is the current implementation, doc 11 defines the target architecture with proper async messaging, capability-based routing, and delivery guarantees.

**Correction:** Add NATS + JetStream to the architecture diagram as the agent communication bus. Note OpenClaw subagents as the Phase 1 implementation with NATS as the Phase 2+ target.

---

## 6. Agent Roster — 🟡 INCOMPLETE

**Blueprint says:** 13+ agents: "Atlas · Rex · Forge · Scout · Hawk · Pixel · Blaze · Echo · Chase · Finn · Dash · Dot · Law" with Kev as orchestrator
**Doc 03 says:** 14 agents with Rex as orchestrator, Atlas as personal assistant, and Ally (support agent) included

**Discrepancies:**
- Blueprint lists Kev as the orchestrator. Doc 03 has Rex as the orchestrator. Kev appears to be an OpenClaw-level agent not defined in doc 03.
- Blueprint omits **Ally** (support/customer success agent, using claude-haiku-3.5).
- Blueprint says "13+" but doc 03 defines exactly 14 agents.
- The Kev/Rex orchestrator role overlap is unresolved — is Kev the human-facing orchestrator (OpenClaw) while Rex is the task-decomposition orchestrator (factory internal)?

**Correction:** Clarify the Kev vs Rex distinction. Add Ally to the agent roster diagram. List all 14 agents explicitly. If Kev and Rex are separate roles, document the boundary.

---

## 7. Security Model — 🟡 OVERSIMPLIFIED

**Blueprint says:** "Scoped API tokens + filesystem permissions + egress filtering" (§4, §7.1)
**Doc 07 says:** Full capability-based access control (ocap), Docker/Firecracker sandboxing, HashiCorp Vault for secrets, trust levels L0–L4, dual-LLM pattern for prompt injection defense, per-agent network namespaces, capability tokens with 15-min TTL

**Details:** The Blueprint's security description is a one-line summary that misses the majority of doc 07's security architecture. Critical omissions: capability tokens (not just "scoped API tokens"), sandbox tiers (Standard/Hardened/Maximum), secret management via Vault (agents never see raw secrets), and the trust ladder system.

**Correction:** Expand §7.1 to reference capability-based access control, sandbox tiers, Vault-based secret management, and the trust level system. At minimum, the tech stack table should list HashiCorp Vault and Docker/Firecracker.

---

## 8. MCP Integration — 🟡 OVERSIMPLIFIED

**Blueprint says:** MCP mentioned in interface contracts table (§2.2) with one line
**Doc 14 says:** Centralized MCP Client Gateway, tool namespacing (server.tool_name), role-based + tool-level authorization, custom internal MCP servers (task-queue, memory, deploy, agents, metrics), credential management via Vault

**Details:** MCP is a major architectural layer documented in 10+ pages in doc 14. The Blueprint gives it one row in a table. The MCP Client Gateway is an important component that should appear in the architecture diagrams — it's the layer between agents and all external tools.

**Correction:** Add the MCP Client Gateway to the architecture diagram. List the 5 custom internal MCP servers as components. Reference doc 14 for the full specification.

---

## 9. Local Inference — 🟡 INACCURATE

**Blueprint says:** "Ollama / llama.cpp" (§1, §3.1, §5)
**Doc 13 says:** llama.cpp specifically chosen over Ollama for lower VRAM overhead, direct control, and better single-user performance. Ollama listed as alternative but not recommended.

**Details:** Doc 13 explicitly evaluates Ollama vs llama.cpp and chooses llama.cpp. The Blueprint lists them as equivalent alternatives. Additionally, doc 13 specifies a Model Manager service for GPU model swapping that isn't mentioned in the Blueprint.

**Correction:** Change "Ollama / llama.cpp" to "llama.cpp (with model manager service)" as the primary local inference stack. Keep Ollama as a noted alternative.

---

## 10. Data Analytics Platform — 🟡 MISSING

**Blueprint says:** No mention of TimescaleDB, business analytics, or the Dash agent's analytical role
**Doc 18 says:** Full data analytics platform with TimescaleDB, ingestion pipeline from 10+ sources, continuous aggregates, automated reporting, ROI calculation engine, Grafana dashboards

**Details:** Doc 18 defines an entire analytics platform that doesn't appear in the Blueprint at all. TimescaleDB is a significant infrastructure component running on dreamteam. The data ingestion pipeline, business KPIs, and automated reporting are major factory capabilities.

**Correction:** Add TimescaleDB to the Shared State Layer and deployment topology. Add the data ingestion pipeline to the data flow diagram. Reference doc 18 in the tech stack decisions table.

---

## 11. Self-Improvement System — 🟡 MISSING

**Blueprint says:** No mention
**Doc 19 says:** Full self-improvement loop with Signal detection, A/B testing framework, knowledge distillation, anti-pattern library, post-mortem agent

**Details:** The self-improvement system is described as the factory's most important long-term capability. It should at least appear in the Build Order (§8) as a future phase and in the Cross-Cutting Concerns (§7).

**Correction:** Add self-improvement to §7 cross-cutting concerns. Add to Phase 4+ in the build order.

---

## 12. Cross-Reference Table — 🟡 CONFUSING

**Blueprint §9** maps "Decision Area" to numbered research docs [01]–[20]. But **§2.2** interface contracts table references [01], [02] etc. as if they're architecture docs. The numbering systems conflict — are [01]–[20] research docs or architecture docs?

**Correction:** Clarify that §9 references research docs in `/research/` while §2.2 should reference architecture docs in `/architecture/`. Use distinct notation (e.g., "R-01" for research, "A-01" for architecture).

---

## 13. Provider/Model References — 🟡 OUTDATED

**Blueprint says:** "Cerebras (free fan-out)" prominently featured (§1, §4, §6)
**Doc 02 says:** Groq hosts gpt-oss-120b for speed tier. Cerebras not listed as a provider. Together AI and Fireworks also listed.

**Details:** The Blueprint positions Cerebras as a major provider. Doc 02's provider registry doesn't include Cerebras at all — the speed/free tier is served by Groq and Google. Either the Blueprint or doc 02 needs updating.

**Correction:** Align provider list between Blueprint and doc 02. If Cerebras is a desired provider, add to doc 02's registry. If Groq replaced Cerebras, update the Blueprint.

---

## Verified Alignments (🟢)

These areas are consistent across documents:

- **Deployment platforms:** Cloudflare Workers/Pages, Railway, Vercel, GitHub Actions — consistent across docs 06, 10, 20
- **CI/CD pipeline structure:** Fast gates → tests → security → build → E2E → deploy — consistent between docs 06, 09, 20
- **Human-in-the-loop tiers:** Auto-approve / notify-after / approve-before / never-auto — consistent between docs 07, 12, 20
- **Testing strategy:** Playwright for E2E, Vitest/pytest for unit, Stryker for mutation — consistent between docs 09, 17, 20
- **Build order critical path:** Task Queue → Routing → First Pipeline → Ship Product — directionally consistent
- **Local-first philosophy:** Default to local compute, cloud for frontier quality — consistent between docs 13, 20
- **Git as source of truth:** GitOps model for deployments — consistent between docs 06, 15, 20

---

## Recommended Actions

### Immediate (Before Build Starts)

1. **Fix the Big 5:** Task queue (PostgreSQL), memory (Qdrant+Neo4j), LLM routing (LiteLLM), monitoring (Langfuse+Prometheus+Grafana+TimescaleDB), agent comms (NATS target)
2. **Clarify Kev vs Rex:** Document the orchestrator role split
3. **Add missing agent (Ally)** to the roster

### Short-Term

4. **Expand security section** with capability tokens, Vault, sandbox tiers
5. **Add MCP Gateway** to architecture diagrams
6. **Fix cross-reference numbering** (research vs architecture docs)
7. **Align provider list** (Cerebras vs Groq)

### Before Phase 3+

8. **Add TimescaleDB / analytics platform** to topology
9. **Add self-improvement system** to build phases
10. **Add NATS** to future architecture diagrams

---

*Verification completed 2026-02-08. The Blueprint captures the right vision but lags behind the detailed architecture decisions. A refresh pass incorporating the corrections above would bring it into alignment.*
