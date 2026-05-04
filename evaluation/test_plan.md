# Test Plan — ILM Artefact v1.0.0

**Version:** 1.0.0  
**Test environment:** Okta developer sandbox + ServiceNow developer instance  
**Test data:** synthetic_identities.json  
**All test cases use synthetic/anonymised data only**

---

## Test categories

| Category | Test cases | Covers |
|---|---|---|
| Joiner functional | TC-J-01 to TC-J-06 | FR1.1–FR1.6 |
| Mover functional | TC-M-01 to TC-M-05 | FR2.1–FR2.6 |
| Leaver functional | TC-L-01 to TC-L-05 | FR3.1–FR3.6 |
| Approval flow | TC-A-01 to TC-A-04 | FR4.1–FR4.4 |
| Exception scenarios | TC-J-05, TC-J-06, TC-A-04, TC-L-05 | FR5.1–FR5.4 |
| Evidence and reporting | Covered by all above | FR6.1–FR6.3 |
| Error and reliability | TC-E-01 to TC-E-04 | NFR3.1–NFR3.3 |

---

## TC-J-01: Standard Joiner, HRIS trigger, Package B

**Identity:** EMP-0001 (Marta Kalniņa, Engineering)  
**Trigger:** HRIS webhook, eventSource: hris  
**Preconditions:** No existing Okta user with EMP-0001  

**Steps:**
1. Submit valid Joiner payload for EMP-0001 to JML-J webhook endpoint
2. Wait for flow execution to complete (max 2 minutes)

**Expected outcomes:**
- [ ] Okta user created with status `Active`
- [ ] Profile attributes match payload (email, department, title, managerId)
- [ ] ilm_status = `active`, ilm_lastEventType = `joiner`
- [ ] BL-* baseline groups assigned (6 groups)
- [ ] PKG-B-* package groups assigned (6 groups)
- [ ] ROLE-Engineering-Developer assigned
- [ ] No ITSM tasks created (all systems automated for Engineering)
- [ ] ilm_tx_log row: status = `Completed`, groupsAdded contains all assigned groups
- [ ] ilm_evidence_archive row present with same txId
- [ ] Okta System Log: user.lifecycle.create + group membership events, all carrying txId in debugContext
- [ ] Transaction time < 60 seconds

**Pass criteria:** All checkboxes above confirmed.  
**FR coverage:** FR1.1, FR1.2, FR1.3, FR1.4, FR1.5, FR1.6

---

## TC-J-02: Re-hire, suspended account

**Identity:** EMP-0001 (re-hire after suspension)  
**Preconditions:** Okta user EMP-0001 exists with status `Suspended`, has Finance-dept groups from previous employment

**Steps:**
1. Manually create suspended user with Finance groups in test tenant
2. Submit Joiner payload for EMP-0001 with department = Engineering

**Expected outcomes:**
- [ ] Flow takes re-hire path (not duplicate-detected)
- [ ] Previous Finance groups cleared (ROLE-Finance-*, PKG-A-*)
- [ ] Engineering baseline and Package B groups assigned
- [ ] ilm_tx_log eventSource = `rehire`
- [ ] No Finance groups remain on user after completion

**FR coverage:** FR1.2

---

## TC-J-03: Future start date, pre-provisioning

**Identity:** EMP-0003 (startDate = 10 days from today)  

**Expected outcomes:**
- [ ] Okta user created with status `Staged`
- [ ] ilm_tx_log status = `Pending-activation`
- [ ] Scheduled flow registered
- [ ] When scheduled flow fires: account transitions to `Active`, groups assigned, status = `Completed`
- [ ] Activation occurs within ±5 minutes of startDate

**FR coverage:** FR1.2

---

## TC-J-04: Manual application path

**Identity:** EMP-0004 (Sales — includes LegacyApp-Manual in system class)

**Expected outcomes:**
- [ ] Automated groups assigned normally
- [ ] ITSM provisioning task created for LegacyApp-Manual
- [ ] Task addressed to `legacy-app-owner` in ITSM
- [ ] Task body contains employeeNumber, txId, displayName, dueDate
- [ ] ilm_pendingTasks = true on Okta user
- [ ] Transaction status = `Pending-manual-tasks`
- [ ] When ITSM task closed: SH-1a callback received; transaction transitions to `Completed`

**FR coverage:** FR1.5

---

## TC-J-05: Contractor onboarding, ITSM trigger

