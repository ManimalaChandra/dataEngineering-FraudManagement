# Data Engineering — Fraud Management

A reference architecture for **real-time and batch fraud detection pipelines**, covering both the current state (streaming + ETL) and the future agentic state (LLM-powered agents with tool orchestration).

## Use Case

```
POS Terminal → API Gateway → Application → Fraud Risk Score → APPROVE / DENY
```

Fraud management requires two complementary lanes:

| Lane | Purpose | Latency Target |
|---|---|---|
| **Real-Time Streaming** | Per-transaction fraud decisioning | < 300ms |
| **Batch Processing** | Historical analysis, model feedback, BI reporting | Minutes to hours |

---

## Repository Structure

```
dataEngineering-FraudManagement/
├── current-state/
│   ├── overview.md                 # Current architecture summary
│   ├── real-time-streaming.md      # Kafka → Flink → Redis → ML pipeline
│   └── batch-processing.md         # Kafka → Spark → ETL → BI pipeline
│
├── future-state/
│   ├── overview.md                 # Agentic architecture summary
│   ├── real-time-agentic.md        # LLM Agent with tool-chain decisioning
│   ├── batch-analytics-agent.md    # Batch agent: patterns, drift, calibration
│   ├── feedback-loop.md            # Shared context store + feedback design
│   └── tool-taxonomy.md            # All agent tools: type, latency, purpose
│
└── architecture/
    └── diagrams.md                 # Full architecture diagrams (ASCII)
```

---

## Key Architectural Shift

| Dimension | Current State | Future State (Agentic) |
|---|---|---|
| **Decisioning** | Single ML model call | Agent chains 3–5 tools (ReAct loop) |
| **Explainability** | None (black box score) | Natural language rationale per decision |
| **Adaptability** | Static rules, manual threshold updates | Batch agent recalibrates thresholds continuously |
| **Escalation** | Hard deny or approve | Soft decline + OTP step-up authentication |
| **Feedback Loop** | None | Batch insights update real-time agent context |
| **ML Governance** | Manual | Agent detects drift; humans approve retraining |

---

## Technologies Referenced

- **Streaming**: Apache Kafka, Apache Flink
- **Batch**: Apache Spark
- **Feature Store**: Redis, Feast
- **Graph Analytics**: Neo4j / Amazon Neptune
- **ML Platform**: MLflow (model registry), custom scoring service
- **LLM Agent Runtime**: Claude (`claude-haiku-4-5` for real-time, `claude-sonnet-4-6` for batch)
- **Observability**: Case management DB, audit trail store

---

## Sections

- [Current State →](current-state/overview.md)
- [Future State →](future-state/overview.md)
- [Architecture Diagrams →](architecture/diagrams.md)
