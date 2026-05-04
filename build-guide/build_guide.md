# Artefact - Build Guide

**Version:** 1.0.0  
**Target platform:** Okta Workflows (Okta Identity Engine, Workforce Identity tier or above)  
**Estimated build time:** 4-6 hours for a practitioner familiar with Okta Workflows

---

## Overview

This guide walks through constructing the complete ILM Workflows artefact from scratch in an Okta tenant. Follow each section in order. Do not activate any flow until its section instructs you to - flows that call helpers must be built before the helpers are activated.

---

## Part 1 - Prerequisites and tenant setup

### 1.1 Verify tenant prerequisites

Before beginning, confirm all of the following in your Okta admin console:

- [ ] Okta Workflows is enabled: **Admin Console → Platform → Workflows**
- [ ] You have the Workflow Administrator role (or Super Administrator for initial setup)
- [ ] Okta Identity Engine is active (check under **Settings → Account**)
- [ ] Universal Directory custom attributes are provisioned (see Section 1.3)

### 1.2 Create the service application

1. Go to **Applications → Applications → Create App Integration**
2. Select **API Services**
3. Name the application `ILM-Workflows-ServiceApp`
4. Under **Okta API Scopes**, grant the following scopes:
   - `okta.users.manage`
   - `okta.groups.manage`
   - `okta.apps.manage`
   - `okta.logs.read`
   - `okta.workflows.invoke`
5. Save the **Client ID** and generate a **Client Secret**
6. Do **not** assign an administrator role to this application
7. In Okta Workflows, create a new **Okta connection** using these credentials. Name it `ILM-Service-Connection`.

### 1.3 Add Universal Directory custom attributes

Navigate to **Directory → Profile Editor → User (default)**. Add the following custom attributes:

| Display Name | Variable Name | Data Type | Default |
|---|---|---|---|
| ILM Status | ilm_status | String | active |
| ILM Last Event ID | ilm_lastEventId | String | |
| ILM Last Event Type | ilm_lastEventType | String | |
| ILM Last Event Timestamp | ilm_lastEventTs | String | |
| ILM Access Package | ilm_accessPackage | String | |
| ILM Pending Tasks | ilm_pendingTasks | Boolean | false |

### 1.4 Create Okta groups

Create all groups listed in `Config-Templates/group_inventory.md` (Appendix C of the thesis). Ensure group names match exactly - they are case-sensitive in API calls.

### 1.5 Create Workflows tables

In the Workflows console, navigate to **Tables** and create six tables with the following names. Leave them empty for now - they will be populated in Part 3.

- `ilm_access_map`
- `ilm_system_class`
- `ilm_exceptions`
- `ilm_tx_log`
- `ilm_intervention_queue`
- `ilm_evidence_archive`

Refer to `Design-Spec/data_model_and_attribute_schema.md` for the column definitions of each table.

---

## Part 2 - Build flows (folder by folder)

### Folder: /ILM-Utilities/

Build these first - they are called by everything else and have no dependencies.

#### UTL-1: Username generator
**Purpose:** Derives Okta `login` from `firstName` and `lastName`.

1. Create a new flow named `UTL-1 Username generator`
2. Trigger: **Helper Flow** (called by other flows)
3. Inputs: `firstName` (Text), `lastName` (Text), `domain` (Text)
4. Logic:
   - **Text → Lowercase** card on `firstName` → output `fn`
   - **Text → Lowercase** card on `lastName` → output `ln`
   - **Text → Concatenate** card: `fn` + `.` + `ln` + `@` + `domain` → output `login`
5. Output: `login` (Text)
6. Save. Do **not** activate yet.

#### UTL-2: Date formatter and SLA calculator
**Purpose:** Parses ISO 8601 dates; calculates SLA due dates; compares dates to today.

1. Create flow named `UTL-2 Date formatter and SLA calculator`
2. Trigger: **Helper Flow**
3. Inputs: `inputDate` (Text), `slaHours` (Number), `action` (Text - values: `parse`, `addHours`, `isToday`, `isFuture`)
4. Logic: Use **Date & Time** cards:
   - `parse`: **Text → Date** on `inputDate`; output `parsedDate`
   - `addHours`: **Date → Add time** (slaHours × 3600 seconds); output `dueDate`
   - `isToday`: **Date → Format** today's date; **Text → Compare** with `inputDate`; output `isToday` (Boolean)
   - `isFuture`: **Date → Compare** `inputDate` > today; output `isFuture` (Boolean)
