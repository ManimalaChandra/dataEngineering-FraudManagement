# Architecture Diagrams

## 1. End-to-End System Overview

```
╔══════════════════════════════════════════════════════════════════════════════╗
║                     FRAUD MANAGEMENT PLATFORM                              ║
╠══════════════════════════════════════════════════════════════════════════════╣
║                                                                              ║
║  ┌─────────────┐     ┌──────────────┐     ┌────────────────────────────┐   ║
║  │ POS Terminal│────►│ API Gateway  │────►│ Kafka: raw-transactions     │   ║
║  └─────────────┘     └──────────────┘     └────────────┬───────────────┘   ║
║                                                         │                   ║
║                                                         ▼                   ║
║                                           ┌────────────────────────────┐   ║
║                                           │ Flink: Feature Extraction   │   ║
║                                           │  - velocity windows         │   ║
║                                           │  - geo-distance             │   ║
║                                           │  - device hash              │   ║
║                                           └────────────┬───────────────┘   ║
║                                                         │                   ║
║                                           ┌─────────────▼──────────────┐   ║
║                                           │ Risk Tier Router            │   ║
║                                           │  LOW ──► Auto-Approve       │   ║
║                                           │  MED/HIGH ──► Agent         │   ║
║                                           └─────────────┬──────────────┘   ║
║                                                         │                   ║
║                         ┌───────────────────────────────▼────────────────┐ ║
║                         │        Real-Time Fraud Decisioning Agent        │ ║
║                         │        Model: claude-haiku-4-5                  │ ║
║                         │  ┌─────────────────────────────────────────┐   │ ║
║                         │  │ Tools (parallel where possible):         │   │ ║
║                         │  │  ├── feature_store_query  (Redis, 5ms)  │   │ ║
║                         │  │  ├── ml_score              (ML, 30ms)   │   │ ║
║                         │  │  ├── rule_engine_check     (Rules, 5ms) │   │ ║
║                         │  │  ├── enrich_context        (API, 50ms)  │   │ ║
║                         │  │  └── graph_risk_check      (Graph, 20ms)│   │ ║
║                         │  └─────────────────────────────────────────┘   │ ║
║                         │              decide_and_act ↓                   │ ║
║                         └─────────────────────┬──────────────────────────┘ ║
║                                               │                            ║
║                          ┌──────────┬─────────┴─────────┐                 ║
║                          ▼          ▼                    ▼                 ║
║                       APPROVE  SOFT DECLINE            DENY               ║
║                                (OTP step-up)                               ║
║                                    │                                       ║
║                          Notification Service                              ║
║                          SMS / Push OTP ──► Customer                       ║
║                          Retry with VERIFIED flag                          ║
║                                                                            ║
╠══════════════════════════════════════════════════════════════════════════════╣
║                          SHARED CONTEXT STORE                              ║
║           Redis (fast KV)  +  Vector DB (semantic search)                  ║
║     Thresholds │ Threat Intel │ Compromised Merchants │ Fraud Patterns      ║
║                       ▲                    │                               ║
║                       │ writes             │ reads                         ║
╠══════════════════════════════════════════════════════════════════════════════╣
║                                            ▼                               ║
║                         ┌────────────────────────────────────────────────┐ ║
║                         │        Batch Analytics Agent                    │ ║
║                         │        Model: claude-sonnet-4-6                 │ ║
║                         │  ┌─────────────────────────────────────────┐   │ ║
║                         │  │ Tools:                                   │   │ ║
║                         │  │  ├── pattern_analysis                   │   │ ║
║                         │  │  ├── model_drift_detection               │   │ ║
║                         │  │  ├── threshold_calibration               │   │ ║
║                         │  │  ├── report_generation                   │   │ ║
║                         │  │  └── threat_alert                        │   │ ║
║                         │  └─────────────────────────────────────────┘   │ ║
║                         └──────────────────┬───────────────────────────┘  ║
║                                            │                               ║
║                       ┌──────────┬─────────┴─────────┐                   ║
║                       ▼          ▼                    ▼                   ║
║               Updated        BI Reports          ML Drift Alert            ║
║               Feature        Regulatory          (JIRA + Slack)            ║
║               Store          Reports             ML Team ► Approve          ║
║                                                                            ║
║  ┌─────────────────────────────────────────────────────────────────────┐  ║
║  │ App Data → Kafka: batch-events → Spark ETL → Data Warehouse          │  ║
║  └─────────────────────────────────────────────────────────────────────┘  ║
╚══════════════════════════════════════════════════════════════════════════════╝
```

---

## 2. Current State vs Future State (Side by Side)

