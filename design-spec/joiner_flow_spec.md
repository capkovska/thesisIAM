# JML-J: Joiner Orchestrator — Flow Specification

**Version:** 1.0.0  
**Folder:** /ILM-Orchestrators/  
**Trigger types:** HRIS webhook (primary), ITSM webhook (exception), Scheduled (pre-provisioning)

---

## Purpose
Provisions a new identity from event receipt through baseline access assignment, optional elevated access approval, ITSM task generation for manual systems, and evidence recording.

---

## Inputs (from trigger payload)
See Appendix B / `hris_webhook_payload_schema.json` for the full schema.

| Field | Required | Notes |
|---|---|---|
| `eventType` | Yes | Must be `joiner` |
| `eventSource` | Yes | `hris` or `itsm` |
| `identity.employeeNumber` | Yes | Unique HRIS identifier |
| `identity.firstName` | Yes | |
| `identity.lastName` | Yes | |
| `identity.email` | Yes | Primary work email |
| `identity.department` | Yes | Must match `ilm_access_map` key |
| `identity.employeeType` | Yes | `employee`, `contractor`, `service-account` |
| `identity.managerId` | Yes | `employeeNumber` of manager |
| `identity.startDate` | Yes | ISO 8601 date |
| `identity.location` | No | Used for location-specific baseline groups |
| `flags.vip` | No | Triggers VIP handling |
| `flags.contractor` | No | Applies contractor baseline policy |
| `flags.expiryDate` | No | For contractor end-of-engagement scheduling |

---

## Steps

### J-1: Validate and normalise
- **Helper called:** SH-4
- **Action:** Validate all required fields; normalise `department` to canonical form matching `ilm_access_map`; derive `displayName`
- **On failure:** Write `Clarification-required` to `ilm_tx_log`; call SH-2 to notify HR ops; halt — no Okta user created

### J-2: Duplicate check
- **Helper called:** UTL-4
- **Action:** Query Okta UD for user with matching `employeeNumber`
- **Branch — Not found:** Continue to J-3
- **Branch — Suspended (re-hire):** Update profile from new payload; clear existing groups; reactivate account; log `eventSource: rehire`; skip to J-4
- **Branch — Active (duplicate):** Write `Duplicate-detected` error; alert ops team; halt

### J-3: Create Okta user
- **Helper called:** UTL-4
- **Action:** Create user via Okta Users API
- **Status:** `Staged` if `startDate` is future; `Active` if `startDate` is today or past
- **If Staged:** Register scheduled flow via UTL-2 to activate on `startDate ± 5 min`
- **Output:** Okta `userId` written to `ilm_tx_log` row

### J-4: Assign baseline groups
- **Helpers called:** CFG-1, UTL-4, SH-7
- **Action:** Call CFG-1 with `employeeType` and `location`; assign returned `BL-*` groups via UTL-4 wrapped in SH-7 retry
- **Output:** Groups appended to `groupsAdded` in `ilm_tx_log`

### J-5: Resolve standard package
- **Helper called:** CFG-1
- **Action:** Call CFG-1 with `department` and `employeeType`; if matching row found, add `PKG-A-*` or `PKG-B-*` groups to assignment queue
- **Output:** `ilm_accessPackage` set to `A`, `B`, or `individual`

### J-6: Approval gate (conditional)
- **Helpers called:** SH-1, SH-1a (async callback), SH-1b (escalation timer)
- **Condition:** Only invoked if any access item in queue maps to Medium/High/Critical sensitivity in `ilm_system_class`
- **If Low only:** Skip; proceed directly to J-7
- **Action:** Call SH-1 to emit ITSM approval task(s); set transaction state to `Pending-approval`; pause at Wait-for-Response card
- **Resume:** SH-1a receives callback; resumes flow with approved/rejected set per item

### J-7: Assign approved access
- **Helpers called:** UTL-4, SH-6, SH-7
- **Action:** For each approved group → assign via UTL-4; for each manual system → call SH-6 to generate ITSM provisioning task
- **Output:** `groupsAdded` updated; ITSM task IDs stored in `itsm_tasks`; `ilm_pendingTasks` set to `true` if tasks exist

### J-8: Write evidence and close
- **Helper called:** SH-3, UTL-4
- **Action:** SH-3 assembles EV-J-01 through EV-J-05; writes to `ilm_tx_log` and `ilm_evidence_archive`; UTL-4 updates profile attributes: `ilm_status=active`, `ilm_lastEventType=joiner`, `ilm_lastEventId=txId`, `ilm_lastEventTs=now`
- **Terminal state:** `Completed` (no ITSM tasks) or `Pending-manual-tasks` (tasks open)

---

## Outputs
- Okta user account created/reactivated, `Active` or `Staged`
- Baseline and package group memberships assigned
- ITSM provisioning tasks for manual systems
- Evidence records EV-J-01 through EV-J-05 in `ilm_tx_log` and `ilm_evidence_archive`
- Okta System Log entries cross-referenced by `txId`

---

## Error states
| State | Trigger | Action |
|---|---|---|
| `Clarification-required` | Missing required field | Notify HR ops; no user created |
| `Duplicate-detected` | Active user with same `employeeNumber` | Alert ops; halt |
| `Pending-approval` | Approval gate active | Wait for SH-1a callback |
| `Pending-manual-tasks` | ITSM tasks open | Monitor via SH-1b; close when tasks confirmed |
| `Pending-intervention` | SH-7 retry exhausted | Add to `ilm_intervention_queue`; alert ops |
