# Phase 3 — Conditional Access Policies

> **Project:** Zero Trust Identity Governance Lab
> **Certification:** Microsoft SC-300 (Identity and Access Administrator)
> **Environment:** Microsoft Entra ID (M365 Developer Program Tenant)
> **Status:** ✅ Complete (CA003 intentionally in Report-only — see design decisions)

---

## Objective

Deploy four production-grade Conditional Access policies that enforce the Zero Trust principle of *never trust, always verify* — using the authentication strengths and named locations built in Phase 2 as Grant controls and Conditions respectively.

This phase is the enforcement layer of the identity stack. Phases 1 and 2 defined *who users are* and *what authentication methods are available*. Phase 3 defines *under what circumstances access is granted or denied*.

---

## Environment

| Setting | Value |
|---|---|
| Tenant type | Microsoft Entra ID (M365 E5 Dev Program) |
| Licenses required | Free tier (CA001, CA003, CA004) + Entra ID P2 (CA002 — Identity Protection) |
| Break-glass account | breakglass.admin@… (Global Admin, excluded from all CA policies) |

---

## Pre-requisite: Break-Glass Account

Before building any Conditional Access policy, a break-glass emergency access account was created and **excluded from every CA policy in this phase**.

| Setting | Value |
|---|---|
| Account | breakglass.admin@yourdomain.onmicrosoft.com |
| Role | Global Administrator (permanent, not via PIM) |
| MFA | Not enrolled — intentional, credentials stored offline |
| Excluded from | All CA policies (CA001–CA004) |

**Why this matters:** A misconfigured CA policy can block all users from signing in — including all administrators. Without a break-glass account excluded from CA policies, there is no recovery path short of raising a Microsoft support case. This is a Microsoft-documented best practice and appears as a scenario in SC-300 exam questions.

---

## What I Built

### Policy Overview

| Policy ID | Name | Scope | Condition | Grant | State |
|---|---|---|---|---|---|
| CA001 | Require MFA for All Users | All users | None | Developer Standard Auth strength | ✅ On |
| CA002 | Block High-Risk Sign-Ins | All users | Sign-in risk: High, Medium | Block access | ✅ On |
| CA003 | Phishing-Resistant MFA for Admins | SG-Admins | None | Privileged Admin Auth strength | 🔶 Report-only |
| CA004 | Block Legacy Authentication | All users | Client apps: EAS + Other clients | Block access | ✅ On |

> All policies exclude `breakglass.admin@…` from scope.

---

### CA001 — Require MFA for All Users

**Purpose:** Ensures every user must satisfy multi-factor authentication on every sign-in, regardless of location or device.

**Configuration:**

| Setting | Value |
|---|---|
| Users | All users |
| Exclude | breakglass.admin@… |
| Target resources | All cloud apps |
| Conditions | None |
| Grant | Require authentication strength: Developer Standard Auth |
| Session | Default |
| State | On |

**Grant control detail:** Rather than using the generic "Require multi-factor authentication" checkbox, this policy uses the custom *Developer Standard Auth* strength created in Phase 2. This strength permits Microsoft Authenticator (with number matching) and FIDO2, but excludes SMS and voice call. This is a deliberate narrowing of acceptable MFA methods at the policy level — not just at the auth method policy level.

**Validation:** Signed in as `james.foster@…` in report-only mode and confirmed CA001 appeared in the Conditional Access tab of the sign-in log with result *Would have succeeded*. Switched to On and confirmed MFA challenge (number matching) was presented on next sign-in.

---

### CA002 — Block High-Risk Sign-Ins

**Purpose:** Uses Microsoft Entra Identity Protection's real-time risk engine to block sign-ins that exhibit high or medium risk signals, regardless of whether the user has valid credentials and MFA.

**Configuration:**

| Setting | Value |
|---|---|
| Users | All users |
| Exclude | breakglass.admin@… |
| Target resources | All cloud apps |
| Conditions | Sign-in risk: High, Medium |
| Grant | Block access |
| State | On |

**Risk simulation:** To generate a real sign-in risk event, signed in as `priya.sharma@…` using **Tor Browser**. Tor exit nodes are flagged by Microsoft's threat intelligence as anonymising proxies, generating an *Anonymous IP address* risk detection. The risk event appeared in `Security → Identity Protection → Risky sign-ins` within 10 minutes with risk level Medium.

After switching CA002 to On, repeated the Tor sign-in and received a hard block: *"Your sign-in was blocked."* The CA002 policy appeared in the sign-in log Conditional Access tab with result *Block*.

**Post-test cleanup:** Dismissed the risk on Priya's account via `Identity Protection → Risky users → Dismiss user risk` to prevent the risk score from carrying forward.

**Sign-in risk vs User risk:** This policy targets *sign-in risk* — the per-authentication real-time signal. *User risk* is a separate, cumulative score reflecting the probability that a user's identity is compromised over time (e.g. leaked credentials). A separate user risk policy (requiring password change) would be the complementary control. Both are configurable via Identity Protection and Conditional Access.

---

### CA003 — Phishing-Resistant MFA for Admins

