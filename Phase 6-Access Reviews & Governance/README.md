# Phase 6 — Access Reviews & Governance

> **Project:** Zero Trust Identity Governance Lab
> **Certification:** Microsoft SC-300 (Identity and Access Administrator)
> **Environment:** Microsoft Entra ID (M365 Developer Program Tenant)
> **Status:** ✅ Complete

---

## Objective

Validate that existing access assignments remain appropriate using three distinct types of Microsoft Entra access review — covering group membership, PIM eligible role assignments, and entitlement management access packages.

Granting access is only half of the governance problem. Access Reviews address the other half: periodically challenging whether access that was once appropriate is still appropriate today. This phase closes the loop on Phases 1–5 by introducing a recurring challenge mechanism across every layer of the identity stack built so far.

---

## Environment

| Setting | Value |
|---|---|
| Tenant type | Microsoft Entra ID (M365 E5 Dev Program) |
| License required | Entra ID P2 |
| Reviews created | 3 (group, Entra ID role, access package) |
| Review outcomes | 1 approved, 1 denied (with auto-removal), 1 pending annual cycle |

---

## Access Review Strategy

Before building, the three review types and their governance intent were defined:

| Layer | What is being reviewed | Why |
|---|---|---|
| SG-Admins (Group) | Group membership | Privileged group used for PIM scoping — stale membership expands the eligible assignment surface |
| Security Administrator (Role) | PIM eligible assignments | Validates the right to activate the role, not just current active holders |
| New Developer Onboarding (Package) | Entitlement management assignments | Ensures permanent employee access is periodically revalidated |

**Key design decision — "If reviewers don't respond":**
Rather than using a single default across all reviews, each review was configured with a no-response behaviour calibrated to the risk level of what is being reviewed:

