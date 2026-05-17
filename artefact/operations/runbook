# ILM Artefact - Operational Runbook

**Version:** 1.0.0  
**Audience:** IAM operations team (L1, L2 support)  
**Artefact version this runbook covers:** 1.0.0

---

## Section F.1 - Daily monitoring checklist

Perform these checks at the start of each business day:

1. **Flow status check**
   - Open Okta Workflows console → `/ILM-Orchestrators/` folder
   - Confirm JML-J, JML-M, JML-L are all in `On` status
   - If any flow is `Off`: check execution history for the reason before re-enabling

2. **Intervention queue review**
   - Open Workflows Tables → `ilm_intervention_queue`
   - Filter `status = open`
   - For each open row: read `errorDetail` and initiate recovery procedure (Section F.3)

3. **Stale pending-manual-tasks review**
   - Query `ilm_tx_log` for rows with `status = Pending-manual-tasks` AND `createdAt` > 48 hours ago
   - Confirm the corresponding ITSM tasks are visible and assigned
   - If an ITSM task is missing: re-invoke SH-6 with the `txId` to regenerate it

4. **Alert inbox review**
   - Check operations team inbox for SH-2 alerts
   - Critical alerts (Leaver suspension failure, critical system deprovisioning pending) require immediate L2 escalation
   - Warning alerts (approval SLA breaches, duplicate detection) can be addressed within the business day

---

## Section F.2 - Responding to a Pending-approval timeout

When SH-1b fires on an approval SLA breach:

**For manager-only approvals:**
1. Locate the ITSM approval task via `ilm_tx_log.approvalRef`
2. Confirm whether the manager received the task (check ITSM assignment queue)
3. If not received: contact manager directly; re-invoke SH-1 with `txId` to regenerate the task
4. If manager is on leave: escalate to manager's manager; update `approverOverride` in `ilm_tx_log`

**For manager-and-owner or manager-and-security approvals:**
- Do not re-route without documented authority
- Raise an L2 incident
- Await guidance from IAM team lead before modifying any approval requirement
- Document the decision in `ilm_tx_log.errorDetail`

**Escalation contacts:**
- L1 escalation: IAM team lead
- L2 escalation: CISO for security-tier approvals; Department head for owner-tier approvals

---

## Section F.3 - Recovering a Pending-intervention transaction

**Step 1: Identify the failed step**
```
Open ilm_intervention_queue → find row with txId
Read: failedFlow, failedCard, errorDetail, retryCount
```

**Step 2: Resolve root cause**

| Error pattern | Likely cause | Resolution |
|---|---|---|
| HTTP 401 on Okta API | Service account OAuth token expired | Rotate client secret in Okta service app; update Workflows connection secret |
| HTTP 429 on Okta API | Rate limit reached | Wait 60 seconds; if persistent, check for runaway retry loops |
| HTTP 503 on ITSM API | ITSM maintenance window | Wait for ITSM to return; check ITSM status page |
| `user not found` in UTL-4 | User manually deleted from Okta | Re-create via Joiner flow if appropriate; if termination, close as exception-manual-close |
| `group not found` in UTL-4 | Group renamed or deleted | Re-create group with correct name; verify ilm_access_map consistency |
| `connection timeout` | Network interruption between Workflows and external API | Wait for network recovery; then re-invoke |

**Step 3: Re-invoke the failed helper**
1. Navigate to the flow named in `failedFlow` in the Workflows console
2. Click **Test** (manually invoke) with `txId` as input
3. UTL-4's idempotency check will skip already-completed actions
4. Confirm the transaction transitions to `Completed` or `Pending-manual-tasks`

**Step 4: Close the intervention record**
1. Open `ilm_intervention_queue` table
2. Set `status = resolved`
3. Write resolution note in `resolutionNote` field including: what was done, when, and by whom

---

## Section F.4 - Updating the access mapping table

When a new department is created, renamed, or changes its standard access package:

1. Apply the change in the **test tier** first
2. Execute TC-M-01 (Mover - department transfer) for a test identity in the affected department
3. Confirm correct groups in `groupsAdded` in the test tier `ilm_tx_log`
4. Apply change in production (`ilm_access_map` table)
5. Record the change in `Config-Templates/changelog.md`
6. Monitor the next three production transactions involving the affected department

**No flow redeployment is needed.** Changes to `ilm_access_map` take effect immediately for subsequent transactions.

---

## Section F.5 - Adding a new manually managed application

1. Add a row to `ilm_system_class` in the Workflows table:
   - `systemName`: unique identifier (no spaces)
   - `integrationMethod`: `manual`
   - `sensitivityTier`: Low / Medium / High / Critical
   - `appOwnerItsm`: the application owner's ITSM user ID
   - `provisionSla`: hours allowed for provisioning task closure
   - `deprovisionSla`: hours allowed for deprovisioning task closure
   - `isCritical`: true if this system appears in the critical systems check during Leaver
   - `approvalTier`: see approval_routing_matrix.md

