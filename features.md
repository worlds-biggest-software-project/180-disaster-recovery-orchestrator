# Disaster Recovery Orchestrator — Feature & Functionality Survey

> Candidate #180 · Researched: 2026-05-03

## Solutions Analysed

| Tool | Type | Licence / Model | URL |
|------|------|-----------------|-----|
| Veeam Recovery Orchestrator (VRO) | SaaS / On-prem | Commercial (bundled with Veeam Data Platform Premium) | https://www.veeam.com/products/veeam-data-platform/recovery-orchestration.html |
| HPE Zerto (from Commvault) | SaaS / On-prem | Commercial, enterprise pricing | https://www.zerto.com/ |
| AWS Elastic Disaster Recovery (DRS) | SaaS (cloud-native) | Commercial, pay-per-resource ($0.028/hr/server) | https://aws.amazon.com/disaster-recovery/ |
| Azure Site Recovery (ASR) | SaaS (cloud-native) | Commercial ($25/instance/month) | https://azure.microsoft.com/en-us/products/site-recovery |
| Commvault Disaster Recovery | SaaS / On-prem | Commercial, enterprise pricing | https://www.commvault.com/ |
| Rubrik Cyber Recovery | SaaS | Commercial, enterprise pricing | https://www.rubrik.com/products/cyber-recovery |
| Druva Cloud Disaster Recovery | SaaS | Commercial, from $2/VM/month | https://www.druva.com/use-cases/cloud-disaster-recovery |
| HYCU R-Cloud | SaaS | Commercial, from $2/VM/month | https://www.hycu.com/solutions/disaster-recovery |
| Cutover | SaaS | Commercial, enterprise pricing | https://cutover.com/ |
| LitmusChaos | Open source | Apache 2.0 | https://litmuschaos.io/ |

---

## Feature Analysis by Solution

### Veeam Recovery Orchestrator (VRO)

**Core features**
- Automated runbook creation, execution, and management for multi-tier application recovery
- Non-disruptive DR testing in isolated clean-room environments without production impact
- Application dependency modelling to enforce correct failover sequencing across VMs, databases, and services
- Scheduled or on-demand DR testing with automated result capture, timing, and gap analysis
- Compliance documentation generation — automatic runbook reports with approvals and evidence for auditors
- Hyper-V support (v7.2, February 2025) and VMware support across on-premises and cloud targets
- RTO/RPO monitoring and SLA dashboards
- Integration with Veeam Backup & Replication as the underlying replication engine

**Differentiating features**
- Full audit trail: captures each test run with timestamps, pass/fail status, and approver sign-off, ready for SOC 2 / ISO 27001 evidence
- One-click runbook execution for declared disaster scenarios
- Automatically detects configuration drift between runbook steps and live infrastructure

**UX patterns**
- Wizard-driven runbook builder guides operators through dependency mapping
- Dashboards show readiness scores per application group and gap-to-SLA
- Portal-based web UI; relatively dated compared to cloud-native tools

**Integration points**
- REST API (OAuth 2.0-based, HAL format) for programmatic orchestration and integration
- Veeam Data Platform ecosystem (Backup & Replication, ONE monitoring)
- PowerShell scripting for custom recovery steps

**Known gaps**
- Heavy on-premises heritage; multi-cloud orchestration less mature than cloud-native competitors
- Limited public cloud workload coverage beyond lift-and-shift VMs
- UI perceived as dated relative to SaaS-native alternatives

**Licence / IP notes**
- Proprietary commercial licence; bundled in Veeam Data Platform Premium Edition

---

### HPE Zerto (from Commvault)

**Core features**
- Continuous journal-based replication (CDP) enabling near-zero RPO (seconds) and fast RTO (minutes)
- Automated failover and failback orchestration with pre-defined protection groups
- Non-disruptive DR testing that runs concurrently with live replication
- Workload mobility and migration orchestration in addition to DR
- Multi-cloud and hybrid cloud support (Azure, AWS, on-premises VMware, Hyper-V)
- Analytics and reporting on replication health, journal history, and RTO/RPO performance
- Integration with Commvault platform post-acquisition (December 2025 GA)

**Differentiating features**
- Journal-based CDP means any point-in-time recovery within the journal window, not just snapshot-based restore
- Simultaneous failover of multiple VMs in a consistency group with application-aware sequencing
- Zerto In-Cloud: standalone AWS DR using EBS snapshots and AWS API, no agent required

**UX patterns**
- Web-based UI with protection group management and replication topology maps
- Recovery site wizards for configuring network mappings and VM settings at the target

