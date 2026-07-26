# Zero Trust Identity Governance Lab

> **Certification:** Microsoft SC-300 — Identity and Access Administrator  
> **Platform:** Microsoft Entra ID (M365 E5 Developer Program Tenant)  
> **Status:** ✅ All 7 phases complete  
> **Estimated build time:** 14–20 hours

A hands-on identity governance lab built from scratch in Microsoft Entra ID, covering all four SC-300 exam domains across seven sequential phases — from tenant population and MFA hardening through to application registration and OAuth 2.0 flow testing.

Every phase includes portal configuration, end-to-end testing, and audit log evidence. The project is designed to demonstrate practical IAM skills to hiring managers, not just exam knowledge.

---

## Why This Project Exists

Most SC-300 candidates study the theory. This lab demonstrates the practice.

Hiring managers at MSSPs, banks, and super funds reviewing IAM Analyst applications see hundreds of CVs listing Microsoft certifications. What separates candidates is evidence of hands-on work: real configurations, real audit trails, real design decisions with documented rationale.

This project covers every major SC-300 domain in a single connected architecture — the same users and groups created in Phase 1 appear in Conditional Access policies in Phase 3, PIM assignments in Phase 4, access packages in Phase 5, access reviews in Phase 6, and a Graph API response in Phase 7. That cross-phase continuity is what makes this a portfolio piece rather than a collection of isolated labs.

---

## Prerequisites

### Licences & Accounts

| Requirement | Cost | Notes |
|---|---|---|
| Microsoft 365 Developer Program | Free | Provides an E5 tenant with 25 users and full Entra ID P2 — covers all phases. Sign up at developer.microsoft.com/microsoft-365/dev-program |
| Azure Free Account | Free | Required for Phase 4 (PIM for Azure resources). $200 credit, 12 months free services. portal.azure.com |
| Postman | Free | Required for Phase 7 OAuth 2.0 client credentials flow testing. postman.com |
| Tor Browser | Free | Required for Phase 3 risk event simulation. torproject.org |

### Licences Required by Phase

| Phase | Free Tier | Entra ID P2 |
|---|---|---|
| Phase 1 — Users & Groups | ✅ | — |
| Phase 2 — MFA Hardening | ✅ | — |
| Phase 3 — Conditional Access | ✅ (CA001, CA003, CA004) | ✅ CA002 (Identity Protection) |
| Phase 4 — PIM | — | ✅ All PIM features |
| Phase 5 — Entitlement Management | — | ✅ All EM features |
| Phase 6 — Access Reviews | — | ✅ All review features |
| Phase 7 — App Registration | ✅ | — |

> The M365 Developer Program E5 tenant includes Entra ID P2, covering all P2-dependent phases at no cost.

### Tools & Portals

| Tool | Purpose | URL |
|---|---|---|
| Microsoft Entra Admin Centre | Primary identity administration portal | entra.microsoft.com |
| Azure Portal | Azure resource management and PIM for Azure | portal.azure.com |
| My Access Portal | End-user self-service for access packages and reviews | myaccess.microsoft.com |
| Microsoft Graph Explorer | Interactive Graph API testing | developer.microsoft.com/graph/graph-explorer |
| jwt.ms | JWT access token decoder | jwt.ms |
| Postman | HTTP client for OAuth 2.0 flow testing | postman.com |
| draw.io | Architecture diagram creation | app.diagrams.net |

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                   ZeroTrustLabCo Tenant                         │
│                                                                 │
│  ┌─────────────────┐    ┌──────────────────────────────────┐   │
│  │   IDENTITIES    │    │      AUTHENTICATION LAYER         │   │
│  │  Phase 1        │    │      Phase 2 + 3                  │   │
│  │                 │    │                                   │   │
│  │  7 Users        │───▶│  Auth Methods Policy              │   │
│  │  4 Sec Groups   │    │  2 Custom Auth Strengths          │   │
│  │  1 M365 Group   │    │  2 Named Locations                │   │
│  │  SSPR (SG-Dev)  │    │  4 Conditional Access Policies    │   │
│  └─────────────────┘    └──────────────────────────────────┘   │
│                                          │                      │
│  ┌─────────────────┐    ┌───────────────▼──────────────────┐   │
│  │  GOVERNANCE     │    │    PRIVILEGED ACCESS LAYER        │   │
│  │  Phase 6        │    │    Phase 4                        │   │
│  │                 │    │                                   │   │
│  │  Group Review   │    │  PIM — Entra ID Roles             │   │
│  │  Role Review    │    │  PIM — Azure Resources            │   │
│  │  Package Review │◀───│  JIT + Approval + MFA             │   │
│  │  Auto-remediate │    │  Audit Logging                    │   │
│  └─────────────────┘    └──────────────────────────────────┘   │
│                                                                 │
│  ┌─────────────────┐    ┌──────────────────────────────────┐   │
│  │  APPLICATIONS   │    │    ENTITLEMENT LAYER              │   │
│  │  Phase 7        │    │    Phase 5                        │   │
│  │                 │    │                                   │   │
│  │  App Reg        │    │  Catalog: Project Alpha           │   │
│  │  OAuth 2.0      │    │  Package: Developer Onboarding    │   │
│  │  App Roles RBAC │    │  Package: Contractor Access       │   │
│  │  Graph API      │    │  My Access Portal                 │   │
│  └─────────────────┘    └──────────────────────────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## Phase Summary

