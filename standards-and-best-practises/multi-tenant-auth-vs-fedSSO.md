# SaaS Application Authentication
### Multi-Tenant Auth (as-is) vs Federated SSO (App Reg in Your Tenant)
*Identity & Access Management | Advisory Reference | Audience: IT, Security & Leadership*

---

## Section 1 — Context: Two Types of Multi-Tenant Apps

Before focusing on the core topic, it is important to clarify that the term 'multi-tenant app' covers two distinct scenarios in Microsoft Entra ID. This document is focused on Scenario B — where your organisation's users are consumers of a SaaS application.

| | **Type A — You Publish the App** | **Type B — You Consume the App (This Document)** |
|---|---|---|
| Who owns the app reg? | Your organisation | SaaS Vendor |
| Who are the users? | Your users + other tenant users | Your users only |
| Typical use case | You are building/hosting an app for customers or partners | Your users log into a third-party SaaS platform |
| Your control level | Full — you own all aspects of the app registration | None — vendor owns all aspects |

> This document focuses entirely on **Type B** — where your users are consumers of a SaaS vendor's application — and evaluates two approaches: leaving auth as-is (vendor-owned) versus creating an app registration in your own tenant for that SaaS app.

---

## Section 2 — The Two Approaches for Type B (SaaS Consumer)

When your users need to access a SaaS application, there are two approaches to how authentication is handled. The difference comes down to whether your organisation owns the app registration for that SaaS app or not.

### Multi-Tenant Auth (as-is)
- SaaS vendor has created the app registration in their own tenant.
- Your users click **'Sign in with Microsoft'** on the SaaS platform.
- Authentication flows through Microsoft's `/common` shared endpoint.
- No app object exists in your tenant for this SaaS app.
- Vendor controls the client ID, secret, redirect URIs, and permissions.
- Your users get access — but your organisation has no ownership.

### Federated SSO (App Reg in Your Tenant)
- Your organisation creates an app registration in your Entra ID tenant.
- You share the Client ID and Secret (or Certificate) with the SaaS vendor.
- Vendor configures their platform to use your SSO endpoint.
- Authentication flows through your tenant's dedicated endpoint.
- Your users click **'Sign in with SSO'** — auth hits your tenant directly.
- You own the app object, credentials, and all associated controls.

---

## Section 3 — What Works the Same in Both Approaches

Several important identity controls function correctly in both approaches. This is because the user's identity resides in your tenant regardless of which approach is used.

| **Control** | **Multi-Tenant Auth (as-is)** | **Federated SSO (App Reg in Your Tenant)** |
|---|---|---|
| Conditional Access Policies | ✅ Applies — enforced on service principal in your tenant | ✅ Applies — enforced on service principal in your tenant |
| Sign-in Logs in Entra ID | ✅ Full visibility for user sign-ins and CA evaluation | ✅ Full visibility for user sign-ins and CA evaluation |
| MFA Enforcement | ✅ Enforced via Conditional Access policy | ✅ Enforced via Conditional Access policy |
| Disabled User Blocked | ✅ Entra ID blocks at the identity layer | ✅ Entra ID blocks at the identity layer |
| Password & Account Lockout Policies | ✅ Enforced at identity layer in your tenant | ✅ Enforced at identity layer in your tenant |
| Token Lifetime Policies | ✅ Configured in your tenant, applied to user session regardless of app reg location | ✅ Configured in your tenant, applied to user session regardless of app reg location |
| Identity Protection (Sign-in & User Risk) | ✅ Entra ID evaluates risk on the user identity, not the app object | ✅ Entra ID evaluates risk on the user identity, not the app object |
| Admin Consent Workflow | ⚠️ Controlled — because user consent is restricted, users cannot consent on their own. Admins approve or reject the scope set the vendor has defined. You hold the veto, not the pen. | ✅ Controlled — because user consent is restricted. Your organisation defines and manages the permission scopes from the start. |

---

## Section 4 — Where They Differ: Security

The meaningful differences emerge in areas of security posture, organisational control, and identity lifecycle management — all of which become increasingly critical as the sensitivity of the SaaS application increases.

| **Security Area** | **Multi-Tenant Auth (as-is)** | **Federated SSO (App Reg in Your Tenant)** |
|---|---|---|
| Secret / Credential Ownership | ❌ Vendor owns and manages the client secret. You cannot enforce rotation frequency, expiry, or storage standards per your security policy. | ✅ You own the secret or certificate. Rotate on your schedule. Your security policy is enforceable end to end. |
| Redirect URI Integrity | ❌ Vendor controls redirect URIs. A misconfiguration or vendor-side compromise can expose your users' authentication tokens. | ✅ You define and control redirect URIs in your tenant. No changes occur without your explicit action. |
| Incident Response | ⚠️ If vendor credentials are compromised, user authentication in your tenant remains unaffected. However, remediation, credential rotation, and service recovery depend entirely on the vendor's detection, response, and timeline. | ✅ You detect and respond immediately. You can revoke or rotate credentials, reconfigure the app, and restore access under your own incident response process and SLA. |
| Trust Boundary | ❌ You are trusting the vendor's security posture to protect the app registration that grants access to your users' sessions. | ✅ Your Entra ID tenant is the trust authority. No dependency on vendor security practices for the authentication layer. |
| User Assignment & Role-Based Access Control | ⚠️ User/group assignment is supported. Role-based access depends on vendor integration — roles are typically passed via claims, SCIM, or app roles defined by the vendor. | ⚠️ User/group assignment is supported. Role-based access depends on app design — roles are defined and managed via claims or app roles in your own app registration. |

---

## Section 5 — Recommendation

Based on the controls and differences outlined in this document, the following approach is recommended:

### ⚠️ Multi-Tenant Auth (as-is) — Acceptable under specific conditions only

Multi-Tenant Auth may be retained where there is a clear business requirement and the SaaS vendor is a well-known, enterprise-grade vendor. It should not be the default approach for every independent SaaS vendor. Where this approach is used, ensure Conditional Access, MFA, and Admin Consent Workflow controls are actively maintained.

### ✅ Federated SSO (App Reg in Your Tenant) — Recommended as the standard approach

For all other SaaS applications — particularly those handling sensitive organisational data — Federated SSO with an app registration in your tenant is the recommended approach. This ensures your organisation owns the trust boundary, controls credentials, and can enforce your security policies end to end.

---

## Section 6 — Next Steps

- Classify your SaaS applications by risk profile — identify which require Federated SSO.
- Request the SaaS vendor to support OIDC or SAML SSO using an app registration you create.
- Create the App Registration in your Entra ID tenant. Generate Client ID and Certificate (preferred over secret).
- Share the credentials with the vendor. Configure SSO on their platform. Disable the generic 'Sign in with Microsoft' option for your users.
- Apply a dedicated Conditional Access policy targeting your owned app registration.
- Establish a credential rotation schedule aligned to your organisation's security policy.
- Integrate the app registration with your IDM / IGA tool via SCIM for full Joiner-Mover-Leaver lifecycle automation.

---

> **⚠️ Important Boundary:** App registration ownership controls who can authenticate to the SaaS platform. It does not protect data already stored within the vendor's infrastructure. Data protection requires complementary controls: Data Processing Agreement (DPA), Vendor Security Assessment, SOC 2 / ISO 27001 certification review, and contractual data residency and breach notification obligations.

---
