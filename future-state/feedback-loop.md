# Future State — Feedback Loop Design

## The Missing Piece in Current Architecture

Today, the batch and real-time pipelines are **siloed**. Insights discovered in batch (emerging fraud patterns, updated risk profiles, model degradation signals) take hours to influence real-time decisions — and only via the scheduled feature store refresh, not through intelligent signal propagation.

The feedback loop is the architectural bridge that makes the agentic system **continuously learning**.

---

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    Real-Time Lane                           │
│                                                             │
│  POS → API → Kafka → Flink → Fraud Decisioning Agent        │
│                                    │                        │
│                                    │  Writes:               │
│                                    │  - Edge case logs      │
│                                    │  - Uncertain decisions │
│                                    │  - Tool call traces    │
│                                    ↓                        │
└─────────────────────────────────────────────────────────────┘
                                     │
                                     ↓
               ┌─────────────────────────────────────┐
               │        Shared Context Store          │
               │                                     │
               │  Redis (fast key-value lookup):      │
               │    - Thresholds per segment          │
               │    - Compromised merchant list       │
               │    - Device blacklist                │
               │    - IP reputation overrides         │
               │                                     │
               │  Vector DB (semantic search):        │
               │    - Historical fraud case summaries │
               │    - Pattern descriptions            │
               │    - Agent reasoning traces          │
               └─────────────────────────────────────┘
                                     ↑
                                     │
┌─────────────────────────────────────────────────────────────┐
│                    Batch Lane                               │
│                                                             │
│  App Data → Kafka → Spark → Batch Analytics Agent           │
│                                    │                        │
│                                    │  Writes:               │
│                                    │  - Updated thresholds  │
│                                    │  - New threat intel    │
│                                    │  - Pattern embeddings  │
│                                    │  - Merchant risk scores│
│                                    ↓                        │
└─────────────────────────────────────────────────────────────┘
```

---

## What Flows Through the Loop

### Batch Agent → Shared Context Store → Real-Time Agent

| Signal | Store | Latency | Real-Time Impact |
|---|---|---|---|
| Updated risk thresholds (per segment) | Redis | ~10ms write | Agent reads on next decision; no redeploy |
| Compromised merchant list | Redis (set) | ~10ms write | `graph_risk_check` tool reads this |
| Emerging fraud pattern embeddings | Vector DB | ~100ms write | Agent semantic search during investigation |
| Device fingerprint fraud patterns | Redis | ~10ms write | `enrich_context` tool cross-checks |
| IP reputation overrides | Redis | ~10ms write | `enrich_context` tool cross-checks |
| Geo fraud wave alert | Redis + Vector DB | ~50ms write | Agent context prompt enriched |

### Real-Time Agent → Shared Context Store → Batch Agent

| Signal | Store | Purpose |
|---|---|---|
| Edge case transaction logs | Kafka topic: `agent-edge-cases` | Batch agent analyzes uncertain decisions |
| Tool call traces (uncertain outcomes) | Case management DB | Pattern analysis input |
| OTP step-up outcomes (verified vs abandoned) | Kafka topic: `otp-outcomes` | Label generation for batch training |
| Novel fraud pattern flags | Kafka topic: `agent-flags` | Batch agent prioritizes investigation |

---

## Shared Context Store — Data Model

### Redis Structure

```
# Risk thresholds (updated by batch agent, read by real-time agent)
threshold:{segment}                    → float  (e.g., threshold:premium → 0.72)
threshold:default                      → float  (e.g., 0.65)

# Threat intel (updated by batch agent)
compromised:merchant:{merchant_id}     → timestamp + severity
fraudulent:device:{fingerprint_hash}  → timestamp + confidence
suspicious:ip:{ip_address}            → timestamp + risk_score

# Velocity counters (written by Flink, read by feature_store_query tool)
velocity:1m:{customer_id}             → count (TTL: 60s)
velocity:5m:{customer_id}             → count (TTL: 300s)
velocity:1h:{customer_id}             → count (TTL: 3600s)

# Customer risk profile (updated by batch Spark job)
profile:{customer_id}                 → hash: avg_spend, risk_band, segment, last_updated
```

### Vector DB Structure (for semantic fraud pattern search)

```
Collection: fraud_patterns
Documents:
  {
    id: "pattern_20260228_skimming_chicago",
    text: "Card skimming event detected at gas stations in Chicago metro area.
           Affected merchant category: 5541 (Gas Stations). 47 confirmed fraud cases.
           Device fingerprint pattern: [hash]. Geo cluster: 41.8N, 87.6W ±20mi.",
    embedding: [...],
    metadata: {
      severity: "HIGH",
      created_at: "2026-02-28T06:00:00Z",
      expires_at: "2026-03-14T06:00:00Z",
      source: "batch_analytics_agent"
    }
  }
```

The real-time agent queries the vector DB with semantic similarity to the current transaction context, retrieving relevant historical patterns to inform its reasoning.

---

## Feedback Loop Latency

| Update type | Batch agent write | Real-time agent reads | Total lag |
|---|---|---|---|
| Threshold calibration | ~10ms | Next transaction | ~10ms |
| Compromised merchant | ~10ms | Next transaction | ~10ms |
| Fraud pattern (vector) | ~100ms | Next transaction | ~100ms |
| Chargeback-driven alert | Event-triggered (~minutes) | Immediately after write | Minutes |
| Daily feature refresh | Hourly Spark job | After job completes | 0–60 min |

This is a dramatic improvement over the current state where batch insights have a minimum lag of the next scheduled job (1+ hours).

---

## Operational Considerations

- **Context store TTL**: Threat intel entries expire (e.g., 14-day TTL) to prevent stale signals from persisting
- **Conflict resolution**: If batch agent writes a new threshold that conflicts with a recent real-time signal, the batch value wins (batch has full historical context)
- **Observability**: All writes to the shared context store are logged with source, timestamp, and confidence score — enabling audit of what the real-time agent "knew" at decision time
- **Rollback**: If a batch agent threshold update causes a significant approval rate drop, the previous threshold value is retained in Redis with a versioned key for fast rollback