### [Phase 1 — Tenant & Identity Foundation](./phase-1/README.md)

> **Est. time:** 2–3 hrs | **Licence:** Free | **Screenshots:** 11

Populated the tenant with a realistic organisational structure — 7 simulated users across IT, Engineering, SOC, HR, Finance, and External roles, assigned to 4 Security Groups and 1 Microsoft 365 Group. Configured Self-Service Password Reset scoped to SG-Developers with phishing-resistant authentication methods (Authenticator app + email, SMS and security questions excluded). Tested the full SSPR registration and reset flow end-to-end, captured the audit log as evidence.

**Key outputs:** User population, group structure, SSPR policy, audit log evidence  
**Zero Trust principle:** Verify explicitly — every user has a defined identity and authentication requirement

---

### [Phase 2 — Authentication & MFA Hardening](./phase-2/README.md)

> **Est. time:** 2–3 hrs | **Licence:** Free | **Screenshots:** 13

Configured the tenant's authentication layer using the modern Authentication Methods Policy blade. Enabled Microsoft Authenticator with number matching (MFA fatigue prevention), configured Temporary Access Pass scoped to SG-Admins for break-glass and onboarding scenarios. Built a tiered authentication strength model: *Privileged Admin Auth* (FIDO2 and CBA only) for administrators and *Developer Standard Auth* (Authenticator + FIDO2) for developers. Defined two Named Locations: a trusted IP-based office network and a country-based high-risk block list.

**Key outputs:** Auth method policy, 2 custom auth strengths, 2 named locations, MFA registration evidence  
**Zero Trust principle:** Verify explicitly — authentication method quality is calibrated to user risk tier

---

### [Phase 3 — Conditional Access Policies](./phase-3/README.md)

> **Est. time:** 3–4 hrs | **Licence:** Free + P2 (CA002) | **Screenshots:** 15

Deployed 4 production-grade Conditional Access policies, each starting in Report-only mode with sign-in log validation before enforcement. Created a break-glass emergency access account excluded from all policies. Simulated a high-risk sign-in using Tor Browser to generate an Identity Protection risk event, then confirmed the block. Left CA003 (phishing-resistant auth for admins) in Report-only — the correct deployment posture pending FIDO2 provisioning — and documented the *Would have failed* sign-in log result as architectural evidence.

| Policy | Condition | Grant | State |
|---|---|---|---|
| CA001 — Require MFA for All Users | None | Developer Standard Auth strength | ✅ On |
| CA002 — Block High-Risk Sign-Ins | Sign-in risk: High, Medium | Block | ✅ On |
| CA003 — Phishing-Resistant Auth for Admins | None (SG-Admins scope) | Privileged Admin Auth strength | 🔶 Report-only |
| CA004 — Block Legacy Authentication | Client apps: EAS + Other | Block | ✅ On |

**Key outputs:** 4 CA policies, Tor risk event, audit evidence, break-glass account  
**Zero Trust principle:** Verify explicitly — every sign-in evaluated against risk, location, and auth strength

---

### [Phase 4 — Privileged Identity Management](./phase-4/README.md)

> **Est. time:** 2–3 hrs | **Licence:** P2 | **Screenshots:** 15

