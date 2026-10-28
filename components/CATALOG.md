# Component Catalog

Every ZIONOS component mapped to its layer, its cross-cutting integrations,
and its current status.

## Layer 1 — Trust & Identity

| Component | Role | Integrations | Status |
|-----------|------|--------------|--------|
| `zion-aid-v2` | Biscuit token issuer, audit trail, governance engine | Tenant claims, RLS, governance hooks | Branch protection pending; Vault prod-mode pending |
| `agent-z-os` | eBPF policy enforcement, OverlayFS isolation, RLS at process boundary | Every vertical agent | Stub |
| `zpm` | DAG orchestration, multi-tenant routing, governance loop | Layer 1 substrate | Stub |

## Layer 2 — Memory & Cognition

| Component | Role | Integrations | Status |
|-----------|------|--------------|--------|
| `memory-core-mcp` | Oracle ADB vector search + semantic layer | Semantic layer (Spec 1), RLS (Spec 2) | Vector search in flight; semantic layer PoC next |
| `ai-memory-vault` | Curated memory mirror + knowledge graph | Knowledge graph (Spec 1) | Stub → knowledge graph migration next |
| `Z-Claude` | Evaluation engine, quant-mind, drift detection | Drift detection (Spec 4) | Stub |
| Event bus (Neon `event_log`) | Event-sourced projections, replay | OTel traces (Spec 3) | Operational |

## Layer 3 — Agent Operations

| Component | Role | Integrations | Status |
|-----------|------|--------------|--------|
| `agent-mcp-ops` | Multi-agent swarm pattern, MCP integration | OTel (Spec 3), memory MCP tools | Branch protection pending |
| `agent-zero` | Autonomous agent framework, HITL gates | Governance (Spec 4) | Stub |
| `open-agent-platform` | No-code agent builder | Governance hooks (Stage 3) | Stub |
| `ottomator-agents` | Pre-built agent library | Capability tokens (Stage 3) | Stub |

## Layer 4 — Vertical Applications

| Component | Domain | Cross-cutting integration |
|-----------|--------|---------------------------|
| `fincept-ai-ops` | Trading | RLS, semantic layer, knowledge graph, OTel |
| `chainlingo` | Localization | Semantic layer, terminology graph |
| `tokenVoice` | DAO Governance | Knowledge graph, audit trail |
| `changelogLingo` | DevTools | Audit trail integration |
| `glossaryGuard` | Terminology | Semantic layer enforcement |
| `a11yAgent` | Accessibility | Compliance reporting |
| `agentReceipt` | Cost tracking | Token cost metrics, OTel cost attribution |
| `promptLedger` | Prompt versioning | Drift detection |
| `vibevoice` | Voice AI | Distributed tracing |
| `Z-Puter` | Decentralized compute | Capability tokens, federated learning |
| `locally-uncensored` | Local AI desktop | Federated learning client |
| `gemma-chat` | Apple Silicon AI | Edge inference, federated learning client |

## Layer 5 — Observability & Tracing (NEW)

| Component | Role |
|-----------|------|
| OTel collector | Receives OTLP from every component |
| Trace store | Stores spans with tenant/agent attributes |
| Anomaly detector | Latency, cost, error rate signals |
| Grafana dashboards | Agent performance, cost, errors |
| `agentReceipt` v2 | Trace-aware cost attribution |

## Layer 6 — Governance & Compliance (NEW)

| Component | Role |
|-----------|------|
| EU AI Act dashboard | High-risk classification, conformity assessment |
| GDPR AI compliance | Data minimization, purpose limitation, right to explanation |
| Drift detector (Spec 4) | `promptLedger` + `Z-Claude` evaluation engine |
| Bias monitor | Fairness checks on agent outputs |
| HITL workflow | Human sign-off gates for high-risk decisions |
| Audit reconstruction | Versioned records, decision paths, interventions |
