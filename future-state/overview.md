# Future State — Agentic Architecture Overview

## The Core Shift

The future state replaces single-shot ML scoring with **LLM-powered agents** that reason across multiple tool calls before making a fraud decision. The two data lanes are preserved (streaming and batch) but are enriched with agent layers that enable multi-step investigation, natural language explainability, adaptive thresholds, and a closed feedback loop between batch and real-time.

```
REAL-TIME (Agentic):
POS → API → Kafka → Flink → [Fraud Decisioning Agent]
                                   ├── Tool: Feature Store Query
                                   ├── Tool: ML Score
                                   ├── Tool: Rule Engine Check
                                   ├── Tool: Context Enrichment
                                   ├── Tool: Graph Risk Check
                                   └── Tool: Decide & Act
                              → APPROVE / SOFT DECLINE (OTP) / DENY

BATCH (Agentic):
App Data → Kafka → Spark → [Analytics Agent]
                                   ├── Tool: Pattern Analysis
                                   ├── Tool: Model Drift Detection
                                   ├── Tool: Threshold Calibration
                                   ├── Tool: Report Generation
                                   └── Tool: Threat Alert
                              → Updated feature store, BI reports, ML drift alerts

FEEDBACK LOOP:
Batch Agent ──► Shared Context Store ◄──► Real-Time Agent
```

---

## Comparison: Current vs Future

| Dimension | Current State | Future State |
|---|---|---|
| **Decisioning model** | Single ML model call → score | Agent: ReAct loop over 3–5 tools |
| **Explainability** | None (opaque score) | Natural language rationale per decision |
| **Threshold management** | Static, engineering-deployed | Batch agent recalibrates per segment continuously |
| **Edge case handling** | Hard deny or approve | Soft decline + OTP step-up authentication |
| **Feedback loop** | None | Batch insights update real-time agent's context store |
| **ML governance** | Manual retraining cycle | Agent detects drift; ML team approves; staged rollout |
| **Emerging threats** | Detected at next batch run (hours) | Batch agent pushes threat intel in near-real-time |
| **Audit trail** | Score + binary outcome | Full tool call trace + natural language reasoning |
| **Multi-signal investigation** | Not possible | Agent chains graph + ML + rules + enrichment |

---

## Latency Strategy

**SLA: 100ms – 300ms end-to-end** for the real-time lane.

| Risk Tier | Agent Behavior | Tools Invoked | Target Latency |
|---|---|---|---|
| Low risk (score < 0.3) | Auto-approve; agent skipped | None | < 50ms |
| Medium risk (0.3–0.7) | Agent: feature store + ML + rules | 3 tools | < 200ms |
| High risk (score > 0.7) | Agent: full chain incl. graph | 4–5 tools | < 300ms |
| Ambiguous / novel | Soft decline + OTP step-up | Async notify | Immediate deny, async resolution |

---

## Model Selection

| Lane | Model | Rationale |
|---|---|---|
| Real-time agent | `claude-haiku-4-5` | Low latency, low cost, sufficient for tool orchestration |
| Batch analytics agent | `claude-sonnet-4-6` | Higher reasoning capability for pattern analysis and reporting |

---

## Key Benefits

1. **Explainability** — Every fraud decision includes a natural language rationale stored in the case management system. Supports customer disputes, regulatory audits, and analyst review.

2. **Multi-hop Investigation** — The agent can chain graph risk, ML scoring, and IP enrichment sequentially — something impossible in the current single-call model.

3. **Closed Feedback Loop** — Batch agent insights (new patterns, threshold adjustments, threat intel) flow into the shared context store, which the real-time agent reads — no redeployment needed.

4. **Adaptive Thresholds** — Batch agent recalibrates risk thresholds by customer segment based on observed false positive/negative rates.

5. **Intelligent Escalation** — Ambiguous cases are soft-declined with step-up authentication (OTP), reducing both fraud loss and unnecessary friction for legitimate customers.

6. **Auditability** — Tool call traces provide a complete audit trail of the agent's reasoning steps — critical for AML compliance and regulatory requirements.

---

## Pipeline Details

- [Real-Time Agentic Pipeline →](real-time-agentic.md)
- [Batch Analytics Agent →](batch-analytics-agent.md)
- [Feedback Loop Design →](feedback-loop.md)
- [Tool Taxonomy →](tool-taxonomy.md)