**Integration points**
- REST API with significant expansion in Zerto 10 (2024); all core APIs ported to Linux-based ZVM appliance
- OpsRamp integration for monitoring and alerting
- Commvault Command Center integration post-acquisition

**Known gaps**
- Acquisition by HPE then transfer to Commvault creates integration uncertainty for long-term roadmap
- Requires dedicated ZVM appliance; higher infrastructure footprint than cloud-native alternatives
- Replication agent required on source workloads; agentless option limited

**Licence / IP notes**
- Proprietary commercial licence; now part of the Commvault portfolio (December 2025)

---

### AWS Elastic Disaster Recovery (DRS)

**Core features**
- Continuous block-level replication to AWS staging area with crash-consistent RPO of seconds
- Automated failover with RTO typically 5–20 minutes
- Point-in-time recovery for ransomware protection (select a clean point in the replication journal)
- Automated server conversion (physical, VMware, or other cloud to EC2)
- Post-launch actions via AWS Systems Manager integration for automated validation scripts
- Application dependency grouping for sequenced, batched recovery across hundreds of servers
- Failback to original on-premises or other cloud environments

**Differentiating features**
- Native AWS service — no separate orchestration appliance; full AWS IAM, CloudTrail, and CloudWatch integration
- Scale support: 20 concurrent jobs, up to 500 servers across active jobs; automation via Step Functions
- Ransomware-specific point-in-time recovery flow built into the service

**UX patterns**
- AWS Management Console UI; also fully API-driven via boto3 / AWS CLI
- Uses familiar AWS paradigms (Security Groups, IAM roles, VPCs) for target configuration

**Integration points**
- Full REST API (AWS DRS API, latest reference January 2026)
- Python SDK (boto3), all AWS language SDKs supported
- AWS Step Functions for recovery sequence automation
- Systems Manager for post-launch validation hooks

**Known gaps**
- AWS destination only — no cross-cloud failover capability
- Vendor lock-in concern for organisations with multi-cloud or hybrid mandates
- Limited orchestration for non-AWS-native workloads without agent installation
- No built-in runbook documentation or compliance reporting

**Licence / IP notes**
- AWS proprietary service; open-source SDKs (Apache 2.0)

---

### Azure Site Recovery (ASR)

**Core features**
- Replication of Azure VMs between regions; on-premises VMs and physical servers to Azure
- Three failover types: test failover (isolated, no production impact), planned failover, and unplanned failover
- Automated failover and failback orchestration via recovery plans with custom scripts
- Native Azure Monitor integration for replication health alerts (unhealthy replication, failover failures, agent expiry)
- Support for NVMe-enabled Generation 2 VM families for high-performance workloads (2025)
- Business Continuity Center dashboard for unified DR visibility across subscriptions
- Application-consistent replication using VSS (Windows) and pre/post scripts (Linux)

**Differentiating features**
- Deep Azure ecosystem integration — ARM templates, Azure Policy, Azure Monitor, and Azure Automation
- Capacity reservation at the target region for guaranteed resource availability during failover
- Recovery plan sequencing with PowerShell and Azure Automation runbook hooks

**UX patterns**
- Azure Portal-based UI; supports both self-service and managed configurations
- Replication topology and health presented as a map within the Recovery Services vault

**Integration points**
- Site Recovery REST API (latest version 2026-01-01)
- Azure SDKs (Go, .NET, Python, Java, JavaScript) via azure-sdk-for-go and azure-sdk-for-net
- Azure Automation for custom recovery step scripts
- Azure Monitor for alerting and dashboards

**Known gaps**
- Azure destination required — not suitable for cross-cloud or on-premises-to-on-premises orchestration
- Recovery plans limited to Azure Automation scripts; no native integration with third-party ITSM for approvals
- Limited application-specific orchestration depth compared to dedicated DR orchestrators for complex multi-tier apps

**Licence / IP notes**
- Azure proprietary service; SDKs open source (MIT)

---

### Commvault Disaster Recovery

**Core features**
- Recovery Runbooks: automated multi-tier recovery playbooks with interactive sequencing and validation steps
- Cleanroom Recovery: pre-recovery environment orchestration for safe, auditable clean-site recovery
- Arlie Recover AI agent (2025): guides teams through policy-based recovery after cyber events
- Continuous replication for VMware workloads with orchestration to Azure and AWS
- Automated testing with scheduled runs validating backup integrity, RTO, and application functionality
- Multi-cloud orchestration: on-premises, Azure, and AWS supported as recovery targets
- Built-in compliance reporting with audit trail for SOC 2 and ISO 27001 evidence

