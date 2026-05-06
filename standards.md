# Standards & API Reference

> Project: Disaster Recovery Orchestrator · Generated: 2026-05-03

## Industry Standards & Specifications

### ISO Standards

**ISO 22301:2019 — Business Continuity Management Systems**
- URL: https://www.iso.org/standard/75106.html
- The primary international standard for business continuity management. Requires organisations to plan, implement, test, and continuously improve processes to protect against, respond to, and recover from disruptive incidents. Defines the framework within which DR orchestration tools must operate. Amended in February 2024 (ISO 22301:2019/Amd 1:2024) to incorporate climate action requirements. Specifies that DR plans must be regularly tested and evidence retained — directly driving demand for automated compliance evidence generation in DR tools.

**ISO/IEC 27001:2022 — Information Security Management Systems**
- URL: https://www.iso.org/standard/27001
- The global standard for information security management. Requires documented and tested IT continuity and DR procedures as part of Annex A controls (specifically A.17: Information Security Aspects of Business Continuity Management). Auditors require evidence of DR test execution, results, and remediation — a key output an AI-native DR orchestrator should automate. Maps closely to SOC 2 Trust Services Criteria, enabling dual-certification evidence reuse.

**ISO 22313:2020 — Business Continuity Management Systems: Guidance**
- URL: https://www.iso.org/standard/75107.html
- Companion guidance standard to ISO 22301. Provides detailed implementation guidance for business continuity planning, testing, and exercising. Useful reference for structuring DR runbook design, test scenarios, and recovery exercise documentation.

**ISO/IEC 27031:2011 — Guidelines for ICT Readiness for Business Continuity**
- URL: https://www.iso.org/standard/44374.html
- Specifically addresses ICT readiness within business continuity programmes. Defines the concept of ICT Disaster Recovery and provides guidance on developing ICT continuity capabilities, including recovery time objectives and recovery point objectives. Directly relevant to the metrics and reporting an orchestrator should expose.

---

### W3C & IETF Standards

**RFC 7231 — Hypertext Transfer Protocol (HTTP/1.1): Semantics and Content**
- URL: https://datatracker.ietf.org/doc/html/rfc7231
- Foundational HTTP standard governing the REST API design of all DR orchestration platforms. Defines GET, POST, PUT, DELETE semantics used across Veeam VRO, Azure Site Recovery, AWS DRS, and Commvault APIs.

**RFC 6749 — The OAuth 2.0 Authorization Framework**
- URL: https://datatracker.ietf.org/doc/html/rfc6749
- OAuth 2.0 is the standard authentication framework used by Veeam Recovery Orchestrator REST API, Azure Site Recovery, and AWS DRS. An AI-native DR orchestrator integrating with these platforms must implement OAuth 2.0 client credentials or authorisation code flows.

**RFC 7519 — JSON Web Tokens (JWT)**
- URL: https://datatracker.ietf.org/doc/html/rfc7519
- JWT is the token format used with OAuth 2.0 authentication in most DR platform APIs. Required for implementing secure API authentication in integrations with Veeam VRO (which returns access and refresh tokens in JWT format) and cloud-native DR APIs.

**RFC 5246 / RFC 8446 — TLS (Transport Layer Security)**
- URL: https://datatracker.ietf.org/doc/html/rfc8446
- TLS 1.3 is mandatory for securing DR API communications. Veeam Recovery Orchestrator REST API explicitly requires HTTPS; AWS and Azure APIs enforce TLS. Critical for any DR tool handling sensitive infrastructure configurations.

**RFC 8288 — Web Linking**
- URL: https://datatracker.ietf.org/doc/html/rfc8288
- Defines hypermedia linking semantics. Veeam Recovery Orchestrator REST API is built on Hypertext Application Language (HAL), which extends RFC 8288. Relevant for implementing API clients that navigate the VRO API resource graph.

**W3C JSON-LD 1.1**
- URL: https://www.w3.org/TR/json-ld11/
- JSON-LD is relevant for representing infrastructure dependency graphs and DR plan metadata in a machine-readable, interoperable format. Potential standard for describing DR plan schemas in an open-source AI-native tool.

---

### Data Model & API Specifications