Implemented Just-In-Time privileged access for Entra ID roles and Azure resource roles, eliminating permanent admin assignments. Configured tiered role settings: Global Administrator (1-hour max, approval required, MFA, ticket number) and Security Administrator (4-hour max, self-approve, MFA). Executed a complete JIT lifecycle across two browser sessions — eligible assignment → activation request → approval → time-bound active role → early deactivation → audit log. Extended PIM governance to the Azure subscription Owner role.

**Key JIT lifecycle evidence:**

```
Eligible Assignment → Activation Request → Pending Approval
       → Approved → Active (1hr countdown) → Deactivated → Audit Log
```

**Key outputs:** PIM role settings, eligible assignments, full activation lifecycle, PIM audit log  
**Zero Trust principle:** Least privilege — no persistent admin access; every elevation is time-bound, approved, and logged

---

### [Phase 5 — Entitlement Management & Access Packages](./phase-5/README.md)

> **Est. time:** 2–3 hrs | **Licence:** P2 | **Screenshots:** 17

Built a self-service access governance system using a Project Alpha Resources catalog with Linda Wu (HR Manager) as delegated catalog owner. Created two access packages with deliberately different governance postures: *New Developer Onboarding* (permanent access, manager approval, bundling M365 Group + Security Group) and *Contractor Project Access* (90-day expiry, narrower resource scope). Tested the full request-to-provisioning workflow via the My Access portal as both the requestor (Priya Sharma) and approver (Linda Wu). Confirmed both group memberships provisioned automatically from a single approval.

**Key outputs:** Catalog, 2 access packages, My Access portal workflow, audit log, 90-day contractor assignment  
**Zero Trust principle:** Least privilege — access is time-bound, scoped by user type, and requires documented justification

---

### [Phase 6 — Access Reviews & Governance](./phase-6/README.md)

> **Est. time:** 2–3 hrs | **Licence:** P2 | **Screenshots:** 11

Deployed three distinct access review types targeting every layer of the identity stack:

| Review | Type | Outcome |
|---|---|---|
| SG-Admins Membership | Group — Quarterly | ✅ Approved with justification |
| Security Admin Eligible Assignment | Entra ID Role — 6-monthly | ❌ Denied — auto-removed from PIM |
| New Developer Onboarding | Access Package — Annual | ⏳ Pending annual cycle |

The Security Administrator denial demonstrated auto-apply: Marcus O'Brien's eligible assignment was automatically removed from PIM without any additional admin action. The removed assignment was confirmed in PIM → Eligible assignments — closing the loop from Phase 4.

**Key outputs:** 3 access reviews, denial with auto-removal, PIM assignment confirmed removed  
**Zero Trust principle:** Assume breach — existing access is periodically challenged, not assumed valid indefinitely

---

### [Phase 7 — App Registration & OAuth 2.0](./phase-7/README.md)

> **Est. time:** 2–3 hrs | **Licence:** Free | **Screenshots:** 18

Registered *ZeroTrustLabApp* as a single-tenant application and configured delegated permissions (User.Read, GroupMember.Read.All, Mail.Send) and application permissions (User.Read.All, Group.Read.All, AuditLog.Read.All) with admin consent. Defined two custom app roles (Analytics.Read, Data.Write) and assigned them via group (SG-Developers) and direct user (Priya Sharma) provisioning patterns. Tested the Authorization Code flow (decoded JWT showing `scp` and `roles` claims) and Client Credentials flow (decoded JWT showing `idtyp=app`, no `scp`, application permission `roles`). Called Graph API with the application token and received a 200 OK response listing Phase 1 groups. Enabled `Assignment Required = Yes` and confirmed unassigned users are blocked with AADSTS50105.

**Key outputs:** App registration, API permissions, app roles, client credentials token, Graph API call, RBAC enforcement  
**Zero Trust principle:** Verify explicitly + Least privilege — application identity verified via OAuth, access scoped to minimum permissions

---

## SC-300 Domain Coverage

| SC-300 Exam Domain | Weight | Phases |
|---|---|---|
| Implement and manage user identities | ~25% | Phase 1, 2 |
| Implement authentication and access management | ~25% | Phase 2, 3 |
| Plan and implement workload identities | ~20% | Phase 7 |
| Plan and implement identity governance | ~30% | Phase 4, 5, 6 |

**Detailed objective coverage:**

