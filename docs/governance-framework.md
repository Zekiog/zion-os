# AI Governance Framework Spec (Spec 4)

## Why

EU AI Act and GDPR require decision reconstruction, drift detection, bias
monitoring, and human-in-the-loop for high-risk decisions. Real audits need
**versioned records**, **decision paths**, and **human intervention points**
— not a checkbox.

## Four Planes

| Plane | Question it answers | Implementation |
|-------|---------------------|----------------|
| **Govern** | Who is accountable? Who approved what? | `zion-aid-v2` policy engine + audit trail |
| **Map** | What risk applies to this action? | Knowledge graph queries against `ai-memory-vault` |
| **Measure** | How is this agent performing? | `promptLedger` + `Z-Claude` evaluation engine |
| **Manage** | What do we do when something is wrong? | Auto-remediation + HITL escalation |

## Reference Implementation (zion-aid-v2)

```python
# zion-aid-v2/governance.py
from dataclasses import dataclass

@dataclass
class GovernanceDecision:
    action: str   # ALLOW | BLOCK | ESCALATE_TO_HUMAN
    reason: str
    drift: float
    bias: float
    risk: str


class AIGovernanceFramework:
    def __init__(self, policy_engine, risk_graph, drift_engine, bias_engine):
        self.policy = policy_engine
        self.risk = risk_graph
        self.drift = drift_engine
        self.bias = bias_engine

    def govern(self, agent_id: str, action: str, ctx: dict) -> GovernanceDecision:
        policy_result = self.policy.evaluate(agent_id, action)
        risk_level = self.risk.assess(agent_id, action, ctx)
        drift_score = self.drift.detect(agent_id)
        bias_score = self.bias.check(agent_id, action, ctx)

        if drift_score > 0.8 or bias_score > 0.7:
            return GovernanceDecision(
                action="ESCALATE_TO_HUMAN",
                reason="High drift/bias",
                drift=drift_score,
                bias=bias_score,
                risk=risk_level,
            )
        if policy_result == "DENY":
            return GovernanceDecision(
                action="BLOCK",
                reason="Policy violation",
                drift=drift_score,
                bias=bias_score,
                risk=risk_level,
            )
        return GovernanceDecision(
            action="ALLOW",
            reason="Compliant",
            drift=drift_score,
            bias=bias_score,
            risk=risk_level,
        )
```

## Compliance Dashboard

```python
# zion-aid-v2/compliance_dashboard.py
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

    def gdpr_ai_report(self, tenant_id: str) -> Report:
        return Report(
            data_minimization=self.check_data_minimization(tenant_id),
            purpose_limitation=self.check_purpose_limitation(tenant_id),
            right_to_explanation=self.check_right_to_explanation(tenant_id),
        )
```

## Drift Detection

- `promptLedger` records every prompt version with hash, timestamp, author.
- `Z-Claude` evaluation engine runs a held-out eval set nightly.
- Drift = cosine distance between current prompt's outputs and the baseline.
- Threshold: >5% drift in 24h → page on-call + create incident.

## Bias Monitoring

- For classification agents: demographic parity, equal opportunity, calibration.
- For generative agents: toxicity classifier + fairness benchmark.
- Threshold: bias score >0.7 → ESCALATE_TO_HUMAN.

## HITL Workflow

```
[Governance Decision] → ESCALATE_TO_HUMAN
        ↓
[Create Approval Request] → audit trail entry
        ↓
[Notify on-call] → Slack/email
        ↓
[Human reviews context] → audit trail entry
        ↓
[Approve / Reject] → capability token issued for re-execution OR
                       agent quarantined + postmortem scheduled
```

## Audit Trail Reconstruction

Every governance decision MUST be reconstructible. The audit trail record
contains:

```jsonc
{
  "trace_id": "7f3e...",
  "tenant_id": "acme-corp",
  "agent_id": "fincept-ai-ops",
  "action": "execute_trade",
  "policy_result": "DENY",
  "drift_score": 0.12,
  "bias_score": 0.05,
  "risk_level": "HIGH",
  "decision": "BLOCK",
  "reason": "Position size exceeds per-trade limit",
  "token_hash": "abc...",
  "human_intervention": null,
  "ts": "2026-08-31T17:21:41Z"
}
```

## Adoption Checklist

- [ ] Implement four-plane governance in `zion-aid-v2`
- [ ] Build EU AI Act + GDPR dashboard
- [ ] Wire drift detection via `promptLedger` + `Z-Claude`
- [ ] Wire bias monitoring with fairness checks
- [ ] Wire HITL approval workflow (Slack/email + audit entry)
- [ ] Persist every governance decision in the audit trail
- [ ] Quarterly: replay last 90 days of decisions and prove reconstruction

## Failure Modes

| Failure | Detection | Recovery |
|---------|-----------|----------|
| Drift detector offline | Eval job hasn't run in 24h | Page SRE + quarantine agent |
| Bias score missing for new agent | CI scan | Block deploy |
| Audit trail gap | Replay count != issued token count | Block deploy + investigate |
| HITL timeout (>2h) | SLA alert | Auto-reject + notify compliance |
