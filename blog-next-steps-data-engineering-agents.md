# Next Steps for Data Engineering: Workflow Agents and Tools

> *Using fraud management as a real-world lens on the agentic pipeline shift*

---

Data engineering has always been about moving data reliably, fast, and at scale. Kafka replaced message queues. Flink and Spark replaced batch-only ETL. The feature store replaced ad-hoc Redis hacks. Each wave simplified how we connect raw data to business decisions.

The next wave is already here — and it doesn't look like a new database or a faster stream processor. It looks like an **agent with tools**.

This post walks through what that shift means in practice, using a fraud management pipeline as a concrete example. Not a toy example — a real one: point-of-sale transactions, Kafka topics, Flink jobs, Redis feature stores, and ML models making millisecond decisions.

---

## The Pipeline That Works (But Has Ceilings)

Most modern fraud pipelines follow a pattern that data engineers know well:

```
POS Terminal → API → Kafka → Flink → Redis → ML Model → ALLOW / DENY
```

And for batch:

```
App Data → Kafka → Spark → ETL → Data Warehouse → BI Reports
```

This architecture is solid. Kafka decouples producers from consumers. Flink handles stateful stream processing with millisecond latency. Redis serves pre-computed features in under 5ms. The ML model wraps years of labeled fraud data into a score.

It works. And it has real ceilings.

**Ceiling 1: Explainability.** The ML model produces a score — say, 0.73. The system denies the transaction. Why? There's no answer that a human can read. A customer dispute? A regulatory audit? A fraud analyst's review? All of them hit the same wall: a number with no narrative.

**Ceiling 2: Static thresholds.** The deny/approve cutoff (e.g., score > 0.65) is engineered into config or code. Changing it for a specific customer segment, a new fraud pattern, or a seasonal shift requires a deployment. Fraud doesn't wait for deployment windows.

**Ceiling 3: Single-shot decisioning.** The ML model gets one chance — one input, one output. There's no mechanism to say: "This score is 0.71. I'm not sure. Let me check whether this merchant is connected to a known fraud ring before I decide." The pipeline can't investigate. It can only score.

**Ceiling 4: No closed loop.** The batch pipeline produces chargeback reconciliation data, fraud labels, and updated risk profiles. But those insights don't reach the real-time pipeline until the next scheduled job — at best, an hour later. Emerging fraud patterns discovered at 2am don't influence real-time decisions until 3am.

These aren't flaws in the engineering. They're constraints of the paradigm — deterministic pipelines making single, irreversible decisions.

---

## What Changes with Workflow Agents

An **agent** is a system that takes an input, reasons over it, and decides which actions (tools) to invoke — iteratively — until it reaches a conclusion. This is the ReAct pattern: **Reason, Act, Observe, Repeat**.

In a data pipeline context, the tools are already familiar:

- **Feature store query** — read from Redis
- **ML model invocation** — call the scoring service
- **Rule engine check** — evaluate business rules
- **Enrichment service** — call IP reputation or device intelligence APIs
- **Graph lookup** — traverse a fraud network graph

What's new is that a reasoning layer — typically a large language model — sits between the incoming event and these tools. Instead of calling them all in sequence regardless of context, the agent **decides which tools to call, in what order, based on what it observes at each step**.

Here's what that looks like in practice for a flagged transaction:

```
INPUT: $847 transaction at a new merchant, 340 miles from the customer's last transaction,
       on a device the system has never seen before.

THINK: "Multiple anomaly signals. ML score will likely be elevated. Let me check."

ACT:   ml_score(transaction)
OBS:   fraud_probability: 0.71 — high, but not conclusive

THINK: "Score is high but the customer has a 5-year history. Before hard denying,
       check whether this merchant is in a known fraud network."

ACT:   graph_risk_check(merchant_id)
OBS:   "Merchant connected to 2 fraud cases in last 30 days. Network risk: ELEVATED."

THINK: "Combined signal is strong. But a hard deny for a loyal customer with a clean
       history creates friction. Use step-up authentication — let the customer verify."

ACT:   decide_and_act(SOFT_DECLINE, rationale="Geo anomaly + new device + elevated
       merchant network risk. OTP step-up required.")
```

This is not possible in a deterministic pipeline. The ML model can't pause, notice uncertainty, and decide to run an additional check. The rule engine can't read its own output and say "I flagged this, but is that flag actually meaningful given the customer's history?" The agent can do both.

