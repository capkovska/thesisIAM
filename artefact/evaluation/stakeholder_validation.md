# Stakeholder Validation

**Version:** 1.0.0
**Validation type:** Structured practitioner review
**Number of reviewers:** 2
**Format:** Pre-read of artefact bundle + structured question session based on success criteria

---

## Overview

In addition to the structured technical evaluation, two practitioners with direct experience in enterprise IAM and identity automation reviewed the artefact design, test results, and evidence outputs. This validation addresses SC6 (operational readiness) and the broader DSR question of whether the artefact is useful and consistent with real-world practice.

Both practitioners had previously contributed to the requirements elicitation phase of this research (workshops and semi-structured interviews). This is acknowledged as a potential source of confirmation bias: practitioners who helped shape the requirements may be more inclined to judge the artefact favourably against those same requirements. This limitation is noted in the evaluation conclusions.

---

## Reviewer profiles

**Reviewer 1 - Senior IAM Architect**
Extensive experience designing and implementing enterprise IAM programs across multiple organizations. Familiar with Okta, SailPoint, and Azure AD. Has led identity lifecycle automation projects and IAM audit engagements.

**Reviewer 2 - IT Specialist **
Responsible for access governance processes in a company using ITSM-based approval workflows. Experience with ServiceNow-based provisioning request handling and manual deprovisioning processes. Not an IAM specialist but a direct end-user of IAM governance outputs.

---

## Session structure

Before the review session, both practitioners were provided with:
- The full artefact bundle (all files in this repository)
- A summary of the evaluation results (traceability matrix verdicts and performance table)
- The sample evidence outputs (`Evaluation/sample_evidence_outputs.json`)

Feedback was gathered using a structured question set mapped to the six success criteria. The questions and key responses are documented below.

---

## SC1 and SC2 - Functional completeness and mixed-trigger support

**Question:** Does the JML coverage address the operational problems you encounter or have encountered in practice? Is the dual-trigger model (HRIS + ITSM) realistic?

**Reviewer 1 (IAM Architect):**
Both the coverage and the dual-trigger model were confirmed as realistic. The architect noted that in practice, no enterprise achieves 100% HRIS automation coverage - there are always edge cases that require an ITSM-initiated path. The quote recorded during the session:

> "Organizations rarely achieve 100% HRIS automation coverage. The ITSM exception path is not a workaround, it is a first-class requirement."

The functional scope was judged as appropriate for the described environment.

**Reviewer 2 (IT Specialist):**
Agreed that the JML coverage maps to the most common operational problems. Specifically highlighted that the manual ITSM task path for non-integrated applications reflects real-world conditions in most mid-sized organizations, where not all applications have a provisioning connector.

---

## SC3 - Approval enforcement

**Question:** Does the tiered approval model and the asynchronous approval pattern address real-world approval workflow problems?

**Reviewer 1 (IAM Architect):**
Confirmed that the tiered sensitivity model (`manager-only`, `manager-and-owner`, `manager-and-security`) reflects standard IAM governance practice. Noted that the Critical tier enforcement with no override option is the correct design for privileged access.

**Reviewer 2 (IT Specialist):**
Strongly supported the non-blocking asynchronous approval pattern (SH-1). Explained from direct operational experience that synchronous approval gates - where the entire provisioning process halts until an approval is received - are a frequent failure mode in approval-gated provisioning systems. When approval response times are slow or approvers are unavailable, synchronous gates cause transaction backlogs that are then cleared manually, defeating the purpose of automation. The asynchronous SH-1 design, where the workflow parks itself and resumes on callback, was identified as a direct and effective solution to this failure mode.

---

## SC4 - Evidence generation

**Question:** Would the evidence records produced by the artefact meet the requirements of a typical IAM audit request?

**Reviewer 1 (IAM Architect):**
Reviewed the TC-J-01 and TC-L-01 evidence records from `Evaluation/sample_evidence_outputs.json`. Confirmed that the fields in the `ilm_tx_log` row and the `ilm_evidence_archive` entry would likely satisfy a typical IAM audit request covering who was provisioned, what access was granted, who approved it, and when.

