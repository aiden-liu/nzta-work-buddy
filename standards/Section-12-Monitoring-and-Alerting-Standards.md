## 12. Monitoring & Alerting Standards
[[_TOC_]]

Monitoring and alerting ensure that the Data Platform operates reliably and that issues are detected and resolved before they impact downstream consumers. This section establishes a consistent baseline that makes the platform governable, auditable, and safe to operate at scale.

Without a shared standard, monitoring is left to individual effort and project timelines — leading to inconsistency across teams, which cascades into governance gaps and silent failures. This standard addresses four failure modes:

- **Silent failures** — pipelines fail or produce bad data with no detection
- **Drift without detection** — pipelines technically succeed but data meaning quietly degrades (late arrivals, sum mismatches, quality drift). No alert fires because nothing "broke"
- **Governance gaps** — no audit trail to prove monitoring was in place
- **Inconsistency** — teams monitor differently, making platform-wide visibility impossible

Monitoring covers infrastructure, platform services, and data pipelines, in compliance with the **NZISM**, NZTA ICT monitoring policies, and the platform's observability standards (Section 4).

---

### 12.1 Monitoring Scope

Monitoring covers three layers of the platform: **infrastructure** (compute, storage, networking), **platform services** (Databricks, Unity Catalog, ADLS, Power BI), and **data pipelines** (ADF pipelines, Databricks jobs). All monitoring is centralised in approved observability platforms to ensure consistent coverage and governance oversight.

**Ownership** is split across two tiers:

- The **platform team** is accountable for infrastructure and platform service monitoring, and for providing the shared monitoring frameworks and tooling that domain and project teams build upon. The platform team is also responsible for monitoring the integration seams between platform services (e.g. ADF triggering a Databricks job).
- The **domain team** is accountable for pipeline and data product monitoring within the frameworks the platform team provides. Monitoring ownership must be formally transferred from the project team to the domain team upon project completion.

Monitoring is active across all environments, with requirements scaled to environment risk:

| Environment | Requirement |
|---|---|
| **Dev** | Structured logging must be present; alerting is optional |
| **Test** | Structured logging required; alerting on pipeline failures required |
| **Prod** | Full monitoring required; all critical alerts active and routed to ServiceNow |

---

### 12.2 Metrics & Telemetry

Metrics are quantified, time-series measurements derived from aggregating raw events, used to understand system behaviour over time and trigger automated responses when thresholds are breached. Telemetry is the continuous, automated collection and transmission of observability data from systems to a centralised platform — it is the pipeline that carries the signal. Alerting, dashboards, and audit trails are all consumers of that signal.

Telemetry is active across all environments (Dev, Test, Prod). Retention requirements apply to Production only — telemetry data is retained for a minimum of 180 days to support audit and forensic analysis (see Section 4.4).

Metrics and telemetry cover the three pillars of observability (see Section 4):

- **Metrics** — system utilisation, job success/failure, error rates, latency, throughput, and queue depths
- **Logs** — structured logs emitted by all pipelines and services in alignment with Section 4.1
- **Traces** — all pipelines emit a consistent correlation identifier (e.g. `run_id`, `pipeline_id`) to enable future end-to-end trace correlation across platform services. Full distributed tracing is a platform maturity goal.

Data freshness and completeness metrics are collected to validate the timeliness and completeness of ingestion and processing. Freshness and completeness thresholds should be defined and registered in Unity Catalog, typically by the domain team responsible for the data product.

Telemetry is reliable, consistent, and tamper-evident. If telemetry is absent or broken, all downstream consumers — alerts, dashboards, audit trails — are blind. Telemetry health is therefore a first-class platform concern.

*See the [Section 12 Implementation Guide](./Section-12-Implementation-Guide.md) for tool-specific metric examples across ADF, Databricks, ADLS, and Power BI.*

---

### 12.3 Alerting & Escalation

Alerting translates monitoring signals into timely, actionable notifications that reach the right people with enough context to respond effectively. Alerts integrate with **ServiceNow** for incident management, ownership assignment, and escalation tracking.

**Alert severity levels** are defined by impact and urgency:

| Level | Characteristics | Response |
|---|---|---|
| **Critical** | Platform or pipeline completely down, data unavailable, production impacted, or SLO breach occurring | Immediate response during business hours; ServiceNow incident auto-created |
| **High** | Significant degradation, data delayed or partially incorrect, risk of downstream impact if unresolved | Prompt response; ServiceNow incident created; domain team notified |
| **Medium** | Non-critical anomaly, pipeline degraded but still producing, investigation warranted | Reviewed within business hours; ServiceNow ticket created |
| **Low** | Informational, no immediate impact, trend worth monitoring | Logged and reviewed on a regular cadence; alerts that consistently produce no action are tuned or suppressed |

**Runbooks** exist for all critical alerts, defining investigation and remediation steps. Runbook ownership follows the two-tier model — the platform team maintains runbooks for infrastructure and platform service alerts; domain teams maintain runbooks for pipeline and data product alerts.

**Alert noise** is actively managed. Thresholds are tuned and correlated to prevent alert fatigue — an alert that fires constantly without producing action is a misconfigured alert, not a monitoring success. Low severity alerts are reviewed regularly and either escalated, tuned, or suppressed.

---

### 12.4 Data Pipeline Health

Pipeline health monitoring covers three categories, in order of increasing difficulty and domain specificity:

- **Execution health** — whether the pipeline ran, completed within expected duration, and exited without errors. This is the minimum baseline, monitored for all pipelines across all environments.
- **Volume health** — whether the expected amount of data arrived. Automated alerts trigger on unexpected record count drops, missing source files, processing backlogs, or late data arrival beyond defined thresholds. Volume thresholds should be defined and registered in Unity Catalog by the domain team.
- **Correctness health** — whether the data is semantically valid. Quality validation results (see Section 8) are integrated into pipeline health monitoring. Correctness rules are defined by the domain team and enforced within the pipeline; failures surface as alerts rather than being silently swallowed.

Automated alerts are the primary detection mechanism for pipeline health. Alerts carry sufficient context for an engineer to begin investigation immediately — including pipeline identifier, failure stage, expected vs actual values, last successful run time, and a correlation identifier.

Observability dashboards provide pipeline health visibility for team leads and platform oversight — serving as an investigation and situational awareness tool rather than a substitute for automated alerting.

The platform team provides reusable frameworks for volume and correctness health monitoring to ensure detection capability is consistent across pipelines and independent of individual project delivery timelines. Domain teams engage with these frameworks to define and maintain rules for their data products. Monitoring without domain-defined rules is incomplete.

---

### 12.5 Governance & Continuous Improvement

Monitoring and alerting standards are reviewed annually at minimum, and triggered by significant platform or organisational changes — such as new tooling adoption, team restructuring, or major pipeline redesigns.

When post-mortems occur following incidents, they include an assessment of whether monitoring and alerting coverage was adequate — specifically whether the right alerts existed, fired in time, carried sufficient context, and reached the right people. Lessons are fed back into the standard or the implementation guide as appropriate.
