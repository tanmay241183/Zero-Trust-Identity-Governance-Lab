# Phase 7 — App Registration & OAuth 2.0

> **Project:** Zero Trust Identity Governance Lab
> **Certification:** Microsoft SC-300 (Identity and Access Administrator)
> **Environment:** Microsoft Entra ID (M365 Developer Program Tenant)
> **Status:** ✅ Complete

---

## Objective

Register an enterprise application in Microsoft Entra ID, configure delegated and application API permissions with admin consent, implement application-layer Role-Based Access Control using custom app roles, and validate both the Authorization Code and Client Credentials OAuth 2.0 flows with JWT token inspection.

This phase demonstrates the identity layer that underpins every application integrated with Entra ID — closing the project with the developer and architect perspective that complements the administrator and governance work of Phases 1–6.

---

## Environment

| Setting | Value |
|---|---|
| Tenant type | Microsoft Entra ID (M365 E5 Dev Program) |
| License required | Free tier — no P2 required |
| Application name | ZeroTrustLabApp |
| Redirect URI | https://jwt.ms (Microsoft token decoder) |
| Tools used | Entra admin portal, Postman, jwt.ms, PowerShell |

---

## Object Model — App Registration vs Service Principal

Before building, it is important to establish the distinction between the two objects involved:

```
App Registration (Application object)
│ Created once. Defines the app's identity globally.
│ Contains: Client ID, redirect URIs, API permissions
│ requested, app roles defined, certificates and secrets.
│
└── Service Principal (Enterprise application)
Created automatically in each tenant that registers
or consents to the app.
Contains: Permissions actually granted, user/group
assignments, SSO settings, sign-in activity.
```

**Single-tenant app:** one App Registration → one Service Principal in the same tenant.
**Multi-tenant app:** one App Registration → one Service Principal per tenant that consents.

This lab registers a single-tenant app — the most appropriate model for internal enterprise tooling.

---

## What I Built

### 7A — App Registration

| Setting | Value |
|---|---|
| Application name | ZeroTrustLabApp |
| Supported account types | Single tenant (this directory only) |
| Redirect URI | https://jwt.ms (Web platform) |
| Application (client) ID | [redacted] |
| Directory (tenant) ID | [redacted] |

**Why single-tenant?** Single-tenant restricts authentication to users in this directory only. Multi-tenant would allow any Microsoft account from any organisation to attempt sign-in — appropriate for SaaS products, not internal tooling. Choosing the most restrictive account type by default is a secure-by-design decision.

---

### 7B — API Permissions & Admin Consent

**Delegated permissions** (act on behalf of a signed-in user):

| Permission | API | Admin consent required |
|---|---|---|
| User.Read | Microsoft Graph | No — user self-consents |
| User.ReadBasic.All | Microsoft Graph | Yes |
| GroupMember.Read.All | Microsoft Graph | Yes |
| Mail.Send | Microsoft Graph | No — but pre-consented |

**Application permissions** (act as the application itself, no user present):

| Permission | API | Admin consent required |
|---|---|---|
| User.Read.All | Microsoft Graph | Yes — always for app permissions |
| Group.Read.All | Microsoft Graph | Yes |
| AuditLog.Read.All | Microsoft Graph | Yes |

Admin consent was granted for all permissions via `API permissions → Grant admin consent for [tenant]`. Status confirmed: all permissions showing green tick — *Granted for [tenant name]*.

---

### 7C — App Roles (Application-Layer RBAC)

Two custom app roles were defined in the application manifest:

| Role display name | Value (token claim) | Member types | Description |
|---|---|---|---|
| Analytics Reader | `Analytics.Read` | Users/Groups | Read access to analytics dashboards |
| Data Writer | `Data.Write` | Users/Groups | Write access to data pipeline configurations |

**Role assignments (Service Principal → Users and groups):**

| Assignee | Assignment type | Role granted |
|---|---|---|
| SG-Developers | Group | Analytics Reader (Analytics.Read) |
| Priya Sharma | Direct user | Data Writer (Data.Write) |

**Effective permissions by user:**

| User | Roles in token |
|---|---|
| Priya Sharma | Analytics.Read (via SG-Developers) + Data.Write (direct) |
| James Foster | Analytics.Read (via SG-Developers) |
| Marcus O'Brien | None — not assigned to app |

---

### 7D — Application Credentials

