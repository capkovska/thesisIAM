# Orchestrator Execution Template

**Version:** 1.0.0
**Applies to:** JML-J (Joiner), JML-M (Mover), JML-L (Leaver)

---

## Overview

All three JML orchestrators follow the same six-stage execution template. This shared structure ensures consistent processing, auditability, and error handling across all lifecycle event types. It also makes the flows easier to understand and extend — a practitioner reading any one orchestrator will find the same logical sections in the same order.

The six stages run sequentially. Each stage has a defined input, a set of helper or utility flows it calls, and a defined output or exit condition. Failures at any stage are handled by SH-7 and result in a documented state in ilm_tx_log rather than an undefined error.

---

## Stage 1 — Receive and validate

**Helper called:** SH-4

The entry point receives the event payload from one of three trigger sources: HRIS webhook, ITSM webhook, or scheduled flow. SH-4 is called immediately with the raw inbound payload.

SH-4 performs the following operations:
- Validates that all required fields are present and correctly typed
- Normalises the `department` value to the canonical form used as a key in `ilm_access_map`
- Derives `displayName` from `firstName` and `lastName` if absent
- Checks that `startDate` (joiner) or `endDate` (leaver) is a valid ISO 8601 date

If validation fails, SH-4 returns an error object. The orchestrator uses this to write a `Clarification-required` record to `ilm_tx_log` via UTL-3, calls SH-2 to notify the HR operations contact, and halts. No Okta user object or group change is made at this point.

If validation passes, SH-4 returns a normalised identity object that is stored in a flow variable and used by all subsequent stages.

---

## Stage 2 — Load configuration

**Helpers called:** CFG-1, CFG-2

The orchestrator calls CFG-1 and CFG-2 to load the configuration data relevant to this transaction. These reads happen once per transaction and the results are stored in flow variables for the remainder of execution — they are not re-queried at each step.

- **CFG-1** (Access mapping table reader): called with `department` and `employeeType` to retrieve the applicable `baselineGroups`, `packageGroups`, and `roleGroups` lists from `ilm_access_map`.
- **CFG-2** (System classification table reader): called to retrieve the full `ilm_system_class` table. The orchestrator uses this data to determine which systems require approval, which are manual, and which are critical for leaver completeness checks.

For mover events, CFG-1 is called twice: once for the current (old) department/type combination and once for the new combination. Both results are passed to SH-5 in Stage 4 for delta calculation.

---

## Stage 3 — Resolve approvals

**Helpers called:** SH-1, SH-1a (async callback), SH-1b (escalation timer)

This stage is conditional. It is only executed if any access item in the pending assignment queue maps to a Medium, High, or Critical sensitivity tier in `ilm_system_class`.

If all items are Low sensitivity, this stage is skipped entirely and execution proceeds directly to Stage 4.

When approval is required:
1. SH-1 is called with the approval request object: `txId`, `employeeNumber`, `displayName`, list of access items requiring approval, and the computed `approvalTier`
2. SH-1 creates one ITSM approval task per required approver and writes the task references to `ilm_tx_log.approvalRef`
3. The transaction state is set to `Pending-approval`
4. The orchestrator pauses at Okta Workflows' built-in **Wait for Response** card, freeing the execution thread

The flow resumes when SH-1a receives the approval callback from the ITSM platform. SH-1a validates the approver identity, records the decision per access item, and calls the Okta Workflows Resume API to restart the paused orchestrator.

For multi-approver tiers (`manager-and-owner`, `manager-and-security`), SH-1a collects approvals incrementally and only resumes the orchestrator once all required approvers have responded.

SH-1b is registered by SH-1 at approval request time. It fires at the configured SLA deadline. If not all approvals have arrived, SH-1b either escalates to the next approver level or cancels the pending items and resumes the orchestrator with a rejected status, depending on the escalation policy configured in `ilm_system_class`.

**Note on removals:** Access removals (mover and leaver) never require approval and always bypass this stage. Only additions are gated.

---

## Stage 4 — Execute identity actions

**Helpers called:** UTL-4, SH-5 (mover only), SH-7

The orchestrator applies the group membership changes determined in Stage 2 (and for mover events, validated against the delta produced by SH-5).