5. Use **IF/ElseIF** card to branch on `action`
6. Output: `result` (Text or Boolean depending on action)
7. Save.

#### UTL-3: Payload logger
**Purpose:** Writes a row to `ilm_tx_log`. Called at transaction open and on each state change.

1. Create flow named `UTL-3 Payload logger`
2. Trigger: **Helper Flow**
3. Inputs: `txId`, `eventType`, `eventSource`, `employeeNumber`, `status`, `groupsAdded` (Text/JSON), `groupsRemoved` (Text/JSON), `errorDetail`, `itsm_tasks` (Text/JSON)
4. Logic: **Okta Workflows Tables → Upsert Row** card targeting `ilm_tx_log`; key field = `txId`; map all input fields to table columns; set `createdAt` on first write using **Date → Now** card; set `completedAt` on terminal states
5. Save.

#### UTL-4: Okta API wrapper
**Purpose:** Generic user and group operations with idempotency checks.

1. Create flow named `UTL-4 Okta API wrapper`
2. Trigger: **Helper Flow**
3. Inputs: `action` (Text - values: `createUser`, `updateUser`, `suspendUser`, `deactivateUser`, `addToGroup`, `removeFromGroup`, `removeAllGroups`, `revokeTokens`, `getUser`, `getGroups`), plus action-specific fields (`userId`, `groupId`, `profile`, etc.)
4. For each action, use the corresponding **Okta** connector card from the `ILM-Service-Connection`:
   - `createUser` → **Okta → Create User**; before creating, call `getUser` to check for existing `employeeNumber` (idempotency)
   - `suspendUser` → **Okta → Suspend User**; guard with status check first
   - `deactivateUser` → **Okta → Deactivate User**
   - `addToGroup` → **Okta → Assign User to Group**; guard with group membership check
   - `removeFromGroup` → **Okta → Remove User from Group**
   - `removeAllGroups` → **Okta → List User's Groups** then loop with **removeFromGroup**
   - `revokeTokens` → **Okta → Clear User Sessions** + **Okta → Revoke Tokens for User**
5. Wrap all external calls in error handling (see SH-7 pattern below)
6. Save.

---

### Folder: /ILM-Config/

#### CFG-1: Access mapping table reader
1. Create flow `CFG-1 Access map reader`
2. Trigger: **Helper Flow**
3. Inputs: `department` (Text), `employeeType` (Text)
4. Logic: **Tables → Search Rows** on `ilm_access_map` where `department` = input AND `employeeType` = input; output first matching row
5. Output: `baselineGroups` (List), `packageGroups` (List), `roleGroups` (List)
6. Save.

#### CFG-2: System classification reader
1. Create flow `CFG-2 System class reader`
2. Trigger: **Helper Flow**
3. Inputs: `systemName` (Text) OR `getCriticalSystems` (Boolean)
4. Logic:
   - If `systemName` provided: **Tables → Search Rows** on `ilm_system_class` where `systemName` = input; return matching row
   - If `getCriticalSystems` = true: **Tables → Search Rows** where `isCritical` = true; return all matching rows as List
5. Save.

#### CFG-3: Exception registry reader
1. Create flow `CFG-3 Exception registry reader`
2. Trigger: **Helper Flow**
3. Inputs: `employeeNumber` (Text)
4. Logic: **Tables → Search Rows** on `ilm_exceptions` where `employeeNumber` = input; return all matching rows
5. Output: `exceptions` (List); `hasVip` (Boolean); `hasBreakGlass` (Boolean); `hasContractor` (Boolean)
6. Save.

---

### Folder: /ILM-Helpers/

#### SH-4: Input validator and normaliser
1. Create flow `SH-4 Validator normaliser`
2. Trigger: **Helper Flow**
3. Inputs: Raw trigger payload fields
4. Logic:
   - **Text → Is Empty** check on each required field; collect missing fields into a list
   - If any required field missing: set `isValid = false`; set `errorDetail = "Missing fields: " + list`
   - **Text → Uppercase/Lowercase** and **Text → Trim** on `department` for canonical form
   - **Text → Concatenate** to derive `displayName` if absent
   - **Date → Parse** on `startDate` / `endDate` to validate format
5. Outputs: `isValid` (Boolean), `errorDetail` (Text), `normalisedPayload` (Object)
6. Save.