**Purpose:** Enforces the *Privileged Admin Auth* custom authentication strength (created in Phase 2) for all members of SG-Admins, requiring FIDO2 security key or Certificate-Based Authentication only. Standard Authenticator push is not sufficient for admin sign-ins under this policy.

**Configuration:**

| Setting | Value |
|---|---|
| Users | SG-Admins |
| Exclude | breakglass.admin@… |
| Target resources | All cloud apps |
| Conditions | None |
| Grant | Require authentication strength: Privileged Admin Auth |
| State | 🔶 Report-only |

**Why Report-only?** In this lab environment, FIDO2 security keys have not been provisioned for admin accounts. Enabling CA003 in enforced mode would immediately block `alex.chen@…` from signing in, as Authenticator push does not satisfy the *Privileged Admin Auth* strength.

Validated in report-only mode: signed in as `alex.chen@…` and confirmed CA003 appeared in the sign-in log with result *Report-only: Would have failed*. This is the expected and correct result — it proves the policy evaluated accurately and identified the authentication gap.

**Production deployment path:**
1. Enable in Report-only (current state)
2. Identify all admin accounts that would be blocked
3. Issue FIDO2 security keys or configure Certificate-Based Authentication
4. Confirm method registration via `Security → Authentication methods → User registration details`
5. Switch policy to On

---

### CA004 — Block Legacy Authentication

**Purpose:** Blocks all legacy authentication protocols that cannot perform modern MFA — closing the attack vector used in password spray campaigns.

**Configuration:**

| Setting | Value |
|---|---|
| Users | All users |
| Exclude | breakglass.admin@… |
| Target resources | All cloud apps |
| Conditions | Client apps: Exchange ActiveSync clients, Other clients |
| Grant | Block access |
| State | On |

**Why legacy auth can't be protected by MFA:** Legacy protocols (Basic Auth, SMTP Auth, IMAP, POP3, Exchange ActiveSync, older MAPI clients) authenticate with a username and password at the protocol level. There is no mechanism in these protocols to prompt for or relay a second factor. A CA policy requiring MFA will not evaluate against a legacy auth session — it simply does not apply. The only effective control is to block these client app types entirely.

**Impact:** Any mail client, script, or device using Basic Auth or SMTP Auth will stop working. In a production rollout this would require an impact assessment (identifying which services use legacy auth via sign-in logs filtered by client app type) before enabling. In this lab, no legacy auth clients are in use, so the policy was switched to On immediately after a short report-only observation period.

---

## Key Design Decisions

**Why use a named authentication strength in CA001 instead of the generic MFA checkbox?**
The generic "Require multi-factor authentication" Grant control allows any method that satisfies MFA — including SMS and voice call, which were deliberately excluded from the tenant's authentication method policy in Phase 2. Using the *Developer Standard Auth* strength in the Grant control enforces that exclusion at the policy level as well, providing defence-in-depth. If someone re-enables SMS at the method policy level, CA001 still won't accept it.

**Why is the break-glass account excluded from all policies and not enrolled in MFA?**
The break-glass account is an emergency recovery tool, not a daily-use account. If it were enrolled in MFA and the MFA system itself had a failure (e.g. Microsoft Authenticator outage, lost device), the break-glass account would be inaccessible when most needed. Credentials are stored offline (physically secured) and sign-ins by this account should trigger an alert — any use outside a declared emergency is itself an incident.

**Why does Policy 2 target sign-in risk rather than user risk?**
Sign-in risk is evaluated in real time at the moment of authentication — it is the most actionable signal for a block policy. User risk accumulates over time and is better suited to a remediation policy (requiring password change or admin confirmation) rather than an outright block, since a single historical risk signal should not permanently deny access. The two controls are complementary: block the session now (sign-in risk), and remediate the account over time (user risk).

**Why is CA003 left in Report-only rather than enabled?**
Enabling CA003 without first provisioning FIDO2 keys would immediately lock administrators out. The correct sequence — report-only, identify gap, remediate, enforce — is the production-grade deployment approach. Documenting this gap explicitly (and showing the *Would have failed* log result) is more valuable than falsely switching the policy to On. It demonstrates understanding of the deployment lifecycle, not just the configuration syntax.

**How does this align with the ACSC Essential Eight?**
These four CA policies directly address two Essential Eight controls:

- **Multi-factor authentication (ML2/ML3):** CA001 enforces MFA for all users using strong methods. CA003 enforces phishing-resistant MFA for privileged users — meeting the Essential Eight requirement at Maturity Level 3 once FIDO2 is provisioned.
- **Restrict administrative privileges:** CA003 applies a stricter authentication requirement to admin accounts, reducing the risk of admin credential compromise.

CA002's risk-based blocking also aligns with the Essential Eight's intent around restricting access from untrusted sources.

---

## Evidence

