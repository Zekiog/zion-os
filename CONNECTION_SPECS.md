# Connection Specs

This document defines the five research-driven integration contracts that connect
ZIONOS layers. Each spec has a **Motivation**, **Contract** (data + protocol),
**Reference Implementation**, and **Adoption Checklist**.

| # | Integration | Layers | Anchor Components |
|---|-------------|--------|-------------------|
| 1 | Semantic Layer + Knowledge Graph | L2 ↔ L3, L4 | `memory-core-mcp`, `ai-memory-vault`, `chainlingo`, `glossaryGuard`, `tokenVoice` |
| 2 | Multi-Tenant Isolation + RLS | L1 ↔ L2, L4 | `zion-aid-v2`, `memory-core-mcp`, `fincept-ai-ops` |
| 3 | Distributed Tracing + OTel | L3 → L5 | `agent-mcp-ops`, `agentReceipt` |
| 4 | AI Governance + Compliance | L6 wraps L1–L5 | `zion-aid-v2`, `promptLedger`, `Z-Claude` |
| 5 | Federated Learning + Edge Coordination | L1 ↔ L4 | `locally-uncensored`, `gemma-chat`, `Z-Puter` |

---

## Spec 1 — Semantic Layer + Knowledge Graph

**Motivation.** Vector similarity returns chunks that look alike, not the answer.
Agents need entities (`Customer`, `TradingStrategy`, `LocalizationProject`) with
metrics (`lifetime_value`, `sharpe_ratio`, `terminology_consistency_score`) and
relationships (`Agent → uses → Skill → requires → Permission`).

**Contract.**

```python
# memory-core-mcp semantic layer
class SemanticLayer:
    def define_entity(self, name: str, schema: dict, metrics: list[str]): ...
    def translate_to_sql(self, business_query: str) -> str: ...
    def resolve(self, entity: str, metric: str) -> Any: ...

# ai-memory-vault knowledge graph
class KnowledgeGraph:
    def add_entity(self, entity_id: str, entity_type: str, attributes: dict): ...
    def add_relationship(self, from_id: str, relation: str, to_id: str): ...
    def query(self, cypher: str) -> list[dict]: ...
```

**Adoption checklist.**
- [ ] Define entity catalog (`Customer`, `TradingStrategy`, `LocalizationProject`, `Agent`, `Skill`, `Permission`).
- [ ] Wire `chainlingo` and `glossaryGuard` to shared terminology graph.
- [ ] Wire `tokenVoice` entity mapping for DAO governance.
- [ ] Add `metric` resolution to `memory-core-mcp` `search_memory` MCP tool.

See [`docs/semantic-layer.md`](./docs/semantic-layer.md) for full schema and
migration steps.

---

## Spec 2 — Multi-Tenant Isolation + RLS

**Motivation.** Enterprise customers pay for database-level guarantees. The
Biscuit token must carry the tenant; every read must enforce it.

**Contract.**

```sql
-- memory-core-mcp: Oracle ADB row-level security
CREATE POLICY tenant_isolation ON memory_entries
USING (tenant_id = sys_context('ZION_CTX', 'tenant_id'));
```

```jsonc
// zion-aid-v2: Biscuit token shape
{
  "agent_id": "fincept-ai-ops",
  "tenant_id": "customer-acme-corp",
  "capabilities": ["trade:read", "trade:write", "memory:read"],
  "attenuation": ["parent_token_hash", "expiry_timestamp"]
}
```

**Adoption checklist.**
- [ ] Add `tenant_id` claim to all Biscuit tokens in `zion-aid-v2`.
- [ ] Enforce `X-Tenant-ID` header on every agent request.
- [ ] Deploy Oracle ADB RLS policies on `memory_entries`, `knowledge_graph_edges`.
- [ ] Adopt **bridge schema-per-tenant** in `fincept-ai-ops` portfolio schemas.

See [`docs/multi-tenant-rls.md`](./docs/multi-tenant-rls.md).

---

## Spec 3 — Distributed Tracing + OpenTelemetry

**Motivation.** Production debugging requires the full call graph from user
request → orchestrator → sub-agent → tool call → LLM generation.

**Contract.**

```python
from opentelemetry import trace

tracer = trace.get_tracer("agent-mcp-ops")

def execute_agent_task(task: Task):
    with tracer.start_as_current_span("agent_task") as span:
        span.set_attribute("agent_id", task.agent_id)
        span.set_attribute("tenant_id", task.tenant_id)
        span.set_attribute("task_type", task.type)

        with tracer.start_as_current_span("sub_agent_call") as sub:
            sub.set_attribute("sub_agent_id", task.sub_agent_id)
            result = call_sub_agent(task)

        with tracer.start_as_current_span("llm_generation") as llm:
            llm.set_attribute("model", "gpt-4")
            llm.set_attribute("token_count", result.token_usage)
            response = generate_with_llm(task.prompt)

        span.set_attribute("total_cost_usd", calculate_cost(span))
        return result
```