**Identity:** EMP-0005 (contractor, Engineering)  
**Trigger:** ITSM webhook, eventSource: itsm

**Expected outcomes:**
- [ ] Restricted baseline groups only (BL-Email, BL-ITSM-User, BL-MFA-Policy, BL-SSO-Portal)
- [ ] BL-Intranet and BL-HR-Portal NOT assigned
- [ ] ilm_exceptions entry created with expiryDate
- [ ] Leaver flow scheduled for endDate

**FR coverage:** FR5.1

---

## TC-J-06: VIP classification

**Identity:** EMP-0006 (VIP flag in flags.vip = true)

**Expected outcomes:**
- [ ] CFG-3 detects VIP exception
- [ ] Manager confirmation requested before any access assignment (even baseline)
- [ ] Evidence record contains vip_handling = true
- [ ] ilm_exceptions entry for VIP classification

**FR coverage:** FR5.2

---

## TC-M-01: Department transfer, non-empty delta, approval required

**Identity:** EMP-0007 (Finance → Engineering transfer)  
**Preconditions:** EMP-0007 exists as active user with Finance groups

**Steps:**
1. Submit Mover payload with department changed from Finance to Engineering
2. Submit manager approval in ITSM
3. Submit app owner approval in ITSM

**Expected outcomes:**
- [ ] Delta calculation: ROLE-Finance-*, PKG-A-* in toRemove; PKG-B-*, ROLE-Engineering-* in toAdd
- [ ] Finance groups removed BEFORE Engineering groups added (check System Log timestamps)
- [ ] Approval tasks created for both manager and app owner
- [ ] Flow pauses until both approvals received
- [ ] After approval: Engineering groups added, Finance groups absent
- [ ] ilm_tx_log groupsRemoved and groupsAdded both populated
- [ ] Profile attributes updated (department, managerId)

**FR coverage:** FR2.1, FR2.2, FR2.3, FR2.4, FR2.6

---

## TC-M-02: Title change only, empty delta

**Identity:** EMP-0008 (title change only)

**Expected outcomes:**
- [ ] SH-5 returns isDeltaEmpty = true
- [ ] No group changes made
- [ ] Profile updated (title)
- [ ] Transaction closed as `Completed` with no-access-change note
- [ ] groupsAdded and groupsRemoved both empty arrays in log

**FR coverage:** FR2.2

---

## TC-M-05: Time-bound access grant

**Identity:** EMP-0009 (flags.expiryDate set 30 days from today)

**Expected outcomes:**
- [ ] Access granted after manager approval
- [ ] Scheduled reversal flow registered for expiryDate
- [ ] ilm_exceptions entry created
- [ ] When reversal fires: access removed, references original txId

**FR coverage:** FR2.5

---

## TC-L-01: Standard termination, same-day

**Identity:** EMP-0011 (endDate = today)

**Steps:**
1. Submit Leaver payload for EMP-0011

**Expected outcomes:**
- [ ] L-3 suspension within 30 seconds of payload receipt
- [ ] Okta System Log shows user.lifecycle.suspend event
- [ ] L-4: user.session.end event in System Log
- [ ] L-5: All group membership removal events in System Log (all carry txId)
- [ ] L-6: ITSM deprovisioning tasks created for any manual systems
- [ ] L-7: High-priority alert sent for critical systems
- [ ] L-8: Account in Deprovisioned status
- [ ] Evidence records EV-L-01 through EV-L-05 all present

**FR coverage:** FR3.1, FR3.2, FR3.3, FR3.4, FR3.6

---

## TC-L-02: Future end date, scheduled execution

**Identity:** EMP-0012 (endDate = 5 days from today)

**Expected outcomes:**
- [ ] ilm_tx_log status = `Pending-activation` immediately after payload receipt
- [ ] Account remains Active during intervening period
- [ ] When scheduled trigger fires: full Leaver sequence executes as per TC-L-01

**FR coverage:** FR3.1

---

## TC-L-04: Critical system ITSM task overdue

**Identity:** EMP-0013 (has ERP-Finance-Posting access — Critical)

**Steps:**
1. Submit Leaver payload
2. Deliberately do NOT close the ERP ITSM deprovisioning task
3. Allow SLA to expire

**Expected outcomes:**
- [ ] SH-1b fires on SLA expiry
- [ ] Critical alert sent to app owner's manager AND security ops contact
- [ ] ilm_tx_log remains in `Pending-manual-tasks`
- [ ] Escalation noted in transaction record