---

## The Architecture, Updated

The two-lane architecture stays. Streaming and batch are still the right tools for their respective jobs. What changes is what sits at the decision points.

```
REAL-TIME LANE

POS → API → Kafka → Flink (feature extraction)
                         │
                    Risk Tier Router
                    ├── LOW  → Auto-approve (no agent, <50ms)
                    └── MED/HIGH → Fraud Decisioning Agent
                                      ├── feature_store_query
                                      ├── ml_score
                                      ├── rule_engine_check
                                      ├── enrich_context
                                      └── graph_risk_check
                                      ↓
                                  APPROVE / SOFT DECLINE / DENY
                                  + Natural Language Explanation
```

```
BATCH LANE

App Data → Kafka → Spark → Analytics Agent
                                ├── pattern_analysis
                                ├── model_drift_detection
                                ├── threshold_calibration
                                ├── report_generation
                                └── threat_alert
                                ↓
                        Updated thresholds + BI reports + ML drift alerts
```

And the piece that connects them — the **feedback loop**:

```
Batch Agent ──► Shared Context Store (Redis + Vector DB) ◄──► Real-Time Agent
```

The batch agent writes; the real-time agent reads. Threshold updates, emerging threat intel, newly compromised merchants, geo fraud waves — all of it flows from batch analysis into real-time context within minutes, without a deployment.

---

## The Feedback Loop Is the Real Innovation

Every data engineer has seen the frustration: the batch job discovers a new fraud pattern at 3am. The on-call analyst gets an alert at 8am. The ML team triages it by noon. A model retrain is scheduled for next week. The fraud pattern is exploited all week long.

The agentic feedback loop collapses that timeline:

1. **Batch agent** detects pattern at 3am (event-triggered by chargeback spike)
2. **`threat_alert` tool** writes to shared context store at 3:00:01am
3. **Real-time agent** reads updated context at 3:00:02am on the next transaction
4. Fraud pattern is immediately factored into decisioning — no deployment, no meeting, no retrain

This is the shift from a pipeline that *reports* to one that *adapts*.

The context store itself is a hybrid:

- **Redis** for fast key-value lookups: thresholds by segment, compromised merchant flags, device blacklists
- **Vector DB** for semantic search: the real-time agent queries "is there a known fraud pattern similar to this transaction?" and retrieves narrative descriptions of recent fraud incidents — context that the agent uses in its reasoning

---

## Latency Is Still Real

The obvious question: doesn't adding LLM reasoning blow the latency budget?

It doesn't have to — if you architect around risk tiers.

| Risk Tier | Behavior | Latency |
|---|---|---|
| Low (score < 0.3) | Auto-approve; agent never invoked | < 50ms |
| Medium (0.3–0.7) | Agent: 3 parallel tool calls (feature store + ML + rules) | < 200ms |
| High (> 0.7) | Agent: full chain including graph and enrichment | < 300ms |
| Ambiguous | Soft decline + OTP step-up | Immediate deny, async resolution |

The key insight: **low-risk transactions are the majority**. For an average pipeline, 70–80% of transactions can be auto-approved without touching the agent. The agent concentrates its reasoning on the transactions where investigation actually changes the outcome.

For the agent itself, `claude-haiku-4-5` is the right choice for real-time — it's fast and cheap, and tool orchestration doesn't require deep reasoning capability. Save `claude-sonnet-4-6` for the batch analytics agent where richer reasoning on complex pattern analysis adds real value.

And tool calls can be parallelized where there's no dependency. `feature_store_query`, `ml_score`, and `rule_engine_check` can all run concurrently — the wall clock time is the slowest single call (~30ms), not their sum (~40ms).

---

## Escalation Becomes Intelligent

Today's pipelines have two outcomes: approve or deny. Fraud near the threshold is a coin flip — approve and risk fraud loss; deny and risk customer friction.

Step-up authentication changes this:

```
Agent: SOFT_DECLINE with OTP required
    ↓
Customer receives SMS: "Unusual activity detected. Enter code XXXXXX at terminal."
    ↓
Customer enters OTP → transaction retried with VERIFIED flag
    ↓
Agent re-evaluates with elevated trust → likely APPROVE
```

For legitimate customers with anomalous transactions (travel, large purchase, new device), this is the right outcome: a small friction bump that preserves security without a hard deny. For actual fraud attempts, the OTP is never entered — the transaction dies there.