```
CURRENT STATE                          FUTURE STATE
─────────────────────────────          ─────────────────────────────
Real-Time:                             Real-Time:

POS                                    POS
 │                                      │
API Gateway                            API Gateway
 │                                      │
Kafka                                  Kafka
 │                                      │
Flink ──► Redis ──► ML Model           Flink
 │              └──► Score              │
 │                    │                Risk Tier Router
 │             ALLOW / DENY             │
 │                                     Agent (ReAct)
 │                                      ├── feature_store_query
 │                                      ├── ml_score
 │                                      ├── rule_engine_check
 │                                      ├── enrich_context
 │                                      └── graph_risk_check
 │                                      │
 │                                     APPROVE / SOFT DECLINE / DENY
 │                                      │ (with NL explanation)
─────────────────────────────          ─────────────────────────────
Batch:                                 Batch:

App Data                               App Data
 │                                      │
Kafka                                  Kafka
 │                                      │
Spark ETL                              Spark ETL
 │                                      │
Data Warehouse                         Analytics Agent
 │                                      ├── pattern_analysis
BI Reports                             ├── model_drift_detection
(static)                               ├── threshold_calibration
                                        ├── report_generation
                                        └── threat_alert
                                        │
No Feedback Loop                       ┌─────────────────────┐
                                       │  SHARED CONTEXT      │
                                       │  STORE               │
                                       │  (Redis + Vector DB) │
                                       └─────────────────────┘
                                          ↑ writes    ↓ reads
                                       Batch Agent  Real-Time Agent
```

---

## 3. Agent ReAct Reasoning Loop

```
┌──────────────────────────────────────────────────────────┐
│                   Agent ReAct Loop                       │
│                                                          │
│  ┌──────────────┐                                        │
│  │   OBSERVE    │  ◄─── Enriched transaction event       │
│  └──────┬───────┘                                        │
│         │                                                │
│         ▼                                                │
│  ┌──────────────┐                                        │
│  │    THINK     │  "ML score is 0.71 — high. Check graph."│
│  └──────┬───────┘                                        │
│         │                                                │
│         ▼                                                │
│  ┌──────────────┐                                        │
│  │     ACT      │  ──► Tool: graph_risk_check(...)       │
│  └──────┬───────┘                                        │
│         │                                                │
│         ▼                                                │
│  ┌──────────────┐                                        │
│  │   OBSERVE    │  ◄─── "Merchant: 2 fraud connections"  │
│  └──────┬───────┘                                        │
│         │                                                │
│         ▼                                                │
│  ┌──────────────┐                                        │
│  │    THINK     │  "Strong multi-signal. But loyal       │
│  │              │   customer — use soft decline."        │
│  └──────┬───────┘                                        │
│         │                                                │
│         ▼                                                │
│  ┌──────────────┐                                        │
│  │     ACT      │  ──► Tool: decide_and_act(SOFT_DECLINE)│
│  └──────────────┘                                        │
└──────────────────────────────────────────────────────────┘
```

---

## 4. Feedback Loop Detail

```
Real-Time Agent                    Shared Context Store
      │                           ┌────────────────────────┐
      │  writes edge cases ──────►│ Redis                  │
      │                           │  threshold:{segment}   │
      │  reads on each txn ◄──────│  compromised:merchant  │
      │                           │  suspicious:ip         │
      │                           │  velocity:{customer}   │
      │                           ├────────────────────────┤
      │                           │ Vector DB              │
      │                           │  fraud_patterns        │
      │  semantic search ◄────────│  case_summaries        │
      │                           │  threat_intel          │
Batch Agent                       └────────────────────────┘
      │                                        ▲
      │  writes thresholds ─────────────────── │
      │  writes threat intel ───────────────── │
      │  writes pattern embeddings ─────────── │
```

---

## 5. ML Model Governance Flow

```
Batch Agent
    │
    ├── Computes: PSI, F1 delta, chargeback rate
    │
    ▼
Drift Detected? (PSI > 0.2 or F1 drop > 5%)
    │
    YES ──► JIRA ticket created
    │       Slack alert to #ml-fraud-team
    │       Drift report attached
    │
    ▼
ML Team reviews ──► Approve? ──NO──► Monitor (no action)
    │
   YES
    │
    ▼
Retraining pipeline
    │
    ▼
Staging deployment (shadow mode, 10% traffic)
    │
    ▼
Metrics pass? ──NO──► Roll back to current model
    │
   YES
    │
    ▼
Production rollout
    │
    ▼
Real-time agent ml_score tool ──► new model endpoint
```