| Credential | Type | Description | Expiry |
|---|---|---|---|
| ZeroTrustLabApp-Secret-01 | Client secret | Used for client credentials flow testing | 6 months |
| ZeroTrustLabApp-Cert-01 | Self-signed certificate | Uploaded for certificate-based auth demonstration | 1 year |

**Secret vs certificate — production guidance:** Client secrets are passwords that can be leaked through logs, environment variables, or code repositories. Certificates use public-key cryptography — the private key never leaves the machine or key vault, and a signed JWT assertion is sent to the token endpoint instead of the secret value. For production workloads, Microsoft recommends certificates or managed identities over client secrets.

---

### 7E — OAuth 2.0 Flow Testing

#### Flow 1 — Authorization Code (Delegated)

Constructed the authorization URL manually and signed in as Priya Sharma. Token decoded at `jwt.ms`.

**Key claims in the delegated access token:**

| Claim | Value | Significance |
|---|---|---|
| `idtyp` | `user` | Confirms this is a user token |
| `upn` | priya.sharma@… | Signed-in user's UPN |
| `scp` | `User.Read GroupMember.Read.All` | Delegated scopes consented to |
| `roles` | `Analytics.Read Data.Write` | App roles held by Priya |
| `aud` | https://graph.microsoft.com | Token intended for Graph API |

The `roles` claim confirms that app roles configured in Section 7C are being delivered in the token — the RBAC model is working end-to-end.

#### Flow 2 — Client Credentials (Application)

Sent a POST request from Postman to the token endpoint with `grant_type=client_credentials`. Token decoded at `jwt.ms`.

**Key claims in the application access token:**

| Claim | Value | Significance |
|---|---|---|
| `idtyp` | `app` | Confirms this is an application token — no user |
| `scp` | *Not present* | No delegated scopes — app tokens use `roles` for permissions |
| `roles` | `User.Read.All Group.Read.All AuditLog.Read.All` | Application permissions granted via admin consent |
| `oid` | Service principal object ID | The app's identity, not a user's |
| `appid` | ZeroTrustLabApp client ID | The requesting application |

The absence of `scp` and presence of `roles` (application permissions) in the app token is the definitive visual proof of the delegated vs application permission distinction.

#### Graph API Call

Used the client credentials access token to call `GET /v1.0/groups` in Postman. Received a `200 OK` response containing all tenant groups — including `SG-Admins`, `SG-Developers`, and `Project Alpha Team` created in Phase 1.

This closes the full project loop: groups provisioned in Phase 1 are readable via the application identity registered in Phase 7, using permissions granted and consented in Phase 7B.

---

### 7F — Access Control: Assignment Required

| Setting | Value |
|---|---|
| Assignment required | Yes |
| Visible to users | Yes (appears in MyApps portal) |

**Why this matters:** By default, any user in the tenant can authenticate to a registered application — regardless of whether they have a role assignment. Enabling `Assignment Required = Yes` restricts sign-in to only users and groups explicitly listed in the Enterprise application's Users and groups tab.

**Validation:** Attempted sign-in as `marcus.obrien@…` (not assigned to ZeroTrustLabApp). Received error:

> `AADSTS50105: The signed-in user is not assigned to a role for the application.`

This confirms the access control is enforced at the authentication layer — Marcus cannot even present credentials to the application. The error screenshot is the evidence that the control has teeth, not just configuration.

---

## Key Design Decisions

**Why use `https://jwt.ms` as the redirect URI?**
`jwt.ms` is a Microsoft-owned token inspection tool that decodes JWT access tokens and displays all claims in a human-readable format. Using it as the redirect URI for lab testing means the token is automatically decoded after a successful authorization code flow — no additional tooling required. In a real application, the redirect URI would be the application's own callback endpoint. `jwt.ms` is safe for lab use as it is operated by Microsoft and the tokens are not stored or shared.

**Why configure both delegated and application permissions?**
The two permission types serve fundamentally different scenarios. Delegated permissions enable user-facing features — reading the signed-in user's profile, sending email on their behalf, reading groups they belong to. Application permissions enable automation and background services — batch processing of user data, reading audit logs for SIEM ingestion, synchronising group memberships to external systems. An IAM engineer needs to understand both models and know when to use each. Configuring both in the same application demonstrates that understanding.

**Why use app roles rather than groups for application-layer authorisation?**
Groups appear in tokens as GUIDs via the `groups` claim — unintelligible to application code without a Graph API lookup. App roles appear as meaningful string values (`Analytics.Read`) that application code can directly evaluate. Additionally, tenants with many groups can hit the 200-group claim limit in tokens, causing groups to be omitted silently. App roles have no such limit. App roles are also scoped to the application — they don't bleed into other applications' authorisation decisions the way tenant-wide groups can.