**OpenAPI Specification 3.1**
- URL: https://spec.openapis.org/oas/v3.1.0
- The de-facto standard for REST API description. Azure Site Recovery REST API is documented with OpenAPI-compatible specifications (azure-rest-api-specs repository). An AI-native DR orchestrator should publish an OpenAPI 3.1 schema to enable ecosystem integration, SDK generation, and API consumer tooling.

**AsyncAPI 2.x / 3.x**
- URL: https://www.asyncapi.com/docs/reference/specification/v3.0.0
- Standard for event-driven API specifications (webhooks, pub/sub). Relevant for defining the event schemas emitted by a DR orchestrator (failover initiated, test completed, RTO breach detected). Complements OpenAPI for the event-driven integration surface.

**CloudEvents 1.0**
- URL: https://cloudevents.io/
- CNCF standard for describing event data in a common format. Relevant for standardising the event payloads emitted by an AI-native DR orchestrator when integrated into cloud-native observability and automation pipelines (Kubernetes, Knative, AWS EventBridge).

**JSON Schema Draft 2020-12**
- URL: https://json-schema.org/specification
- Used to define and validate the structure of DR plan definitions, runbook schemas, and recovery state payloads. Should be used to define the canonical schema for DR plan representations in an open-source tool.

**Terraform Provider Schema**
- URL: https://developer.hashicorp.com/terraform/plugin/framework
- Many organisations define infrastructure-as-code using Terraform. A DR orchestrator that can parse Terraform state and plan files to infer application dependency graphs and target infrastructure would address a significant gap. The Terraform Plugin Framework defines the schema for custom providers.

---

### Security & Authentication Standards

**OAuth 2.0 (RFC 6749) and OpenID Connect 1.0**
- URL: https://openid.net/developers/how-connect-works/
- Used by all major DR platform APIs for authentication and authorisation. An AI-native orchestrator must support OAuth 2.0 for API-to-API authentication when integrating with Veeam VRO, Azure ASR, and cloud provider DR services.

**NIST SP 800-34 Rev. 1 — Contingency Planning Guide for Federal Information Systems**
- URL: https://csrc.nist.gov/pubs/sp/800/34/r1/upd1/final
- The US government's definitive contingency planning framework. Defines a seven-step process: policy development, BIA, preventive controls, recovery strategies, plan development, testing, and maintenance. An AI-native DR orchestrator aligned with NIST SP 800-34 would be positioned for US federal and regulated-industry buyers. Last updated: April 2021.

**NIST SP 800-53 Rev. 5 — Security and Privacy Controls for Information Systems**
- URL: https://csrc.nist.gov/publications/detail/sp/800-53/rev-5/final
- Defines the CP (Contingency Planning) control family, which mandates documented and tested DR plans, alternate processing sites, system backups, and recovery procedures. DR orchestration tools serve as the implementation evidence for CP controls in FedRAMP and federal compliance programmes.

**NIST Cybersecurity Framework (CSF) 2.0**
- URL: https://www.nist.gov/cyberframework
- The CSF Recover function directly maps to DR orchestration capabilities: recovery planning, improvements, and communications. Aligning an AI-native DR tool to CSF 2.0 Recover categories provides a widely recognised compliance reference.

**OWASP API Security Top 10 (2023)**
- URL: https://owasp.org/API-Security/editions/2023/en/0x00-header/
- Directly relevant to securing the REST API surface of a DR orchestration platform. DR platforms handle sensitive infrastructure credentials and failover triggers — a compromised DR API could be weaponised by attackers. OWASP API Top 10 risks (Broken Object Level Authorisation, Excessive Data Exposure, etc.) must be addressed in API design.

**EU Digital Operational Resilience Act (DORA) — Regulation (EU) 2022/2554**
- URL: https://www.digital-operational-resilience-act.com/
- Entered application January 17, 2025. Mandatory for financial entities operating in the EU. Requires: documented and regularly tested DR plans, ability to recover data within two hours of an incident, at least annual manual DR testing, and immutable third backup copy segregated from primary/secondary. A DR orchestrator targeting EU financial services must support DORA evidence generation and two-hour RTO target monitoring.

