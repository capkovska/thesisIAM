# ILM Artefact Bundle — v1.0.0

**Thesis:** The low-code/no-code (LCNC) solution for identity lifecycle management using Okta Workflows  
**Author:** Endija Dārija Čapkovska  
**Institution:** Riga Technical University, Faculty of Computer Science, Information Technology and Energy  
**Year:** 2026  
**Artefact version:** 1.0.0

---

## What this bundle contains

This artefact package is the deliverable described in Section 3.8 of the thesis. It contains everything needed to understand, reproduce, deploy, and operate the ILM automation solution.

---

## Directory structure

```
/Design-Spec/       — Flow specifications, data model, attribute schema
/Config-Templates/  — Workflows table definitions, approval matrix, payload schema
/Build-Guide/       — Step-by-step build instructions and prerequisites checklist
/Evaluation/        — Synthetic test dataset, test plan, traceability matrix, KPI definitions, sample evidence
/Operations/        — Operational runbook, release and change checklist
```

---

## Quick start

1. Read `Build-Guide/prerequisites_checklist.md` — ensure your Okta tenant meets all prerequisites
2. Follow `Build-Guide/build_guide.md` — build all flows in order (Utilities → Config → Helpers → ApprovalEngine → Orchestrators)
3. Import `Config-Templates/ilm_access_map.json` and `Config-Templates/ilm_system_class.json` adapted to your organisation
4. Run the pre-activation test cases from `Evaluation/test_plan.md` in your test tier
5. Promote to production following `Operations/release_checklist.md`



---

## Limitations and environment notes

- This artefact was built and validated in an Okta developer sandbox. No real personal data was used at any point.
- All synthetic identities in `Evaluation/synthetic_identities.json` are fictitious.
- Connection secrets (OAuth client credentials, ITSM API tokens) are NOT included in this bundle. Each deployment must create its own service application and store credentials as Workflows connection secrets.
- Licence reclamation (FR3.5) is designed correctly but requires a live SCIM provisioning connector to a Microsoft 365 tenant for end-to-end verification.

---

## Version history

| Version | Date | Notes |
|---|---|---|
| 1.0.0 | 2026-01-15 | Initial release — thesis submission |