| SC-300 Objective | Phase |
|---|---|
| Implement and manage users and groups | 1 |
| Implement and manage SSPR | 1 |
| Plan and implement MFA and authentication methods | 2 |
| Implement authentication strengths | 2 |
| Plan and implement Conditional Access | 3 |
| Implement Identity Protection risk policies | 3 |
| Plan and implement PIM for Entra ID roles | 4 |
| Plan and implement PIM for Azure resources | 4 |
| Plan and implement entitlement management | 5 |
| Manage access packages and policies | 5 |
| Plan and implement access reviews | 6 |
| Plan and implement lifecycle workflows | 6 |
| Register and manage applications | 7 |
| Implement API permissions and admin consent | 7 |
| Implement OAuth 2.0 authorisation flows | 7 |

---

## Framework Alignment

### ACSC Essential Eight

| Essential Eight Control | Phase | Evidence |
|---|---|---|
| Multi-factor authentication (ML2) | 2, 3 | Auth methods policy, CA001 enforcement |
| Multi-factor authentication (ML3) | 2, 3 | Phishing-resistant auth strength, CA003 (report-only pending FIDO2) |
| Restrict administrative privileges | 4, 6 | PIM eligible-only assignments, quarterly SG-Admins review |
| Application control | 7 | Assignment Required = Yes, AADSTS50105 block |
| Patch applications / User application hardening | 3 | CA004 blocking legacy authentication protocols |

### ISO 27001

| Control | Phase | Evidence |
|---|---|---|
| A.9.1.2 — Access to networks and services | 3 | CA policies enforcing MFA and blocking risky/legacy auth |
| A.9.2.1 — User registration and de-registration | 1, 5 | User provisioning, access package request workflow with justification |
| A.9.2.3 — Management of privileged access | 4 | PIM JIT with approval, time limits, and MFA on activation |
| A.9.2.5 — Review of user access rights | 6 | Three access reviews — group, role, and package |
| A.9.2.6 — Removal of access rights | 6 | Auto-apply with Remove access on no response |
| A.9.4.2 — Secure log-on procedures | 2, 3, 4 | MFA enforcement, CA policies, PIM activation MFA |
| A.12.4 — Logging and monitoring | 3, 4, 5, 6 | CA sign-in logs, PIM audit log, entitlement audit log, review results |

---

## Repository Structure

```
entra-ztlab/
│
├── README.md                          ← You are here
│
├── phase-1/                           ← Users, Groups & SSPR
│   ├── README.md
│   ├── users-and-groups/
│   └── sspr/
│
├── phase-2/                           ← MFA & Auth Hardening
│   ├── README.md
│   ├── mfa/
│   ├── auth-strengths/
│   └── named-locations/
│
├── phase-3/                           ← Conditional Access
│   ├── README.md
│   ├── policies/
│   └── [before/after overview shots]
│
├── phase-4/                           ← Privileged Identity Management
│   ├── README.md
│   ├── entra-roles/
│   ├── activation-workflow/
│   └── azure-resources/
│
├── phase-5/                           ← Entitlement Management
│   ├── README.md
│   ├── catalog/
│   ├── access-packages/
│   ├── request-workflow/
│   └── contractor/
│
├── phase-6/                           ← Access Reviews
│   ├── README.md
│   ├── reviews/
│   └── [overview shots]
│
└── phase-7/                           ← App Registration & OAuth
    ├── README.md
    ├── app-registration/
    ├── api-permissions/
    ├── app-roles/
    ├── credentials/
    ├── oauth-flows/
    └── access-control/
```

**Total screenshots across all phases:** ~106  
**Total estimated lab time:** 14–20 hours

---

## Simulated Organisation

All users were created in the lab tenant to simulate a realistic organisational structure. No real personal data was used.

| User | Role | Department | Key Involvement |
|---|---|---|---|
| Alex Chen | IT Administrator | IT | SG-Admins, PIM Global Admin (Phase 4), SG-Admins access review |
| Priya Sharma | Senior Developer | Engineering | SG-Developers, SSPR test, CA risk simulation, access package requestor, app role: Analytics.Read + Data.Write |
| Marcus O'Brien | Security Analyst | SOC | SG-SOC, PIM Security Admin eligible (denied in Phase 6 review) |
| Linda Wu | HR Manager | Human Resources | Catalog owner (Phase 5), access package approver, access review reviewer |
| James Foster | Data Analyst | Finance | SG-Developers, Project Alpha Team, app role: Analytics.Read |
| External Contractor | Contractor | External | SG-Contractors, 90-day Contractor Project Access package (Phase 5) |
| SVC Automation | Service Account | IT | Scoped for Phase 7 workload identity scenarios |

