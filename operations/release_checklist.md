# Release and Change Checklist

**Version:** 1.0.0  
**Use:** Complete before any change to flow logic or configuration tables in production.

---

## Type 1 — Configuration table change (L2 — no flow redeployment)

Applies to: adding/editing rows in `ilm_access_map`, `ilm_system_class`, `ilm_exceptions`

- [ ] Change tested in test tier (Mover or Joiner scenario for affected dept/system)
- [ ] Test result documented (screenshot or log export)
- [ ] Change reviewed by IAM team lead
- [ ] Change recorded in `Config-Templates/changelog.md` before applying in production
- [ ] Change applied in production
- [ ] First 3 production transactions involving affected dept/system monitored for correct behaviour
- [ ] Changelog entry updated with production apply date

---

## Type 2 — Helper or utility flow change (L3 — requires test tier validation)

Applies to: any edit to flows in `/ILM-Helpers/`, `/ILM-Config/`, `/ILM-Utilities/`, `/ILM-ApprovalEngine/`

- [ ] Change developed and unit-tested in dev tier
- [ ] Affected test cases re-run in test tier (minimum: the test cases that call this flow)
- [ ] All test cases pass
- [ ] Performance impact assessed (no increase in API call count per transaction)
- [ ] Change reviewed and approved by IAM team lead
- [ ] Artefact patch version incremented (e.g. 1.0.0 → 1.0.1)
- [ ] `Design-Spec/` updated if flow contract changed
- [ ] Release notes added to this document (see Section below)
- [ ] Change deployed to production during low-traffic window
- [ ] Post-deployment: execute TC-J-01 and TC-L-01 in production (smoke test)
- [ ] Monitoring for 24 hours post-deployment confirmed assigned

---

## Type 3 — Orchestrator flow change (L3 — full test suite)

Applies to: any edit to JML-J, JML-M, or JML-L

- [ ] All Type 2 steps above completed
- [ ] **Full test suite** run in test tier (all TC-J, TC-M, TC-L, TC-A, TC-E cases)
- [ ] Performance: mean T2P and T2R within targets
- [ ] Security review: no new API scopes required; no credentials in flow logic
- [ ] Artefact minor version incremented (e.g. 1.0.x → 1.1.0)
- [ ] Thesis appendix / design spec updated if flow structure changed
- [ ] Change reviewed by IAM team lead AND security ops contact
- [ ] Deployment scheduled and communicated to operations team

---

## Type 4 — Table schema change (requires migration)

Applies to: adding/removing columns from any Workflows table

- [ ] Migration procedure documented before any schema change
- [ ] All flows that read/write the changed table identified and updated
- [ ] Artefact **major version** incremented (e.g. 1.x.x → 2.0.0)
- [ ] Migration tested end-to-end in dev and test tiers
- [ ] Data migration script tested with synthetic data
- [ ] Rollback procedure documented
- [ ] Full Type 3 checklist also completed
- [ ] Production migration window scheduled (expect 2–4 hour maintenance)

---

## Release history

| Version | Date | Type | Summary | Released by |
|---|---|---|---|---|
| 1.0.0 | 2026-01-15 | Initial | Artefact v1.0.0 initial release | IAM Team |
