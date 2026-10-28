# ZIONOS Roadmap

Six phases, each gated by the layer beneath it. Each phase has explicit
deliverables, adoption checks, and measurable outcomes.

---

## Stage 1 — Trust Fabric + Capability Security

**Goal:** Single sign-on + audit trail + tenant isolation foundation.

**Deliverables:**
1. `zion-aid-v2` branch protection + Vault prod-mode
2. `memory-core-mcp` branch protection
3. `agent-mcp-ops` branch protection
4. **NEW:** `zion-aid-v2` Biscuit tokens with `tenant_id` claims
5. **NEW:** `memory-core-mcp` Biscuit token authentication
6. **NEW:** `agent-mcp-ops` capability-based access control

**Adoption checks:**
- [ ] All Biscuit tokens carry `tenant_id` and `agent_id` claims
- [ ] All agent requests carry `X-Tenant-ID` header (enforced at gateway)
- [ ] Audit log entries are append-only and queryable by tenant

**Outcome:** Foundation for Layers 2–6.

---

## Stage 2 — Memory Mesh + Semantic Layer

**Goal:** Shared context + business meaning + entity relationships.

**Deliverables:**
1. `memory-core-mcp` Wave 1 (Oracle ADB vector search)
2. `memory-core-mcp` Wave 2 (N8N workflows)
3. **NEW:** `memory-core-mcp` semantic layer (entity definitions, metric translations)
4. **NEW:** `ai-memory-vault` knowledge graph (entity mapping, relationship tracking)
5. **NEW:** `chainlingo` + `glossaryGuard` shared terminology graph
6. **NEW:** `tokenVoice` + knowledge graph DAO entity mapping

**Adoption checks:**
- [ ] Semantic layer catalog has ≥10 entity types
- [ ] Knowledge graph captures `Agent → uses → Skill → requires → Permission` edges
- [ ] `chainlingo` and `glossaryGuard` resolve terminology from the shared graph

**Outcome:** Agents retrieve meaning, not just similar chunks.

---

## Stage 3 — Agent Operations + Multi-Agent Swarm

**Goal:** Collaborative agents + governed creation + memory-aware execution.

**Deliverables:**
1. `agent-mcp-ops` multi-agent swarm pattern implementation
2. `agent-zero` autonomous agent framework + HITL gates
3. **NEW:** `open-agent-platform` governance hooks (policy evaluation before agent creation)
4. **NEW:** `ottomator-agents` capability tokens (each agent has unforgeable authority)
5. **NEW:** `agent-mcp-ops` + `memory-core-mcp` MCP tool integration (`search_memory`, `write_memory`)
6. **NEW:** `fincept-ai-ops` + `memory-core-mcp` trading context storage

**Adoption checks:**
- [ ] Swarm pattern supports all-to-all communication with back-pressure
- [ ] HITL gates fire on >$10k position size, >3σ anomaly, bias >0.7
- [ ] Every agent carries capability tokens on creation

**Outcome:** Orchestrators compose vertical agents with shared memory.

---

## Stage 4 — Observability + Distributed Tracing

**Goal:** Production debugging + cost attribution + performance optimization.

**Deliverables:**
1. **NEW:** OpenTelemetry collector deployment (all agents instrumented)
2. **NEW:** Root spans per session/task (complete execution trees)
3. **NEW:** LLM generation spans (token usage, latency, cost attribution)
4. **NEW:** `agentReceipt` trace-aware cost tracking (`GET /traces/{trace_id}/cost`)
5. **NEW:** Anomaly detection (latency spikes, token cost anomalies, error rate increases)
6. **NEW:** Grafana dashboard (agent performance, cost breakdown, error rates)

**Adoption checks:**
- [ ] Every agent entry point emits a root span
- [ ] Every LLM call carries `model`, `token_count`, `cost_usd`
- [ ] Anomaly detector pages on-call for >3σ latency, >5× cost, >10% error rate