**EU NIS2 Directive — Directive (EU) 2022/2555**
- URL: https://digital-strategy.ec.europa.eu/en/policies/nis2-directive
- Transposed by EU Member States from October 18, 2024. Applies to essential and important entities (digital services, cloud providers, energy, healthcare, banking). Requires backup management, disaster recovery processes, and regular testing with documented results. ENISA technical guidance (June 2025) recommends the 3-2-1 backup rule and full backup solution review with integrity verification. An AI-native DR tool aligned to NIS2 requirements would address the growing EU compliance burden.

**SOC 2 Trust Services Criteria — Availability Category**
- URL: https://www.aicpa-cima.com/resources/landing/system-and-organization-controls-soc-suite-of-services
- US attestation framework. The Availability TSC requires: documented recovery procedures, regular DR testing with evidence, and performance monitoring against RTO/RPO objectives. Auditors specifically require test logs, results, and evidence of remediation. DR orchestrators that auto-generate SOC 2 evidence packages significantly reduce audit preparation burden.

**ITIL 4 — Service Continuity Management Practice**
- URL: https://www.axelos.com/certifications/itil-service-management/itil-4-foundation
- IT service management framework governing DR planning, testing, and continuous improvement practices within the ITIL lifecycle. Many enterprise IT organisations use ITIL terminology for DR programme governance; alignment with ITIL 4 service continuity management practice is expected by enterprise buyers.

---

### MCP Server Specifications

**Model Context Protocol (MCP) 1.x**
- URL: https://modelcontextprotocol.io/
- Anthropic's open protocol for connecting AI language models to external tools and data sources. An AI-native DR orchestrator built with MCP server support would enable LLM-powered agents to query DR plan status, trigger test runs, inspect recovery readiness, and generate compliance reports via natural language interfaces. MCP tooling is directly applicable to the AI-augmented DR use cases identified in research — dependency discovery, runbook generation, and post-incident reporting.

---

## Similar Products — Developer Documentation & APIs

### Veeam Recovery Orchestrator (REST API)
- **Description:** Automated DR runbook execution, non-disruptive testing, and compliance reporting for VMware and Hyper-V workloads. REST API exposes full plan management, test execution, and reporting capabilities.
- **API Documentation:** https://helpcenter.veeam.com/docs/vro/rest/overview.html
- **Version 13 Reference:** https://helpcenter.veeam.com/references/vro/13/rest/tag/SectionOverview/index.html
- **Developer Hub:** https://helpcenter.veeam.com/category/development.html
- **Standards:** REST/JSON, HAL (Hypertext Application Language), HTTPS only
- **Authentication:** OAuth 2.0 (access token + refresh token)

### AWS Elastic Disaster Recovery (DRS API)
- **Description:** AWS-native continuous block-level replication and automated failover service. Full API for managing replication, failover, failback, and point-in-time recovery across hundreds of servers.
- **API Documentation:** https://docs.aws.amazon.com/drs/latest/APIReference/Welcome.html
- **Python SDK (boto3):** https://boto3.amazonaws.com/v1/documentation/api/latest/reference/services/drs.html
- **Developer Guide:** https://docs.aws.amazon.com/drs/
- **Standards:** REST/JSON, AWS Signature Version 4
- **Authentication:** AWS IAM (access key / role-based)

### Azure Site Recovery (REST API)
- **Description:** Microsoft-native DR with automated failover and failback for Azure and on-premises workloads. Full ARM-based REST API for replication policy management, failover plan creation, and test execution.
- **API Documentation:** https://learn.microsoft.com/en-us/rest/api/site-recovery/
- **SDK Links:** Azure SDK for Go: `github.com/Azure/azure-sdk-for-go/sdk/resourcemanager/recoveryservices/armrecoveryservicessiterecovery/v3`; .NET, Python, Java, JS SDKs also available
- **Developer Guide:** https://learn.microsoft.com/en-us/azure/site-recovery/
- **REST API Specs (OpenAPI):** https://github.com/Azure/azure-rest-api-specs
- **Standards:** REST/JSON, Azure Resource Manager API conventions, OpenAPI-compatible spec
- **Authentication:** Azure Active Directory (OAuth 2.0 / MSAL)

