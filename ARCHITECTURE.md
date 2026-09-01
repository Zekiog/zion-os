# ZIONOS Architecture

## 1. Layer Diagram

```
┌─────────────────────────────────────────────────────────────────────────┐
│  LAYER 6: GOVERNANCE & COMPLIANCE (AI Governance Framework)            │
│  • zion-aid-v2 audit trail + policy enforcement                        │
│  • NEW: EU AI Act + GDPR compliance dashboard                           │
│  • NEW: Drift detection + bias monitoring                               │
│  • NEW: Human-in-the-loop (HITL) approval workflows                     │
├─────────────────────────────────────────────────────────────────────────┤
│  LAYER 5: OBSERVABILITY & TRACING (OpenTelemetry Stack)                │
│  • NEW: OTel collector + distributed tracing                            │
│  • NEW: Root span → LLM spans → tool spans                              │
│  • NEW: Metrics aggregation (latency, token cost, error rates)          │
│  • NEW: Anomaly detection for abnormal agent behavior                   │
├─────────────────────────────────────────────────────────────────────────┤
│  LAYER 4: VERTICAL APPLICATIONS (Domain-Specific Agents)               │
│  • fincept-ai-ops (Trading) + multi-tenant isolation (RLS)             │
│  • chainlingo (Localization) + semantic layer (terminology graph)      │
│  • tokenVoice (DAO Governance) + knowledge graph (entity mapping)      │
│  • changelogLingo (DevTools) + audit trail integration                 │
│  • glossaryGuard (Terminology) + semantic layer enforcement            │
│  • a11yAgent (Accessibility) + compliance reporting                    │
│  • agentReceipt (Cost Tracking) + token cost metrics                   │
│  • promptLedger (Prompt Versioning) + drift detection                  │
│  • vibevoice (Voice AI) + distributed tracing                          │
│  • Z-Puter (Decentralized Compute) + capability tokens                 │
│  • locally-uncensored (Local AI Desktop) + federated learning          │
│  • gemma-chat (Apple Silicon AI) + edge inference                      │
├─────────────────────────────────────────────────────────────────────────┤
│  LAYER 3: AGENT OPERATIONS (Orchestration + Skills)                    │
│  • agent-mcp-ops: Multi-agent swarm pattern + MCP integration          │
│  • agent-zero: Autonomous agent framework + HITL gates                 │
│  • open-agent-platform: No-code agent builder + governance hooks       │
│  • ottomator-agents: Pre-built agent library + capability tokens       │
├─────────────────────────────────────────────────────────────────────────┤
│  LAYER 2: MEMORY & COGNITION (Semantic Layer + Knowledge Graph)        │
│  • memory-core-mcp: Oracle ADB vector search + semantic layer          │
│  • ai-memory-vault: Curated memory mirror + knowledge graph            │
│  • Z-Claude: Evaluation engine + quant-mind + drift detection          │
│  • Event bus: Neon event_log (event-sourced) + OpenTelemetry traces    │
├─────────────────────────────────────────────────────────────────────────┤
│  LAYER 1: TRUST & IDENTITY (Capability-Based Security Fabric)          │
│  • zion-aid-v2: Biscuit tokens + capability-based security             │
│  • agent-z-os: eBPF policy + OverlayFS isolation + RLS enforcement     │
│  • zpm: Governance loop + DAG orchestration + multi-tenant routing    │
└─────────────────────────────────────────────────────────────────────────┘
```

## 2. Layer Responsibilities

### Layer 1 — Trust & Identity

The substrate every other layer depends on. Without capability tokens, multi-tenant
isolation, and an audit trail, no agent can act safely.

- **Capability issuance:** `zion-aid-v2` mints Biscuit tokens with `tenant_id`,
  `agent_id`, and a capability set. Tokens attenuate on delegation; revocation
  propagates immediately.
- **Process isolation:** `agent-z-os` enforces eBPF policy and OverlayFS namespace
  boundaries per agent run.
- **Routing:** `zpm` resolves a tenant-scoped DAG plan and routes it across the
  agent fleet.

### Layer 2 — Memory & Cognition

Gives agents shared meaning instead of isolated vector lookups.

- **Semantic layer:** Entities, metrics, and translation rules. A "Customer" entity
  knows it has `lifetime_value`; a "TradingStrategy" knows it has `sharpe_ratio`.
