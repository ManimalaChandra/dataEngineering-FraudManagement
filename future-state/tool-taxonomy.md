# Future State — Agent Tool Taxonomy

All tools available to the fraud management agents are defined here: their type, purpose, inputs, outputs, latency, and which agent uses them.

---

## Real-Time Agent Tools

These tools are invoked by the **Fraud Decisioning Agent** (Model: `claude-haiku-4-5`) during per-transaction evaluation.

### Tool 1: `feature_store_query`

| Attribute | Value |
|---|---|
| **Type** | Read (cached) |
| **Latency** | ~5ms P99 |
| **Store** | Redis |

**Purpose**: Retrieve pre-computed customer and merchant features from the feature store.

**Inputs**:
```json
{
  "customer_id": "string",
  "merchant_id": "string"
}
```

**Outputs**:
```json
{
  "customer_avg_spend_30d": 123.45,
  "customer_velocity_1h": 3,
  "customer_velocity_1d": 12,
  "customer_risk_band": "MEDIUM",
  "merchant_risk_score": 0.42,
  "device_trust_history": ["fingerprint_hash_1"],
  "last_transaction_geo": { "lat": 37.7, "lon": -122.4 }
}
```

---

### Tool 2: `ml_score`

| Attribute | Value |
|---|---|
| **Type** | Compute |
| **Latency** | ~30ms P99 |
| **Serving** | REST API (containerized, auto-scaled) |

**Purpose**: Invoke the production ML fraud scoring model to get a base fraud probability.

**Inputs**: Full enriched transaction event (from Flink output)

**Outputs**:
```json
{
  "fraud_probability": 0.71,
  "top_features": ["geo_distance", "new_device", "velocity_1h"],
  "model_version": "v2.4.1",
  "confidence": "HIGH"
}
```

**Note**: This tool wraps the existing ML model — no change to the model itself. The agent provides context for *how* to interpret the score.

---

### Tool 3: `rule_engine_check`

| Attribute | Value |
|---|---|
| **Type** | Read (deterministic) |
| **Latency** | ~5ms P99 |
| **Store** | Rule engine service (in-memory) |

**Purpose**: Evaluate configurable business rules that are non-negotiable (regulatory limits, velocity caps, jurisdiction restrictions).

**Inputs**:
```json
{
  "transaction": { "amount": 847.00, "currency": "USD", "merchant_category": "5411" },
  "customer_profile": { "segment": "premium", "country": "US", "kyc_status": "verified" }
}
```

**Outputs**:
```json
{
  "rules_triggered": ["new_device_high_amount"],
  "rules_passed": ["aml_limit", "velocity_cap", "jurisdiction_check"],
  "recommended_action": "FLAG",
  "override_required": false
}
```

---

### Tool 4: `enrich_context`

| Attribute | Value |
|---|---|
| **Type** | External API (read) |
| **Latency** | ~50ms P99 |
| **Source** | IP reputation service, device intelligence service |

**Purpose**: Enrich the transaction with external signals about the IP address and device.

**Inputs**:
```json
{
  "ip_address": "192.168.x.x",
  "device_fingerprint": "hash_string",
  "geo_coordinates": { "lat": 41.8, "lon": -87.6 }
}
```

**Outputs**:
```json
{
  "ip_reputation_score": 0.3,
  "ip_is_vpn": false,
  "ip_is_proxy": false,
  "ip_associated_fraud_cases": 0,
  "device_trust_score": 0.15,
  "device_is_known": false,
  "device_fraud_associations": 0
}
```

---

### Tool 5: `graph_risk_check`

| Attribute | Value |
|---|---|
| **Type** | Read (graph traversal) |
| **Latency** | ~20ms P99 |
| **Store** | Graph DB (Neo4j / Amazon Neptune) |

**Purpose**: Check for network-level risk — fraud rings, mule account networks, compromised merchants.

**Inputs**:
```json
{
  "customer_id": "string",
  "merchant_id": "string"
}
```

**Graph queries**:
- Merchant: connected fraud cases within 2 hops (last 30 days)
- Customer: shared device or IP with confirmed fraud accounts
- Merchant: connected to known compromised merchant network

