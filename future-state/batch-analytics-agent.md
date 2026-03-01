# Future State — Batch Analytics Agent

## Flow

```
Operational Data Sources
    ├── Transaction DB (chargebacks, confirmations, reversals)
    ├── Customer profiles
    └── Fraud case outcomes (analyst verdicts)
    │
    │  CDC / scheduled export
    ↓
Apache Kafka  [Topic: batch-events, chargeback-events, label-updates]
    │
    ↓
Apache Spark  (ETL & Feature Engineering)
    │  ├── Chargeback reconciliation (match to original transactions)
    │  ├── False positive / false negative labeling
    │  ├── Rolling feature aggregations (7d, 30d, 90d profiles)
    │  ├── Customer segment risk band computation
    │  └── Merchant risk score refresh
    │
    ↓
┌───────────────────────────────────────────────────────────┐
│             Batch Analytics Agent                          │
│             Model: claude-sonnet-4-6                       │
│             Trigger: Scheduled (hourly/daily) OR           │
│                      Event-driven (chargeback spike)       │
│                                                           │
│  ACT → Tool 1: pattern_analysis                            │
│    Scans recent transactions for emerging fraud patterns  │
│    (new merchant categories, geo clusters, device types)  │
│                                                           │
│  ACT → Tool 2: model_drift_detection                      │
│    Computes PSI, KL-divergence on prediction distribution │
│    Compares F1/precision/recall against baseline          │
│                                                           │
│  ACT → Tool 3: threshold_calibration                      │
│    Adjusts real-time risk thresholds per customer segment │
│    Balances false positive rate vs fraud loss             │
│                                                           │
│  ACT → Tool 4: report_generation                          │
│    Produces BI dashboards, executive summaries,           │
│    regulatory reports (SAR, AML), model performance       │
│                                                           │
│  ACT → Tool 5: threat_alert                               │
│    Pushes emerging threats to shared context store        │
│    (e.g., new compromised merchant list, geo fraud wave)  │
└───────────────────────────┬───────────────────────────────┘
                            │
            ┌───────────────┼────────────────────┐
            ↓               ↓                    ↓
    Updated Feature   BI Dashboards +      ML Drift Alert
    Store (Redis)     Regulatory Reports   (JIRA / Slack)
            │               │                    │
    Real-time agent   Analyst / Exec        ML Team reviews
    reads updated     consume reports       → approves
    thresholds                              retraining
```

---

## Tool Details

### Tool 1: `pattern_analysis`

Detects emerging fraud patterns not yet captured in existing rules or the ML model.

**Inputs**: Time window, customer segment, merchant category
**Logic**:
- Clusters recent denied transactions by feature similarity
- Identifies statistical anomalies vs. 30-day baseline
- Flags new merchant categories with rising chargeback rates
- Detects geo-temporal fraud waves (e.g., city-level card skimming events)

**Output**: Structured pattern report + severity classification (LOW / MEDIUM / HIGH)

---

### Tool 2: `model_drift_detection`

Monitors the production ML fraud model for degradation.

**Inputs**: Model ID, recent prediction window (e.g., last 7 days)
**Metrics computed**:
- **PSI (Population Stability Index)**: feature distribution shift
- **KL-Divergence**: prediction score distribution shift
- **F1 / Precision / Recall**: vs. labeled chargeback ground truth
- **Approval rate delta**: vs. 30-day baseline by segment

**Thresholds**:
- PSI > 0.2 → Model drift warning
- F1 drop > 5% → Drift alert, retraining recommended
- Chargeback spike > 2σ → Immediate alert

---

### Tool 3: `threshold_calibration`

Adjusts real-time fraud scoring thresholds without requiring model retraining or engineering deployment.

**Inputs**: Customer segment, observed false positive rate, observed fraud rate
**Logic**:
- Optimizes threshold per segment using precision-recall trade-off
- Constrains: false positive rate < 2%, fraud loss < business tolerance
- Writes updated thresholds to Redis (key: `threshold:{segment}`)
- Real-time agent reads from Redis on each decision — picks up changes immediately

**Output**: Updated threshold config in Redis; audit log of changes

---

### Tool 4: `report_generation`

Generates structured reports for business and regulatory consumers.

**Report types**:

| Report | Cadence | Audience |
|---|---|---|
| Executive fraud dashboard | Daily | C-suite, Risk team |
| Analyst operations report | Daily | Fraud ops team |
| Model performance summary | Weekly | ML team |
| SAR (Suspicious Activity Report) | As needed | Regulators |
| AML compliance summary | Monthly | Compliance team |

**Output**: Reports pushed to data warehouse + dashboard (Tableau/Looker) and delivered via scheduled email.

---

### Tool 5: `threat_alert`

Publishes emerging threats discovered by the batch agent to the shared context store, where the real-time agent can act on them immediately.

**Examples of threats pushed**:
- Newly compromised merchant list (e.g., merchant data breach detected in chargeback data)
- Geo fraud wave (e.g., card skimming cluster in a specific city)
- Device fingerprint pattern linked to fraud ring
- Mule account network detected via graph analysis

**Output**: Written to Redis + vector DB (for semantic lookup by real-time agent)

---

## ML Model Governance Flow

```
Batch Agent detects model drift (PSI > 0.2 or F1 drop > 5%)
    ↓
Tool: model_drift_alert(metrics, severity)
    │
    ├── Creates JIRA ticket with:
    │     - Drift metrics report
    │     - Suggested retraining data window
    │     - Feature importance delta (what shifted)
    │     - Recommended new threshold for validation
    │
    └── Posts Slack alert to #ml-fraud-team channel
    ↓
ML Team reviews drift report
    ↓
Approves retraining job (or rejects if drift is acceptable)
    ↓
Retraining pipeline executes:
    - Data slicing (configurable window)
    - Feature engineering (same Spark jobs)
    - Model training + hyperparameter optimization
    - Validation on holdout set
    ↓
Staging deployment → A/B test (10% traffic shadow)
    ↓
Performance validated → ML Team approves production rollout
    ↓
Real-time agent's ml_score tool updated to point to new model endpoint
```

---

## Batch Agent Cadence

| Run | Trigger | Primary Tools | Output |
|---|---|---|---|
| Hourly | Scheduled | threshold_calibration | Updated Redis thresholds |
| Daily | Scheduled | pattern_analysis, report_generation | BI reports, pattern alerts |
| Daily | Scheduled | model_drift_detection | Drift metrics, ML team alert if needed |
| Event-driven | Chargeback spike (+2σ) | pattern_analysis, threat_alert | Immediate threat push to context store |
| Weekly | Scheduled | report_generation | Regulatory reports, model performance |
