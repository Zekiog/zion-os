# Semantic Layer Spec (Spec 1)

## Why

Vector similarity returns chunks that look alike, not the right answer. A
semantic layer adds **entities** (Customer, TradingStrategy, LocalizationProject)
and **metrics** (lifetime_value, sharpe_ratio, terminology_consistency_score)
on top of raw vector data so agents can answer business questions, not just
retrieve similar strings.

## Catalog

Initial entities to model in `memory-core-mcp`:

| Entity | Key attributes | Metrics |
|--------|---------------|---------|
| Customer | id, tenant_id, segment, signup_date | lifetime_value, churn_risk |
| TradingStrategy | id, tenant_id, market, asset_class | sharpe_ratio, max_drawdown, win_rate |
| LocalizationProject | id, tenant_id, source_lang, target_langs | terminology_consistency_score, throughput |
| Agent | id, tenant_id, role, layer | success_rate, p95_latency_ms |
| Skill | id, name, version, capability_set | invocation_count, error_rate |
| Permission | id, scope, attenuation_chain | grant_count, revoke_count |
| MarketData | symbol, exchange, last_price | spread, volume_24h |
| Connector | id, type, source, last_heartbeat | uptime_pct, error_rate |
| DAOMember | address, tenant_id, voting_power | participation_rate |
| AuditEvent | id, tenant_id, agent_id, action, ts | cost_usd, governance_decision |

## Reference Implementation (memory-core-mcp)

```python
# memory-core-mcp/semantic_layer.py
from dataclasses import dataclass, field
from typing import Callable, Any

@dataclass
class Entity:
    name: str
    schema: dict
    metrics: list[str] = field(default_factory=list)
    resolver: Callable[[str, dict], Any] | None = None


class SemanticLayer:
    def __init__(self):
        self.entities: dict[str, Entity] = {}

    def define_entity(self, name: str, schema: dict, metrics: list[str]):
        self.entities[name] = Entity(name=name, schema=schema, metrics=metrics)

    def translate_to_sql(self, business_query: str) -> str:
        # Example: "high-value customers" ->
        #   SELECT * FROM customers WHERE lifetime_value > 10000
        # Use the resolver attached to the entity.
        entity_name, predicate = self._parse(business_query)
        entity = self.entities[entity_name]
        return entity.resolver(predicate)

    def resolve(self, entity: str, metric: str) -> Any:
        return self.entities[entity].metrics.__contains__(metric)
```

## Knowledge Graph (ai-memory-vault)

```python
# ai-memory-vault/knowledge_graph.py
class KnowledgeGraph:
    def add_entity(self, entity_id: str, entity_type: str, attributes: dict): ...
    def add_relationship(self, from_id: str, relation: str, to_id: str): ...
    def query(self, cypher: str) -> list[dict]: ...

# Example wiring
kg = KnowledgeGraph()
kg.add_entity("fincept-ai-ops", "TradingAgent", {"strategy": "momentum"})
kg.add_relationship("fincept-ai-ops", "trades_on", "BTC-USD")
kg.add_relationship("fincept-ai-ops", "uses", "memory-core-mcp")
kg.add_relationship("memory-core-mcp", "requires", "tenant_isolation_policy")
```

## MCP Tool Surface (memory-core-mcp)

| Tool | Purpose |
|------|---------|
| `search_memory(query, entity?, metric?)` | Semantic + vector hybrid retrieval |
| `resolve_metric(entity, metric)` | Look up a metric definition |
| `add_entity(entity_type, attributes)` | Insert entity (subject to RLS) |
| `add_relationship(from, relation, to)` | Insert edge (subject to RLS) |
| `query_graph(cypher)` | Run a graph query |

## Adoption Checklist

- [ ] Define entity catalog (10 entities minimum)
- [ ] Implement `SemanticLayer.define_entity`, `translate_to_sql`, `resolve`
- [ ] Migrate `ai-memory-vault` from vector mirror to `KnowledgeGraph` backend
- [ ] Wire `chainlingo` and `glossaryGuard` to the shared terminology graph
- [ ] Wire `tokenVoice` to entity mapping for DAO governance
- [ ] Expose MCP tools on `memory-core-mcp`

## Open Questions

- Which graph backend? (Neo4j, SurrealDB, Oracle PGX, in-house)
- Metric storage: same DB as entities, or separate metrics store?
- Resolver authoring: SQL only, or include DSL?