One improvement was suggested: adding the full group membership state (not just the access delta) to the leaver evidence record. This would allow an auditor to confirm what access the user held immediately before deprovisioning, without needing to reconstruct it from the `groupsRemoved` delta and a prior transaction record. This suggestion has been recorded as a candidate improvement for artefact version 1.1.0.

**Reviewer 2 (IT Specialist):**
No specific feedback on evidence format. Confirmed that the ITSM task records generated by SH-6 contain sufficient information for application owners to complete deprovisioning tasks without needing to contact the IAM team for context.

---

## SC5 - Integrated and manual target handling

**Question:** Is the split between automated (SCIM/AD) and manual (ITSM task) execution paths appropriate and practical?

**Reviewer 1 (IAM Architect):**
Confirmed that the three-way integration model (SCIM, AD group-based, manual ITSM task) covers the integration patterns present in most enterprise environments. Noted that the ability to add a new manual system to `ilm_system_class` without modifying any flow logic is a significant practical advantage over hardcoded workflow implementations.

**Reviewer 2 (IT Specialist):**
Confirmed from direct experience that manual deprovisioning tasks are unavoidable in environments with legacy applications. The structured ITSM task format - with a clear due date, application owner assignment, and transaction ID - was identified as a material improvement over the informal email-based notification approach used in typical as-is processes.

---

## SC6 - Operational readiness and LCNC accessibility

**Question:** Is the artefact operationally complete? Could an IAM practitioner without software development expertise deploy and operate it?

**Reviewer 1 (IAM Architect):**
Reviewed the build guide and confirmed that the step-by-step structure is sufficient for a practitioner with Okta administration experience. Noted that the five-folder structure and consistent naming conventions make the flow set navigable without requiring familiarity with the full design specification.

**Reviewer 2 (IT Specialist):**
Reviewed the operational runbook. Confirmed it is sufficient for L1 and L2 operations - routine monitoring, task follow-up, and standard error recovery. Specifically noted that the intervention queue recovery procedure is clearly described and that the table-driven configuration model significantly reduces the L3 change burden compared to hardcoded workflow implementations, where a new department or system typically requires a developer to modify flow logic.

**On LCNC accessibility (both reviewers):**
Both practitioners were asked directly whether someone without programming knowledge could deploy and operate the artefact using the build guide and configuration templates. Both agreed that the artefact is accessible to IAM practitioners with system administration experience who are not software developers. This directly supports the core argument of the thesis regarding the practical utility of LCNC tooling for workforce ILM.

---

## Shared limitation identified

Both practitioners independently identified the same limitation: the absence of a genuine end-to-end SCIM integration test introduces uncertainty about the behaviour of the SCIM provisioning connector in complex deprovisioning scenarios - specifically regarding the consistent application of soft-delete semantics across different SaaS vendors. This is acknowledged as an evaluation limitation in the thesis (Chapter 5.7) and in the paper. It does not affect the correctness of the artefact design; it reflects an environmental constraint of the sandbox evaluation.

---

## Summary

| Success criterion | Reviewer 1 verdict | Reviewer 2 verdict |
|---|---|---|
| SC1 - JML functional coverage | Appropriate for described environment | Covers most common operational problems |
| SC2 - Mixed-trigger support | Realistic; ITSM path is first-class requirement | Reflects real-world mixed environments |
| SC3 - Approval enforcement | Tiered model correct; async pattern well-designed | Async pattern solves known synchronous bottleneck |
| SC4 - Evidence generation | Likely meets audit requirements; v1.1.0 suggestion noted | ITSM task format sufficient for app owners |
| SC5 - Integrated + manual execution | Config-driven manual system addition is practical advantage | Structured ITSM tasks improve on email-based as-is |
| SC6 - Operational readiness and LCNC accessibility | Build guide sufficient for Okta admins; runbook covers L1/L2 | Table-driven config reduces L3 burden vs hardcoded implementations |

Both practitioners confirmed that the artefact is consistent with enterprise IAM practice and accessible to the intended non-developer audience. The primary shared limitation identified - absence of a live SCIM integration test - is environmental rather than architectural.

---

## References

- Sample evidence outputs: `Evaluation/sample_evidence_outputs.json`
- Test plan: `Evaluation/test_plan.md`
- Requirements traceability: `Evaluation/requirements_traceability_matrix.md`
- Thesis: Chapter 5.6 - Stakeholder validation