**Differentiating features**
- Cleanroom Recovery distinguishes Commvault — pre-verifies the recovery environment before initiating workload failover, reducing failed recovery scenarios
- AI-assisted recovery orchestration via Arlie agent helps operators navigate complex recovery sequences
- Deep integration with HPE Zerto continuous replication after acquisition

**UX patterns**
- Commvault Command Center web UI; unified control plane for backup, replication, and orchestration
- Wizard-driven runbook builder with drag-and-drop task sequencing

**Integration points**
- REST API portal (api.commvault.com) with specific DR endpoints (test boot, failover, backup options)
- Postman public workspace for API exploration
- Integrations with ServiceNow, BMC Remedy, and other ITSM platforms for approval workflows
- OpsRamp monitoring integration

**Known gaps**
- Complex licensing; understanding what is included in which edition can be difficult
- Heavy platform with significant implementation and management overhead
- Arlie AI agent is nascent and not yet deeply integrated across all recovery workflow steps

**Licence / IP notes**
- Proprietary commercial licence; Zerto acquisition (completed December 2025) adds CDP capability to portfolio

---

### Rubrik Cyber Recovery

**Core features**
- Threat detection and anomaly analysis (mass encryption detection) to identify last known good backup point
- Guided workflows for file-level, object-level, application-level, or system-wide restore
- Automated isolation of infected workloads and staging to a clean recovery environment
- Cyber Recovery Simulation: test recovery plans in an isolated environment without production impact
- Immutable, air-gapped backups (cannot be encrypted or deleted during ransomware attack)
- Pre-emptive recovery planning: identifies failure points and establishes clean recovery environments in advance
- Google Cloud integration for ransomware-resistant database protection at scale (April 2026)

**Differentiating features**
- Ransomware-first design philosophy: entire workflow optimised for cyber incident recovery, not just infrastructure failure
- AI-powered anomaly detection flags deviation from normal backup patterns to identify attack blast radius
- Validates recovered workloads in an isolated staging environment before production cutover

**UX patterns**
- Clean, modern SaaS UI with dashboard showing protection coverage, threat detections, and recovery readiness
- Incident response playbooks guide operators through the recovery process step-by-step

**Integration points**
- Rubrik REST API for programmatic access
- SIEM integrations (Splunk, Microsoft Sentinel) for threat feed correlation
- SOAR platform integrations for automated response triggering
- Cloud-native integrations (AWS, Azure, Google Cloud)

**Known gaps**
- Backup-first design means DR orchestration depth (dependency management, RTO/RPO modelling) is less mature than dedicated DR orchestrators
- Less suited to infrastructure failure DR (network outage, datacenter loss) compared to ransomware scenarios
- Limited support for non-hypervisor workloads (bare metal, mainframe)

**Licence / IP notes**
- Proprietary commercial licence; IPO'd April 2024 at ~$6B valuation

---

### Druva Cloud Disaster Recovery

**Core features**
- Single-click failover via pre-configured DR plans covering VM selection, network settings, instance types, and failover order
- Automated guest OS readiness checks and AWS environment health checks before failover
- Dependency-based failover sequencing with boot scripts for EC2 recovery
- On-demand and scheduled DR testing with result capture
- Proactive failover and failback readiness checks
- Integration with ServiceNow and BMC Remedy for automated runbook initiation
- AWS-based recovery target; workloads from on-premises VMware and physical servers replicated to AWS

**Differentiating features**
- Proactive health checks validate both guest OS readiness and AWS environment readiness before initiating failover
- DR plans treat the AWS target environment as first-class: pre-configures Security Groups, VPCs, and instance types as part of the plan definition

**UX patterns**
- SaaS web console; simplified onboarding with wizard for DR plan creation
- Dashboard shows per-VM protection status and last test results

**Integration points**
- AWS-native integration (CloudFormation, IAM, EC2, VPC)
- ServiceNow and BMC Remedy for ITSM workflow integration
- REST API for programmatic orchestration

**Known gaps**
- AWS-only recovery target limits multi-cloud flexibility
- Orchestration depth (dependency graphing, complex multi-tier apps) is less granular than Veeam VRO or Commvault
- Limited coverage for containerised workloads (Kubernetes)

**Licence / IP notes**
- Proprietary SaaS; raised $500M+, valued at $2B

---

### HYCU R-Cloud

