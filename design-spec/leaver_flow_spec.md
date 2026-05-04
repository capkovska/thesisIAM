# JML-L: Leaver Orchestrator — Flow Specification

**Version:** 1.0.0  
**Folder:** /ILM-Orchestrators/  
**Trigger types:** HRIS webhook (termination), ITSM webhook (offboarding request), Scheduled (future end date)

---

## Purpose
Deprovisiones a departing identity. Central account suspension is unconditionally the first action taken, ensuring the security-critical outcome is achieved regardless of any downstream failures.

---

## Inputs
| Field | Required | Notes |
|---|---|---|
| `eventType` | Yes | Must be `leaver` |
| `eventSource` | Yes | `hris` or `itsm` |
| `identity.employeeNumber` | Yes | |
| `identity.endDate` | Yes | ISO 8601 date — determines immediate vs scheduled execution |
| `flags.urgent` | No | If true, bypass scheduled activation; execute immediately |

---

## Steps

### L-1: Validate and check endDate
- **Helpers:** SH-4, UTL-2
- **Action:** Validate payload; compare `endDate` to today
- **If endDate is future (and urgent=false):** Create `Pending-activation` record; register scheduled trigger for `endDate`; exit current execution
- **If endDate is today or past:** Continue immediately

### L-2: Locate and verify identity
- **Helper:** UTL-4
- **Branch — Not found:** Write warning; alert ops; halt
- **Branch — Already Deprovisioned:** Write `duplicate-leaver` note; halt cleanly
- **Branch — Active or Suspended:** Continue

### L-3: Suspend central identity ⚠️ PRIORITY STEP
- **Helper:** UTL-4
- **Action:** Call Okta Users API to set status to `Suspended`
- **Effect (immediate):** All active SSO sessions terminated; MFA factors disabled; user cannot authenticate
- **Output:** Suspension timestamp written to `ilm_tx_log` (EV-L-02)
- **Note:** This step is intentionally placed before any downstream action. If all subsequent steps fail, the account is still suspended.

### L-4: Revoke sessions and tokens
- **Helper:** UTL-4
- **Action:** Call Okta Sessions API to terminate all active sessions; clear all refresh tokens
- **Output:** Revocation timestamp (EV-L-03)

### L-5: Deprovision integrated applications
- **Helper:** UTL-4
- **Action:** Remove user from ALL Okta groups (including BL-*); remove all direct application assignments
- **Effect:** SCIM-connected apps receive deprovisioning signal automatically
- **Output:** `groupsRemoved` list (EV-L-04)

### L-6: Generate ITSM deprovisioning tasks for manual applications
- **Helper:** SH-6, CFG-2
- **Action:** Query `ilm_system_class` for manual systems; generate one ITSM task per system with due date = now + `deprovisionSla`
- **Output:** Task IDs in `itsm_tasks`; `ilm_pendingTasks = true`

### L-7: Critical systems completeness check
- **Helpers:** CFG-2, SH-2
- **Action:** For each critical system (isCritical=true in `ilm_system_class`):
  - SCIM-connected: verify deprovisioning signal was sent (confirm in System Log)
  - Manual: SH-2 emits high-priority alert to app owner AND security ops contact
- **Purpose:** Ensures no critical system is silently missed

### L-8: Deactivate Okta account
- **Helper:** UTL-4
- **Action:** Transition user from `Suspended` to `Deprovisioned`
- **Note:** Deprovisioned accounts cannot be reactivated; re-hire triggers JML-J re-hire path

### L-9: Write evidence and close
- **Helper:** SH-3
- **Action:** Assemble EV-L-01 through EV-L-05; write to both log tables; set `ilm_status=deprovisioned`

---

## SLA enforcement
- Each ITSM task carries a due date from `ilm_system_class.deprovisionSla`
- SH-1b registered for each task's due date
- On expiry without closure: escalate to app owner's manager + security ops; keep transaction in `Pending-manual-tasks`