#### SH-5: Group delta calculator
1. Create flow `SH-5 Group delta calculator`
2. Trigger: **Helper Flow**
3. Inputs: `currentGroups` (List), `targetGroups` (List)
4. Logic:
   - Filter `currentGroups` to exclude `BL-*` prefix items → `currentNonBaseline`
   - Filter `targetGroups` to exclude `BL-*` prefix items → `targetNonBaseline`
   - `toAdd` = items in `targetNonBaseline` NOT in `currentNonBaseline` (**List → Filter** + **List → Contains**)
   - `toRemove` = items in `currentNonBaseline` NOT in `targetNonBaseline`
   - `retained` = items in both lists (no-op)
5. Outputs: `toAdd` (List), `toRemove` (List), `retained` (List), `isDeltaEmpty` (Boolean)
6. Save.

#### SH-6: ITSM task generator
1. Create flow `SH-6 ITSM task generator`
2. Trigger: **Helper Flow**
3. Inputs: `txId`, `employeeNumber`, `displayName`, `department`, `systemName`, `action` (provision/deprovision/update), `dueDate`, `appOwnerItsm`
4. Logic: **HTTP → Post** to ITSM API endpoint `/api/now/table/sc_task` (ServiceNow) or equivalent; body includes all input fields plus a structured description; authentication via ITSM API token connection secret
5. Output: `taskId` (Text), `taskUrl` (Text)
6. Save.

#### SH-7: Error handler and retry coordinator
1. Create flow `SH-7 Error handler retry`
2. Trigger: **Helper Flow**
3. Inputs: `flowName`, `cardName`, `txId`, `retryCount` (Number), `maxRetries` (Number - default 3), `baseDelaySeconds` (Number - default 10)
4. Logic:
   - If `retryCount` < `maxRetries`: **Flow Control → Wait** for `baseDelaySeconds * (2 ^ retryCount)` seconds; increment `retryCount`; output `shouldRetry = true`
   - If `retryCount` >= `maxRetries`: Write to `ilm_intervention_queue` via **Tables → Create Row**; call SH-2 with `severity=critical`; output `shouldRetry = false`
5. Outputs: `shouldRetry` (Boolean), `updatedRetryCount` (Number)
6. Save.

#### SH-2: Notifier
1. Create flow `SH-2 Notifier`
2. Trigger: **Helper Flow**
3. Inputs: `severity` (info/warning/critical), `txId`, `employeeNumber`, `message`, `recipients` (List)
4. Logic:
   - **Send Email** card to `recipients` with subject `[ILM-{severity.toUpperCase()}] {message}` and body including `txId`, `employeeNumber`, timestamp
   - If `severity = critical`: additionally create ITSM incident via SH-6 equivalent with priority=1
5. Save.

#### SH-3: Evidence writer
1. Create flow `SH-3 Evidence writer`
2. Trigger: **Helper Flow**
3. Inputs: All `ilm_tx_log` fields (pass full transaction object)
4. Logic:
   - **UTL-3** to update `ilm_tx_log` row (upsert)
   - **Tables → Create Row** on `ilm_evidence_archive` (insert only - never upsert; this maintains append-only integrity)
5. Save.

---

### Folder: /ILM-ApprovalEngine/

#### SH-1: Approval router
1. Create flow `SH-1 Approval router`
2. Trigger: **Helper Flow**
3. Inputs: `txId`, `employeeNumber`, `displayName`, `accessItems` (List), `approvalTier`
4. Logic:
   - Look up approver ITSM IDs based on `approvalTier` (from CFG-2 data passed in)
   - For each required approver: call SH-6 to create an ITSM approval task; store task IDs
   - **UTL-3** to set `status = Pending-approval` and store `approvalRef`
   - Register SH-1b scheduled flow via **Flow Control → Schedule** for `now + SLA hours`
5. Output: `approvalTaskIds` (List)
6. Save.

#### SH-1a: Callback handler
1. Create flow `SH-1a Approval callback handler`
2. Trigger: **API Endpoint** (HTTP POST - this is the inbound webhook from ITSM)
3. Inputs from webhook body: `txId`, `approverItsm`, `decision` (approved/rejected), `itemId`
4. Logic:
   - **Tables → Search Rows** on `ilm_tx_log` to validate `txId` is in `Pending-approval` state
   - Validate `approverItsm` matches expected approver for tier
   - Self-approval check: if `approverItsm` matches the user's `employeeNumber` in the transaction → reject with error
   - Update per-item approval count in transaction record
   - If all required approvers have responded: **Flow Control → Resume Flow** using the Okta Workflows Resume API targeting the paused orchestrator's instance ID (stored in `ilm_tx_log.approvalRef`)