- **Knowledge graph:** Entities and relationships (`Agent → uses → Skill →
  requires → Permission`). Surfaces lineage and ownership.
- **Event log:** Event-sourced projections in Neon (`event_log`) preserve full
  history and enable replay.

### Layer 3 — Agent Operations

The execution plane. Orchestrators, autonomous loops, and no-code builders.

- **Multi-agent swarm:** All-to-all communication pattern, hybrid reactive-
  deliberative architecture. Production standard for 2026.
- **HITL gates:** Critical decisions pause for human sign-off.
- **No-code builder:** `open-agent-platform` composes agents from a governed
  capability library.

### Layer 4 — Vertical Applications

Twelve domain agents. Each maps to one or more cross-cutting capabilities
(RLS, semantic layer, knowledge graph, audit trail, distributed tracing,
capability tokens, federated learning, edge inference).

See [`components/CATALOG.md`](./components/CATALOG.md) for the mapping table.

### Layer 5 — Observability & Tracing

A production system is invisible until it is observable. OpenTelemetry gives us
the substrate; the span model gives us the call graph.

- **Root span** per session/task
- **Sub-spans** for orchestrator, sub-agent, tool call, LLM generation
- **Attributes** carry `agent_id`, `tenant_id`, `task_type`, `model`,
  `token_count`, `cost_usd`
- **Anomaly detection** runs on the metric stream: latency spikes, cost
  anomalies, error-rate increases

### Layer 6 — Governance & Compliance

Wraps everything. Four functions, four planes:

| Plane | Function | Implementation |
|-------|----------|----------------|
| **Govern** | Accountability, roles, approvals | `zion-aid-v2` policy engine + audit trail |
| **Map** | Risk contextualization (which agent, which data, which policy) | Knowledge graph queries |
| **Measure** | Drift detection, bias checks, performance monitoring | `promptLedger` + `Z-Claude` eval engine |
| **Manage** | Auto-remediation, human escalation | HITL workflows + `agent-z-os` process isolation |

## 3. Request Flow Example

A `fincept-ai-ops` trade request from tenant `acme-corp`:

```
User (acme-corp)
   │
   ▼
[Layer 1] zion-aid-v2 → Biscuit token check (tenant_id, capabilities)
   │
   ▼
[Layer 1] zpm → DAG plan: fincept-ai-ops → memory-core-mcp → market-data → LLM
   │
   ▼
[Layer 3] agent-mcp-ops → orchestrator spawns the swarm
   │
   ▼
[Layer 4] fincept-ai-ops → strategy evaluation
   │            │
   │            ├── [Layer 2] memory-core-mcp → semantic layer (TradingStrategy, sharpe_ratio)
   │            ├── [Layer 2] ai-memory-vault → knowledge graph (agent lineage, recent trades)
   │            └── [Layer 4] market-data connector → live price
   │
   ▼
[Layer 5] Every span emitted with tenant_id, agent_id, cost_usd
   │
   ▼
[Layer 6] zion-aid-v2 → policy check (position size, drawdown limits)
            ↓
        Govern decision: ALLOW | BLOCK | ESCALATE_TO_HUMAN
   │
   ▼
Response back to user with trace_id
```

## 4. Layer Dependencies

```
L6 depends on L1, L2, L5  (audit trail + drift detection need traces + memory)
L5 depends on all layers  (traces span every layer)
L4 depends on L1, L2, L3  (vertical agents use memory and orchestration)
L3 depends on L1, L2      (orchestrators need tokens and semantic memory)
L2 depends on L1          (memory access is tenant-scoped via tokens)
L1 depends on nothing     (substrate)
```

## 5. Anti-Patterns to Avoid

| Anti-pattern | Why it fails | ZIONOS guard |
|--------------|--------------|--------------|
| Building vertical agents before L1 | No isolation; data leaks across tenants | L1 first, always |
| Vector search without semantic layer | Agents retrieve similar chunks, not the right answer | `memory-core-mcp` semantic layer |
| Multi-agent swarm without governance | Unbounded blast radius, cost overruns | `open-agent-platform` governance hooks + `ottomator-agents` capability tokens |
| Cost tracking without traces | You know the bill but not which agent caused it | OTel spans → `agentReceipt` cost attribution |
| Compliance as a checklist | Real audits need decision reconstruction | `zion-aid-v2` audit trail + governance framework |

See [`CONNECTION_SPECS.md`](./CONNECTION_SPECS.md) for the integration contracts
between layers and components.