**Core features**
- Cross-cloud DR across all major public clouds (AWS, Azure, GCP) — unique multi-cloud flexibility
- On-premises to cloud, cloud-to-cloud, and cross-region failover from a single platform
- 90+ integrations covering VMs, file shares, IaaS, DBaaS, PaaS, SaaS, and AI workloads
- Agentless protection for Hyper-V (2025), VMware, and public cloud VMs
- Azure SQL Managed Instance protection and unified Azure multi-cloud view
- One-click recovery with application-consistent backups
- SaaS data protection alongside infrastructure DR (unique combined coverage)

**Differentiating features**
- True cross-cloud DR (any cloud to any cloud) rather than cloud-vendor-specific destination
- Unified management of SaaS data (Microsoft 365, Salesforce) alongside infrastructure DR from one console
- No specialised DR knowledge required for onboarding — simplified self-service model

**UX patterns**
- Clean SaaS UI; designed for simplicity over advanced configuration
- Unified workload view combines on-premises and cloud assets in a single inventory

**Integration points**
- REST API
- Native integrations with public cloud providers
- SaaS connectors for Microsoft 365, Salesforce, and others

**Known gaps**
- Less orchestration depth than Veeam VRO or Commvault for complex multi-tier dependency management
- Smaller brand recognition limits enterprise trust in mission-critical DR scenarios
- Compliance documentation and audit reporting less mature than enterprise-grade incumbents

**Licence / IP notes**
- Proprietary SaaS; commercial pricing from $2/VM/month

---

### Cutover

**Core features**
- Collaborative runbook automation for IT DR, cloud migrations, and technology change events
- Node maps for process visualisation and dependency modelling
- Real-time dashboards with task completion tracking, milestone status, and recovery stream progress
- Immutable audit logs for regulatory compliance evidence
- REST API with pre-built integrations (Jira, ServiceNow, Jenkins, Ansible, Slack, Microsoft Teams)
- Conditional logic for dynamic task flows (branch on result, skip on condition)
- AI-powered runbook creation and improvement suggestions (2025): generates runbooks from scratch, summarises content, identifies bottlenecks
- Mobile UI for field technicians during active recovery events
- Post-incident review reports and analytics

**Differentiating features**
- Human-machine collaboration focus: designed for the people executing recovery, not just automation scripts
- AI agent in runbook platform reduces manual effort and surfaces improvement suggestions during and after recovery events
- Linked runbooks with primary-secondary relationships enable modular, reusable recovery building blocks

**UX patterns**
- Clean modern SaaS UI built for operator usability under pressure
- Runbook builder with visual task flow and dependency mapping
- Mobile-first design for teams executing recovery tasks in the field

**Integration points**
- REST API for full programmatic control
- Native connectors: Jira, ServiceNow, Jenkins, Ansible, Slack, Microsoft Teams
- Webhook support for event-driven trigger integration

**Known gaps**
- Does not perform replication or data protection — orchestration layer only; must integrate with a replication tool
- Less suited to fully automated, lights-out DR execution (designed around human-in-the-loop workflows)
- Less deep integration with infrastructure-layer DR tools (Veeam, Zerto) compared to the vendors' own orchestration products

**Licence / IP notes**
- Proprietary SaaS; commercial enterprise pricing

---

### LitmusChaos (Open Source)

**Core features**
- Chaos engineering experiments (fault injection) to validate system resilience and DR runbook effectiveness
- Kubernetes-native chaos experiments (pod failure, node drain, network partition, CPU stress)
- Experiment scheduling and automated resilience scoring
- Integration with CI/CD pipelines to gate deployments on resilience thresholds
- Community hub with 50+ pre-built chaos experiment templates
- Observability integration (Prometheus, Grafana) for chaos impact monitoring

**Differentiating features**
- Open source with Apache 2.0 licence — no vendor lock-in
- Native Kubernetes and cloud-native workload support
- FireDrill methodology: controlled outages to validate runbooks under realistic conditions

**UX patterns**
- Portal UI for experiment management and scheduling
- YAML-based experiment definition for GitOps workflows

**Integration points**
- Kubernetes-native (CRDs for experiment definitions)
- Prometheus / Grafana for metrics
- GitHub Actions, Jenkins, and other CI/CD platforms
- REST API for programmatic experiment management

**Known gaps**
- Limited to cloud-native / Kubernetes workloads; no support for VMs, physical servers, or legacy applications
- No built-in DR runbook execution or compliance reporting
- Chaos testing tool, not a DR orchestrator; must be combined with separate DR platform

**Licence / IP notes**
- Apache 2.0 open source; CNCF sandbox project

---

## Cross-Cutting Feature Themes

