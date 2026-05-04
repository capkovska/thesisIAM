# Requirements Traceability Matrix

**Version:** 1.0.0  
**Maps:** Each FR and NFR → implementation location → test case(s) → verdict

---

## Functional Requirements

| Req ID | Description (summary) | Implemented in | Test case(s) | Verdict |
|---|---|---|---|---|
| FR1.1 | Joiner — dual trigger (HRIS + ITSM) | JML-J steps J-1, trigger config | TC-J-01, TC-J-05 | Pass |
| FR1.2 | Create or activate identity | JML-J steps J-2, J-3 | TC-J-01, TC-J-02 | Pass |
| FR1.3 | Attribute validation + normalisation | SH-4, JML-J J-1 | TC-J-01 (positive), TC-E-01 (negative) | Pass |
| FR1.4 | Baseline access assignment | JML-J J-4, ilm_access_map BL-* rows | TC-J-01 | Pass |
| FR1.5 | Integrated + manual execution paths | JML-J J-7, SH-6, ilm_system_class | TC-J-01, TC-J-04 | Pass |
| FR1.6 | Evidence record per Joiner transaction | SH-3, ilm_tx_log, ilm_evidence_archive | TC-J-01 | Pass |
| FR2.1 | Mover — dual trigger | JML-M trigger config | TC-M-01, TC-M-05 | Pass |
| FR2.2 | Attribute update | JML-M step M-7, UTL-4 | TC-M-01, TC-M-02 | Pass |
| FR2.3 | Remove-before-add sequencing | JML-M steps M-5 before M-6 | TC-M-01 | Pass |
| FR2.4 | Integrated + manual mover paths | JML-M M-5/M-6, SH-6 | TC-M-01 | Pass |
| FR2.5 | Time-bound access | JML-M flags.expiryDate handling, ilm_exceptions | TC-M-05 | Pass |
| FR2.6 | Evidence record per Mover | SH-3, EV-M-01…04 | TC-M-01 | Pass |
| FR3.1 | Leaver — dual trigger + scheduling | JML-L L-1, UTL-2 | TC-L-01, TC-L-02 | Pass |
| FR3.2 | Central suspension first | JML-L L-3 (priority step) | TC-L-01 | Pass |
| FR3.3 | Session and token revocation | JML-L L-4, UTL-4 | TC-L-01 | Pass |
| FR3.4 | Integrated + manual deprovisioning | JML-L L-5, L-6, SH-6 | TC-L-01 | Pass |
| FR3.5 | Licence reclamation | JML-L L-5 (group removal triggers SCIM) | TC-L-01 | Partial* |
| FR3.6 | Critical systems completeness check | JML-L L-7, CFG-2, SH-2 | TC-L-04 | Pass |
| FR4.1 | Tiered approval routing | SH-1, ilm_system_class.approvalTier | TC-A-01, TC-A-02, TC-A-03 | Pass |
| FR4.2 | No execution before approval | JML-J J-6 Wait-for-Response card | TC-A-01 | Pass |
| FR4.3 | Separation of request and execution | SH-1 (async), orchestrator pause | TC-A-01 | Pass |
| FR4.4 | Approval evidence fields | SH-1a callback handler, ilm_tx_log.approvalRef | TC-A-01 | Pass |
| FR5.1 | Contractor lifecycle | JML-J contractor path, ilm_exceptions, scheduled Leaver | TC-J-05, TC-L-05 | Pass |
| FR5.2 | VIP classification | CFG-3, JML-J VIP branch | TC-J-06 | Pass |
| FR5.3 | Break-glass access | TC-A-04, ilm_system_class, SH-1 Critical tier | TC-A-04 | Pass |
| FR5.4 | Exception workflow states | ilm_tx_log status values, CFG-3 | TC-J-06, TC-A-04 | Pass |
| FR6.1 | Evidence package per transaction | SH-3, EV-J/M/L records in ilm_evidence_archive | TC-J-01, TC-M-01, TC-L-01 | Pass |
| FR6.2 | Export formats (CSV, JSON) | ilm_tx_log export, ilm_evidence_archive export | — (inspection) | Pass |
| FR6.3 | Traceability chain (event → evidence) | txId in debugContext, dual-log design | TC-J-01 | Pass |

*FR3.5 Partial: Group removal confirmed in Okta (initiates SCIM deprovisioning signal). Downstream licence removal in M365 not directly observable in sandbox — environmental constraint, not design deficiency.

---

## Non-Functional Requirements

| Req ID | Description (summary) | Implemented in | Evidence | Verdict |
|---|---|---|---|---|
| NFR1.1 | Least-privilege service account | Table 4.1 scopes only, no admin role | Scope inspection | Pass |
| NFR1.2 | Secure credential storage | Workflows connection secrets, no hardcoded values | Flow inspection | Pass |
| NFR1.3 | Approval bypass prevention | SH-1a validates approver ID; self-approval check | TC-A-01 | Pass |
| NFR1.4 | Privileged access time-limiting | APP-*-Admin max 90 days; break-glass 4h expiry | TC-A-04 | Pass |
| NFR2.1 | Full transaction traceability | ilm_tx_log + Okta System Log cross-ref by txId | TC-J-01, TC-L-01 | Pass |
| NFR2.2 | Minimum evidence set per transaction | SH-3, EV-J/M/L records | All main TCs | Pass |
| NFR2.3 | Evidence readable by governance stakeholders | ilm_evidence_archive CSV export | Practitioner review | Pass |
| NFR3.1 | Idempotency — no duplicates on retry | UTL-4 existence checks; J-2 duplicate guard | TC-E-01, TC-E-03 | Pass |
| NFR3.2 | Error handling — common failure modes | SH-7 retry/backoff, intervention queue, SH-2 alerts | TC-E-02 | Pass |
| NFR3.3 | Consistent state on partial failure | Partial completion recoverable via re-invoke | TC-E-04 | Pass |
| NFR4.1 | Timeliness KPIs measurable | Table 5.2, KPI-1/KPI-2 definitions | Test execution | Pass |
| NFR4.2 | Scalable to 50 events/month | API call count per event ~10; well below rate limits | Performance analysis | Pass |
| NFR4.3 | Rate limit awareness | SH-7 backoff; UTL-4 existence checks reduce unnecessary calls | TC-E-02 | Pass |
| NFR5.1 | Modular decomposition | 5-folder structure, 25 distinct flows | Architecture review | Pass |
| NFR5.2 | Config-driven extensibility | ilm_access_map, ilm_system_class tables | Config change test | Pass |
| NFR5.3 | Versioning practices | Semantic versioning, changelog.md | Documentation | Pass |
| NFR5.4 | Understandable workflow logic | Card annotations, naming conventions, build guide | Practitioner review | Pass |
| NFR6.1 | Operational logging | ilm_tx_log all states, txId in System Log | All main TCs | Pass |
| NFR6.2 | Failure visibility and escalation | SH-7 → intervention queue → SH-2 alerts | TC-E-02 | Pass |
| NFR6.3 | Operational runbook | Operations/runbook.md, Appendix F | Documentation | Pass |
| NFR7.1 | Minimal personal data handling | Only lifecycle-required attributes used | Design review | Pass |
| NFR7.2 | Academic/production separation | Sandbox + synthetic data throughout evaluation | Environment setup | Pass |
| NFR7.3 | Evidence retention documented | Table 3.2, Appendix D, runbook Section F.6 | Documentation | Pass |
