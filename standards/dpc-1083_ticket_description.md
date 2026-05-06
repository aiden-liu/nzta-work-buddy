📖 User Story
As a platform data engineer, I would like to author and publish Section 12: Monitoring & Alerting Standards to the Data Platform Confluence knowledge base so that all platform and product teams have a clear, governance-approved reference for how monitoring, alerting, and observability must be implemented and maintained across the platform.

✅ Acceptance Criteria
12.1 — Monitoring Scope
Document that monitoring must cover infrastructure (compute, storage, networking), platform services (Databricks, Unity Catalog, ADLS, Power BI), and data pipelines.

Document that monitoring coverage must extend across Dev, Test, UAT, and Production environments, with higher rigour applied to UAT and Production.

Document that all monitoring must be centralised in approved observability platforms to ensure consistent coverage and governance oversight.

12.2 — Metrics & Telemetry
Document that monitoring must capture the three pillars of observability: metrics, logs, and traces (cross-referencing Section 4).

Document that standard metrics must include system utilisation, job success/failure, error rates, latency, throughput, and queue sizes.

Document that data freshness and completeness metrics must be collected to validate timeliness of ingestion and processing.

Document that telemetry data must be retained for a minimum of 180 days in alignment with Section 4 for audit and forensic purposes.

12.3 — Alerting & Escalation
Document that alerts must be configured for critical system failures, SLA/SLO breaches, and data quality incidents.

Document that four alert severity levels (critical, high, medium, low) must be defined with corresponding response time targets for each.

Document that alerts must integrate with IvantiCloud ITSM for incident management and escalation workflows.

Document that runbooks must exist for all critical alerts, defining investigation and remediation steps.

Document that alert noise must be minimised through threshold tuning and event correlation to prevent alert fatigue.

12.4 — Data Pipeline Health
Document that pipelines must be monitored for latency, throughput, error rates, and data freshness to ensure reliable operation.

Document that automated alerts must trigger on unexpected delays, backlogs, or failure patterns in pipeline execution.

Document that observability dashboards must visualise pipeline health in real time for both platform and product teams.

Document that quality validation results (cross-referencing Section 8) must be integrated into pipeline health monitoring.

12.5 — Governance & Continuous Improvement
Document that monitoring and alerting standards must be reviewed annually to align with evolving NZISM and Stats NZ ICT policies.

Document that incident post-mortems must include an assessment of monitoring and alerting coverage, with lessons fed back into improvements.

Document that product teams must regularly review alert dashboards to ensure pipeline-specific monitoring remains relevant.

Document that platform-level monitoring metrics and compliance must be reported to Stats NZ governance forums.

Publication
Verify that the completed Section 12 page is published under the correct parent page in the Data Platform Confluence space and is accessible to all relevant teams at 
[DIP - Monitoring & Alerting Standards](https://waka-kotahi.atlassian.net/wiki/x/IQCX-/)

🚫 Out of Scope
Authoring the detailed content of individual runbooks (runbook existence is in-scope; their detailed step-by-step content is not).

Configuration or deployment of monitoring tooling or dashboards (standards documentation only).

Non-Azure or non-approved platform components not referenced in the document.

Defining specific numeric SLA/SLO thresholds or response time targets.

Incident response procedures beyond the alerting and escalation standards described in 12.3.

🔗 Dependencies
Section 4 — Observability Standards (upstream): Section 12 directly references Section 4 for the three pillars of observability and the 180-day telemetry retention requirement. Section 4 must be published before Section 12 can be finalised.

Section 8 — Data Quality Validation (upstream): Section 12.4 requires quality validation results to be integrated into pipeline health monitoring. Section 8 must be published and its outputs understood before Section 12 can be completed.

🎤 Presentation
Sprint review walkthrough of the published Confluence page at 
Draft Section 12: Monitoring & Alerting Standards