**Outputs**:
```json
{
  "merchant_fraud_connections": 2,
  "merchant_network_risk": "ELEVATED",
  "customer_shared_device_fraud": 0,
  "customer_ip_fraud_associations": 0,
  "network_risk_score": 0.68
}
```

---

### Tool 6: `decide_and_act`

| Attribute | Value |
|---|---|
| **Type** | Write (action + audit) |
| **Latency** | ~5ms |
| **Output** | Kafka, Case Management DB |

**Purpose**: Record the agent's final decision with full rationale. This is the terminal tool in the real-time agent's chain.

**Inputs**:
```json
{
  "transaction_id": "string",
  "decision": "APPROVE | SOFT_DECLINE | DENY",
  "risk_score": 0.71,
  "rationale": "Natural language explanation string",
  "otp_required": true,
  "tools_invoked": [...]
}
```

**Side effects**:
- Publishes to Kafka topic `fraud-decisions`
- Writes full audit record to Case Management DB
- If SOFT_DECLINE: triggers Notification Service for OTP flow

---

## Batch Analytics Agent Tools

These tools are invoked by the **Analytics Agent** (Model: `claude-sonnet-4-6`) during scheduled or event-triggered runs.

### Tool 7: `pattern_analysis`

| Attribute | Value |
|---|---|
| **Type** | Compute (Spark query) |
| **Latency** | Seconds to minutes |
| **Source** | Data warehouse + feature store |

**Purpose**: Identify emerging fraud patterns not yet captured in rules or the ML model.

**Outputs**: Structured pattern report with severity, affected segment, and recommended real-time action (threshold adjustment, merchant flag, etc.)

---

### Tool 8: `model_drift_detection`

| Attribute | Value |
|---|---|
| **Type** | Compute |
| **Latency** | Minutes |
| **Source** | Model registry (MLflow), chargeback labels |

**Purpose**: Detect when the production ML model has drifted from its training distribution.

**Metrics**: PSI, KL-divergence, F1 delta, precision/recall delta, approval rate delta

**Output**: Drift report; triggers `model_drift_alert` JIRA ticket and Slack notification if thresholds exceeded

---

### Tool 9: `threshold_calibration`

| Attribute | Value |
|---|---|
| **Type** | Write (Redis) |
| **Latency** | ~10ms write |
| **Output** | Redis key: `threshold:{segment}` |

**Purpose**: Recalibrate per-segment fraud scoring thresholds based on observed false positive/negative rates.

**Logic**: Optimizes Precision-Recall trade-off per segment, constrained by business tolerance for fraud loss and false positive rate (< 2%).

---

### Tool 10: `report_generation`

| Attribute | Value |
|---|---|
| **Type** | Write (data warehouse + BI) |
| **Latency** | Minutes |
| **Output** | DWH tables, Tableau/Looker dashboards, email |

**Purpose**: Generate business, operations, and regulatory reports.

---

### Tool 11: `threat_alert`

| Attribute | Value |
|---|---|
| **Type** | Write (Redis + Vector DB) |
| **Latency** | ~100ms write |
| **Output** | Shared context store |

**Purpose**: Push newly discovered threat intelligence to the shared context store so the real-time agent acts on it immediately — without waiting for the next model retrain or deployment.

---

## Tool Latency Summary

| Tool | Agent | Type | P99 Latency |
|---|---|---|---|
| feature_store_query | Real-time | Read (Redis) | ~5ms |
| ml_score | Real-time | Compute | ~30ms |
| rule_engine_check | Real-time | Read | ~5ms |
| enrich_context | Real-time | External API | ~50ms |
| graph_risk_check | Real-time | Read (Graph DB) | ~20ms |
| decide_and_act | Real-time | Write | ~5ms |
| pattern_analysis | Batch | Compute | Seconds–minutes |
| model_drift_detection | Batch | Compute | Minutes |
| threshold_calibration | Batch | Write (Redis) | ~10ms |
| report_generation | Batch | Write (DWH) | Minutes |
| threat_alert | Batch | Write (Redis + VDB) | ~100ms |

> Real-time agent tools are designed to be **parallelizable** where there are no inter-dependencies. For example, `feature_store_query`, `ml_score`, and `rule_engine_check` can be issued concurrently, reducing wall-clock time to the slowest single call (~30ms) rather than their sum (~40ms).
