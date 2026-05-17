# Configuration Change Log

**Artefact version:** 1.0.0  
**Format:** Date | Changed by | Table/File | Change description | Reason

---

## Change history

| Date | Changed by | Target | Change | Reason |
|---|---|---|---|---|
| 2026-01-15 | IAM Team | `ilm_access_map.json` | Initial population - Engineering, Finance, Sales, HR, Legal dept mappings | Artefact v1.0.0 release |
| 2026-01-15 | IAM Team | `ilm_system_class.json` | Initial population - 9 systems classified | Artefact v1.0.0 release |
| 2026-01-15 | IAM Team | `ilm_exceptions.json` | Initial population - example entries only (synthetic data) | Artefact v1.0.0 release |

---

## How to record a change

When modifying any configuration table in production, add a row to this log **before** applying the change. Follow the process in the runbook (Operations/runbook.md Section F.4).

Fields:
- **Date**: ISO 8601 date
- **Changed by**: Name or role of person making the change
- **Target**: Which file or Workflows table was changed
- **Change**: Brief description of what was added, removed, or modified
- **Reason**: Business justification (ticket reference preferred)

---

## Versioning policy

Configuration changes do **not** increment the artefact version number unless they modify the table schema (column additions or removals). Schema changes increment the major version and require a migration procedure.

Content-only changes (adding/editing rows) are tracked in this log only.
