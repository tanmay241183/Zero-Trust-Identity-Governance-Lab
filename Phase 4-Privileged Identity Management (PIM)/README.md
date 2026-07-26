# Phase 4 — Privileged Identity Management (PIM)

> **Project:** Zero Trust Identity Governance Lab
> **Certification:** Microsoft SC-300 (Identity and Access Administrator)
> **Environment:** Microsoft Entra ID (M365 Developer Program Tenant)
> **Status:** ✅ Complete

---

## Objective

Implement Just-In-Time (JIT) privileged access for both Microsoft Entra ID roles and Azure resource roles using Privileged Identity Management — eliminating permanent privileged assignments and replacing them with time-bound, approval-gated, MFA-enforced eligible assignments with full audit logging.

This phase applies the Zero Trust principle of *least privilege* specifically to administrative identities — the highest-value targets in any organisation's identity environment.

---

## Environment

| Setting | Value |
|---|---|
| Tenant type | Microsoft Entra ID (M365 E5 Dev Program) |
| License required | Entra ID P2 (required for all PIM functionality) |
| Roles managed | Global Administrator, Security Administrator (Entra ID) + Owner (Azure subscription) |
| Users in scope | Alex Chen (SG-Admins), Marcus O'Brien (SG-SOC) |

---

## What I Built

### 4A — PIM Baseline

Onboarded Privileged Identity Management for the tenant and reviewed the starting state:

- All existing role assignments were permanent active assignments (no PIM management)
- Eligible assignments tab was empty — no JIT controls existed before this phase
- PIM dashboard confirmed identity governance scope: Entra ID roles + Azure resource roles

### 4B — Role Settings (Tiered Model)

Configured PIM role settings for two Entra ID roles before creating any assignments. Settings define the rules that govern every future activation of that role.

| Setting | Global Administrator | Security Administrator |
|---|---|---|
| Max activation duration | 1 hour | 4 hours |
| Require MFA on activation | ✅ Yes | ✅ Yes |
| Require justification | ✅ Yes | ✅ Yes |
| Require ticket number | ✅ Yes | ❌ No |
| Require approval | ✅ Yes | ❌ No (self-approve) |
| Allow permanent eligible | ❌ No | ❌ No |
| Eligible assignment expiry | 365 days | 180 days |
| Allow permanent active | ❌ No | ❌ No |

**Design intent:** Global Administrator carries the highest blast radius of any Entra ID role — full tenant control, ability to remove all other admins, access to all data. It therefore requires the most friction: approval from a second person, maximum 1-hour window, MFA re-challenge, and a change ticket. Security Administrator (read access to security signals, ability to manage security policies) warrants strong controls but allows self-approval with a 4-hour window to support SOC analyst workflows where raising an approval request mid-incident is operationally impractical.

### 4C — Eligible Assignments

| User | Role | Assignment Type | Expiry | Can Self-Approve |
|---|---|---|---|---|
| Alex Chen | Global Administrator | Eligible | 365 days | ❌ No — requires approval |
| Marcus O'Brien | Security Administrator | Eligible | 180 days | ✅ Yes |

Neither user holds their privileged role at rest. Both must actively request activation through PIM, satisfy MFA, and provide a justification. Alex's requests additionally require approval from a designated approver before the role becomes active.

### 4D — JIT Activation Lifecycle (End-to-End Test)

Executed a complete activation lifecycle for Alex Chen's Global Administrator role across two browser sessions (requestor and approver).

**Lifecycle steps executed and evidenced:**

| Step | Actor | Action | Outcome |
|---|---|---|---|
| 1 | Admin | Created eligible assignment | Alex Chen → Global Admin, Eligible |
| 2 | Alex Chen | Requested activation (1 hr, with justification + ticket INC-20250718) | Status: Pending approval |
| 3 | PIM system | Sent approval notification to designated approver | Email notification triggered |
| 4 | Admin (approver) | Reviewed request, approved with justification | Role activated immediately |
| 5 | Alex Chen | Confirmed role active with 1-hour expiry countdown | Global Admin active |
| 6 | Alex Chen | Deactivated role early after task completion | Role returned to Eligible |
| 7 | PIM system | Wrote complete audit trail to Entra roles audit log | All 5 events logged with timestamps |

### 4E — PIM for Azure Resources

Extended PIM governance to the Azure subscription layer:

| Setting | Value |
|---|---|
| Resource onboarded | Azure subscription (Free tier) |
| Role managed | Owner |
| Eligible user | Alex Chen |
| Activation max duration | 1 hour |
| Approval required | ✅ Yes |
| MFA on activation | ✅ Yes |

This demonstrates that PIM is not limited to Entra ID directory roles — it extends to Azure RBAC, providing unified JIT governance across both identity and infrastructure.

