# Phase 5 — Entitlement Management & Access Packages

> **Project:** Zero Trust Identity Governance Lab
> **Certification:** Microsoft SC-300 (Identity and Access Administrator)
> **Environment:** Microsoft Entra ID (M365 Developer Program Tenant)
> **Status:** ✅ Complete

---

## Objective

Build a self-service access governance system using Microsoft Entra Entitlement Management — replacing ad-hoc, admin-driven group assignments with a structured catalog, access packages, approval workflows, and automatic time-limited provisioning for contractors.

This phase addresses a fundamental gap in most identity environments: access is granted quickly when someone joins a project, but rarely reviewed or removed when they leave. Entitlement Management solves this by making access time-bound, request-driven, and fully auditable from day one.

---

## Environment

| Setting | Value |
|---|---|
| Tenant type | Microsoft Entra ID (M365 E5 Dev Program) |
| License required | Entra ID P2 |
| Catalog | Project Alpha Resources |
| Access packages | 2 (Developer Onboarding, Contractor Project Access) |
| Users tested | Priya Sharma (requestor), Linda Wu (approver), External Contractor (admin-assigned) |

---

## Object Model

Before building, it is worth establishing the hierarchy that Entitlement Management uses:

```
Catalog ──────────────────────────────────────────────────────
│ Project Alpha Resources
│
├── Resources
│ ├── Project Alpha Team (M365 Group)
│ └── SG-Developers (Security Group)
│
├── Access Package: New Developer Onboarding
│ ├── Resource roles: Project Alpha Team (Member) + SG-Developers (Member)
│ └── Policy: All members → Manager approval → No expiry
│
└── Access Package: Contractor Project Access
├── Resource roles: Project Alpha Team (Member) only
└── Policy: Admin assigns → Linda Wu approval → 90-day expiry
```

**Key principle:** A single access package request grants all bundled resources simultaneously through one approval — no separate requests per group.

---

## What I Built

### 5A — Entitlement Management Settings

Reviewed global Entitlement Management settings before building:

- External users: Blocked — access packages are restricted to internal tenant members only
- Delegation: Catalog owners can manage their own catalogs without Global Administrator rights
- This established the governance boundary before creating any resources

### 5B — Catalog: Project Alpha Resources

| Setting | Value |
|---|---|
| Catalog name | Project Alpha Resources |
| Description | Catalog for Project Alpha team onboarding and contractor access |
| Enabled | Yes |
| Enabled for external users | No |
| Catalog owner | Linda Wu (HR Manager) — delegated without Global Admin |

**Resources added to catalog:**

| Resource | Type | Used in Package |
|---|---|---|
| Project Alpha Team | Microsoft 365 Group | Both packages |
| SG-Developers | Security Group | Developer Onboarding only |

**Delegation design:** Linda Wu was assigned as Catalog owner. This means HR manages the access lifecycle for Project Alpha without needing IT admin involvement for day-to-day approvals and package management. This models the intended Entitlement Management governance pattern — business owners manage access to their own resources.

### 5C — Access Packages

**Package 1: New Developer Onboarding**

| Setting | Value |
|---|---|
| Catalog | Project Alpha Resources |
| Resources granted | Project Alpha Team (Member) + SG-Developers (Member) |
| Who can request | All directory members |
| Approval | Required — Linda Wu (HR Manager) |
| Requestor question | "What project will you be working on and what is your start date?" |
| Request expiration | 14 days to respond |
| Assignment expiry | None — permanent for full-time employees |
| Access reviews | Configured in Phase 6 |

**Package 2: Contractor Project Access**

| Setting | Value |
|---|---|
| Catalog | Project Alpha Resources |
| Resources granted | Project Alpha Team (Member) only |
| Who can request | Admin assigns on behalf of contractor |
| Approval | Required — Linda Wu (HR Manager) |
| Requestor question | "Contractor company name, engagement reference, and expected end date" |
| Assignment expiry | 90 days from assignment date |
| Access reviews | Not configured — access ends at expiry |

**Package design rationale:** Contractors receive a deliberately narrower resource set than employees. Package 1 grants both the collaboration group and the security group used for CA policy scoping. Package 2 grants the collaboration group only — contractors get Project Alpha team access but are not added to SG-Developers, so they do not benefit from any developer-tier Conditional Access allowances. The 90-day hard expiry on Package 2 means no manual cleanup is needed when a contractor engagement ends — access is removed automatically.

### 5D — Request and Approval Workflow (My Access Portal)

Executed the full self-service request lifecycle using `myaccess.microsoft.com`:

**Priya Sharma (requestor) perspective:**

1. Signed in to `myaccess.microsoft.com` as priya.sharma@…
2. Located New Developer Onboarding in the Access packages list
3. Submitted request with custom question answered: *"Project Alpha migration work. Start date: 21 July 2025. Team lead: Alex Chen."* Business justification: *"Required for Phase 2 of Project Alpha data migration — dev environment access needed from day one."*
4. Received confirmation: *Your request has been submitted*
5. Status showed Pending approval — Priya did not yet have access