2. No flow changes required

3. Notify the application owner that they will receive ITSM ILM tasks and confirm the closure procedure with them

4. Add the system to `Config-Templates/changelog.md`

---

## Section F.6 - Audit evidence export

To produce an evidence package for a specific user or time period:

**From ilm_evidence_archive (recommended for audit):**
1. Navigate to Workflows Tables → `ilm_evidence_archive`
2. Filter by `employeeNumber` and/or `createdAt` date range
3. Export as CSV (File → Export CSV)
4. The CSV is a point-in-time snapshot of completed transactions; rows are never updated after initial write

**From Okta System Log (for independent verification):**
1. Open Okta Admin Console → Reports → System Log
2. Filter: `debugContext.debugData.requestId eq "{txId}"`
3. This returns all API events associated with that transaction
4. Export as CSV

**Combining both sources:**
The `ilm_evidence_archive` row and the System Log export together constitute the complete evidence package for a transaction, as specified in Table 3.2 of the thesis.

**Retention:**
- Identity action records (Joiner, Leaver): retain 7 years
- Mover and execution summary records: retain 3 years
- Exception and error records: retain 3 years

Export `ilm_evidence_archive` to a durable external store (object storage, SIEM, or secure file archive) at least monthly. The Workflows table store does not have built-in archival; exports ensure evidence is preserved beyond the platform retention window.

---

## Section F.7 - Handling a contractor expiry

When a contractor's `endDate` arrives and the scheduled Leaver flow fires:

1. The flow executes automatically - no manual action required for the automation path
2. Check `ilm_tx_log` to confirm the contractor's transaction completed successfully
3. Confirm the contractor's `ilm_exceptions` entry has been removed or flagged inactive
4. If the contractor's engagement is extended before the endDate:
   - Update `endDate` in HRIS (will trigger a Mover event updating the Okta profile)
   - Update `expiryDate` in `ilm_exceptions` to match the new endDate
   - Cancel the previously scheduled Leaver trigger (in Workflows execution history, find and stop the scheduled flow)
   - Document the extension in `Config-Templates/changelog.md` with the approving manager

---

## Section F.8 - Flow version promotion checklist (dev → test → production)

Complete all items before promoting to the next tier.

**Pre-promotion (all tiers):**
- [ ] All flows in source tier are `On` with no draft changes pending
- [ ] `ilm_system_class` entries present and complete for all in-scope systems
- [ ] Service application scopes verified (Table 4.1)
- [ ] HRIS and ITSM connector credentials tested with live ping
- [ ] AD Agent connectivity verified

**Dev → Test:**
- [ ] TC-J-01 passes (standard Joiner)
- [ ] TC-J-02 passes (re-hire)
- [ ] TC-M-01 passes (department transfer)
- [ ] TC-L-01 passes (same-day Leaver)
- [ ] TC-E-01 passes (validation failure)
- [ ] TC-E-02 passes (API retry)

**Test → Production:**
- [ ] All dev→test test cases above pass in test tier
- [ ] Joiner mean time < 15 minutes measured over 5+ test runs
- [ ] Leaver suspension mean time < 5 minutes measured over 5+ test runs
- [ ] Evidence records complete and correct for each test
- [ ] `ilm_evidence_archive` confirmed append-only
- [ ] System Log txId cross-reference verified
- [ ] Operations team briefed on any changes from previous version
- [ ] Changelog updated

---

## Section F.9 - Escalation contacts template

*Replace with your organisation's actual contacts before production deployment.*

| Role | ITSM user ID | Email | When to contact |
|---|---|---|---|
| IAM Team Lead | iam-lead | iam-lead@org.example | L1 escalation; approval routing questions |
| Security Ops Contact | sec-ops | sec-ops@org.example | Critical system alerts; break-glass approvals |
| CISO | ciso-user | ciso@org.example | Security-tier approval escalations; risk acceptance |
| IT Ops | it-ops | it-ops@org.example | AD Agent issues; connector failures |
| Finance App Owner | finance-app-owner | finance-apps@org.example | ERP access tasks |
| Cloud Platform Owner | cloud-platform-owner | cloud-ops@org.example | CloudConsole access tasks |

---

## Section F.10 - Known limitations and workarounds

| Limitation | Workaround |
|---|---|
| SCIM licence reclamation not verifiable in sandbox | In production, verify via Microsoft 365 admin portal that M365 licence is released within 24 hours of Leaver execution |
| Concurrent high-volume events (>10 simultaneous) not load-tested | Monitor API rate limit events in Okta System Log; if persistent 429 errors appear, space event submissions via HRIS batch schedule |
| `ilm_evidence_archive` has no built-in expiry | Export monthly to external storage per Section F.6 |
| Break-glass expiry is policy-enforced, not hard-enforced | Monitor `ilm_exceptions` for overdue break-glass entries; if reversal flow did not fire, manually re-invoke via Workflows console |
