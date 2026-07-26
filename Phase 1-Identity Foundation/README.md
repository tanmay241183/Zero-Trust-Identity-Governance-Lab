# Phase 1 — Identity Foundation

> **Project:** Zero Trust Identity Governance Lab
> **Certification:** Microsoft SC-300 (Identity and Access Administrator)
> **Environment:** Microsoft Entra ID 
> **Status:** ✅ Complete

---

## Objective

Establish the base identity environment by provisioning a realistic set of users and groups, and configuring Self-Service Password Reset (SSPR) with appropriately scoped policies and phishing-resistant authentication methods.

This phase underpins all subsequent phases — every Conditional Access policy, PIM assignment, and access package in Phases 2–7 targets the users and groups created here.

---

## Environment

| Setting | Value |
|---|---|
| Tenant type | Microsoft Entra ID |
| Tenant domain | zerotrust-lab-[redacted].onmicrosoft.com |
| Admin account | Global Administrator |
| Region | Australia |

> **Note:** Sensitive values (Tenant ID, domain name) have been redacted in screenshots.

---

## What I Built

### 1B — Users & Groups

Created 7 simulated users representing a realistic organisational structure:

| Display Name | UPN | Job Title | Department | Group Membership |
|---|---|---|---|---|
| Alex Chen | alex.chen@… | IT Administrator | IT | SG-Admins |
| Priya Sharma | priya.sharma@… | Senior Developer | Engineering | SG-Developers, Project Alpha Team |
| Marcus O'Brien | marcus.obrien@… | Security Analyst | SOC | SG-SOC |
| Linda Wu | linda.wu@… | HR Manager | Human Resources | *(Phase 5 scope)* |
| James Foster | james.foster@… | Data Analyst | Finance | SG-Developers, Project Alpha Team |
| External Contractor | contractor.ext@… | Contractor | External | SG-Contractors |
| SVC Automation | svc.automation@… | Service Account | IT | *(Phase 7 scope)* |

Created 5 groups:

| Group Name | Type | Members | Purpose |
|---|---|---|---|
| SG-Admins | Security | Alex Chen | PIM scope (Phase 4) |
| SG-Developers | Security | Priya Sharma, James Foster | SSPR pilot, CA scope |
| SG-SOC | Security | Marcus O'Brien | SOC-specific policy scope |
| SG-Contractors | Security | External Contractor | Entitlement Management (Phase 5) |
| Project Alpha Team | Microsoft 365 | Priya Sharma, James Foster | Access Package resource (Phase 5) |

### 1C — Self-Service Password Reset (SSPR)

Configured and tested end-to-end SSPR for the SG-Developers group.

| Setting | Value | Rationale |
|---|---|---|
| SSPR enabled for | SG-Developers (Selected) | Controlled pilot rollout before org-wide enablement |
| Methods required to reset | 2 | Dual-method verification reduces risk of account takeover |
| Allowed methods | Microsoft Authenticator app, Email | Phishing-resistant and low SIM-swap risk |
| Excluded methods | SMS, Security questions | SMS vulnerable to SIM-swapping; questions guessable from OSINT |
| Require registration on sign-in | Yes | Ensures all in-scope users register before needing to reset |
| Re-confirmation frequency | 180 days | Balances security with user friction |
| Notify users on reset | Yes | Alerts user if account is reset without their knowledge |
| Notify admins on admin reset | Yes | Maintains audit trail for privileged account changes |

---

## Key Design Decisions

**Why scope SSPR to SG-Developers only?**
Microsoft's own deployment guidance recommends piloting SSPR with a non-privileged group before enabling org-wide. Scoping to SG-Developers allows validation of the registration and reset flow without exposing the entire tenant. In a production environment, this would be followed by a staged rollout tracked via the SSPR usage report.

**Why was SMS excluded as an authentication method?**
SMS-based verification is vulnerable to SIM-swapping attacks, where an attacker social-engineers a mobile carrier into transferring a victim's phone number. This is a well-documented attack vector used against high-value targets. Microsoft Authenticator app codes (TOTP) and email are retained as they do not rely on the carrier network.

**Why exclude Security Questions?**
Security question answers are frequently derivable from public social media profiles (pet names, schools, hometowns). They provide a false sense of security and are not considered a strong authentication factor under NIST SP 800-63B.

**Why use a Microsoft 365 Group for Project Alpha Team rather than a Security Group?**
Microsoft 365 Groups provision a shared mailbox, SharePoint site, and Teams workspace automatically. This is required for Entitlement Management access packages in Phase 5, where SharePoint site access is bundled into the package alongside group membership. Security Groups do not provision these collaboration resources.

---

## Evidence

| # | Screenshot | Description |
|---|---|---|
| 1 | `users-and-groups/04-users-list-all.png` | All 7 simulated users in Entra ID Users list |
| 2 | `users-and-groups/05-security-groups-list.png` | 4 Security Groups with correct membership types |
| 3 | `users-and-groups/06-m365-group-overview.png` | Project Alpha Team — Microsoft 365 Group type |
| 4 | `users-and-groups/07-sg-admins-members.png` | SG-Admins membership showing Alex Chen |
| 5 | `users-and-groups/08-org-structure-diagram.png` | User-to-group mapping diagram |
| 6 | `sspr/09-sspr-properties-selected-group.png` | SSPR scoped to SG-Developers |
| 7 | `sspr/10-sspr-auth-methods.png` | Authentication methods — Authenticator + Email only |
| 8 | `sspr/11-sspr-registration-settings.png` | Registration policy — require on sign-in, 180-day re-confirm |
| 9 | `sspr/12-sspr-registration-complete.png` | End-user registration confirmation as Priya Sharma |
| 10 | `sspr/13-sspr-reset-success.png` | Successful self-service password reset |
| 11 | `sspr/14-sspr-audit-log.png` | SSPR audit log showing reset event with timestamp and method |

![SSPR Audit Log](./sspr/14-sspr-audit-log.png)
*SSPR audit log confirming a successful self-service reset by Priya Sharma via Microsoft Authenticator.*

![Users List](./users-and-groups/04-users-list-all.png)
*All 7 simulated users provisioned in the ZeroTrustLabCo tenant.*

---

## SC-300 Domain Coverage

| SC-300 Objective | Covered in this phase |
|---|---|
| Implement and manage user identities | ✅ User creation, attributes, usage location |
| Implement and manage groups | ✅ Security Groups, M365 Groups, membership assignment |
| Implement authentication methods | ✅ SSPR auth method policy |
| Plan and implement SSPR | ✅ Scoped enablement, registration, end-to-end testing, audit logs |

---

## What I Learned

- **SSPR audit logs live under Password Reset, not the main Audit Logs blade.** This is easy to miss and worth knowing for exam scenarios that ask where to investigate a failed reset.
- **Microsoft 365 Groups vs Security Groups is a meaningful distinction for downstream resources.** You cannot use a Security Group as the basis for a SharePoint access package — the M365 Group is required.
- **Usage location must be set on users before assigning licenses.** If you skip this, license assignment will fail silently in some configurations.
- **Requiring two authentication methods for SSPR is more defensible than one**, even for a lab. It mirrors real-world policy and demonstrates understanding of layered verification.

---