**Linda Wu (approver) perspective:**

1. Signed in to `myaccess.microsoft.com` as linda.wu@…
2. Navigated to Approvals tab — Priya's request visible with full justification and custom question answers
3. Reviewed request details, approved with justification: *"Confirmed with Project Alpha team lead — access required for migration work starting Monday."*
4. Priya's group memberships provisioned automatically within minutes

**Post-approval verification (admin view):**

- Confirmed Priya Sharma appeared in Project Alpha Team → Members
- Confirmed Priya Sharma appeared in SG-Developers → Members
- Both memberships provisioned by a single approval — no additional admin action
- Assignment visible in Entitlement Management → New Developer Onboarding → Assignments with status: Delivered

### 5E — Contractor Assignment and Audit Log

**Contractor access:**

- Admin assigned Contractor Project Access to `contractor.ext@…` on behalf of the user
- Assignment confirmed with status: Delivered
- Assignment expiry date: 90 days from creation — visible in the Assignments tab
- Contractor has Project Alpha Team membership only — SG-Developers not granted

**Audit log:**

Reviewed the Entitlement Management audit log covering the full Priya Sharma request lifecycle:

| Event | Actor | Timestamp |
|---|---|---|
| Access package assignment request created | Priya Sharma | Day 1 |
| Access package assignment request approved | Linda Wu | Day 1 |
| Access package assignment granted | System | Day 1 |
| Access package assignment delivered | System | Day 1 |

---

## Key Design Decisions

**Why use Entitlement Management instead of direct group assignment?**
Direct group assignment by an admin grants access with no documented business justification, no approval workflow, no automatic expiry, and no self-service capability. When the employee leaves or moves teams, the access persists until someone manually removes it — which often never happens. Entitlement Management addresses all of these gaps: the requestor provides a business justification, a business owner approves, access expires automatically, and the full lifecycle is auditable. This is the difference between access management as a security control versus access management as an afterthought.

**Why is the M365 Group a better catalog resource than a Security Group for collaboration access?**
A Microsoft 365 Group provisions a shared mailbox, SharePoint site, Teams workspace, and OneNote automatically when a user is added. A Security Group grants membership only — no collaboration resources are created. For Project Alpha, where team members need a shared workspace and document library, the M365 Group is the correct resource type. SG-Developers is also included in Package 1 as a Security Group, giving employees the group membership used for Conditional Access scoping — contractors in Package 2 deliberately do not receive this.

**Why require a custom question on both access packages?**
The custom question creates a documented business justification attached to every assignment record. When an auditor or access reviewer asks "why does this user have access?", the answer is in the assignment audit log — not just a date and an approver's name, but the specific project, start date, and engagement reference provided at request time. This directly satisfies ISO 27001 A.9.2.1 (User registration and de-registration) and ACSC Essential Eight requirements around logging access grants with sufficient context.

**Why delegate catalog ownership to Linda Wu (HR) rather than keeping it with IT admins?**
Entitlement Management is designed to shift access governance from IT to business owners. HR is the appropriate owner of onboarding and contractor access packages — they know who is joining, on what terms, and for how long. By delegating catalog ownership to Linda Wu, HR can manage the access lifecycle for their own resources without raising IT tickets for every approval. IT retains control over what resources are in the catalog and what the tenant-wide settings are. This separation of duties is a real enterprise pattern and models what SC-300 expects you to understand about delegated identity governance.

**Why does the contractor package expire at 90 days without an access review?**
Contractor engagements have a defined end date built into the commercial arrangement. A 90-day hard expiry is appropriate because the access should end regardless of whether a reviewer remembers to action a review. If the engagement is extended, the contractor or their sponsor submits a new request — which creates a new audit trail entry. Access reviews (Phase 6) are better suited to permanent employee access, where the question "does this person still need this?" cannot be answered by a date alone.

**How does this phase align with the Zero Trust principle of least privilege?**
Three least-privilege controls are applied simultaneously:

1. **Scope:** Contractors receive Project Alpha Team only, not SG-Developers. Employees receive both. Each package grants the minimum required for that user type.
2. **Duration:** Contractor access is time-bound to 90 days. Employee access is permanent but subject to access reviews in Phase 6. No access is permanent without a review mechanism.
3. **Justification:** Every grant requires a documented business justification. There is no path to access that bypasses the approval workflow and the justification question.

---

## Evidence

