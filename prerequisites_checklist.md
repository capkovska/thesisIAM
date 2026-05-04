# Pre-Deployment Prerequisites Checklist

**Artefact version:** 1.0.0  
Complete all items before importing flows into any tier.

---

## Okta tenant

- [ ] Okta Identity Engine tenant provisioned
- [ ] Workflows feature enabled (Admin Console → Platform → Workflows)
- [ ] Workflow Administrator role assigned to build/deploy account
- [ ] ILM-Workflows-ServiceApp created with all 5 API scopes (see build_guide.md §1.2)
- [ ] ILM-Service-Connection created in Workflows using ServiceApp credentials
- [ ] All 6 Universal Directory custom attributes added (ilm_status, ilm_lastEventId, ilm_lastEventType, ilm_lastEventTs, ilm_accessPackage, ilm_pendingTasks)
- [ ] All groups from group_inventory (Appendix C) created in Okta
- [ ] All 6 Workflows tables created (empty): ilm_access_map, ilm_system_class, ilm_exceptions, ilm_tx_log, ilm_intervention_queue, ilm_evidence_archive

## HRIS integration

- [ ] HRIS webhook endpoint URL confirmed (Okta Workflows API trigger URL)
- [ ] HRIS configured to POST to endpoint on employee status changes: hire, transfer, termination
- [ ] Payload format validated against hris_webhook_payload_schema.json
- [ ] Test payload submitted and received by Workflows (check execution history)
- [ ] HRIS contact confirmed for Clarification-required notifications

## ITSM integration

- [ ] ITSM API token created with minimum scope: create/read tickets in ILM project space
- [ ] ITSM API token stored as Workflows connection secret (never in flow logic)
- [ ] ITSM callback webhook configured: POST to SH-1a endpoint on approval task action
- [ ] ITSM approval task template created and tested
- [ ] ITSM operations team inbox confirmed for alert routing

## Active Directory integration

- [ ] Okta AD Agent installed on domain-joined Windows server
- [ ] AD Agent outbound connectivity to *.okta.com:443 verified
- [ ] Test group membership write from Okta to AD confirmed
- [ ] AD groups corresponding to all Okta groups created in AD

## Network and security

- [ ] All Workflows connections use HTTPS (no plain HTTP endpoints)
- [ ] Service account credentials NOT embedded in any flow logic
- [ ] No credentials included in artefact export bundle or screenshots
- [ ] Credential rotation schedule documented (recommended: 90 days max for API tokens)

## Configuration tables

- [ ] ilm_system_class populated from ilm_system_class.json (adapted to your systems)
- [ ] ilm_access_map populated from ilm_access_map.json (adapted to your departments)
- [ ] ilm_exceptions cleared of synthetic example data before production use

## Test tier sign-off

- [ ] All 6 test cases from build_guide.md Part 5 passed in test tier
- [ ] Performance: Joiner mean time < 15 minutes, Leaver suspension < 5 minutes
- [ ] Evidence records present and complete for each test scenario
- [ ] ilm_evidence_archive rows confirmed as append-only (no update API available)