5. Save and **Activate** (this must be active to receive callbacks).

#### SH-1b: Escalation timer
1. Create flow `SH-1b Escalation timer`
2. Trigger: **Scheduled** (registered dynamically by SH-1)
3. Inputs: `txId`, `approvalTier`, `escalationPolicy` (escalate/reject)
4. Logic:
   - Check if `ilm_tx_log` row for `txId` is still in `Pending-approval`; if not (already resolved) - exit cleanly
   - If `escalationPolicy = escalate`: re-route approval request to approver's manager; reset SLA clock; call SH-2 with warning
   - If `escalationPolicy = reject`: write `timeout` to transaction; call Workflows Resume API with all items set to `rejected`; call SH-2 with critical alert
5. Save.

---

### Folder: /ILM-Orchestrators/

#### JML-J: Joiner orchestrator
Follow the step-by-step logic in `Design-Spec/joiner_flow_spec.md`. Key card sequence:

1. **API Endpoint trigger** (HRIS webhook) + **Scheduled trigger** variant
2. **Call Flow** → SH-4 (validate)
3. **If** isValid = false → **Call Flow** UTL-3 (write error) → **Call Flow** SH-2 (notify) → **End**
4. **Call Flow** UTL-4 (getUser by employeeNumber)
5. Branch on user status (not found / suspended / active)
6. **Call Flow** UTL-4 (createUser or reactivateUser)
7. **Call Flow** CFG-1 (get baseline groups)
8. **For Each** group in baselineGroups: **Call Flow** UTL-4 (addToGroup) wrapped in SH-7
9. **Call Flow** CFG-1 (get package groups)
10. **If** approval required: **Call Flow** SH-1 → **Wait for Response** card
11. **For Each** approved group: **Call Flow** UTL-4 (addToGroup)
12. **For Each** manual system: **Call Flow** SH-6 (create ITSM task)
13. **Call Flow** SH-3 (write evidence)
14. **Call Flow** UTL-4 (updateUser - set ilm_* attributes)

Activate after full build and test tier validation.

#### JML-M: Mover orchestrator
Follow `Design-Spec/mover_flow_spec.md`. Key additions vs Joiner:
- Step M-3 calls SH-5 for delta; add **If** isDeltaEmpty branch
- Step M-5 (remove) always precedes Step M-6 (add)
- Step M-7 profile update is last

#### JML-L: Leaver orchestrator
Follow `Design-Spec/leaver_flow_spec.md`. Key note:
- Step L-3 (suspend) must be placed **before** any ITSM task generation or downstream calls
- Use **Error Handler** around L-3 with immediate retry (no backoff) - this step must not silently fail

---

## Part 3 - Populate configuration tables

Import the JSON files from `Config-Templates/` into the corresponding Workflows tables using **Tables → Import CSV** (convert JSON to CSV first) or via the Workflows API.

Order:
1. `ilm_system_class.json` → `ilm_system_class` table
2. `ilm_access_map.json` → `ilm_access_map` table
3. `ilm_exceptions.json` → `ilm_exceptions` table (remove synthetic example rows before production use)

---

## Part 4 - Activation sequence

Activate flows in this order to avoid broken references:

1. All UTL-* flows
2. All CFG-* flows  
3. SH-4, SH-5, SH-6, SH-7, SH-2, SH-3
4. SH-1a (approval callback - must be active before SH-1)
5. SH-1, SH-1b
6. JML-J, JML-M, JML-L

---

## Part 5 - Pre-activation test checklist

Run these tests in the test tier before activating in production:

- [ ] TC-J-01: Standard Joiner, HRIS trigger → account created, baseline groups assigned, evidence written
- [ ] TC-J-02: Re-hire (suspended account) → profile updated, old groups cleared, re-activated
- [ ] TC-M-01: Department transfer → correct delta, remove-before-add, evidence written
- [ ] TC-L-01: Same-day Leaver → suspension within 30 seconds, all groups removed, ITSM tasks created
- [ ] TC-E-01: Incomplete payload → error record written, no account created
- [ ] TC-E-02: ITSM API unavailable → retry loop executes, intervention queue populated

All tests must pass before production activation.
