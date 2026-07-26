# Phase 2 — Authentication & MFA Hardening

> **Project:** Zero Trust Identity Governance Lab
> **Certification:** Microsoft SC-300 (Identity and Access Administrator)
> **Environment:** Microsoft Entra ID (M365 Developer Program Tenant)
> **Status:** ✅ Complete

---

## Objective

Configure the authentication layer of the tenant by enabling and hardening MFA through the modern Authentication Methods Policy, building a tiered authentication strength model, and defining named locations that will act as signals in Conditional Access policies in Phase 3.

This phase establishes **what authentication methods are available** and **where users are signing in from** — both of which are conditions that Conditional Access evaluates before granting access.

---

## Environment

| Setting | Value |
|---|---|
| Tenant type | Microsoft Entra ID (M365 E5 Dev Program) |
| Licenses required | Free tier (MFA methods, Named Locations, Auth Strengths) |
| Identity Protection (risk-based) | Requires Entra ID P2 — configured in Phase 3 |

---

## What I Built

### 2A — Authentication Method Policy

Configured authentication methods using the modern **Authentication Methods Policy** blade (`Security → Authentication methods → Policies`) rather than the legacy per-user MFA portal. This is the Microsoft-recommended approach and aligns with current SC-300 exam objectives.

| Method | Status | Scope | Key Settings |
|---|---|---|---|
| Microsoft Authenticator | ✅ Enabled | All users | Number matching ON, app name ON, location ON |
| Temporary Access Pass (TAP) | ✅ Enabled | SG-Admins | 60 min default, 480 min max, one-time use |
| FIDO2 Security Key | ✅ Enabled | All users | Default settings |
| SMS | ❌ Disabled | — | Excluded — SIM-swap risk |
| Voice call | ❌ Disabled | — | Excluded — low assurance |

Validated MFA registration end-to-end as **Alex Chen** via `aka.ms/mfasetup`, confirming Microsoft Authenticator (number matching) and email were both registered. Confirmed registration status via the **User registration details** report (`Security → Authentication methods → User registration details`).

### 2B — Authentication Strengths

Created a tiered authentication strength model with two custom strengths on top of the three Microsoft built-in strengths.

**Built-in strengths (Microsoft-provided):**

| Strength | Allowed Methods |
|---|---|
| Multifactor authentication | Broad set — Authenticator push, TOTP, SMS, voice, FIDO2 |
| Passwordless MFA | Authenticator passwordless sign-in, FIDO2, passkey |
| Phishing-resistant MFA | FIDO2, passkey, certificate-based authentication (CBA) |

**Custom strengths created:**

| Strength Name | Allowed Methods | Assigned To |
|---|---|---|
| Privileged Admin Auth | FIDO2, CBA (multi-factor) only | SG-Admins (Phase 3 CA policy) |
| Developer Standard Auth | Authenticator (push + passwordless), FIDO2 | SG-Developers (Phase 3 CA policy) |

### 2C — Named Locations

Created two named locations to act as Conditional Access signals in Phase 3.

| Location Name | Type | Setting | Purpose |
|---|---|---|---|
| Trusted Office Network | IP ranges | Marked as trusted ✅ | Reduced friction from known network |
| High-Risk Countries | Countries | Include unknown regions ✅ | Block access signal for Phase 3 |

**Countries included in High-Risk Countries:** North Korea, Iran, Russia, China, Belarus.

Validated that location signals resolve correctly by inspecting a sign-in event in `Monitoring → Sign-in logs` and confirming the Location tab showed Australia with the expected IP range.

---

## Key Design Decisions

**Why use the Authentication Methods Policy blade instead of per-user MFA?**
The legacy per-user MFA portal is being deprecated by Microsoft. The Authentication Methods Policy provides a centralised, group-scoped, method-level control plane — it is the current recommended approach and is what SC-300 exams test. Configuring MFA here rather than per-user also means enforcement can be delegated to Conditional Access policies (Phase 3), which gives far more granular control over when MFA is required and with what strength.