### Table-Stakes Features
- Automated failover and failback with dependency-ordered sequencing
- Non-disruptive DR testing in isolated environments
- RTO and RPO monitoring and alerting
- Automated runbook generation or templating
- Compliance documentation and audit trails (evidence for SOC 2, ISO 22301, ISO 27001)
- REST API for programmatic integration and orchestration
- ITSM integration (ServiceNow, Jira) for approval workflows and incident ticketing
- Scheduled and on-demand DR test execution with result capture

### Differentiating Features
- CDP / journal-based point-in-time recovery (Zerto)
- AI-powered anomaly detection for ransomware blast radius analysis (Rubrik)
- True cross-cloud (any-to-any) DR orchestration (HYCU)
- Human-machine collaboration runbooks with mobile UI for field execution (Cutover)
- Cleanroom Recovery with pre-verification of the target environment (Commvault)
- Chaos engineering integration for proactive resilience validation (LitmusChaos)
- AI-assisted runbook creation and bottleneck identification (Cutover)

### Underserved Areas / Opportunities
- Multi-cloud orchestration with intelligent routing: no tool offers seamless failover sequencing that dynamically selects the optimal target cloud based on cost, latency, and capacity at the time of disaster
- AI-driven dependency discovery: most tools require manual mapping of application dependencies; AI could infer the dependency graph from infrastructure telemetry, network flows, and application performance metrics
- Proactive RTO/RPO gap detection: continuous simulation of recovery scenarios to surface SLA gaps before an incident reveals them; only 32% of organisations have a readiness dashboard
- Automated compliance attestation: generating audit-ready DR evidence packages (ISO 22301, SOC 2, DORA, NIS2) automatically from test run data, without manual assembly
- Containerised and serverless workload DR: most tools focus on VMs; Kubernetes workloads, serverless functions, and managed databases lack deep orchestration support
- Plain-language incident communication: automated stakeholder updates and regulator notifications generated from recovery event logs

### AI-Augmentation Candidates
- Dependency graph inference from network flow and APM telemetry — replacing manual runbook dependency mapping
- Failure prediction from storage health, network anomaly, and system telemetry patterns — enabling pre-emptive failover
- Intelligent recovery sequencing that adapts to actual infrastructure state at the time of disaster, not static runbook steps
- Natural language runbook generation from infrastructure-as-code definitions and deployment manifests
- Post-incident report generation: automated root cause analysis, timeline reconstruction, and regulatory attestation from recovery event logs
- Continuous RTO/RPO simulation with ML-based gap scoring against declared SLAs

---

## Legal & IP Summary

All major solutions analysed are proprietary commercial products with custom enterprise licensing. No patents on specific DR orchestration techniques were identified in publicly available sources, though algorithmic claims around journal-based replication (Zerto) and anomaly detection (Rubrik) may be subject to existing patents not publicly documented in product materials. The only open-source tool surveyed (LitmusChaos) is Apache 2.0 licensed, permitting free use, modification, and distribution. An AI-native DR orchestrator built on open-source foundations and avoiding direct replication of proprietary UI or API designs would carry no identified copyright or licence compatibility risk. No specific concerns were found that would prevent building an open-source alternative in this space.

---

## Recommended Feature Scope

**Must-have (MVP)**
- Multi-tier DR runbook builder with dependency-ordered failover sequencing
- Non-disruptive DR test execution in isolated environments, with automated result capture
- RTO/RPO monitoring dashboard with live recovery readiness scoring
- AI-assisted dependency discovery from infrastructure telemetry (network flows, APM)
- REST API and webhook support for integration with replication engines and ITSM platforms
- Compliance evidence package generation (test logs, approvals, timings) for SOC 2 / ISO 22301 audits

**Should-have (v1.1)**
- Multi-cloud failover target selection (AWS, Azure, GCP) with intelligent routing based on cost and capacity
- Failure prediction engine analysing system health indicators to trigger pre-emptive alerts
- Natural language runbook generation from IaC definitions (Terraform, CloudFormation, ARM templates)
- ServiceNow / Jira integration for approval workflows and incident ticket creation
- Mobile UI for field operators executing recovery tasks

**Nice-to-have (backlog)**
- Chaos engineering experiment integration (LitmusChaos / Chaos Mesh) to gate runbook readiness on resilience scores
- Automated post-incident report generation with root cause analysis and regulatory attestation text
- Kubernetes-native DR orchestration for containerised workloads
- AI chatbot interface for operators querying recovery status and DR plan guidance in natural language
- Vendor-neutral replication adapter layer supporting Veeam, Zerto, AWS DRS, and Azure ASR as pluggable backends