| # | Screenshot | Description |
|---|---|---|
| 1 | `01-entitlement-mgmt-overview.png` | Entitlement Management dashboard — before state |
| 2 | `02-entitlement-mgmt-settings.png` | Global settings — external users blocked, delegation reviewed |
| 3 | `catalog/03-catalog-creation-form.png` | Project Alpha Resources catalog creation |
| 4 | `catalog/04-catalog-resources-added.png` | Both resources added to catalog |
| 5 | `catalog/05-catalog-owner-assigned.png` | Linda Wu assigned as catalog owner |
| 6 | `access-packages/06-package1-resource-roles.png` | Package 1 resource roles — M365 Group + Security Group |
| 7 | `access-packages/07-package1-request-policy.png` | Package 1 request policy — approval and custom question |
| 8 | `access-packages/08-package1-created-overview.png` | Package 1 creation confirmed |
| 9 | `access-packages/09-package2-lifecycle-90day.png` | Package 2 lifecycle — 90-day expiry configured |
| 10 | `access-packages/10-access-packages-list.png` | Both packages in list — permanent vs time-limited contrast |
| 11 | `request-workflow/11-myaccess-portal-priya.png` | My Access portal — end-user view as Priya Sharma |
| 12 | `request-workflow/12-access-request-submitted.png` | Request submitted — justification and custom question answered |
| 13 | `request-workflow/13-approval-linda-wu-view.png` | Approver view — Linda Wu's Approvals page |
| 14 | `request-workflow/14-group-membership-provisioned.png` | Project Alpha Team members — Priya confirmed as member |
| 15 | `request-workflow/15-package-assignment-delivered.png` | Assignment status Delivered in admin portal |
| 16 | `contractor/16-contractor-assignment-with-expiry.png` | Contractor assignment — 90-day expiry date visible |
| 17 | `17-entitlement-audit-log.png` | **Full audit trail** — request, approval, grant, delivery events |

![Entitlement Management Audit Log](./17-entitlement-audit-log.png)
*Complete audit trail for the Priya Sharma access package request — every lifecycle event captured with actor, timestamp, and target resource.*

![My Access Portal — Priya Sharma](./request-workflow/11-myaccess-portal-priya.png)
*End-user self-service view at myaccess.microsoft.com — available access packages visible without admin involvement.*

![Access Packages List](./access-packages/10-access-packages-list.png)
*Both access packages in the admin portal — permanent developer onboarding alongside 90-day contractor access.*

---

## SC-300 Domain Coverage

| SC-300 Objective | Covered in this phase |
|---|---|
| Plan and implement entitlement management | ✅ Catalog, resources, two access packages with distinct policies |
| Manage access packages and policies | ✅ Resource roles, approval workflow, custom questions, lifecycle settings |
| Manage access requests and assignments | ✅ Self-service request via My Access, approval, assignment delivery |
| Implement and manage terms of use and access reviews | ✅ Access reviews planned for Phase 6 — referenced in package lifecycle |
| Implement and manage catalog roles and delegation | ✅ Linda Wu assigned as catalog owner without Global Admin rights |

---

## What I Learned

- **The My Access portal (`myaccess.microsoft.com`) is a completely separate interface from the admin portal** — and the one that end users actually interact with. Approvers also use this portal, not the Entra admin centre. Testing from both the requestor and approver perspectives is essential to understanding the full workflow.
- **A resource can only belong to one catalog at a time.** If you add a group to Catalog A, it is not available to add to Catalog B. Plan your catalog boundaries before adding resources, or you will need to remove the resource from one catalog before adding it to another.
- **The custom question field on a policy is not just a UX detail** — it is an audit control. The answer is stored against the assignment record and appears in the audit log. In a compliance context, this is the documented business justification for the access grant.
- **Assignment delivery is not always instant.** After approval, the assignment status moves through Approved → Delivering → Delivered. Group membership provisioning typically takes 1–5 minutes. If the assignment stays in Delivering for more than 10 minutes, check the resource configuration in the catalog — a misconfigured resource role is the most common cause.
- **Catalog owner delegation is one of the most operationally significant Entitlement Management features** and the one most commonly overlooked in labs. In a real organisation, IT cannot be the bottleneck for every access request. Catalog owners shift the governance responsibility to the people who actually understand who should have access to what.

---

## Phase Dependencies

| Phase 1 Input | Used in Phase 5 |
|---|---|
| Project Alpha Team (M365 Group) | Primary resource in both access packages |
| SG-Developers (Security Group) | Resource in Package 1 (Developer Onboarding) |
| Linda Wu (HR Manager user) | Catalog owner + approver |
| External Contractor user | Recipient of admin-assigned Package 2 |
| Priya Sharma | Requestor for Package 1 workflow test |

---

## Previous & Next Phase

⬅️ [Phase 4 — Privileged Identity Management](../phase-4/README.md)

➡️ [Phase 6 — Access Reviews & Lifecycle Workflows](../phase-6/README.md)

Configure scheduled access reviews to periodically validate that existing access assignments are still appropriate — and automate identity lifecycle events using Lifecycle Workflows for joiners and leavers.

---

*Part of the [Zero Trust Identity Governance Lab](../README.md) — an SC-300 portfolio project built in Microsoft Entra ID.*