---

## Key Design Decisions

**Why eliminate permanent privileged role assignments?**
A permanent Global Administrator assignment means the role is active 24/7, regardless of whether it is needed. If the account's credentials are compromised — through phishing, credential stuffing, or token theft — the attacker immediately holds full tenant admin rights with no time constraint. An eligible-only assignment limits the exposure window to the activation period (maximum 1 hour in this lab). The attacker must also defeat the approval workflow and MFA re-challenge at activation time, not just at initial sign-in.

**Why require MFA at activation rather than just at sign-in?**
Conditional Access MFA (Phase 3) verifies identity at sign-in. PIM's MFA-on-activation requirement is a separate, additional challenge that fires specifically at the moment of privilege escalation. This protects against token hijacking scenarios where an attacker captures a valid session token after the initial sign-in MFA — the captured token satisfies CA, but the attacker cannot complete the PIM activation challenge without the user's MFA device.

**Why use a tiered model between Global Admin and Security Admin settings?**
Uniform controls applied to all privileged roles create unnecessary friction for lower-risk roles and may cause analysts to work around PIM rather than through it. A tiered model calibrates friction to risk: Global Admin (full tenant control) gets maximum controls including approval and a 1-hour cap. Security Administrator (read-heavy, investigation-focused) gets self-approval and a 4-hour window that accommodates incident response timelines. The friction is proportional to the blast radius.

**Why require a ticket number for Global Administrator activations?**
Requiring a ticket number integrates the PIM activation event with a change management record. In a production environment this would link to a ServiceNow or Jira ticket, making every privileged access event traceable to an approved change. This is directly relevant to ISO 27001 A.9 (Access Control) and the ACSC Essential Eight's requirement to log and audit privileged access. In this lab the ticket number is simulated (INC-20250718), but the field appears in the PIM audit log alongside the activation event — creating a human-readable audit record.

**Why extend PIM to Azure resource roles, not just Entra ID roles?**
The Owner role on an Azure subscription grants full control over all resources in that subscription — create, modify, delete VMs, storage accounts, databases, networking, and more. It is equally dangerous to a permanent Global Administrator assignment. Extending PIM to Azure resources means a single governance framework covers both the identity plane (Entra ID) and the infrastructure plane (Azure), rather than having JIT controls on directory roles but permanent access to cloud infrastructure.

**What is the correct role for the break-glass account in a PIM environment?**
The break-glass account created in Phase 3 must have a **permanent active** Global Administrator assignment — not an eligible one. If PIM itself experienced a service disruption or misconfiguration, an eligible-only break-glass account could not activate its role and would fail at the exact moment it was most needed. The break-glass account is excluded from all CA policies (Phase 3) and holds permanent admin specifically to provide a recovery path that bypasses all normal controls. Its use should be monitored and any sign-in should trigger an immediate security alert.

**How does this align with ACSC Essential Eight and ISO 27001?**

| Framework | Control | How Phase 4 addresses it |
|---|---|---|
| ACSC Essential Eight | Restrict administrative privileges | Eligible-only assignments eliminate persistent admin access |
| ACSC Essential Eight | Multi-factor authentication (ML2+) | MFA required on every PIM activation, separate from sign-in MFA |
| ISO 27001 A.9.2.3 | Management of privileged access rights | PIM eligible assignments with time limits and approval workflows |
| ISO 27001 A.9.4.2 | Secure log-on procedures | MFA + justification + approval enforced at activation |
| ISO 27001 A.12.4 | Logging and monitoring | Full PIM audit log covering every lifecycle event |

---

## JIT Activation Lifecycle

```
[Eligible Assignment]
│
▼
[User Requests Activation] ←── justification + ticket + MFA
│
▼
[Approval Request Sent] ──► approver receives notification
│
▼
[Approver Reviews & Approves] ←── approver provides justification
│
▼
[Role Active — Time-Bound] ──► 1-hour countdown begins
│
▼
[User Deactivates Early / Role Expires]
│
▼
[Audit Log — All Events Recorded]
│
▼
[Role Returns to Eligible]
```

At no point in this lifecycle does the user hold the privileged role permanently. The role exists in Eligible state by default and transitions to Active only for the approved activation window.

---

## Evidence

