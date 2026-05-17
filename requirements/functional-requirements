# Functional Requirements

## FR1 — Joiner
- **FR1.1** Joiner initiation must be supported from both HRIS-driven events (default path) and ITSM-driven requests (exception path), producing consistent outcomes and evidence in both cases.
- **FR1.2** On a valid joiner event, the solution must create a new identity or activate an existing one in the central identity platform, ensuring a unique identifier and all required core attributes are present.
- **FR1.3** Required attributes must be mapped and applied from the minimum set defined in configuration (legal name, display name, email, username, employee/contractor type, department, team, manager, location, time zone, start date, lifecycle status). Missing or invalid attributes must route the request to a `clarification-required` state.
- **FR1.4** Baseline security and productivity access must be assigned according to company policy (e.g. MFA policy group, baseline collaboration access).
- **FR1.5** Application access must be delivered via one of two paths: **integrated/central** (group or app assignment logic) where available, or **manual** (structured sub-tasks with confirmation and evidence capture) where integrations are absent.
- **FR1.6** The lifecycle record must be updated with: status (`completed`, `pending-manual-tasks`, or `failed`), timestamps, a summary of changes made, and required evidence references.

## FR2 — Mover
- **FR2.1** Mover initiation must be supported from HRIS attribute changes (department, team, manager, location) and/or ITSM-driven access change requests.
- **FR2.2** Identity attributes must be updated in the central platform based on approved changes and defined mapping rules, including validation of allowed values and required fields.
- **FR2.3** Mover processing must support explicit remove-before-add sequencing to prevent residual access from persisting across role transitions.
- **FR2.4** Both integrated/central updates and manual sub-tasks must be supported for systems not centrally integrated, consistent with joiner handling.
- **FR2.5** Temporary access requests must be handled via time-limited assignment where technically possible, or via a controlled manual expiration task with reminders and evidence requirements.
- **FR2.6** The solution must record all access changes (added and removed), approvals used, timestamps, final status, and evidence references, sufficient to reconstruct privilege changes without cross-system manual lookup.

## FR3 — Leaver
- **FR3.1** Leaver initiation must be supported from HRIS and ITSM, covering both immediate termination and scheduled deprovisioning (future end date). Scheduled events must trigger automatically at the correct time.
- **FR3.2** On leaver execution, the identity must be disabled in the central platform (directory/IdP) as the first action. The disablement timestamp must be recorded as audit evidence.
- **FR3.3** Active sessions and authentication tokens must be revoked where the platform and integrations support it, to reduce residual access risk.
- **FR3.4** Application access must be removed via integrated/central deprovisioning where available. Manual removal tasks must be created for non-integrated systems, with confirmation and evidence capture required.
- **FR3.5** License and entitlement removal must be automated where possible. Where automation is unavailable, required manual steps must be explicitly surfaced and tracked.
- **FR3.6** A validation step must be executed against a configured set of critical systems. If residual access is found or cannot be confirmed, remediation tasks must be created and the leaver transaction must remain in a non-final state until resolved or formally accepted as an exception.

## FR4 — Access requests and approvals
- **FR4.1** Approval paths must be configurable per target system and access type: manager-only for low-risk, manager and application owner for delegated systems, manager and security for privileged or high-risk access.
- **FR4.2** No access change may be executed until all required approvals are received and recorded. Denied requests must be closed with a documented reason and denial evidence.
- **FR4.3** The request and approval steps must be separated from execution wherever possible. Approvals must be logged before execution begins; execution must be performed by controlled automation or authorized operators.
- **FR4.4** Approval evidence must include at minimum: approver name and role, approval timestamp, scope of what was approved (apps, groups, roles), and any exception notes.

## FR5 — Exceptions
- **FR5.1** Non-employee identities (contractors) must be supported with defined end dates, lifecycle status tracking, limited and configurable baseline access, and mandatory review and expiration controls.
- **FR5.2** A VIP/special-handling classification must be available, enabling stronger approval requirements, restrictions on automated actions, and enhanced evidence capture.
- **FR5.3** A break-glass emergency access mechanism must be available, with explicit labelling, security oversight, mandatory time-limited access with automatic expiration, and enhanced logging.
- **FR5.4** Exception cases must be routed through explicit workflow states (`exception-approval-required`, `manual-verification-required`, `risk-accepted`) to ensure visibility and auditability.

## FR6 — Audit reporting and evidence export
- **FR6.1** An evidence package must be produced for each JML transaction, containing: initiation source and timestamp, approvals and timestamps, identity and access changes made, completion status and timestamps, and any failure, exception, or remediation records.
- **FR6.2** Evidence must be exportable in audit-ready formats (CSV, JSON, PDF), including periodic summaries of volume, completion rates, exception rates, time-to-access, time-to-revoke, and outstanding manual tasks.
- **FR6.3** Reports must support full traceability from a lifecycle event to its evidence sources (ticket IDs, logs, exports). Evidence retention periods must be configurable and documented in operational guidance.
