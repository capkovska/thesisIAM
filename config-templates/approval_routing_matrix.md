# Approval Routing Matrix

**Version:** 1.0.0

This matrix maps system sensitivity tier to the required approval configuration. The `approvalTier` field in `ilm_system_class.json` must be set to one of the values in column 1.

---

## Approval tiers

| approvalTier value | Approvers required | Approval SLA | Escalation SLA | Notes |
|---|---|---|---|---|
| `none` | None — automatic | — | — | Baseline access, low-risk automated systems |
| `manager-only` | Direct manager | 24 hours | +24 hours to manager's manager | Default for Medium sensitivity |
| `manager-and-owner` | Manager + Application owner | 48 hours | +24 hours each | High sensitivity systems |
| `manager-and-security` | Manager + Security ops | 4 hours | Immediate escalation to CISO | Critical / privileged access |

---

## Sensitivity tier → approval tier mapping

| Sensitivity tier | Default approvalTier | Override allowed? |
|---|---|---|
| Low | `none` | Yes — can require `manager-only` |
| Medium | `manager-only` | Yes — can increase to `manager-and-owner` |
| High | `manager-and-owner` | Yes — can increase to `manager-and-security` |
| Critical | `manager-and-security` | No — minimum enforced |

---

## Special cases

| Scenario | approvalTier | Additional controls |
|---|---|---|
| `APP-*-Admin` group assignment | `manager-and-security` | Mandatory time-limit (max 90 days); auto-revoke at expiry |
| Break-glass access | `manager-and-security` | Max 4-hour expiry; mandatory risk acceptance; enhanced logging |
| VIP employee | Any tier + | Manager confirmation required before ANY access change including baseline |
| Contractor onboarding | As per system class | Expiry date required; leaver scheduled automatically |

---

## Escalation policy per tier

| approvalTier | SLA expires with no response | Escalation action |
|---|---|---|
| `manager-only` | 24 hours | Route to manager's manager; reset 24-hour clock |
| `manager-and-owner` | 48 hours | Route each pending approver to their manager; reset 24-hour clock |
| `manager-and-security` | 4 hours | Immediate CISO notification; policy choice: escalate or auto-reject |

The escalation policy (escalate vs. auto-reject) is configured per `approvalTier` in `ilm_system_class` and can be set globally or per system.
