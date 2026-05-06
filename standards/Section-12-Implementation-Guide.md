---
This guide on remote Confluence is at page 4285792405.
---

# DIP - Monitoring & Alerting Implementation Guide

> This guide is a companion to [Section 12: Monitoring & Alerting Standards](./Section-12%3A-Monitoring-&-Alerting-Standards.md).
> It provides tool-specific implementation detail for the NZTA Data Platform tech stack.
> The standard defines **what must be true**; this guide explains **how to deliver it**.
>
> **Owned by:** Platform Engineering team
> **Change cadence:** Updated as tools and practices evolve — does not require governance approval cycle
> **Stack:** Azure Data Factory, Azure Databricks, ADLS, Power BI, PySpark, SQL, GitHub EMU, ServiceNow

[[_TOC_]]

---

## 12.1 Monitoring Scope — Implementation Notes

### Production Release Readiness Checklist

Before a data pipeline or data product is released to production, the following monitoring checklist must be satisfied:

**Execution monitoring**
- [ ] Pipeline failure alerts are configured and routed to ServiceNow
- [ ] Alert severity is set appropriately (Critical or High for production pipelines)
- [ ] A runbook exists or is linked for each Critical alert

**Volume monitoring**
- [ ] Volume thresholds are defined and registered in Unity Catalog
- [ ] Alerts are configured for unexpected record count drops, missing files, or late arrivals

**Correctness monitoring**
- [ ] Correctness rules are defined by the domain team and registered in Unity Catalog
- [ ] Validation failures surface as pipeline alerts (not silently swallowed)

**Telemetry**
- [ ] Structured logging is enabled and emitting a consistent correlation identifier (e.g. `run_id`)
- [ ] Logs are flowing to the approved observability platform (Azure Log Analytics)

**Ownership**
- [ ] Monitoring ownership is formally assigned to the domain team
- [ ] Alert routing is verified end-to-end (alert fires → ServiceNow ticket created → correct team notified)

This checklist is a release gate. Pipelines that do not satisfy it are not production-ready.

---

## 12.2 Metrics & Telemetry — Implementation Notes

### Azure Data Factory
*(to be populated)*
- Pipeline run status
- Activity duration
- Retry count
- Trigger lag

### Azure Databricks
*(to be populated)*
- Job run duration
- Cluster startup time
- DBU consumption
- Task failure rate

### ADLS
*(to be populated)*
- Read/write latency
- Storage capacity utilisation
- Failed access attempts

### Power BI
*(to be populated)*
- Dataset refresh success/failure
- Refresh duration

---

## 12.3 Alerting & Escalation — Implementation Notes

### ServiceNow Integration

Alert routing to ServiceNow is automated — an alert firing must create a ServiceNow incident without manual intervention. Azure Monitor is the centralised routing layer for both ADF and Databricks pipelines.

The automation chain is:

> **Pipeline failure → Azure Monitor alert rule → Action Group (webhook) → ServiceNow incident**

Implementation steps:

1. Configure Azure Monitor alert rules for each pipeline, scoped to the appropriate severity.
2. Create an Action Group with a webhook action pointing to the ServiceNow REST API endpoint.
3. Map alert severity to ServiceNow incident priority:
   - **Critical** → P1 incident, auto-created and assigned to the on-call team
   - **High** → P2 incident, assigned to the domain team
   - **Medium** → P3 ticket, assigned to the domain team
   - **Low** → logged only; no ServiceNow incident created
4. Ensure the ServiceNow incident is pre-populated with: pipeline identifier, alert severity, failure description, and a correlation identifier (e.g. `run_id`).

This wiring must be in place before a pipeline is released to production (see 12.1 readiness checklist).

### Alert Response SLA Targets

The alert severity table in the standard describes the response process. The following indicative time targets apply in production:

| Severity | Acknowledgement | Investigation Start | Resolution Target |
|---|---|---|---|
| **Critical** | 15 minutes | Immediate | Same business day |
| **High** | 1 hour | Within 2 hours | Within 2 business days |
| **Medium** | 4 hours | Within 1 business day | Within 1 sprint |
| **Low** | Next review cycle | Next review cycle | Best effort; suppress if no action warranted |

