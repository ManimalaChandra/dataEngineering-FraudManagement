# Current State — Overview

## Architecture Summary

The current fraud management platform operates across two independent data lanes: a **real-time streaming lane** for per-transaction decisioning and a **batch processing lane** for analytics and reporting.

```
REAL-TIME:
POS → API Gateway → Kafka → Flink → Redis (features) → ML Model → ALLOW / DENY

BATCH:
Application Data → Kafka → Spark → ETL → BI Reports
```

Both lanes are built on proven open-source data infrastructure (Kafka, Flink, Spark) with Redis as the low-latency feature store and a purpose-built ML model for fraud scoring.

---

## Strengths

- **Low latency** — Flink + Redis enables sub-100ms feature lookup in the real-time path
- **Scalable throughput** — Kafka decouples producers from consumers, handles traffic spikes
- **Proven stack** — Kafka + Spark is a mature, well-understood combination for batch ETL
- **Separation of concerns** — Streaming and batch lanes are independently scalable

## Gaps and Limitations

| Gap | Impact |
|---|---|
| No decision explainability | Cannot justify fraud denials to customers or regulators |
| Static business rules | Rules require engineering deployments to update; lag behind fraud patterns |
| No feedback loop | Batch insights (chargebacks, patterns) don't inform real-time scoring |
| Single ML model call | No multi-signal investigation; a single score drives binary outcome |
| No adaptive thresholds | False positive rate cannot self-correct between model retraining cycles |
| Escalation is binary | No step-up authentication; edge cases are hard-denied or approved |

---

## Pipeline Details

- [Real-Time Streaming Pipeline →](real-time-streaming.md)
- [Batch Processing Pipeline →](batch-processing.md)