**Outcome:** Full call graph, cost attribution per agent, automated anomaly paging.

---

## Stage 5 — Governance + Compliance Framework

**Goal:** Audit-ready AI systems + regulatory compliance + risk management.

**Deliverables:**
1. `zion-aid-v2` AI governance framework (Govern → Map → Measure → Manage)
2. **NEW:** Compliance dashboard (EU AI Act, GDPR AI compliance)
3. **NEW:** Drift detection (`promptLedger` + `Z-Claude` evaluation engine)
4. **NEW:** Bias monitoring (fairness checks on agent outputs)
5. **NEW:** Human-in-the-loop approval workflows (critical decisions require human sign-off)
6. **NEW:** Audit trail reconstruction (versioned records, decision paths, human interventions)

**Adoption checks:**
- [ ] EU AI Act conformity assessment returns COMPLIANT for all high-risk agents
- [ ] Drift detector flags >5% prompt drift in 24h
- [ ] Every BLOCK/ESCALATE decision is human-recoverable from the audit trail

**Outcome:** Audit-ready AI; defensible to regulators and enterprise customers.

---

## Stage 6 — Federated Learning + Edge Coordination

**Goal:** Privacy-preserving AI coordination + edge learning + decentralized intelligence.

**Deliverables:**
1. **NEW:** `locally-uncensored` federated learning client (differential privacy + homomorphic encryption)
2. **NEW:** `gemma-chat` federated learning client (Apple Silicon optimized)
3. **NEW:** Federated aggregation server (secure aggregation, model broadcast)
4. **NEW:** Privacy-preserving coordination protocol (no raw data sharing)
5. **NEW:** Edge AI coordination (`locally-uncensored` + `gemma-chat` + `Z-Puter` distributed compute)
6. **NEW:** Federated learning dashboard (model convergence, participation rates, privacy budget tracking)

**Adoption checks:**
- [ ] All federated updates are homomorphically encrypted
- [ ] Differential privacy epsilon ≤0.1 per round
- [ ] Aggregator cannot inspect individual updates

**Outcome:** Local devices learn from each other without raw data leaving the device.

---

## Quantified Impact Targets

| Metric | Current | After Stage 3 | After Stage 6 |
|--------|----------|---------------|---------------|
| Agent cross-context sharing | 0% | 80%+ (semantic layer) | 95%+ (knowledge graph) |
| Audit trail coverage | 30% | 90%+ (governance framework) | 99%+ (full tracing) |
| Terminology consistency | 40% | 85%+ (shared graph) | 98%+ (semantic enforcement) |
| Trading decision traceability | 20% | 80%+ (distributed tracing) | 95%+ (full execution tree) |
| Prompt version drift | 15% | 5% (drift detection) | 1% (continuous monitoring) |
| Cross-agent memory conflicts | 100% | 10% (RLS isolation) | 2% (capability-based access) |
| Multi-tenant data isolation | 0% | 95%+ (RLS + schema-per-tenant) | 99.9%+ (homomorphic encryption) |
| Privacy-preserving coordination | 0% | 0% | 90%+ (federated learning) |
| EU AI Act compliance | 0% | 70% (governance framework) | 95%+ (audit-ready) |
| Production debugging time | 4+ hours | 30 min (distributed traces) | 5 min (anomaly detection) |

---

## First 14 Days

1. `memory-core-mcp` #6 — Branch protection (5 min)
2. `zion-aid-v2` #1 — Branch protection + Vault prod (15 min)
3. `agent-mcp-ops` #19 — Branch protection (5 min)
4. **NEW:** `zion-os` — this repo: Harmony manifesto v2.0 + connection specs
5. **NEW:** `memory-core-mcp` semantic layer proof-of-concept

## Next 60 Days

Complete Stage 1 + Stage 2. Every ecosystem component ships with single sign-on
plus semantic memory plus knowledge graph.
