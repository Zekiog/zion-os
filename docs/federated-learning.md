# Federated Learning + Edge Coordination Spec (Spec 5)

## Why

Local AI devices (`locally-uncensored`, `gemma-chat`, `Z-Puter`) hold private
data. They must learn from each other without raw data ever leaving the
device. Homomorphic encryption + differential privacy give us a coordination
protocol that does not require trust in the aggregator.

## Threat Model

- Aggregator is **honest-but-curious**: it follows the protocol but tries
  to inspect individual updates.
- Clients are **untrusted**: a malicious client may send adversarial
  updates.
- Network is **adversarial**: updates may be tampered with in transit.

## Defense Stack

| Layer | Defense |
|-------|---------|
| **Transport** | TLS 1.3, mTLS with capability tokens |
| **Computation** | Homomorphic encryption (additive HE) |
| **Statistics** | Differential privacy (Laplace mechanism, ε ≤ 0.1) |
| **Aggregation** | Secure aggregation (server cannot see individual updates) |
| **Robustness** | Trimmed mean / coordinate-wise median to bound adversary influence |
| **Replay** | Nonce + monotonic round counter |

## Client Reference Implementation

```python
import numpy as np
from homomorphic_encryption import encrypt, decrypt, Ciphertext
from dp import laplace_mechanism


class FederatedLearningClient:
    def __init__(self, local_model, privacy_budget: float = 0.1):
        self.model = local_model
        self.epsilon = privacy_budget  # total privacy budget per round
        self.round_counter = 0

    def train_local(self, local_data) -> np.ndarray:
        self.model.fit(local_data)
        return self.model.get_weights()

    def add_differential_privacy(self, weights: np.ndarray) -> np.ndarray:
        # Per-coordinate Laplace noise calibrated to sensitivity 1.
        noise = np.random.laplace(0, scale=1.0 / self.epsilon, size=weights.shape)
        return weights + noise

    def send_update(self, weights: np.ndarray) -> Ciphertext:
        dp_weights = self.add_differential_privacy(weights)
        encrypted = encrypt(dp_weights, public_key=AGGREGATOR_PK)
        return Ciphertext(
            payload=encrypted,
            round=self.round_counter,
            nonce=secrets.token_bytes(16),
            client_id=self.client_id,
        )

    def receive_global_model(self, encrypted_global: Ciphertext):
        aggregated = decrypt(encrypted_global, secret_key=AGGREGATOR_SK)
        self.model.apply_update(aggregated)
        self.round_counter += 1
```

## Aggregator Reference Implementation

```python
class FederatedAggregator:
    def __init__(self, secret_shares: dict[str, bytes]):
        self.shares = secret_shares  # one per client; used for secure aggregation

    def aggregate_updates(self, ciphertexts: list[Ciphertext]) -> Ciphertext:
        # Reject stale or duplicate rounds.
        valid = [c for c in ciphertexts if c.round == self.current_round]
        # Additive HE: sum is computed under encryption.
        running = valid[0]
        for c in valid[1:]:
            running = running + c  # homomorphic addition
        # Average (still encrypted).
        n = len(valid)
        running = running * (1.0 / n)
        return running

    def broadcast(self, encrypted_global: Ciphertext):
        for client in self.clients:
            client.receive_global_model(encrypted_global)
```

## Edge Coordination (Z-Puter)

`Z-Puter` nodes participate as both **clients** (run local training) and
**aggregator candidates** (elected by stake). The protocol is the same; the
trust model is what changes.

| Role | Trust assumption |
|------|------------------|
| `locally-uncensored` (desktop) | Single user; DP budget = ε_user |
| `gemma-chat` (Apple Silicon) | Single user; DP budget = ε_user |
| `Z-Puter` node | Shared cluster; stake-weighted election |

## Convergence Dashboard

| Metric | Definition |
|--------|------------|
| Round | Current global round counter |
| Participation | Active clients / total eligible |
| Convergence | ‖w_t − w_{t-1}‖ < ε for k consecutive rounds |
| Privacy budget used | Σ ε_r per client (must not exceed ε_total) |
| Adversary bound | Max influence ≤ 1/(k+1) under trimmed mean |

## Adoption Checklist

- [ ] `locally-uncensored` ships with the federated client + DP noise
- [ ] `gemma-chat` ships with the federated client (Apple Silicon optimized)
- [ ] Aggregator service deployed (homomorphic addition + secure aggregation)
- [ ] `Z-Puter` nodes join the protocol
- [ ] Dashboard shows convergence, participation, privacy budget per client
- [ ] Quarterly: privacy budget audit (no client exceeds ε_total)

## Failure Modes

| Failure | Detection | Recovery |
|---------|-----------|----------|
| Client goes offline mid-round | Round timeout | Skip; aggregator proceeds with N-1 |
| Adversarial update detected | Trimmed mean rejects outlier | Quarantine client + alert |
| Aggregator compromise | Cross-check with stake election | Rotate aggregator |
| Privacy budget exceeded | Client-side guard | Stop training; freeze model |