The agent decides *when* to escalate. It doesn't escalate everything near the threshold — it escalates transactions where a loyal customer profile makes the investigation ambiguous. That's a judgment call that a static rule engine can't make.

---

## ML Governance Gets a Co-Pilot

Model drift is a silent killer. The fraud landscape shifts — new attack vectors, new merchant categories, new device patterns — and the model's distribution assumptions become stale. Precision drops. False positives creep up. Chargebacks rise.

In most organizations, this is caught in a monthly model review meeting. By then, the damage is done.

The batch analytics agent acts as a continuous co-pilot:

1. Computes drift metrics daily: PSI (Population Stability Index), KL-divergence, F1 delta against chargeback labels
2. If PSI > 0.2 or F1 drops > 5%: creates a JIRA ticket with full drift report, affected features, and suggested retraining window
3. ML team reviews and approves
4. Model retrained, shadow-tested at 10% traffic, validated, promoted to production

**The agent flags. Humans decide.** This is the right model for ML governance — automation for detection, human judgment for action. The agent has the data and the tooling to catch drift faster than any manual review cycle. The humans retain accountability for what gets deployed into production.

---

## What This Means for Data Engineers

The infrastructure you've built doesn't go away. Kafka is still the backbone. Flink still handles stateful stream processing better than anything else. Spark is still the right tool for large-scale batch ETL. Redis is still the right choice for millisecond feature lookups.

What changes is the **decision layer** — the part of the pipeline where raw signals become outcomes. That layer is no longer a static score threshold or a fixed rule set. It's a reasoning agent with access to your existing tools.

For data engineers, this means three new responsibilities:

**1. Tool design becomes a first-class concern.** Every tool the agent uses must be reliable, fast, and clearly defined. A tool that times out, returns inconsistent schemas, or has side effects at the wrong time breaks the agent's reasoning chain. Good tool design is as important as good schema design.

**2. The shared context store is new infrastructure.** The Redis + Vector DB layer that connects batch insights to real-time decisions needs the same rigor as any other production data store: TTLs, versioning, rollback capability, observability. It's not a cache — it's the agent's working memory.

**3. Observability must include reasoning traces.** Traditional pipeline monitoring tracks throughput, latency, and error rates. Agentic pipelines need one more dimension: *why did the agent make this decision?* Tool call traces, rationale strings, and intermediate observations are now part of the audit trail — and they're the primary debugging surface when a decision looks wrong.

---

## The Pattern Is General

Fraud management is one vertical. The same pattern applies wherever a pipeline makes a consequential decision on streaming data:

- **Credit underwriting**: agent chains credit bureau queries, income verification, and risk rules before approving a loan
- **Healthcare claims processing**: agent cross-references coverage rules, medical codes, and prior authorization status before adjudicating a claim
- **Supply chain anomaly detection**: agent investigates sensor spikes by querying inventory levels, supplier history, and logistics data before escalating
- **Content moderation**: agent checks policy rules, user history, and contextual signals before making a moderation decision

In every case, the structure is the same: a streaming event arrives, an agent reasons across multiple tools, and a decision is made with an explanation.

---

## Starting Points

You don't need to replace your entire pipeline to start. The agent layer is additive — it sits on top of your existing infrastructure.

A practical progression:

1. **Define your tools** first. Wrap your existing services (ML model, Redis, rule engine) in clean tool interfaces with consistent schemas. This is good practice regardless of whether you add an agent.

2. **Build the feedback loop** next. Getting batch insights into a shared context store that real-time processes can read is high-value on its own, even before the agent is involved.

3. **Start with the medium-risk tier**. Don't touch auto-approve or hard-deny logic. Apply the agent only to the uncertain middle — the 15–20% of transactions where investigation actually changes the outcome.

4. **Instrument reasoning traces** from day one. The first time a decision looks wrong and you have no trace to inspect, you'll wish you started here.

The pipeline that moves data reliably, fast, and at scale — it's not being replaced. It's getting a reasoning layer. That's the next step.

---

*This post is based on a real architecture design for a fraud management platform. The full reference architecture — including current state documentation, future state pipeline specs, tool definitions, and architecture diagrams — is available at [github.com/ManimalaChandra/dataEngineering-FraudManagement](https://github.com/ManimalaChandra/dataEngineering-FraudManagement).*
