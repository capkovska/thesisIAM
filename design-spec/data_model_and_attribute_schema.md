# Data Model and Attribute Schema

**Version:** 1.0.0

---

## Universal Directory Profile Attributes

All custom attributes use the `ilm_` prefix to avoid collisions with application provisioning connectors.

### HRIS-sourced attributes (written by Joiner/Mover flows; do not overwrite from downstream systems)

| Attribute | Type | Notes |
|---|---|---|
| `login` | String | `firstname.lastname@domain` |
| `email` | String | Primary work email; SSO subject |
| `firstName` | String | |
| `lastName` | String | |
| `displayName` | String | Derived: `firstName + " " + lastName` |
| `department` | String | Canonical form; must match `ilm_access_map` key |
| `title` | String | Informational |
| `employeeNumber` | String | Correlation key between HRIS and Okta |
| `employeeType` | String | `employee`, `contractor`, `service-account` |
| `managerId` | String | Manager's `employeeNumber`; approval routing |
| `startDate` | Date | Joiner effective date |
| `endDate` | Date | Leaver effective date; scheduled suspension |
| `costCenter` | String | Finance reporting and licence allocation |
| `location` | String | Location-specific baseline group assignment |

### Workflow-managed attributes (set by flows; never sourced from HRIS)

| Attribute | Type | Values / Notes |
|---|---|---|
| `ilm_status` | String | `active`, `suspended`, `deprovisioned` |
| `ilm_lastEventId` | String | UUID of last processed lifecycle transaction |
| `ilm_lastEventType` | String | `joiner`, `mover`, `leaver`, `rehire` |
| `ilm_lastEventTs` | DateTime | Timestamp of last successful lifecycle action |
| `ilm_accessPackage` | String | `A`, `B`, or `individual` |
| `ilm_pendingTasks` | Boolean | True when ITSM manual tasks remain open |

---

## Group Naming Convention

| Tier | Pattern | Example | Assigned by |
|---|---|---|---|
| Baseline | `BL-<resource>` | `BL-Email` | Joiner (all employees) |
| Package A | `PKG-A-<resource>` | `PKG-A-CRM-User` | Joiner/Mover (operational roles) |
| Package B | `PKG-B-<resource>` | `PKG-B-DevTools` | Joiner/Mover (technical roles) |
| Role/dept | `ROLE-<dept>-<level>` | `ROLE-Finance-ReadOnly` | Mover delta calculation |
| App admin | `APP-<app>-Admin` | `APP-ERP-Admin` | Explicit request + triple approval |

---

## Workflows Table Schemas

### ilm_access_map
Key: `{department}:{employeeType}`

| Field | Type | Description |
|---|---|---|
| `department` | String | Canonical department name |
| `employeeType` | String | `employee` or `contractor` |
| `baselineGroups` | JSON Array | BL-* groups to assign |
| `packageGroups` | JSON Array | PKG-A-* or PKG-B-* groups |
| `roleGroups` | JSON Array | ROLE-* groups for this dept |

### ilm_system_class
Key: `systemName`

| Field | Type | Description |
|---|---|---|
| `systemName` | String | Unique system identifier |
| `integrationMethod` | String | `scim`, `ad`, or `manual` |
| `sensitivityTier` | String | `Low`, `Medium`, `High`, `Critical` |
| `appOwnerItsm` | String | ITSM user ID of app owner |
| `provisionSla` | Integer | Hours to provision |
| `deprovisionSla` | Integer | Hours to deprovision |
| `isCritical` | Boolean | True = critical systems check in Leaver L-7 |
| `approvalTier` | String | `manager-only`, `manager-and-owner`, `manager-and-security` |

### ilm_exceptions
Key: `{employeeNumber}:{exceptionType}`

| Field | Type | Description |
|---|---|---|
| `employeeNumber` | String | |
| `exceptionType` | String | `contractor`, `vip`, `break-glass`, `time-bound` |
| `expiryDate` | Date | Mandatory for time-bound and break-glass |
| `originalTxId` | String | Transaction that created the exception |
| `riskAccepted` | Boolean | True if exception was risk-accepted |
| `approvedBy` | String | ITSM user ID of approver |

### ilm_tx_log
Key: `txId`

| Field | Type | Description |
|---|---|---|
| `txId` | String (PK) | UUID generated at event receipt |
| `eventType` | String | `joiner`, `mover`, `leaver` |
| `eventSource` | String | `hris` or `itsm` |
| `employeeNumber` | String | Correlation key |
| `status` | String | Current transaction state |
| `approvalRef` | String | ITSM approval ticket ID |
| `itsm_tasks` | JSON | Array of open ITSM task IDs |
| `groupsAdded` | JSON | Groups added in this event |
| `groupsRemoved` | JSON | Groups removed in this event |
| `errorDetail` | String | Last error if Pending-intervention |
| `createdAt` | DateTime | Event receipt timestamp |
| `completedAt` | DateTime | Terminal state timestamp |

### ilm_intervention_queue
Key: `txId`

| Field | Type | Description |
|---|---|---|
| `txId` | String | |
| `failedFlow` | String | Flow name where failure occurred |
| `failedCard` | String | Card name within flow |
| `errorDetail` | String | HTTP status or error message |
| `retryCount` | Integer | Number of retry attempts made |
| `status` | String | `open` or `resolved` |
| `resolutionNote` | String | Operator note on resolution |

### ilm_evidence_archive
Key: `txId` - **append-only; never updated after initial write**

Same schema as `ilm_tx_log`. One row written per transaction at terminal state. Used for audit export only.