**Adoption checklist.**
- [ ] Deploy OTel collector with OTLP receiver.
- [ ] Instrument every agent entry point in `agent-mcp-ops` and `agent-zero`.
- [ ] Wrap every LLM call with `llm_generation` span; carry `model`,
      `token_count`, `cost_usd`.
- [ ] Add `GET /traces/{trace_id}/cost` endpoint in `agentReceipt`.
- [ ] Configure anomaly detectors (latency >3σ, cost >5× average, error rate >10%).

See [`docs/distributed-tracing.md`](./docs/distributed-tracing.md).

---

## Spec 4 — AI Governance + Compliance Framework

**Motivation.** EU AI Act and GDPR require decision reconstruction, drift
detection, bias monitoring, and human-in-the-loop for high-risk decisions.

**Contract.**

```python
# zion-aid-v2 governance
class AIGovernanceFramework:
    def govern(self, agent_id: str, action: str, ctx: dict) -> GovernanceDecision:
        policy = self.evaluate_policy(agent_id, action)
        risk = self.assess_risk(agent_id, action, ctx)
        drift = self.detect_drift(agent_id)         # via promptLedger + Z-Claude
        bias = self.check_bias(agent_id, action)
        if drift > 0.8 or bias > 0.7:
            return GovernanceDecision("ESCALATE_TO_HUMAN", "High drift/bias")
        if policy == "DENY":
            return GovernanceDecision("BLOCK", "Policy violation")
        return GovernanceDecision("ALLOW", "Compliant")


# zion-aid-v2 compliance dashboard
class ComplianceDashboard:
    def eu_ai_act_report(self, tenant_id: str) -> Report:
        high_risk = self.classify_high_risk_agents(tenant_id)
        conformity = self.assess_conformity(high_risk)
        docs = self.generate_technical_documentation(tenant_id)
        return Report(
            high_risk_agents=high_risk,
            conformity_results=conformity,
            technical_documentation=docs,
            compliance_status="COMPLIANT" if all(conformity) else "NON_COMPLIANT",
        )
```

**Adoption checklist.**
- [ ] Implement four-plane governance: Govern → Map → Measure → Manage.
- [ ] Build EU AI Act + GDPR dashboard with decision-reconstruction.
- [ ] Add drift detection via `promptLedger` + `Z-Claude`.
- [ ] Add bias monitoring with fairness checks on agent outputs.
- [ ] Wire HITL approval workflow for high-risk decisions.
- [ ] Persist every governance decision in the audit trail.

See [`docs/governance-framework.md`](./docs/governance-framework.md).

---

## Spec 5 — Federated Learning + Edge Coordination

**Motivation.** Local AI devices (`locally-uncensored`, `gemma-chat`,
`Z-Puter`) hold private data. They must learn from each other without sharing
raw data.

**Contract.**

```python
import numpy as np
from homomorphic_encryption import encrypt, aggregate


class FederatedLearningClient:
    def __init__(self, local_model, privacy_budget: float = 0.1):
        self.model = local_model
        self.epsilon = privacy_budget  # differential privacy

    def train_local(self, local_data):
        self.model.fit(local_data)
        return self.model.get_weights()

    def add_differential_privacy(self, weights):
        noise = np.random.laplace(0, scale=self.epsilon, size=weights.shape)
        return weights + noise

    def send_update(self, encrypted_weights):
        return transmit_to_aggregation_server(encrypted_weights)


class FederatedAggregator:
    def aggregate_updates(self, encrypted_updates: list):
        aggregated = aggregate(encrypted_updates)  # homomorphic addition
        global_model.apply_update(aggregated)
        broadcast_to_clients(global_model)
```

**Adoption checklist.**
- [ ] `locally-uncensored` becomes a federated learning client (DP + HE).
- [ ] `gemma-chat` becomes a federated client (Apple Silicon optimized).
- [ ] Deploy secure aggregation server (homomorphic addition).
- [ ] Coordinate `Z-Puter` decentralized compute nodes into the same protocol.
- [ ] Build federated learning dashboard: convergence, participation, privacy budget.

See [`docs/federated-learning.md`](./docs/federated-learning.md).

---

## Cross-Spec Consistency

- **Tenant context flows through every spec.** Every span, token, semantic
  query, governance decision, and federated update carries `tenant_id`.
- **Audit trail is the single source of truth.** Every cross-spec event is
  appended to the `event_log` in Neon with the trace ID.
- **Capability attenuation is monotonic.** Once a Biscuit token is attenuated,
  no layer may re-widen its capabilities.
- **Cost attribution is trace-derived, never estimated.** `agentReceipt` reads
  spans; it never recomputes cost from logs.