**FR coverage:** FR3.6

---

## TC-A-01: Manager-only approval, approved

**Identity:** EMP-0016 (Low-sensitivity item, manager-only tier)

**Expected outcomes:**
- [ ] One ITSM approval task created for manager only
- [ ] approvalRef in ilm_tx_log matches ITSM task ID
- [ ] Access assigned after approval
- [ ] Evidence fields: approver ID, timestamp, scope, decision all present

**FR coverage:** FR4.1, FR4.2, FR4.3, FR4.4

---

## TC-A-02: Manager + owner, app owner rejects

**Identity:** EMP-0016 (High-sensitivity item added)

**Steps:**
1. Submit access request requiring manager + owner approval
2. Manager approves
3. App owner rejects

**Expected outcomes:**
- [ ] Rejected item NOT assigned
- [ ] ilm_tx_log shows item as `access-denied` with rejecting approver ID
- [ ] Baseline and package access (not rejected) completes normally
- [ ] Transaction closes as `Completed` (not `Failed`)

**FR coverage:** FR4.1, FR4.2

---

## TC-A-03: Approval SLA breach, auto-reject policy

**Identity:** EMP-0017

**Steps:**
1. Submit access request requiring manager approval
2. Allow SLA window to expire without any approver action

**Expected outcomes:**
- [ ] SH-1b fires on expiry
- [ ] Item cancelled with `timeout` record
- [ ] Orchestrator resumed with rejected status
- [ ] Access item NOT assigned

**FR coverage:** FR4.1

---

## TC-A-04: Break-glass access

**Identity:** EMP-0015 (CloudConsole-Admin, Critical)

**Expected outcomes:**
- [ ] Approval tier = manager-and-security (automatic)
- [ ] Critical-priority approval task created
- [ ] Access granted with 4-hour expiry
- [ ] Scheduled reversal registered
- [ ] ilm_exceptions flagged break_glass: true
- [ ] Enhanced evidence logging

**FR coverage:** FR5.3, FR5.4

---

## TC-E-01: Validation failure, incomplete payload

**Identity:** EMP-0020 (submit payload with employeeNumber field omitted)

**Expected outcomes:**
- [ ] SH-4 returns isValid = false
- [ ] ilm_tx_log row: status = `Clarification-required`, errorDetail identifies missing field
- [ ] SH-2 warning notification sent
- [ ] NO Okta user created
- [ ] Re-submitting with corrected payload creates new transaction with new txId, succeeds

**NFR coverage:** NFR3.1

---

## TC-E-02: API retry exhaustion

**Steps:**
1. Configure ITSM mock to return HTTP 503 on all requests
2. Execute TC-L-01 (Leaver for EMP-0011)

**Expected outcomes:**
- [ ] SH-7 executes 3 retry attempts (visible in execution history)
- [ ] After 3rd failure: `Pending-intervention` state
- [ ] ilm_intervention_queue row created with error detail
- [ ] Critical alert sent via SH-2
- [ ] Critically: L-3 suspension and L-5 group removal ALREADY COMPLETED before ITSM failure at L-6
- [ ] After ITSM mock restored and SH-6 re-invoked: ITSM tasks created without duplication

**NFR coverage:** NFR3.2, NFR3.3

---

## TC-E-03: Duplicate webhook, idempotency

**Steps:**
1. Submit TC-J-01 payload for EMP-0001 (creates user successfully)
2. Submit identical payload again within 60 seconds

**Expected outcomes:**
- [ ] Second submission triggers duplicate check in J-2
- [ ] `Duplicate-detected` error written, ops alert sent
- [ ] NO changes made to existing Active account
- [ ] No second Okta user created

**NFR coverage:** NFR3.1

---

## TC-E-04: Partial Mover completion, recovery

**Steps:**
1. Start TC-M-01 (Mover for EMP-0007)
2. Disable Okta Groups API after M-5 (remove) completes but before M-6 (add)
3. Let SH-7 exhaust retries
4. Restore API
5. Re-invoke UTL-4 with txId

**Expected outcomes:**
- [ ] Transaction in `Pending-intervention` with partial state
- [ ] On recovery: idempotency check skips already-removed groups
- [ ] Remaining toAdd groups applied
- [ ] Final transaction state: `Completed`
- [ ] Final log accurately reflects all changes

**NFR coverage:** NFR3.3