These targets apply during business hours. Out-of-hours response for Critical alerts is subject to on-call arrangements defined by the platform team.

SLA targets are reviewed annually alongside the standard.

### Correctness Rule Definition Process

Domain teams are responsible for defining correctness rules for their data products. The process is:

1. Domain team identifies a correctness rule (e.g. no nulls in a key column, sum of daily totals must match source).
2. Domain team raises a Jira story in their backlog to implement the rule, tagging the platform team for framework support if needed.
3. The rule is implemented within the pipeline and registered in Unity Catalog against the relevant table or data product.
4. Failures surface as pipeline alerts — they are never silently swallowed.

The platform team provides reusable validation frameworks (see 12.4). Domain teams are not expected to build detection infrastructure from scratch.

**SLA for rule implementation:** Correctness rules for new data products must be defined and implemented before production release. For existing products, rules are prioritised through the normal team backlog process — there is no platform-mandated SLA, but unmonitored production data products should be treated as a risk item.

### Low-Severity Alert Review Cadence

Low-severity alerts are reviewed on a **monthly** basis. Reviews should be time-boxed and outcome-focused — each alert is either:

- **Kept** — still relevant, no change needed.
- **Tuned** — threshold or condition adjusted.
- **Escalated** — reclassified to a higher severity.
- **Suppressed** — formally documented as producing no action; removed from active alerting.

Suppression is a deliberate decision and must be recorded (e.g. as a Jira ticket or a comment in the alert configuration). Alerts are not left open indefinitely.

The monthly review can be run as a standing agenda item in an existing team sync — it does not require a dedicated meeting.

---

## 12.4 Data Pipeline Health — Implementation Notes

### Volume Threshold Monitoring

Volume thresholds answer the question: *did the right amount of data arrive?* This is a data completeness check — a pipeline can exit without errors and still produce incomplete or missing data if the source sends fewer records than expected, a file arrives late, or an upstream bug causes a silent drop.

The appropriate technique depends on the pipeline type:

- **Row count banding** — for raw ingestion pipelines. Define an acceptable range based on historical averages (e.g. ±20% of rolling 7-day average). Catches sudden drops without requiring an exact expected count.
- **Source-to-target reconciliation** — for transformation or aggregation pipelines. Compare a control total (e.g. sum of a key financial column, count of distinct IDs) between source and target. The row count may differ due to aggregation, but the control total should match.
- **Incremental volume checks** — for daily increment pipelines. Validate today's load against a rolling average of recent days. Catches silent drops or late arrivals.
- **DLT Expectations** — Databricks Delta Live Tables has built-in expectations that enforce completeness rules and surface failures as pipeline metrics natively.

Domain teams select the appropriate technique and define the thresholds. The platform team provides reusable framework components where possible.

### Registering Thresholds in Unity Catalog

Thresholds are stored as **table properties** in Unity Catalog — key-value metadata attached to the target table. This makes thresholds discoverable, queryable by the pipeline validation step, and governed under the same access controls as the table itself.

Agreed naming convention for threshold properties:

```sql
ALTER TABLE <catalog>.<schema>.<table>
SET TBLPROPERTIES (
  'monitoring.volume.method'    = 'row_count_band',  -- or 'reconciliation', 'incremental'
  'monitoring.volume.min_rows'  = '8000',
  'monitoring.volume.max_rows'  = '12000',
  'monitoring.volume.owner'     = 'data-management-team'
);
```

The pipeline validation step reads these properties at runtime rather than hardcoding values — this keeps thresholds maintainable without requiring code changes.

> **Note:** Unity Catalog table properties are a pattern, not a native threshold registry. The team must agree on and adhere to the naming convention above for this to work consistently across pipelines.

---

## 12.5 Governance & Continuous Improvement — Implementation Notes

*(to be populated)*
