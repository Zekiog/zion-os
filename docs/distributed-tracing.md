# Distributed Tracing Spec (Spec 3)

## Why

Production debugging requires the full call graph from user request →
orchestrator → sub-agent → tool call → LLM generation. Without it, an
incident becomes "which agent caused the spike, and why did it cost 5×?".

OpenTelemetry is the substrate. The span model is the contract.

## Span Hierarchy

```
trace_id: 7f3e...
│
├─ span: user_request                (root)
│     attrs: tenant_id, user_id, session_id
│
├─ span: agent_orchestrator          (orchestration layer)
│     attrs: agent_id, task_type, dag_id
│     │
│     ├─ span: sub_agent_call        (sub-agent)
│     │     attrs: sub_agent_id, capability
│     │     │
│     │     ├─ span: tool_call       (MCP / connector)
│     │     │     attrs: tool_name, args_size, result_size
│     │     │
│     │     └─ span: llm_generation  (model)
│     │           attrs: model, prompt_tokens, completion_tokens, cost_usd, latency_ms
│     │
│     └─ span: governance_check      (Layer 6 hook)
│           attrs: decision, drift, bias, policy
│
└─ span: response_sent               (final)
      attrs: trace_id, total_cost_usd, status
```

## Reference Implementation (agent-mcp-ops)

```python
from opentelemetry import trace
from opentelemetry.trace import Status, StatusCode

tracer = trace.get_tracer("agent-mcp-ops")

def execute_agent_task(task: Task) -> Result:
    with tracer.start_as_current_span("agent_task") as span:
        span.set_attribute("agent_id", task.agent_id)
        span.set_attribute("tenant_id", task.tenant_id)
        span.set_attribute("task_type", task.type)

        try:
            with tracer.start_as_current_span("sub_agent_call") as sub:
                sub.set_attribute("sub_agent_id", task.sub_agent_id)
                result = call_sub_agent(task)

            with tracer.start_as_current_span("llm_generation") as llm:
                llm.set_attribute("model", task.model)
                llm.set_attribute("prompt_tokens", result.prompt_tokens)
                llm.set_attribute("completion_tokens", result.completion_tokens)
                llm.set_attribute("cost_usd", result.cost_usd)
                llm.set_attribute("latency_ms", result.latency_ms)

            with tracer.start_as_current_span("governance_check") as gov:
                gov.set_attribute("decision", result.governance.decision)
                gov.set_attribute("drift", result.governance.drift)
                gov.set_attribute("bias", result.governance.bias)
                if result.governance.decision != "ALLOW":
                    gov.set_status(Status(StatusCode.ERROR, result.governance.reason))

            span.set_attribute("total_cost_usd", result.cost_usd)
            return result
        except Exception as exc:
            span.record_exception(exc)
            span.set_status(Status(StatusCode.ERROR, str(exc)))
            raise
```

## OTel Collector Deployment

```yaml
# otel-collector-config.yaml
receivers:
  otlp:
    protocols:
      grpc: { endpoint: 0.0.0.0:4317 }
      http: { endpoint: 0.0.0.0:4318 }

processors:
  batch: { timeout: 5s }
  memory_limiter: { limit_mib: 1024 }
  resource/tenant:
    attributes:
      - key: tenant_id
        from_attribute: $attributes.tenant_id
        action: insert

exporters:
  otlp/tempo:
    endpoint: tempo:4317
    tls: { insecure: true }
  prometheus:
    endpoint: 0.0.0.0:8889

service:
  pipelines:
    traces:
      receivers: [otlp]
      processors: [memory_limiter, batch, resource/tenant]
      exporters: [otlp/tempo]
    metrics:
      receivers: [otlp]
      processors: [batch]
      exporters: [prometheus]
```

## agentReceipt Cost Attribution

```python
# agentReceipt: trace-aware cost
@app.get("/traces/{trace_id}/cost")
def trace_cost(trace_id: str):
    spans = trace_store.query(trace_id=trace_id)
    by_agent = defaultdict(float)
    by_model = defaultdict(float)
    for span in spans:
        if "cost_usd" in span.attributes:
            by_agent[span.attributes["agent_id"]] += span.attributes["cost_usd"]
            by_model[span.attributes["model"]] += span.attributes["cost_usd"]
    return {
        "trace_id": trace_id,
        "total_usd": sum(by_agent.values()),
        "by_agent": dict(by_agent),
        "by_model": dict(by_model),
    }
```

## Anomaly Detection

| Signal | Threshold | Action |
|--------|-----------|--------|
| Latency | >3σ from rolling 1h average | Page on-call |
| Token cost | >5× agent rolling average | Page + auto-throttle |
| Error rate | >10% in 5-minute window | Page + circuit-breaker candidate |
| Governance BLOCK/ESCALATE rate | >20% in 10-minute window | Notify compliance |

## Adoption Checklist

- [ ] Deploy OTel collector with OTLP gRPC + HTTP receivers
- [ ] Instrument every agent entry point (`agent-mcp-ops`, `agent-zero`)
- [ ] Wrap every LLM call with `llm_generation` span and required attributes
- [ ] Add `GET /traces/{trace_id}/cost` in `agentReceipt`
- [ ] Wire anomaly detectors to paging (e.g. PagerDuty / Opsgenie)
- [ ] Grafana dashboards: agent performance, cost breakdown, error rates
- [ ] Sample 100% of traces for first 30 days, then tune

## Performance Budget

| Agent class | Target p95 latency | Cost budget / request |
|-------------|--------------------|-----------------------|
| Read-only retrieval | 200ms | $0.001 |
| Single LLM call | 2s | $0.05 |
| Multi-agent swarm (3 hops) | 6s | $0.20 |
| HITL workflow | 30s (humans are slow) | $0.50 |
