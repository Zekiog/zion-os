# Multi-Tenant Isolation + RLS Spec (Spec 2)

## Why

Enterprise customers pay for database-level guarantees. Application-level
filtering is one bug away from a cross-tenant leak. The Biscuit token must
carry the tenant; every read must enforce it; the database must reject rows
that don't match the session context.

## Three Isolation Models

| Model | Strength | Weakness | ZIONOS use |
|-------|----------|----------|------------|
| Database-per-tenant | Strongest isolation | High ops cost | Premium tier |
| Schema-per-tenant (bridge) | Strong + cost-effective | Cross-schema joins need explicit grants | Default for `fincept-ai-ops` |
| Shared schema + RLS | Cheapest, scales well | Must trust the RLS policy | Default for `memory-core-mcp`, `agentReceipt` |

ZIONOS uses a **bridge** model: shared Oracle ADB database, schemas-per-tenant
where data gravity is high, RLS everywhere else.

## Token Shape (zion-aid-v2)

```jsonc
// Biscuit token claims
{
  "agent_id": "fincept-ai-ops",
  "tenant_id": "customer-acme-corp",
  "capabilities": ["trade:read", "trade:write", "memory:read"],
  "attenuation": ["parent_token_hash", "expiry_timestamp"],
  "issued_at": "2026-08-31T15:00:00Z",
  "expires_at": "2026-08-31T18:00:00Z"
}
```

## RLS Policy (memory-core-mcp on Oracle ADB)

```sql
-- Application sets the context from the Biscuit token
CALL sys_context.set_context('ZION_CTX', 'tenant_id', :tenant_id);

-- Policy rejects any row whose tenant_id does not match
CREATE POLICY tenant_isolation ON memory_entries
USING (tenant_id = sys_context('ZION_CTX', 'tenant_id'));

CREATE POLICY tenant_isolation_kg ON knowledge_graph_edges
USING (
  EXISTS (
    SELECT 1 FROM knowledge_graph_nodes n
    WHERE n.id = knowledge_graph_edges.from_id
      AND n.tenant_id = sys_context('ZION_CTX', 'tenant_id')
  )
);
```

## Header Enforcement

Every agent request MUST carry `X-Tenant-ID` and `X-Biscuit-Token`. The
gateway verifies the token and rejects the request if the claim and the
header disagree.

```python
# agent-z-os gateway
def authorize(request):
    token = verify_biscuit(request.headers["X-Biscuit-Token"])
    if token.tenant_id != request.headers["X-Tenant-ID"]:
        raise PermissionError("tenant mismatch")
    set_session_context("tenant_id", token.tenant_id)
    return token.capabilities
```

## Bridge Schema (fincept-ai-ops)

```sql
-- Shared database, per-tenant schemas for portfolio data
CREATE SCHEMA portfolio_acme;
CREATE TABLE portfolio_acme.positions (
  id UUID PRIMARY KEY,
  symbol TEXT NOT NULL,
  qty NUMERIC NOT NULL,
  entry_price NUMERIC NOT NULL,
  ts TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Cross-schema reads require explicit grant
GRANT SELECT ON portfolio_acme.positions TO zion_runtime;
```

## Adoption Checklist

- [ ] Add `tenant_id` and `agent_id` claims to all Biscuit tokens in `zion-aid-v2`
- [ ] Enforce `X-Tenant-ID` header on the gateway (`agent-z-os`)
- [ ] Deploy RLS policies on `memory_entries` and `knowledge_graph_*` in Oracle ADB
- [ ] Migrate `fincept-ai-ops` to bridge schema-per-tenant
- [ ] Add tenant isolation tests to every component's CI pipeline

## Failure Modes

| Failure | Detection | Recovery |
|---------|-----------|----------|
| Token claim and header disagree | Gateway 401 | Log + alert + revoke parent token |
| RLS policy not applied to new table | CI scan | Block deploy |
| Cross-schema read without grant | DB error → 403 | Audit + revoke grants |
| Tenant context unset | DB returns 0 rows | Caller sees empty result; treat as bug |
