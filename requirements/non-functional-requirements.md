# Non-Functional Requirements

## NFR1 - Security and least privilege
- **NFR1.1** The solution must operate with only the permissions required for each automated action. API tokens and service accounts must be scoped to the minimum necessary per function (read-only where possible, limited write where required).
- **NFR1.2** All credentials, tokens, and secrets must be stored and handled securely. Hardcoded values must not appear in workflows, documentation, screenshots, or exported packages. Secure configuration patterns (environment variables, vault references) must be used and documented instead.
- **NFR1.3** Workflow paths must not allow approvals to be bypassed. A traceable record of who approved what and when must exist for every access change, supporting review and reducing manipulation risk.
- **NFR1.4** High-risk access (admin roles, privileged groups, break-glass accounts) must require stronger approval controls and enhanced logging. Privileged assignments must be time-limited and automatically revoked on expiry.

## NFR2 - Auditability and traceability
- **NFR2.1** Every JML lifecycle transaction must be traceable end-to-end: from initiation (HRIS/ITSM) through approvals to execution results, including which systems were affected and what access changed.
- **NFR2.2** A consistent minimum evidence set must be produced for every transaction: initiation source and timestamp, approval record(s) and timestamp(s), list of access changes made (groups/apps/roles), execution result (success/partial/failure), and exception handling record if applicable.
- **NFR2.3** Evidence exports and reports must be readable by both technical and governance stakeholders (IAM, security, audit), reducing the need to manually correlate data across admin consoles.

## NFR3 - Reliability and resilience
- **NFR3.1** Retrying a failed lifecycle event must not produce duplicate accounts, inconsistent group memberships, or unintended access grants. Where full idempotency cannot be guaranteed due to external system constraints, duplicates must be detected, remediated, and recorded.
- **NFR3.2** Common failure modes must be handled explicitly: connector and API outages, partial provisioning, and timeout errors. Handling must include retry policies where appropriate, escalation paths (e.g. manual remediation tasks), and `pending-intervention` states that prevent silent failure.
- **NFR3.3** When a lifecycle change spans multiple systems, state outcomes must remain consistent. Partial failures must result in either completion after recovery or compensating actions with a documented exception state and clear remediation steps.
- **NFR3.4** Dependency availability (HRIS, ITSM, directory/IdP, application connectors) must be accounted for. The solution must behave predictably when dependent services are unavailable.

## NFR4 - Performance and scalability
- **NFR4.1** Measurable timeliness targets must be supported for lifecycle processing: time-to-access for joiners (baseline access readiness) and time-to-revoke for leavers (central disablement and downstream removal). Where full downstream timing cannot be measured in a prototype environment, the KPI method must be documented and demonstrated on representative test scenarios.
- **NFR4.2** The solution must handle the expected event volume (approximately 50 JML events per month, with seasonal peaks) without manual operational rework becoming the primary bottleneck.
- **NFR4.3** Workflow execution must minimise unnecessary API calls and repeated updates. Where rate limiting applies, backoff and batching strategies must be used.

## NFR5 - Maintainability and extensibility
- **NFR5.1** The solution must be decomposed into reusable modules and sub-flows (e.g. attribute normalisation, group assignment, ticket updates, reporting) to reduce duplication and simplify ongoing maintenance.
- **NFR5.2** Key configuration items - critical systems list, approval routing by system category, access package mappings, exception classifications - must be updatable without modifying workflow logic.
- **NFR5.3** Controlled updates must be supported through documented versioning practices: naming conventions, release notes, and a promotion process. Changes to mapping logic and approval rules must be trackable.
- **NFR5.4** Workflow logic must be understandable to IAM and IT operations staff without software development expertise. The artefact package must include sufficient documentation to deploy, operate, and troubleshoot the solution.

## NFR6 - Observability and operational readiness
- **NFR6.1** Lifecycle actions must be logged at a level sufficient for operational monitoring and auditing, including: transaction IDs for correlation, key decision points (approval route, integrated vs. manual path), per-system outcome status, and actionable error details.
- **NFR6.2** Failures, partial completions, and pending manual tasks must be visible and trigger escalation actions (ticket updates, notifications) so that incomplete lifecycle changes do not go undetected.
- **NFR6.3** An operational runbook must be included covering: monitoring lifecycle execution, recovering or retrying failed transactions, handling exception scenarios (VIP, break-glass, contractor expiry), and retrieving audit evidence on request.

## NFR7 - Privacy and compliance
- **NFR7.1** The solution must process only personal data necessary for lifecycle execution. Evidence exports and reports must avoid unnecessary personal information and apply anonymisation or redaction where possible, particularly in academic contexts.
- **NFR7.2** A clear boundary must exist between academic and production environments. Prototype environments must contain no real personal data or production secrets. All test data must be synthetic or anonymised, with confidentiality measures documented.
- **NFR7.3** Evidence retention expectations must be documented, including what is retained, where it is stored (ticketing system, export logs), and for how long, in line with organisational policy.