**Why assign `Analytics.Read` via a group and `Data.Write` directly to Priya?**
This demonstrates two provisioning patterns that coexist in real environments. Group-based assignment scales — when a new developer joins SG-Developers, they automatically inherit `Analytics.Read` without any additional action. Direct assignment is appropriate for exception cases — Priya has an additional elevated role that other developers don't need. Both patterns produce the same mechanism (claims in the JWT token) but have different management implications. In your token inspection, you can see both `Analytics.Read` and `Data.Write` in Priya's `roles` claim.

**Why set a 6-month expiry on the client secret rather than 24 months?**
Shorter secret lifetimes reduce the window of exposure if a secret is leaked. A 24-month secret that is compromised on day one can be used for nearly two years. A 6-month secret limits that window significantly. In production, secrets should be rotated even more frequently — 3 months is a common enterprise standard — and managed via Azure Key Vault with automatic rotation rather than manual copy-paste. The 6-month choice is a deliberate signal that you understand secret lifecycle management.

**How does this phase connect to Zero Trust principles?**
This phase applies Zero Trust at the application layer:

- **Verify explicitly:** Every token contains claims (user identity, app roles, granted scopes) that the application must verify before granting access. No implicit trust based on network location.
- **Least privilege:** API permissions are scoped to the minimum required. `AuditLog.Read.All` is granted as an application permission for a monitoring use case — not `Directory.ReadWrite.All`. App roles are scoped to specific capabilities, not broad access.
- **Assume breach:** `Assignment Required = Yes` limits blast radius — a compromised account outside SG-Developers cannot authenticate to the application even with valid credentials.

---

## JWT Token Comparison

| Claim | Delegated token (Priya) | Application token | Why it matters |
|---|---|---|---|
| `idtyp` | `user` | `app` | Definitive token type differentiator |
| `scp` | `User.Read GroupMember.Read.All` | *Not present* | Delegated scopes — only in user tokens |
| `roles` | `Analytics.Read Data.Write` | `User.Read.All Group.Read.All AuditLog.Read.All` | App roles (user) vs application permissions (app) |
| `upn` | priya.sharma@… | *Not present* | User identity — absent in app tokens |
| `oid` | Priya's object ID | Service principal object ID | Whose identity this token represents |
| `appid` | ZeroTrustLabApp client ID | ZeroTrustLabApp client ID | The requesting application — same in both |
| `aud` | https://graph.microsoft.com | https://graph.microsoft.com | Target API — same in both |

---

## Evidence

| # | Screenshot | Description |
|---|---|---|
| 1 | `01-app-registrations-overview.png` | App registrations blade — before state |
| 2 | `app-registration/02-app-registration-form.png` | Registration form — single-tenant, jwt.ms redirect URI |
| 3 | `app-registration/03-app-overview-identifiers.png` | App overview — client ID and tenant ID (redacted) |
| 4 | `api-permissions/04-delegated-permissions-added.png` | Delegated permissions — before consent (Not granted) |
| 5 | `api-permissions/05-application-permissions-added.png` | Both permission types visible — before consent |
| 6 | `api-permissions/06-admin-consent-granted.png` | All permissions — green tick, Granted status |
| 7 | `app-roles/07-app-roles-created.png` | Both app roles — Analytics.Read and Data.Write |
| 8 | `app-roles/08-app-role-group-assigned.png` | SG-Developers assigned to Analytics Reader role |
| 9 | `app-roles/09-app-role-assignments-complete.png` | Both assignments — group and individual |
| 10 | `credentials/10-client-secret-created.png` | Client secret — description and 6-month expiry (value redacted) |
| 11 | `credentials/11-cert-and-secret-both.png` | Both credential types — secret and certificate |
| 12 | `oauth-flows/12-jwt-token-decoded-authcode.png` | Delegated token decoded — scp and roles claims visible |
| 13 | `oauth-flows/13-postman-token-request.png` | Client credentials POST — 200 OK with access_token |
| 14 | `oauth-flows/14-jwt-client-credentials-token.png` | Application token decoded — idtyp=app, no scp claim |
| 15 | `oauth-flows/15-graph-api-call-success.png` | Graph API call — 200 OK response listing tenant groups |
| 16 | `access-control/16-assignment-required-enabled.png` | Enterprise app Properties — Assignment required = Yes |
| 17 | `access-control/17-unassigned-user-blocked.png` | AADSTS50105 error — Marcus O'Brien blocked |
| 18 | `access-control/18-enterprise-app-final-overview.png` | Enterprise app overview — sign-in activity and final state |

