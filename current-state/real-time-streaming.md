# Current State — Real-Time Streaming Pipeline

## Flow

```
POS Terminal
    │
    │  (HTTP/gRPC, payment event)
    ↓
API Gateway
    │
    │  Publishes to topic: raw-transactions
    ↓
Apache Kafka
    │
    │  Flink consumer group reads raw-transactions
    ↓
Apache Flink  (Stream Processing)
    │  ├── Parses and validates transaction payload
    │  ├── Computes windowed features:
    │  │       - Transaction velocity (last 1m, 5m, 1h)
    │  │       - Amount deviation from customer baseline
    │  │       - Geo-distance from last transaction
    │  │       - Device fingerprint hash
    │  └── Publishes enriched event to: enriched-transactions
    │
    ↓
Redis  (Feature Store)
    │  ├── Customer historical spend patterns
    │  ├── Merchant risk score (pre-computed)
    │  ├── Recent velocity counters (TTL-based)
    │  └── Device trust history
    │
    ↓
ML Model  (Fraud Scoring Service)
    │  ├── Input: enriched event + Redis features
    │  ├── Output: fraud probability score [0.0 – 1.0]
    │  └── Decision threshold applied (e.g., score > 0.65 → DENY)
    │
    ↓
Decision Engine
    ├── score < threshold  →  ALLOW
    └── score ≥ threshold  →  DENY
    │
    ↓
API Response → POS Terminal
```

---

## Components

### Apache Kafka
- **Topics**: `raw-transactions`, `enriched-transactions`, `fraud-decisions`
- **Retention**: 7 days for raw events; 30 days for decisions (audit)
- **Partitioning**: By `customer_id` to preserve ordering per customer

### Apache Flink
- **Processing model**: Event-time processing with watermarks
- **State backend**: RocksDB (for large stateful aggregations)
- **Key operations**:
  - Tumbling windows (1-minute velocity counts)
  - Sliding windows (5-minute amount averages)
  - Session windows (transaction session grouping)

### Redis (Feature Store)
- **Data model**: Hash maps per `customer_id`, `merchant_id`
- **TTL strategy**: Velocity counters expire after window duration; profile features are long-lived
- **Read latency**: < 5ms P99

### ML Fraud Model
- **Algorithm**: Gradient Boosted Trees (XGBoost / LightGBM)
- **Feature count**: ~120 features (behavioral, transactional, network)
- **Update cadence**: Retrained periodically by ML team
- **Serving**: REST API, containerized, horizontally scaled
- **Inference latency**: ~30ms P99

---

## Latency Budget (End-to-End)

| Stage | Latency |
|---|---|
| API Gateway ingestion | ~10ms |
| Kafka publish + consume | ~15ms |
| Flink feature computation | ~20ms |
| Redis feature lookup | ~5ms |
| ML model inference | ~30ms |
| Decision + API response | ~10ms |
| **Total** | **~90ms P50 / ~150ms P99** |

---

## Failure Modes

| Failure | Current Behavior |
|---|---|
| Redis unavailable | Falls back to ML model with partial features; increased false positives |
| ML model unavailable | Rule-based fallback (hard thresholds only) |
| Flink consumer lag | Decisions delayed; SLA breached |
| Kafka partition rebalance | Brief processing pause (~1-2s) |

---

## Limitations Highlighted

1. **No explainability**: The fraud score has no accompanying rationale. A denied customer cannot be told *why* their transaction was blocked beyond "suspected fraud."
2. **Threshold is static**: A single global threshold (e.g., 0.65) applies to all customers regardless of segment, history, or transaction context.
3. **No multi-signal investigation**: If the ML model is uncertain, there is no mechanism to fetch additional signals (graph relationships, IP reputation) before deciding.
4. **Escalation gap**: Transactions near the threshold are either approved (risk) or denied (friction); no middle path exists.
