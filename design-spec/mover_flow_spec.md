# JML-M: Mover Orchestrator - Flow Specification

**Version:** 1.0.0  
**Folder:** /ILM-Orchestrators/  
**Trigger types:** HRIS webhook (attribute change), ITSM webhook (change request), Scheduled (time-bound reversal)

---

## Purpose
Reconciles a user's current access state with the target access state derived from their new organisational attributes. Enforces remove-before-add sequencing to prevent privilege accumulation.

---

## Inputs
| Field | Required | Notes |
|---|---|---|
| `eventType` | Yes | Must be `mover` |
| `eventSource` | Yes | `hris` or `itsm` |
| `identity.employeeNumber` | Yes | |
| `identity.department` | Yes | New department |
| `identity.employeeType` | Yes | May have changed |
| `identity.managerId` | No | New manager if changed |
| `previousValues.department` | No | Enables delta calculation; if absent, Okta profile is queried |
| `accessChanges.add` | No | Explicit additions (ITSM path) |
| `accessChanges.remove` | No | Explicit removals (ITSM path) |
| `flags.expiryDate` | No | For time-bound access grants |

---

## Steps

### M-1: Validate and normalise
- **Helper:** SH-4
- **Additional:** Extract `previousValues` for `changeSummary` object; if absent, fall back to Okta profile query in M-2

### M-2: Load current identity state
- **Helper:** UTL-4
- **Action:** Retrieve current Okta profile and group memberships
- **Guard:** If user not found or `ilm_status ≠ active` → write error; alert ops; halt

### M-3: Calculate access delta
- **Helper:** SH-5
- **Action:** Compare `currentGroups` to `targetGroups` from CFG-1 lookup; returns `toAdd` and `toRemove` (BL-* excluded from both)
- **Conflict resolution:** Groups in both lists → retain (no-op); log as `retained`
- **If empty delta:** Update profile attributes only; close as `Completed` with `no-access-change` note; skip M-4 through M-7

### M-4: Approval gate (conditional)
- **Helper:** SH-1
- **Condition:** Any item in `toAdd` maps to Medium/High/Critical in `ilm_system_class`
- **Note:** `toRemove` items never require approval

### M-5: Remove old access (remove-before-add)
- **Helpers:** UTL-4, SH-6, SH-7
- **Action:** Remove all groups in `toRemove`; generate ITSM deprovisioning tasks for manual systems
- **Output:** `groupsRemoved` updated in `ilm_tx_log`

### M-6: Add new access
- **Helpers:** UTL-4, SH-6
- **Action:** Assign all approved groups from `toAdd`; generate ITSM provisioning tasks for manual systems
- **Output:** `groupsAdded` updated

### M-7: Update Okta profile attributes
- **Helper:** UTL-4
- **Action:** Write new `department`, `title`, `managerId`, `costCenter`, `location`, `ilm_accessPackage`
- **Note:** Profile update is final step - confirms access state and identity record are consistent

### M-8: Write evidence and close
- **Helper:** SH-3
- **Action:** Assemble EV-M-01 through EV-M-04; write to both log tables; update `ilm_status`, `ilm_lastEventType=mover`

---

## Time-bound access grants
When `flags.expiryDate` is present:
1. Approval proceeds normally
2. On grant, a scheduled reversal flow is registered via UTL-2 for `expiryDate`
3. Entry created in `ilm_exceptions` with `txId` and `expiryDate`
4. Reversal applies same delta logic in reverse; references original `txId`