- **UTL-4** wraps the Okta Groups API and Okta Users API with error handling and idempotency checks. Before making any group assignment or removal call, UTL-4 checks whether the action has already been applied. This prevents duplicate group assignments if the flow is retried.
- **SH-5** (mover only) is called with `currentGroups` (retrieved from Okta) and `targetGroups` (from CFG-1 lookup for the new department/type). It returns `toAdd` and `toRemove` lists, excluding baseline groups from both. Groups that appear in both lists are treated as a no-op and logged as `retained`.
- **SH-7** wraps all external API calls. If a call fails, SH-7 retries up to three times with a 10-second base delay. If all retries fail, SH-7 writes the failure detail to `ilm_tx_log` with status `Pending-intervention`, calls SH-2 to alert operations, and returns an error object to the orchestrator.

For the leaver flow, this stage always executes the central account suspension (step L-3) before any group removal. The suspension call is intentionally placed first and is not wrapped in a conditional — if all subsequent steps fail, the account is still suspended.

For the mover flow, removals in `toRemove` are always executed before additions in `toAdd` (remove-before-add sequencing), regardless of the approval outcome for individual items.

---

## Stage 5 — Emit downstream actions

**Helpers called:** SH-6, CFG-2

For every system in `ilm_system_class` with `integrationMethod: manual`, the orchestrator calls SH-6 to generate a structured ITSM task.

Each ITSM task contains:
- User `displayName` and `employeeNumber`
- Transaction ID (`txId`) for correlation
- Required action (provision / deprovision / update)
- Application owner ITSM user ID (`appOwnerItsm` from `ilm_system_class`)
- Due date calculated from `provisionSla` or `deprovisionSla` via UTL-2

Task IDs returned by the ITSM API are stored in the `itsm_tasks` field of the `ilm_tx_log` row. The Okta user's `ilm_pendingTasks` attribute is set to `true`.

SCIM-connected and AD-connected systems require no action at this stage — Okta's provisioning engine propagates group membership changes to these systems automatically when the group assignments made in Stage 4 take effect.

For the leaver flow, this stage also includes the critical systems completeness check (step L-7): for each system marked `isCritical: true` in `ilm_system_class`, SH-2 emits a high-priority alert to both the application owner and the security operations contact, in addition to the standard ITSM task.

---

## Stage 6 — Close and record evidence

**Helpers called:** SH-3, UTL-4

SH-3 assembles the transaction evidence record for the completed lifecycle event. The evidence fields are defined in Table 3.2 of the thesis (EV-J-01 through EV-J-05 for joiner, EV-M-01 through EV-M-04 for mover, EV-L-01 through EV-L-05 for leaver).

SH-3 writes the evidence to two stores simultaneously:
- **`ilm_tx_log`** — the mutable operational record, updated throughout execution
- **`ilm_evidence_archive`** — append-only; one row written at terminal state, never updated

UTL-4 then updates the Okta user profile lifecycle attributes:
- `ilm_status` → `active` (joiner/mover) or `deprovisioned` (leaver)
- `ilm_lastEventType` → `joiner`, `mover`, or `leaver`
- `ilm_lastEventId` → the transaction UUID
- `ilm_lastEventTs` → current timestamp

The transaction is set to one of two terminal states:
- `Completed` — all group changes applied and no ITSM tasks remain open
- `Pending-manual-tasks` — one or more ITSM tasks are open; the transaction remains in this state until all tasks are closed via the ITSM callback mechanism (SH-1a)

---

## Flow naming and annotation conventions

All flow names follow the format `{folder-prefix}-{sequence} {descriptive-name}`, for example `SH-3 Evidence writer`. This ensures flows are listed in a predictable order within each folder and their purpose is clear without opening the flow definition.

All input and output field names use camelCase, consistent with the Okta API JSON schema. Boolean flags are prefixed with `is` or `has` — for example `isApprovalRequired`, `hasPendingTasks`.

Each card within an orchestrator flow uses the built-in Okta Workflows card annotation field to record the business rule being implemented at that step. This is the primary in-console documentation mechanism and is the first place to look when troubleshooting unexpected behaviour.

---

## References

- Flow specifications: `Design-Spec/joiner_flow_spec.md`, `mover_flow_spec.md`, `leaver_flow_spec.md`
- Data model: `Design-Spec/data_model_and_attribute_schema.md`
- Build guide: `Build-Guide/build_guide.md`
- Thesis: Chapter 4.3 — Orchestrator flows