---

## Key Design Principles

**Phishing-resistant over passwordless over standard MFA.** Authentication strength is tiered by user risk: FIDO2/CBA for administrators, Authenticator (with number matching) for developers, blocked entirely for legacy protocol clients. Each tier is enforced at both the method policy level and the Conditional Access level.

**Eligible-only, never permanent.** All privileged role assignments in PIM are Eligible — no user holds Global Administrator or Security Administrator permanently. Every elevation requires MFA, justification, and (for Global Admin) a second approver. The break-glass account is the only permanent active admin assignment.

**Time-bound access by default.** Contractor access packages expire at 90 days. PIM activations expire in 1–4 hours. Access review denials auto-remove assignments. Nothing granted in this environment persists indefinitely without a periodic review.

**Justification at every layer.** SSPR requires two authentication methods. CA policies require MFA. PIM requires justification text and ticket numbers. Access packages require a business justification question. Access reviews require a reason on approval. Every access decision leaves a documented trail.

**Test before enforce.** Every Conditional Access policy was deployed in Report-only mode first, with sign-in log validation before switching to On. The CA003 policy remains in Report-only — the correct production posture pending FIDO2 provisioning — rather than being falsely enabled or quietly left off.

---

## CV Bullet Points

The following bullet points can be used directly on a CV under a Projects section. Adapt language to match the job description of each role applied for.

> **Zero Trust Identity Governance Lab** — Microsoft Entra ID (SC-300) | *github.com/yourusername/entra-ztlab*

- Designed and implemented a seven-phase Zero Trust identity governance lab in Microsoft Entra ID covering all SC-300 exam domains, from user provisioning through to OAuth 2.0 application integration
- Configured Just-In-Time privileged access using PIM with tiered role settings, approval workflows, MFA-on-activation, and ticket-based justification — eliminating all permanent admin assignments
- Deployed four Conditional Access policies including risk-based sign-in blocking (validated via Tor Browser risk event simulation), phishing-resistant MFA enforcement for administrators, and legacy authentication elimination
- Built a self-service access governance system using Entitlement Management with a catalog, two access packages with distinct governance postures (permanent employee vs 90-day contractor), and a My Access portal workflow tested end-to-end across requestor and approver perspectives
- Implemented three distinct access review types (group membership, PIM eligible role, and access package) with auto-apply remediation — demonstrated by a Security Administrator eligible assignment automatically removed following a denial decision
- Registered an enterprise application, configured delegated and application Microsoft Graph permissions with admin consent, implemented app-role RBAC, and validated both the Authorization Code and Client Credentials OAuth 2.0 flows with JWT token inspection
- Documented all phases with GitHub-committed screenshot evidence, design decision rationale mapped to ACSC Essential Eight and ISO 27001 controls, and cross-phase audit trails connecting user provisioning to application-layer RBAC

---

## How to Navigate This Repository

1. **Start here** — this README gives the architecture overview and project context
2. **Read each phase README** — every phase has its own README with full configuration tables, design decisions, evidence index, and what was learned
3. **Browse the screenshots** — organised by phase and section, numbered sequentially within each subfolder
4. **Check the audit logs** — Screenshots of PIM audit logs, SSPR audit logs, CA sign-in logs, and entitlement management audit logs are the compliance-facing evidence of each phase

If you are reviewing this for a specific capability:

| Looking for... | Go to |
|---|---|
| MFA and authentication configuration | Phase 2 |
| Conditional Access policy design | Phase 3 |
| Privileged access governance (PIM) | Phase 4 |
| Self-service access request workflows | Phase 5 |
| Access review and auto-remediation | Phase 6 |
| OAuth 2.0 and application identity | Phase 7 |
| ACSC Essential Eight evidence | Phase 3, 4 |
| ISO 27001 A.9 access control evidence | Phase 1, 4, 5, 6 |

---

*Built as a portfolio project for an IAM Analyst career transition — demonstrating practical Microsoft Entra ID skills aligned to the SC-300 certification and Australian cybersecurity frameworks.*
