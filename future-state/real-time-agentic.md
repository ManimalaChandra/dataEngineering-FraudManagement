# Future State — Real-Time Agentic Pipeline

## Flow

```
POS Terminal
    │  (payment event, HTTP/gRPC)
    ↓
API Gateway  (~10ms)
    │  Publishes to: raw-transactions
    ↓
Apache Kafka  [Topic: raw-transactions]
    │
    ↓
Apache Flink  (Feature Extraction & Enrichment)
    │  ├── Parses and validates transaction payload
    │  ├── Computes windowed features (velocity, amount delta, geo-distance)
    │  ├── Assigns initial risk tier (LOW / MEDIUM / HIGH) from lightweight heuristic
    │  └── Publishes enriched event to: enriched-transactions
    │
    ↓
Risk Tier Router
    ├── LOW risk (score < 0.3)  →  Auto-approve  →  Response (<50ms)
    │
    └── MEDIUM / HIGH risk  →  Fraud Decisioning Agent
                                    │
                        ┌───────────┴──────────────────────────────┐
                        │     Fraud Decisioning Agent               │
                        │     Model: claude-haiku-4-5               │
                        │     Pattern: ReAct (Reason + Act)         │
                        │                                           │
                        │  THINK: Evaluate enriched event context   │
                        │                                           │
                        │  ACT → Tool 1: feature_store_query        │
                        │    Redis: velocity, spend patterns        │
                        │                                           │
                        │  ACT → Tool 2: ml_score                   │
                        │    Existing ML model: fraud prob [0,1]   │
                        │                                           │
                        │  ACT → Tool 3: rule_engine_check          │
                        │    AML limits, velocity caps, geo rules  │
                        │                                           │
                        │  [If HIGH risk or uncertain:]             │
                        │  ACT → Tool 4: enrich_context             │
                        │    IP reputation, device trust score     │
                        │                                           │
                        │  ACT → Tool 5: graph_risk_check           │
                        │    Fraud rings, mule accounts,           │
                        │    merchant compromise network           │
                        │                                           │
                        │  THINK: Synthesize all signals            │
                        │                                           │
                        │  ACT → Tool 6: decide_and_act             │
                        │    Decision + natural language rationale  │
                        └──────────────────┬────────────────────────┘
                                           │
                              ┌────────────┼────────────────┐
                              ↓            ↓                ↓
                           APPROVE    SOFT DECLINE        DENY
                                      (OTP step-up)
                              │            │                │
                              └────────────┴────────────────┘
                                           │
                                    Kafka [fraud-decisions]
                                    Case Management DB
                                    (decision + rationale)
                                           │
                                    API Response → POS
```

---

## Agent Reasoning Example (ReAct Pattern)

```
INPUT: Transaction $847 at merchant "QuickMart_456", customer C-10293
       Geo: 340 miles from last transaction (2 hours ago)
       Device: new device fingerprint

THINK: "High geo-distance flag. Amount is moderate. Let me get the base ML score first."

ACT:   feature_store_query(customer_id="C-10293", merchant_id="QuickMart_456")
OBS:   { avg_txn: $123, velocity_1h: 3 txns, merchant_risk: 0.4, device_history: [] }

ACT:   ml_score(event=enriched_transaction)
OBS:   { fraud_probability: 0.71, top_features: ["geo_distance", "new_device", "velocity"] }

THINK: "Score 0.71 is high. New device + geo anomaly. Check rule engine and graph."

ACT:   rule_engine_check(transaction, customer_profile)
OBS:   { rules_triggered: ["new_device_high_amount"], action: "FLAG" }

ACT:   graph_risk_check(customer_id="C-10293", merchant_id="QuickMart_456")
OBS:   { merchant_fraud_connections: 2, network_risk: "ELEVATED" }

THINK: "Multiple strong signals: high ML score, new device, geo anomaly, merchant in
       partial fraud network. But this is a $847 transaction and customer has 5yr history.
       Recommend step-up auth rather than hard deny to avoid friction for a loyal customer."

ACT:   decide_and_act(
         action="SOFT_DECLINE",
         explanation="Unusual location (340mi from last transaction) with new device detected.
                      Merchant shows elevated network risk. Step-up authentication required.",
         otp_required=true
       )
```

---

## Escalation Flow (Soft Decline + OTP)

```
Agent decision: SOFT_DECLINE
    ↓
Immediate API Response to POS:
    { status: "soft_decline", reason: "verification_required" }
    ↓
Notification Service (async, <500ms):
    SMS / Push notification:
    "Unusual activity on your card at QuickMart. Was this you?
     Enter code XXXXXX at the terminal to complete your purchase."
    ↓
Customer enters OTP at POS
    ↓
Retry request with flag: { verified: true, otp_token: "..." }
    ↓
Agent re-evaluates with elevated trust:
    THINK: "Customer verified via OTP. Adjusted trust score. Approve."
    → APPROVE
```

---

## Latency Budget

| Stage | Latency |
|---|---|
| API Gateway ingestion | ~10ms |
| Kafka publish + consume | ~15ms |
| Flink feature extraction | ~20ms |
| Risk tier routing | ~2ms |
| Agent LLM reasoning | ~50ms |
| Tool calls (3–5 parallel where possible) | ~60ms total |
| Decision + response | ~10ms |
| **Total (medium/high risk path)** | **~167ms P50 / ~280ms P99** |

> Tool calls for `feature_store_query`, `ml_score`, and `rule_engine_check` are issued in parallel where the agent has no inter-dependency between them.

---

## Failure Modes and Fallbacks

| Failure | Agent Behavior |
|---|---|
| LLM timeout (>250ms) | Fall back to ML score + static threshold (current behavior) |
| Redis unavailable | Agent skips feature_store_query; notes reduced confidence in rationale |
| ML model unavailable | Agent uses rule_engine_check + graph_risk_check; discloses missing signal |
| Graph DB unavailable | Agent proceeds without graph signal; flags rationale as incomplete |
| All tools fail | Hard rule-based fallback; conservative deny for high amounts |

---

## Audit Trail

Every transaction processed by the agent produces:

```json
{
  "transaction_id": "txn-8472634",
  "timestamp": "2026-03-01T14:23:11Z",
  "decision": "SOFT_DECLINE",
  "risk_score": 0.71,
  "tools_invoked": [
    { "tool": "feature_store_query", "latency_ms": 4, "result_summary": "3 velocity hits, new device" },
    { "tool": "ml_score", "latency_ms": 28, "result_summary": "fraud_probability: 0.71" },
    { "tool": "rule_engine_check", "latency_ms": 5, "result_summary": "new_device_high_amount triggered" },
    { "tool": "graph_risk_check", "latency_ms": 19, "result_summary": "merchant: 2 fraud connections" }
  ],
  "rationale": "Unusual location (340mi from last transaction) with new device detected. Merchant shows elevated network risk. Step-up authentication required.",
  "agent_model": "claude-haiku-4-5"
}
```