| # | Screenshot | Description |
|---|---|---|
| 1 | `01-ca-policies-baseline.png` | CA policy list before any policies created — baseline state |
| 2 | `02-p2-license-confirmed.png` | Entra ID P2 licence confirmed active for Identity Protection |
| 3 | `policies/03-breakglass-account-created.png` | Break-glass account with Global Admin role assigned |
| 4 | `policies/04-ca001-require-mfa-config.png` | CA001 full configuration — auth strength Grant visible |
| 5 | `policies/05-ca001-reportonly-signin-log.png` | CA001 report-only result in sign-in log — policy evaluation confirmed |
| 6 | `policies/06-ca001-mfa-enforced.png` | MFA challenge or sign-in log confirming CA001 enforcement |
| 7 | `policies/07-ca002-block-risk-config.png` | CA002 configuration — sign-in risk conditions and Block grant |
| 8 | `policies/08-ca002-risk-event-detected.png` | Identity Protection risky sign-in from Tor Browser — risk detail visible |
| 9 | `policies/09-ca002-block-enforced.png` | Sign-in blocked screen or log showing CA002 Block result |
| 10 | `policies/10-ca002-risk-dismissed.png` | Risk dismissed on Priya's account post-testing |
| 11 | `policies/11-ca003-admin-phishing-resistant-config.png` | CA003 configuration — Privileged Admin Auth strength and SG-Admins scope |
| 12 | `policies/12-ca003-reportonly-would-fail.png` | CA003 report-only log showing *Would have failed* for Alex Chen |
| 13 | `policies/13-ca004-block-legacy-config.png` | CA004 configuration — EAS and Other clients conditions checked |
| 14 | `policies/14-ca004-legacy-auth-blocked.png` | Legacy auth block evidence — sign-in log or enabled policy confirmation |
| 15 | `15-ca-policies-complete-overview.png` | All 4 CA policies in the policy list — final state |

![CA Policies — Complete Overview](./15-ca-policies-complete-overview.png)
*All four Conditional Access policies deployed — CA001, CA002, CA004 enforced; CA003 in report-only pending FIDO2 provisioning.*

![CA002 — Risk Event Detected](./policies/08-ca002-risk-event-detected.png)
*Identity Protection risky sign-in event generated via Tor Browser — Anonymous IP address risk detection confirmed.*

![CA001 — Report-Only Validation](./policies/05-ca001-reportonly-signin-log.png)
*Report-only evaluation of CA001 in the sign-in log — validating policy behaviour before enforcement.*

---

## SC-300 Domain Coverage

| SC-300 Objective | Covered in this phase |
|---|---|
| Plan and implement Conditional Access policies | ✅ 4 policies across all major CA patterns |
| Implement authentication strengths in CA | ✅ Custom strengths used as Grant controls in CA001 and CA003 |
| Implement and manage Identity Protection | ✅ Risk-based CA policy, risk event simulation, risk dismissal |
| Manage CA policy states and report-only mode | ✅ Report-only validation workflow documented for all policies |
| Implement emergency access accounts | ✅ Break-glass account created and excluded from all policies |

---

## What I Learned

- **The Conditional Access tab in Sign-in Logs is the single most useful debugging tool for CA.** Every sign-in shows which policies evaluated, which conditions matched, and what the result was — whether report-only or enforced. Knowing this blade exists and how to read it is essential for both the exam and any real troubleshooting conversation.
- **Authentication strength is a Grant control, not a Condition.** This trips up a lot of SC-300 candidates. Conditions define *when* the policy fires. Grant defines *what must be satisfied* to get access. The strength selector is under Grant → Require authentication strength.
- **Tor Browser does not always generate a risk event immediately.** Microsoft's risk engine has some latency and does not flag every anonymous IP sign-in. If no risk event appears after 15 minutes, a VPN with a non-home-country exit node is a reliable alternative trigger.
- **Legacy auth is still one of the most common enterprise security gaps in 2025.** Many organisations have SMTP relay services, multifunction printers, and legacy line-of-business apps still using Basic Auth. The correct pre-enforcement workflow is to filter sign-in logs by client app type for at least 30 days to identify all legacy auth sources before blocking.
- **CA003's *Would have failed* result is the right outcome to show at this stage.** It demonstrates the policy is working correctly — the admin simply doesn't have the required auth method registered yet. This is more valuable than quietly leaving the policy off or misrepresenting the lab state.

---

## Phase Dependencies

| Phase 2 Input | Consumed by |
|---|---|
| Custom strength: Developer Standard Auth | CA001 — Grant control |
| Custom strength: Privileged Admin Auth | CA003 — Grant control |
| Named location: Trusted Office Network | Available as CA Condition (Phase 3 extension) |
| Named location: High-Risk Countries | Available as CA Condition (Phase 3 extension) |
| MFA registered users (Alex Chen, Priya Sharma) | Test subjects for CA001/CA002 validation |

---

## Previous & Next Phase

⬅️ [Phase 2 — Authentication & MFA Hardening](../phase-2/README.md)

➡️ [Phase 4 — Privileged Identity Management](../phase-4/README.md)

Configure Just-In-Time privileged access using PIM — enforcing time-limited role activation with approval workflows, MFA on activation, and full audit logging for all privileged operations.

---

*Part of the [Zero Trust Identity Governance Lab](../README.md) — an SC-300 portfolio project built in Microsoft Entra ID.*