**Why enable Number Matching on Microsoft Authenticator?**
Without number matching, MFA push notifications can be exploited via MFA fatigue attacks (also called push bombing): an attacker who has obtained valid credentials repeatedly triggers push notifications until a fatigued or distracted user accidentally approves one. Number matching requires the user to enter a 2-digit code displayed on the sign-in screen before the Authenticator notification can be approved — the push alone is not sufficient. Microsoft enabled this by default for all tenants in 2023 following a wave of high-profile push bombing incidents.

**Why was Temporary Access Pass (TAP) scoped to SG-Admins only?**
TAP is a time-limited, one-time-use passcode issued by an administrator. It is most valuable for privileged account bootstrapping (setting up MFA on a new admin account before the admin has an authenticator device) and for break-glass recovery (regaining access without an existing MFA method). Scoping it to SG-Admins reduces the blast radius if a TAP is issued incorrectly — a standard user receiving a TAP cannot self-elevate beyond their normal permissions.

**What is the difference between Phishing-resistant MFA, Passwordless MFA, and Standard MFA?**
This is a commonly tested SC-300 distinction:

- **Standard MFA** (Authenticator push, TOTP codes, SMS) can be intercepted by adversary-in-the-middle (AiTM) proxies such as Evilginx. The attacker relays credentials and the MFA code in real time, capturing a valid session token.
- **Passwordless MFA** (Authenticator phone sign-in) removes the password but the push notification approval can still be proxied — the channel is phishable even without a traditional password.
- **Phishing-resistant MFA** (FIDO2, passkeys, Certificate-Based Authentication) binds the credential cryptographically to the origin domain. A FIDO2 key registered for `login.microsoftonline.com` will not respond to a spoofed login page — the origin check fails at the hardware level. This is the only category that defeats AiTM attacks.

Under the Australian Cyber Security Centre (ACSC) Essential Eight framework, phishing-resistant MFA is required to achieve **Maturity Level 3**.

**Why create two custom authentication strengths instead of using built-in ones?**
The built-in 'Phishing-resistant MFA' strength is appropriate for admins but excludes Authenticator push entirely — too restrictive for developer day-to-day workflows. The built-in 'Multifactor authentication' strength is too permissive (it allows SMS and voice call). Creating custom strengths allows a deliberate tiered model: administrators are held to a higher bar (FIDO2 or CBA only) while developers get strong but practical methods (Authenticator with number matching or FIDO2). This models a real enterprise policy.

**Why mark the Trusted Office Network location as trusted?**
The 'Mark as trusted location' flag is distinct from simply defining an IP range. Conditional Access policies can use 'All trusted locations' as a shorthand condition — this only matches locations explicitly marked as trusted. A named location without the trusted flag will not match this condition, even if the user's IP is within the defined range. Countries-based locations cannot be marked as trusted — only IP range locations have this option.

**Why include unknown countries/regions in the High-Risk Countries location?**
VPN exit nodes, Tor exit relays, and cloud proxy services often resolve to no specific country (or return a private/reserved IP that maps to no geography). Excluding unknown regions would allow an attacker using an anonymising service to bypass a country-based block. Including unknowns ensures the block applies to unattributable traffic, which is itself a risk signal.

---

## Evidence