### Commvault REST API
- **Description:** Comprehensive data management and DR REST API covering failover plan management, test boot, failover execution, and backup operations. Postman workspace available for API exploration.
- **API Documentation:** https://api.commvault.com/
- **Developer Portal:** https://developer.commvault.com/api
- **Postman Workspace:** https://www.postman.com/crimson-shadow-3553/commvault-s-public-workspace/documentation/7sqt79a/commvault-rest-api-public
- **Standards:** REST/JSON
- **Authentication:** Token-based (master permissions required for DR API endpoints)

### Zerto REST API
- **Description:** REST API for managing protection groups, failover tests, live failover, failback, and analytics. Significant expansion in Zerto 10 (2024); Linux-based ZVM appliance supports all core APIs.
- **API Documentation:** https://www.jpaul.me/2024/08/harnessing-zerto-10-api-enhancements-for-superior-disaster-recovery/ (community reference)
- **OpsRamp Integration Docs:** https://docs.opsramp.com/integrations/data-protection/zerto/
- **Standards:** REST/JSON
- **Authentication:** Token-based (bearer token obtained via authentication endpoint)

### Druva REST API
- **Description:** Cloud DR platform API for managing DR plans, triggering failovers, and integrating with AWS environments. Full API access to failover sequencing, network configuration, and test execution.
- **API Documentation:** https://help.druva.com/en/articles/8651935-automated-disaster-recovery-workflow
- **AWS Architecture Blog Reference:** https://aws.amazon.com/blogs/architecture/field-notes-automate-disaster-recovery-for-aws-workloads-with-druva/
- **Standards:** REST/JSON, AWS-native integration
- **Authentication:** OAuth 2.0

### LitmusChaos REST API and Kubernetes CRDs
- **Description:** Open-source chaos engineering platform with Kubernetes CRD-based experiment definitions and a REST API for scheduling and managing chaos experiments. Community hub with 50+ pre-built chaos experiment templates.
- **API Documentation:** https://litmuschaos.io/
- **GitHub:** https://github.com/litmuschaos/litmus
- **Standards:** Kubernetes CRDs (YAML), REST/JSON, Prometheus metrics (OpenMetrics)
- **Authentication:** Kubernetes RBAC; REST API token-based
- **Licence:** Apache 2.0

### Cutover REST API
- **Description:** Collaborative runbook automation platform REST API supporting runbook management, task tracking, approvals, and integration with ITSM and DevOps tools. Webhook support for event-driven trigger integration.
- **API Documentation:** https://cutover.com/
- **Standards:** REST/JSON, webhook event delivery
- **Authentication:** API key / OAuth 2.0
- **Integrations documented:** Jira, ServiceNow, Jenkins, Ansible, Slack, Microsoft Teams

---

## Notes

**Emerging standard — CloudEvents for DR event streaming:** The CloudEvents 1.0 specification is gaining adoption as a standard event envelope format in cloud-native infrastructure tooling (Kubernetes, AWS EventBridge, Azure Event Grid). A DR orchestrator that emits CloudEvents-formatted events for each state transition (replication health change, test initiated, failover triggered, RTO breach) would integrate natively with cloud-native observability pipelines without custom adapters.

**Gap — No open DR plan interchange format:** There is currently no industry standard schema for representing DR plans, runbooks, or recovery dependencies in a portable, tool-neutral format. Each vendor uses proprietary data models. An open-source AI-native DR orchestrator could define and publish a JSON Schema for DR plan definitions, positioning itself as the interoperability layer between replication engines (Veeam, Zerto, AWS DRS, Azure ASR) and operational tooling (ServiceNow, Jira, SIEM platforms).

**Gap — Regulatory evidence package standard:** DORA, NIS2, ISO 22301, and SOC 2 all require DR test evidence, but there is no standard format for packaging and exchanging that evidence with auditors. An AI-native tool that produces structured, machine-readable attestation packages in a consistent format (JSON-LD or OpenAPI-defined schema) would address a significant compliance automation gap.

**Terraform and IaC integration:** No current DR orchestration tool natively parses Terraform state or plan files to auto-generate DR dependency graphs. Given widespread IaC adoption, this represents both a technical opportunity and a potential integration standard to define — a Terraform provider or state parser that maps infrastructure resources to DR protection groups.
