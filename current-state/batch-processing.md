# Current State — Batch Processing Pipeline

## Flow

```
Application Database  (transactions, customer profiles, merchant data)
    │
    │  CDC (Change Data Capture) or scheduled export
    ↓
Apache Kafka  [Topic: batch-events]
    │
    │  Spark structured streaming or batch read
    ↓
Apache Spark  (ETL & Feature Engineering)
    │  ├── Data cleansing and validation
    │  ├── Joins: transactions ↔ customer profiles ↔ merchant master
    │  ├── Chargeback reconciliation (match chargebacks to original transactions)
    │  ├── Feature aggregation:
    │  │       - 7-day, 30-day, 90-day spend profiles
    │  │       - Merchant category risk aggregates
    │  │       - Customer segment risk bands
    │  └── Fraud label propagation (confirmed fraud / chargeback labels)
    │
    ↓
Data Warehouse  (Snowflake / BigQuery / Redshift)
    │  ├── fraud_transactions (labeled)
    │  ├── customer_risk_profiles
    │  ├── merchant_risk_scores
    │  └── daily_fraud_summary
    │
    ↓
BI / Reporting Layer
    ├── Executive dashboards (fraud rate, financial loss, approval rate)
    ├── Operations dashboards (analyst queue, case volume, SLA metrics)
    ├── Regulatory reports (SAR filings, AML summaries)
    └── Model performance reports (precision, recall, F1 by segment)
```

---

## Components

### Apache Kafka (Batch Events)
- **Topics**: `batch-events`, `chargeback-events`, `label-updates`
- **Producer**: Application DB CDC connector (Debezium) or nightly export jobs
- **Consumer**: Spark structured streaming or scheduled Spark batch jobs

### Apache Spark
- **Cluster**: Managed (EMR / Dataproc / Databricks)
- **Job cadence**:
  - Hourly: Feature table refresh (velocity baselines, rolling aggregates)
  - Daily: Chargeback reconciliation, model performance metrics
  - Weekly: Full customer risk profile rebuild
- **Output sinks**: Data warehouse tables, Redis feature store updates

### Data Warehouse
- Stores historical fraud data for model training datasets
- Source of truth for BI and regulatory reporting
- Retention: 7 years (regulatory requirement)

### BI / Reporting
- Dashboards: Tableau / Looker / Power BI
- Report delivery: Scheduled email exports, regulatory filing formats

---

## ETL Job Details

| Job | Cadence | Input | Output |
|---|---|---|---|
| Feature Refresh | Hourly | Raw transactions | Redis feature store |
| Chargeback Reconcile | Daily | Chargebacks + transactions | Labeled dataset in DWH |
| Customer Risk Profile | Daily | 90-day transaction history | Risk band per customer |
| Merchant Risk Score | Daily | Merchant transaction patterns | Merchant risk table |
| Model Performance | Daily | Predictions + actual labels | Performance metrics table |
| Regulatory Report | Weekly / Monthly | Confirmed fraud records | SAR / AML report files |

---

## Data Flow: Feedback to Real-Time (Missing Today)

```
Batch produces:
  ├── Updated merchant risk scores
  ├── Adjusted customer risk bands
  └── New fraud pattern labels

These are written to:
  ├── Data Warehouse  ✓ (for BI)
  └── Redis Feature Store  ✓ (for real-time ML, but delayed by job cadence)

What is NOT happening today:
  ├── ✗ No dynamic threshold adjustment based on batch insights
  ├── ✗ No emerging threat signals pushed to real-time scoring
  └── ✗ No agent context updates between model retraining cycles
```

---

## Limitations Highlighted

1. **Lag in feedback**: Batch insights (new fraud patterns, chargeback spikes) only reach the real-time pipeline at the next job run (hourly at best).
2. **Static BI**: Reports describe what happened; they don't detect emerging patterns or trigger actions.
3. **No closed loop**: The batch pipeline produces model training data but does not trigger retraining or threshold adjustment automatically.
4. **Manual model updates**: The ML team must manually initiate retraining based on reviewing batch reports — not event-driven.