| # | Screenshot | Description |
|---|---|---|
| 1 | `mfa/01-auth-methods-policy-overview.png` | Authentication Methods Policy blade — full method list before changes |
| 2 | `mfa/02-authenticator-policy-configured.png` | Authenticator policy — number matching and contextual info enabled |
| 3 | `mfa/03-tap-policy-configured.png` | TAP policy — scoped to SG-Admins, one-time use, 60 min default |
| 4 | `mfa/04-alex-mfa-registered-methods.png` | Alex Chen security info — Authenticator and email both registered |
| 5 | `mfa/05-mfa-registration-report.png` | User registration details — confirms MFA registered for test users |
| 6 | `auth-strengths/06-builtin-auth-strengths.png` | Three built-in authentication strengths |
| 7 | `auth-strengths/07-custom-strength-privileged-admin.png` | Privileged Admin Auth — FIDO2 and CBA only |
| 8 | `auth-strengths/08-custom-strength-developer.png` | Developer Standard Auth — Authenticator and FIDO2 |
| 9 | `auth-strengths/09-auth-strengths-complete-list.png` | Complete list — 3 built-in + 2 custom strengths |
| 10 | `named-locations/10-named-location-trusted-office.png` | Trusted Office Network — IP range, marked as trusted |
| 11 | `named-locations/11-named-location-high-risk-countries.png` | High-Risk Countries — country list, include unknowns |
| 12 | `named-locations/12-named-locations-complete-list.png` | Named locations summary — both locations, types visible |
| 13 | `named-locations/13-signin-log-location-verified.png` | Sign-in log confirming location resolution to Australia |

![Authentication Strengths List](./auth-strengths/09-auth-strengths-complete-list.png)
*Tiered authentication strength model — 3 built-in strengths plus 2 custom strengths covering admin and developer tiers.*

![Authenticator Policy — Number Matching](./mfa/02-authenticator-policy-configured.png)
*Microsoft Authenticator policy with number matching enabled to prevent MFA fatigue attacks.*

![Named Locations Summary](./named-locations/12-named-locations-complete-list.png)
*Named locations providing trusted network and high-risk country signals for Conditional Access in Phase 3.*

---

## SC-300 Domain Coverage

| SC-300 Objective | Covered in this phase |
|---|---|
| Plan and implement authentication methods | ✅ Modern auth method policy, Authenticator, TAP, FIDO2 |
| Implement and manage authentication strengths | ✅ Built-in review, 2 custom strengths with tiered model |
| Plan Conditional Access policies (signals) | ✅ Named locations — IP-based trusted + country-based block |
| Monitor and report on security posture | ✅ User registration details report, sign-in log validation |

---

## What I Learned

- **The Authentication Methods Policy and the legacy per-user MFA portal are separate systems** — changes in one do not automatically reflect in the other. In a real tenant migrating from legacy to modern, you need to explicitly migrate users and disable the legacy system to avoid a split-brain configuration.
- **Number matching became default-enabled by Microsoft in May 2023** — if your tenant was created after this date, it may already be on. Still worth explicitly confirming in the policy blade and documenting, since some exam questions test whether you know it exists and why it matters.
- **TAP can only be used once per user if configured as one-time use** — if a user fails their MFA registration and needs a new TAP, an admin must issue a new one. Plan for this in any onboarding workflow design.
- **The 'Include unknown countries' checkbox on country-based locations is easy to miss** — and leaving it unchecked creates a meaningful gap if attackers use VPNs or Tor. This is the kind of implementation detail that differentiates a careful engineer from someone who just clicks through the wizard.
- **Authentication strengths are a Grant control in CA, not a Condition** — this matters for exam questions. You assign a strength in the Grant section of a CA policy, not under Conditions. Named locations are a Condition. Easy to mix up.

---

## Phase Dependencies

The outputs of this phase are directly consumed by Phase 3:

| Phase 2 Output | Used in Phase 3 |
|---|---|
| Custom strength: Privileged Admin Auth | CA Policy 3 — Admin phishing-resistant enforcement |
| Custom strength: Developer Standard Auth | CA Policy 1 — MFA for all users (grant control) |
| Named location: Trusted Office Network | CA Policy — location-based condition |
| Named location: High-Risk Countries | CA Policy — country block condition |

---

## Previous & Next Phase

⬅️ [Phase 1 — Tenant & Identity Foundation](../phase-1/README.md)

➡️ [Phase 3 — Conditional Access Policies](../phase-3/README.md)

Deploy Conditional Access policies using the authentication strengths and named locations configured in this phase, including risk-based sign-in blocking, legacy authentication blocking, and phishing-resistant enforcement for administrators.

---

*Part of the [Zero Trust Identity Governance Lab](../README.md) — an SC-300 portfolio project built in Microsoft Entra ID.*