| # | Screenshot | Description |
|---|---|---|
| 1 | `01-pim-overview-dashboard.png` | PIM overview dashboard — navigation and opening state |
| 2 | `entra-roles/02-pim-entra-roles-list.png` | Entra ID roles available in PIM |
| 3 | `entra-roles/03-pim-assignments-baseline.png` | Baseline — empty eligible assignments before configuration |
| 4 | `entra-roles/04-pim-global-admin-role-settings.png` | Global Admin role settings — 1hr, MFA, approval, ticket required |
| 5 | `entra-roles/05-pim-security-admin-role-settings.png` | Security Admin role settings — 4hr, MFA, self-approve |
| 6 | `entra-roles/06-pim-eligible-assignment-globaladmin.png` | Alex Chen eligible assignment creation — Global Admin |
| 7 | `entra-roles/07-pim-eligible-assignments-list.png` | Both eligible assignments created — Alex (Global Admin) + Marcus (Security Admin) |
| 8 | `activation-workflow/08-pim-activation-request-submitted.png` | Alex Chen's activation request form — justification and ticket visible |
| 9 | `activation-workflow/09-pim-approval-pending-request.png` | Approver view — pending request with justification visible |
| 10 | `activation-workflow/10-pim-role-active-with-expiry.png` | Role active under Alex's My roles — expiry countdown visible |
| 11 | `activation-workflow/11-pim-role-deactivated.png` | Role returned to Eligible after early deactivation |
| 12 | `activation-workflow/12-pim-audit-log-full-lifecycle.png` | **Complete PIM audit trail** — all lifecycle events with timestamps |
| 13 | `azure-resources/13-pim-azure-resource-onboarded.png` | Azure subscription onboarded into PIM |
| 14 | `azure-resources/14-pim-azure-owner-role-settings.png` | Azure Owner role settings — mirroring Global Admin controls |
| 15 | `azure-resources/15-pim-azure-eligible-assignments.png` | Alex Chen eligible for Owner on Azure subscription |

![PIM Audit Log — Full Lifecycle](./activation-workflow/12-pim-audit-log-full-lifecycle.png)
*Complete PIM audit trail for Alex Chen's Global Administrator activation — request, approval, activation, and deactivation all captured with timestamps and actor details.*

![PIM Role Active with Expiry](./activation-workflow/10-pim-role-active-with-expiry.png)
*Global Administrator active in Alex Chen's My roles view — time-bound expiry countdown confirms JIT enforcement.*

![Global Admin Role Settings](./entra-roles/04-pim-global-admin-role-settings.png)
*Global Administrator activation settings — 1-hour maximum, MFA required, approval required, justification and ticket enforced.*

---

## SC-300 Domain Coverage

| SC-300 Objective | Covered in this phase |
|---|---|
| Plan and implement Privileged Identity Management | ✅ PIM onboarding, role settings, eligible assignments for Entra ID and Azure roles |
| Configure PIM role settings | ✅ Tiered settings across two roles — approval, MFA, duration, ticket |
| Manage PIM requests and approvals | ✅ Full approval workflow — request, pending, approve, activate |
| Monitor privileged access via PIM audit logs | ✅ Complete lifecycle audit log exported and documented |
| Implement emergency access accounts in a PIM environment | ✅ Break-glass account holds permanent active assignment, excluded from all policies |

---

## What I Learned

- **PIM role settings must be configured before assignments are created.** Settings apply to all future activations — if you create an eligible assignment first and then update role settings, the new settings apply from that point forward but the assignment itself doesn't need to be recreated. However, establishing the governance rules before provisioning access is the correct operational sequence.
- **The approver view in PIM is a separate workflow that many lab walkthroughs skip entirely.** Capturing it from the approver's perspective (PIM → Tasks → Approve requests) is more valuable than the requestor view alone — it shows you understand the full two-party control.
- **PIM audit logs are under PIM → Activity, not under Entra ID → Monitoring → Audit logs.** These are different systems. The Entra ID audit log covers directory operations broadly. The PIM audit log is scoped to privileged role lifecycle events and includes PIM-specific fields like activation duration, justification text, and ticket number that don't appear in the general log.
- **Eligible assignments have their own expiry distinct from activation duration.** The eligible assignment expiry (365 days for Global Admin in this lab) is when the user loses the right to request the role at all. The activation duration (1 hour) is how long each individual activation lasts. These are two separate time controls — easy to conflate in exam questions.
- **PIM for Azure resources is discovered, not automatically inherited.** When you onboard a new Azure subscription, it does not automatically appear in PIM. You must explicitly discover and onboard it via PIM → Manage → Azure resources → Discover resources. In large organisations with many subscriptions, this onboarding step is an ongoing operational task, often automated via policy or management groups.

---

## Previous & Next Phase

⬅️ [Phase 3 — Conditional Access Policies](../phase-3/README.md)

➡️ [Phase 5 — Entitlement Management & Access Packages](../phase-5/README.md)

Build a structured access request and approval workflow using Entitlement Management — creating catalogs, access packages, and policies that automate provisioning and time-limited access for new joiners and contractors.

---

*Part of the [Zero Trust Identity Governance Lab](../README.md) — an SC-300 portfolio project built in Microsoft Entra ID.*