![Delegated Token — jwt.ms](./oauth-flows/12-jwt-token-decoded-authcode.png)
*Decoded authorization code token for Priya Sharma — scp claim (delegated scopes) and roles claim (Analytics.Read, Data.Write) confirming the RBAC model is working in the token.*

![Client Credentials Token — jwt.ms](./oauth-flows/14-jwt-client-credentials-token.png)
*Decoded client credentials token — idtyp=app, no scp claim, roles showing application permissions granted via admin consent.*

![Graph API Call — 200 OK](./oauth-flows/15-graph-api-call-success.png)
*Postman GET /v1.0/groups returning 200 OK — tenant groups from Phase 1 readable via the application identity from Phase 7.*

---

## SC-300 Domain Coverage

| SC-300 Objective | Covered in this phase |
|---|---|
| Register and manage applications | ✅ Single-tenant app registration, redirect URI, identifiers |
| Plan and implement API permissions and admin consent | ✅ Delegated and application permissions, admin consent workflow |
| Implement app roles and group claims | ✅ Two custom app roles, group and direct assignment, token validation |
| Manage application credentials | ✅ Client secret with expiry, self-signed certificate upload |
| Implement OAuth 2.0 authorization flows | ✅ Authorization code flow (delegated) + client credentials flow (application) |
| Monitor and manage enterprise applications | ✅ Assignment required, unassigned user block, sign-in activity |

---

## What I Learned

- **The App Registration and the Enterprise application (Service Principal) are distinct objects configured in different portal blades.** App roles are defined in the App Registration (App roles blade). Role assignments are made in the Enterprise application (Users and groups blade). Confusing these two blades is the most common mistake in this area — the portal navigation itself is designed to trip you up.
- **Admin consent must be granted before application permissions take effect.** Creating an application permission in the App Registration does not grant it — it is a request. Admin consent is the approval. Without it, the token endpoint will return a token without the application permission claims, and Graph API calls will return 403.
- **The `idtyp` claim is the fastest way to distinguish a delegated token from an application token.** `user` = a person signed in. `app` = the application itself. This claim was introduced specifically to make this distinction programmatic — application code can check `idtyp` to verify it received the correct token type before proceeding.
- **`Assignment Required = Yes` is not enabled by default and is not enforced by any Conditional Access policy.** It must be manually enabled on the Service Principal. In a tenant with many app registrations, auditing this setting across all apps is a meaningful security control — all apps should be evaluated for whether open access to all tenant members is appropriate.
- **The Graph API call using the client credentials token seeing Phase 1 groups in the response was a rewarding validation moment.** `SG-Admins`, `SG-Developers`, and `Project Alpha Team` — created in Phase 1 as the identity foundation — appeared in the JSON response from an API call authenticated with the application identity built in Phase 7. This is the full-stack identity story: user provisioning → group structure → conditional access → PIM → entitlement management → access reviews → application integration.
- **Client secrets should never be committed to a Git repository.** Even in a lab environment. Use environment variables, Azure Key Vault references, or a `.env` file with `.gitignore` entries. The habit of keeping secrets out of source control is more important than any individual configuration step.

---

## Project Complete — Full Phase Summary

| Phase | Topic | Key Evidence |
|---|---|---|
| 1 | Users & Groups | 7 users, 5 groups, SSPR audit log |
| 2 | MFA Hardening | Auth methods policy, 2 custom strengths, named locations |
| 3 | Conditional Access | 4 CA policies, Tor risk event, legacy auth block |
| 4 | PIM | JIT activation lifecycle, approval workflow, PIM audit log |
| 5 | Entitlement Management | 2 access packages, My Access portal workflow, 90-day contractor expiry |
| 6 | Access Reviews | 3 review types, denial with auto-removal, PIM assignment removed |
| 7 | App Registration | OAuth 2.0 flows, JWT token inspection, RBAC via app roles |

---

## Previous Phase

⬅️ [Phase 6 — Access Reviews & Governance](../phase-6/README.md)

---

*Part of the [Zero Trust Identity Governance Lab](../README.md) — an SC-300 portfolio project built in Microsoft Entra ID.*