- SG-Admins → **Remove access** (privileged group — err on the side of removal)
- Security Administrator → **Remove access** (eligible role assignment — if the need can't be justified, remove it)
- New Developer Onboarding → **Take recommendations** (lower-risk, permanent employee access — sign-in activity is a reasonable proxy)

---

## What I Built

### Review 1 — SG-Admins Group Membership

**Purpose:** Validate quarterly that all members of SG-Admins still require membership. SG-Admins is the privileged group that scopes PIM eligible assignments and Conditional Access policies — any unnecessary membership here directly widens the privileged access surface.

| Setting | Value |
|---|---|
| Review name | SG-Admins Quarterly Review |
| Review type | Teams + Groups |
| Scope | SG-Admins — All members |
| Reviewer | Alex Chen (group owner) |
| Recurrence | Quarterly |
| Duration | 3 days |
| Require reason on approval | Yes |
| Show recommendations | Yes |
| Auto-apply results | Yes |
| If no response | Remove access |

**Review outcome:** Alex Chen reviewed his own membership (lab simplification — in production a separate reviewer would be assigned) and approved with justification: *"Alex Chen is the primary IT Administrator and requires SG-Admins membership for PIM-scoped role activations. Access confirmed appropriate."*

The reason field is stored against the review record — this is the documented evidence of the access decision, not just a date and a name.

---

### Review 2 — Security Administrator Eligible Assignment (PIM)

**Purpose:** Validate every six months that users holding a PIM eligible assignment for Security Administrator still require the right to activate that role. This review targets eligible assignments — the right to request the role — not currently active holders.

| Setting | Value |
|---|---|
| Review name | Security Admin Role Review |
| Review type | Azure AD roles (PIM) |
| Role | Security Administrator |
| Assignment type | Eligible assignments |
| Reviewer | Admin account |
| Recurrence | Every 6 months |
| Duration | 3 days |
| Require reason on approval | Yes |
| Auto-apply results | Yes |
| If no response | Remove access |

**Review outcome:** Marcus O'Brien was **denied** with the following justification:

> *"Marcus O'Brien has transitioned to a read-only SOC monitoring role — Security Administrator eligible assignment no longer required. Downgrading to Security Reader."*

With auto-apply enabled, Marcus's PIM eligible assignment for Security Administrator was automatically removed when the review period closed. Confirmed via PIM → Microsoft Entra roles → Assignments → Eligible assignments — Marcus O'Brien no longer appeared in the list.

This demonstrates the full access review lifecycle: configure → review → deny → auto-apply → assignment removed — with an audit trail at every step.

---

### Review 3 — New Developer Onboarding Access Package

**Purpose:** Validate annually that active assignments to the New Developer Onboarding access package (created in Phase 5) are still appropriate. This review is configured at the policy level within Entitlement Management, not from the standalone Access Reviews blade.

| Setting | Value |
|---|---|
| Review name | New Developer Onboarding Annual Review |
| Review type | Access package assignments (Entitlement Management) |
| Scope | All active assignments to New Developer Onboarding |
| Reviewer | Linda Wu (HR Manager — original approver) |
| Recurrence | Annually |
| Duration | 14 days |
| If no response | Take recommendations |
| Auto-apply results | Yes |

**Configuration path:** Entitlement Management → Access packages → New Developer Onboarding → Policies → Lifecycle tab → Access reviews: Enabled.

The review appears in Identity Governance → Access Reviews with source column showing "Entitlement Management" — distinguishing it from the standalone group and role reviews.

**Review outcome:** Pending — the annual cycle has not yet triggered a review period. The review configuration is confirmed active.

---

## Key Design Decisions

**Why review group membership, role assignments, and access packages separately?**
Each targets a different layer of the access model and requires different governance logic. Group membership reviews validate who belongs in a security or collaboration group. Role reviews validate who has the right to activate a privileged role — not just who currently holds it. Access package reviews validate bundled resource assignments that were granted through a self-service workflow. Covering all three demonstrates that governance is applied consistently across the identity stack, not just at one layer.

**Why use "Remove access on no response" for privileged groups and roles?**
The alternative — "No change" — means that if a reviewer misses the deadline, stale access persists silently with no record of challenge. For SG-Admins and the Security Administrator eligible assignment, the risk of stale privileged membership outweighs the risk of accidentally removing access that was still needed. If access is incorrectly removed, it can be reinstated quickly and the reinstatement is itself auditable. If stale privileged access is never challenged, it accumulates indefinitely — this is how most privilege escalation paths develop over time.

**Why is Marcus O'Brien's denial the most valuable review result in this phase?**
A review where everyone is approved is indistinguishable from a rubber-stamp exercise. The denial of Marcus O'Brien's eligible assignment — with a substantive reason tied to a role change — demonstrates that the review is meaningful. It also proves the auto-apply mechanism works end-to-end: a governance decision translates directly into a change in the identity system with no additional admin action required. This is the compliance evidence that matters: not that reviews were scheduled, but that reviews produced outcomes and those outcomes were enforced.

**Why configure the access package review at the policy level rather than as a standalone review?**
Access package reviews configured at the policy level recur automatically with each cycle and remain linked to the package for its lifetime. A standalone review created in the Access Reviews blade is one-off unless manually scheduled again. For permanent employee access packages that should be reviewed annually for the foreseeable future, the policy-level configuration is the correct approach. It also means the review scope automatically includes any new assignments to the package — if another employee joins the package between now and the next review cycle, they will be included without any additional configuration.

**How do access reviews and PIM work together?**
PIM (Phase 4) and access reviews operate at different points in the privileged access lifecycle:

- **PIM** controls *when and how* privileged roles are used — time-bounded activation, MFA, approval, justification
- **Access reviews** control *whether the eligible assignment should exist at all* — periodic challenge of the right to request activation

Without access reviews, PIM eligible assignments accumulate indefinitely as organisational roles change. Without PIM, access reviews catch stale access but cannot prevent real-time misuse between review cycles. Together they provide both operational controls (PIM) and governance oversight (reviews) across the full lifecycle of a privileged role assignment.

**How does this align with ACSC Essential Eight and ISO 27001?**

| Framework | Control | How Phase 6 addresses it |
|---|---|---|
| ACSC Essential Eight | Restrict administrative privileges | Quarterly review of SG-Admins with auto-removal on denial |
| ACSC Essential Eight | Regular backups / access validation | Annual access package reviews ensure entitlements remain appropriate |
| ISO 27001 A.9.2.5 | Review of user access rights | Three access reviews covering group, role, and package assignments |
| ISO 27001 A.9.2.6 | Removal of access rights | Auto-apply with "Remove access on no response" enforces timely removal |
| ISO 27001 A.12.4 | Logging and monitoring | Review results, decisions, and reasons are retained in audit logs |

---

## Evidence

| # | Screenshot | Description |
|---|---|---|
| 1 | `01-access-reviews-overview.png` | Access Reviews dashboard — before state, empty review list |
| 2 | `reviews/02-sg-admins-review-config.png` | SG-Admins review — scope, reviewer, and recurrence configured |
| 3 | `reviews/03-sg-admins-review-settings.png` | Settings — auto-apply, remove on no response, require reason |
| 4 | `reviews/04-sg-admins-review-approve.png` | Reviewer view in My Access portal — approve decision with reason |
| 5 | `reviews/05-sg-admins-review-results.png` | Admin results view — Alex Chen approved with justification and timestamp |
| 6 | `reviews/06-security-admin-role-review-config.png` | Security Admin role review — eligible assignments scope, 6-month recurrence |
| 7 | `reviews/07-security-admin-role-review-settings.png` | Settings — auto-apply, remove on no response |
| 8 | `reviews/08-security-admin-review-deny.png` | Deny decision for Marcus O'Brien with role-change justification |
| 9 | `reviews/09-eligible-assignment-removed.png` | PIM eligible assignments — Marcus O'Brien removed after auto-apply |
| 10 | `reviews/10-package-policy-access-review.png` | Entitlement Management policy lifecycle tab — annual review configured |
| 11 | `11-all-active-reviews-list.png` | **All three reviews** in the Access Reviews list — group, role, package |

![All Active Access Reviews](./11-all-active-reviews-list.png)
*Three distinct access review types covering every layer of the identity stack — group membership, PIM eligible roles, and entitlement management access packages.*

![Security Admin Review — Denial](./reviews/08-security-admin-review-deny.png)
*Marcus O'Brien's Security Administrator eligible assignment denied — role change justification documented in the review record.*

![PIM Eligible Assignment Removed](./reviews/09-eligible-assignment-removed.png)
*PIM eligible assignments after auto-apply — Marcus O'Brien's Security Administrator assignment removed following the review denial.*

---

## SC-300 Domain Coverage

| SC-300 Objective | Covered in this phase |
|---|---|
| Plan and implement access reviews | ✅ Three review types, recurrence, scope, reviewer, and completion settings |
| Manage access review results and remediation | ✅ Approve with reason, deny with reason, auto-apply, confirmed removal |
| Monitor identity governance with access reviews | ✅ Results tab, audit log entries, PIM assignment verification post-review |
| Implement access reviews for privileged roles | ✅ PIM eligible assignment review with auto-apply and removal on denial |
| Configure access package lifecycle settings | ✅ Policy-level annual review for Entitlement Management package |

---

## What I Learned

- **Access reviews have three distinct configuration paths** depending on what you are reviewing: Identity Governance → Access Reviews for standalone group and role reviews; Entitlement Management policy lifecycle tab for access package reviews; and PIM role settings for role-specific review frequency defaults. Getting the right path for the right review type is a common SC-300 exam scenario.
- **A Deny outcome with auto-apply is the most valuable evidence you can produce from an access review.** Anyone can configure a review. Showing that a denial decision translated into a real change in the identity system — Marcus O'Brien's eligible assignment removed from PIM — proves the governance mechanism has teeth.
- **The "Take recommendations" no-response option uses sign-in activity as its signal.** Users who have not signed in using the group or app in the past 30 days receive a Deny recommendation; active users receive an Approve recommendation. This is useful for large-scale reviews where manual review of every user is impractical — the system does the triage and reviewers confirm or override.
- **Auto-apply timing is not instant.** After the review period ends, auto-apply runs on a schedule — typically within 24 hours. If you need to see the removal happen faster in a lab, look for an "Apply results" button on the completed review which allows manual triggering of the auto-apply action.
- **The review reason field is the audit trail, not just a UX nicety.** Every reason entered by a reviewer is stored in the review record and visible in the audit log. In an ISO 27001 audit or an APRA CPS 234 assessment, the question "show me evidence that access was reviewed and justified" is answered by the review results with reasons — not just by showing that a review was scheduled.

---

## Review Matrix

| Review | Type | Reviewer | Frequency | No Response | Auto-Apply | Outcome |
|---|---|---|---|---|---|---|
| SG-Admins Membership | Group | Alex Chen | Quarterly | Remove access | Yes | ✅ Approved |
| Security Admin Role | Entra ID Role (PIM) | Admin | Every 6 months | Remove access | Yes | ❌ Denied — removed |
| New Developer Onboarding | Access Package | Linda Wu | Annually | Take recommendations | Yes | ⏳ Pending |

---

## Phase Dependencies

| Previous Phase | Connection to Phase 6 |
|---|---|
| Phase 1 — SG-Admins group | Reviewed quarterly — membership validated |
| Phase 4 — Marcus O'Brien PIM eligible assignment | Reviewed and removed via access review auto-apply |
| Phase 5 — New Developer Onboarding access package | Annual review configured at policy level |
| Phase 5 — Linda Wu (catalog owner/approver) | Assigned as reviewer for access package review |

---

## Previous & Next Phase

⬅️ [Phase 5 — Entitlement Management & Access Packages](../phase-5/README.md)

➡️ [Phase 7 — App Registration & OAuth 2.0](../phase-7/README.md)

Register an enterprise application in Microsoft Entra ID, configure API permissions with admin consent, create app roles for developer group members, and test the OAuth 2.0 client credentials flow against Microsoft Graph.

---

*Part of the [Zero Trust Identity Governance Lab](../README.md) — an SC-300 portfolio project built in Microsoft Entra ID.*