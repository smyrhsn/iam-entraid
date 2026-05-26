# Jamf Pro + Microsoft Intune + Entra ID — Device Compliance Configuration Guide

**Approach:** Multi-Tenant Auth (Option A — Jamf-owned app registration)
**Scope:** iOS device compliance via Jamf Pro as a compliance partner for Microsoft Intune

---

## Architecture overview

```
┌──────────────────────────────┐       ┌─────────────────────────────────────────────────────────┐
│     Jamf (vendor boundary)   │       │                    Your Entra tenant                    │
│                              │       │                                                         │
│  ┌────────────────────────┐  │       │  ┌─────────────────────────┐  ┌──────────────────────┐  │
│  │    App registration    │──┼─ consent 1 ──►  Cloud Connector   │  │    User Reg app      │  │
│  │ Multi-tenant, Jamf's   │──┼─ consent 2 ──────────────────────────►  Device Reg SP       │  │
│  │       tenant           │  │       │  │  Device Compliance SP   │  │                      │  │
│  └───────────┬────────────┘  │       │  └───────────┬─────────────┘  └──────────┬───────────┘  │
│              │ issues        │       │              │ enables                   │ creates       │
│              │ tokens to     │◄──────┼── API trust ─┘ pipeline                 │ device object │
│  ┌───────────▼────────────┐  │       │              │                           │ on reg        │
│  │       Jamf Pro         │  │       │              ▼                           │               │
│  │  MDM + compliance      │  │       │   ┌──────────────────────┐              │               │
│  │       logic            │  │       │   │   Microsoft Intune   │              │               │
│  └───────────┬────────────┘  │       │   │  Compliance partner  │              │               │
│              │ compliance    │       │   └──────────┬───────────┘              │               │
└──────────────┼───────────────┘       │              │ writes isCompliant        │               │
               │                       │              ▼                           │               │
               └───────────────────────┼──────► ┌─────────────────────────┐ ◄───┘               │
                    compliance data    │        │      Device object       │                     │
                    (Intune partner    │        │   isCompliant attribute  │                     │
                         API)         │        └────────────┬────────────┘                     │
                                       │                     │ read at sign-in                   │
                                       │                     ▼                                   │
                                       │        ┌─────────────────────────┐                     │
                                       │        │   Conditional Access    │                     │
                                       │        │     grant or block      │                     │
                                       │        └─────────────────────────┘                     │
                                       └─────────────────────────────────────────────────────────┘
```

> **Two enterprise apps are created in your tenant during setup:**
> - **Cloud Connector for Device Compliance** — authenticates Jamf Pro to call the Intune Partner Compliance API. Jamf Pro uses this SP's token to push compliance data to Intune.
> - **User Registration app for Device Compliance** — used by the end user's device during Entra registration via Self Service. Jamf Pro has no direct connection to this SP.
>
> Both are created automatically by the Jamf wizard during admin consent. They cannot be precreated manually.
>
> **Credential ownership:** Jamf owns the app registration in their tenant. You own the two enterprise apps in your tenant but cannot control the underlying credentials on the Jamf side.

---

## Prerequisites

Before starting, confirm the following are in place:

- Jamf Pro instance (cloud or on-prem) with admin access
- Microsoft Entra ID tenant with Global Administrator or Intune Administrator rights
- Microsoft Intune licence (Intune Plan 1 or included in Microsoft 365 Business Premium / EMS E3+)
- iOS devices enrolled in Jamf Pro

---

## Step 1 — Create smart groups in Jamf Pro

Smart groups drive what Jamf reports to Intune. You need two groups per platform.

**Applicable group** — all iOS devices that need access to company resources, regardless of compliance state.

**Compliance group** — devices that meet your defined compliance criteria. Recommended criteria for iOS:

- OS version (minimum acceptable iOS version)
- Jailbreak detected = false
- Passcode status = enabled
- Last inventory update (within acceptable window)

> Enable **Send email notification on membership change** on the compliance group so you are alerted when a device falls out of compliance.

---

## Step 2 — Configure the Microsoft Entra integration in Jamf Pro

1. In Jamf Pro, go to **Settings → Global → Device compliance**.
2. Click **Edit** and enable the integration using the toggle.
3. Select **iOS/iPadOS** as the platform type.
4. Set the **Compliance group** to the group created in Step 1.
5. Set the **Applicable group** to the applicable group created in Step 1.
6. Select your **Sovereign Cloud** location.
7. Choose a landing page for devices not recognised by Entra ID (default Jamf registration page recommended for initial setup).
8. Click **Save**.

Saving redirects you to the Microsoft application registration page to grant permissions — this is the admin consent step.

---

## Step 3 — Grant admin consent (Entra ID)

Two separate consent prompts are presented by the Jamf wizard. Both require a Global Administrator account.

1. **First prompt** — grants permissions for the **Cloud Connector for Device Compliance** app. This creates the SP that Jamf Pro uses to authenticate against the Intune Partner Compliance API.
2. **Second prompt** — grants permissions for the **User Registration app for Device Compliance**. This creates the SP used during end-user device registration with Entra ID.

After both prompts are accepted, you are redirected to the **Configure Compliance Partner** page in Intune.

> These apps are created by the Jamf wizard and cannot be precreated manually. After setup, both appear in **Entra admin centre → Enterprise applications**.

---

## Step 4 — Create the compliance partner in Microsoft Intune

1. On the Configure Compliance Partner page, click **Open Microsoft Endpoint Manager**.
2. In the Intune portal, go to **Tenant administration → Connectors and tokens → Partner compliance management**.
3. Click **Add compliance partner**.
4. Select **Jamf Device Compliance** from the compliance partner menu.
5. Select **iOS/iPadOS** as the platform and click **Next**.
6. Click **Add groups** and select the Entra ID user groups to scope this compliance policy.

   > **Important:** Do not select "Add all users". This breaks the integration.

7. Click **Next**, review the configuration, and click **Create**.
8. Return to the Jamf Cloud Connector tab and click **Confirm**.

Jamf Pro completes and tests the connection. The result displays on the Device Compliance settings page.

---

## Step 5 — Deploy the Microsoft Authenticator app to iOS devices

The Microsoft Authenticator app is required for iOS device registration with Entra ID.

1. In Jamf Pro, go to **Devices → Mobile Device Apps**.
2. Add the **Microsoft Authenticator** app from the App Store.
3. Scope it to the applicable group created in Step 1.
4. Deploy via a Jamf Pro policy.

---

## Step 6 — End-user device registration with Entra ID

Device registration is a user-triggered action. Once the applicable group and Authenticator app are in place:

1. The **Register with Microsoft** button becomes available in **Jamf Self Service** for iOS.
2. The user taps **Register with Microsoft** and authenticates with their Entra ID credentials via the **User Registration app SP**.
3. A **device object** is created in Entra ID.
4. Jamf Pro sends the device's compliance status to Intune via the Intune Partner Compliance API (authenticated via the Cloud Connector SP).
5. Intune writes the `isCompliant` attribute to the device object in Entra ID.

> **Important:** Users must register via **Jamf Self Service**, not the Company Portal app directly. Using Company Portal results in an `AccountNotOnboarded` error.

---

## Step 7 — Create a Conditional Access policy in Entra ID

Create this policy **after** validating that compliance data is flowing correctly (check a test device in Entra ID → Devices and confirm `isCompliant` is set before enabling the policy).

1. In the Entra admin centre, go to **Protection → Conditional Access → Policies**.
2. Click **New policy** and give it a descriptive name (e.g. `Require compliant device — iOS Jamf`).
3. **Assignments:**
   - Users: target the relevant user groups
   - Target resources: select the cloud apps you want to protect (e.g. Microsoft 365)
4. **Conditions:**
   - Device platforms: include **iOS**
5. **Grant:**
   - Select **Require device to be marked as compliant**
6. Set policy to **Report-only** initially to validate, then switch to **On** once confirmed.

> **Critical:** Exclude the **User Registration app for Device Compliance** from any CA policy that requires compliant devices. Without this exclusion, users will be blocked from registering their devices — creating a chicken-and-egg situation where the device cannot become compliant because it cannot register.

---

## Key notes

**Devices will not appear in Intune's device list.** This is expected — Jamf-managed devices appear in Entra ID under Devices, but are not listed under Devices in Intune. The compliance status is visible in Entra ID.

**isCompliant shows N/A after registration.** This indicates the device did not send compliance data successfully. Remove the device record from Entra ID and re-register.

**Two enterprise apps, not one.** Admin consent creates two separate SPs in your tenant — Cloud Connector (compliance pipeline) and User Reg app (device registration). Both are visible under Entra admin centre → Enterprise applications.

**Credential ownership.** Under Option A (multi-tenant auth), Jamf owns the app registration credentials in their tenant. You own the two enterprise apps in your tenant but cannot control rotation or respond independently to a credential compromise on Jamf's side.

---

## Data sync — identity and API layer

This describes how Jamf Pro maintains awareness of your users, groups, and compliance scope. It is not a full device sync — device inventory stays in Jamf.

### One-time setup

1. Admin grants consent in Entra ID → Jamf wizard creates two enterprise apps (Cloud Connector SP + User Reg app SP) in your tenant. API trust is established between Jamf Pro and your tenant via the Cloud Connector SP.
2. Admin scopes user groups in Intune (**Tenant administration → Partner compliance management**) → defines which users Jamf compliance data applies to.

### Recurring sync

3. Jamf Pro requests an access token from Entra ID using the Cloud Connector SP (OAuth client credentials flow).
4. Entra ID issues the access token.
5. Jamf Pro calls the **Microsoft Graph API** using the token → retrieves user and group data from your tenant.
6. Jamf Pro calls the **Intune Partner Compliance API** → reads the group scope defined in step 2.
7. Jamf Pro combines both → applies group scope to compliance evaluation and policy targeting.

### What is and is not synced

| Data | Synced | Direction | Notes |
|---|---|---|---|
| Users | Yes | Entra → Jamf | Via Graph API |
| Groups | Yes | Entra → Jamf | Via Graph API |
| Compliance group scope | Yes | Intune → Jamf | Via partner API |
| Device inventory | No | — | Jamf owns this, never sent to Entra |
| Full automatic sync | No | — | Triggered by Jamf on its own schedule |

> **Group scope resync:** If you add new groups to the compliance partner scope in Intune after initial setup, those changes do not automatically reflect in Jamf. Go to the Partner Compliance Management blade in Intune and click **Edit → Refresh scope** to resync.

---

## Enrollment flow

This describes the end-to-end journey of a device and user from first contact to fully enrolled and registered. Enrollment (MDM into Jamf) and registration (device object in Entra) are two separate actions — a device can be enrolled in Jamf but not yet registered with Entra.

### 1. User starts enrollment

The user opens **Jamf Self Service** on their iOS device and taps **Register with Microsoft**. This button is only visible once the device is in the applicable smart group scoped during configuration.

### 2. User logs in via Entra

The device presents an Entra ID authentication prompt (OIDC). The user signs in with their corporate credentials. This is handled by the **User Registration app for Device Compliance** SP — the second enterprise app created during admin consent.

### 3. Device enrolled into Jamf

The device receives and installs the MDM profile from Jamf Pro. Jamf creates a device record and begins inventory collection. The device is evaluated against the applicable and compliance smart groups.

### 4. Device registered in Entra

On successful Entra authentication, a **device object** is created in Entra ID. The device is now visible under **Entra admin centre → Devices**. At this point `isCompliant` is initially `N/A` — compliance data has not yet been received.

> Jamf Pro then pushes the device's compliance status to Intune via the Partner Compliance API (authenticated via the Cloud Connector SP). Intune writes `isCompliant = true` or `false` to the device object. This usually completes within a few minutes of registration.

### 5. User ↔ device mapping created

Entra ID links the authenticated user identity to the device object. This mapping is what Conditional Access uses at sign-in time — it evaluates both the user (is this user in scope?) and the device (is this device compliant?) together before granting or blocking access.

> **Important:** Users must register via **Jamf Self Service**, not by opening the Company Portal app directly. Using Company Portal results in an `AccountNotOnboarded` error.

---

## Compliance sending flow

This describes how compliance status travels from the device to Entra ID after enrollment, and how Conditional Access enforces it at every sign-in. This cycle repeats continuously on every MDM check-in.

### 1. Device sends inventory to Jamf

The iOS device checks in with Jamf Pro on its MDM schedule. It sends a full inventory update including OS version, passcode status, jailbreak detection, installed apps, and last check-in time.

### 2. Jamf evaluates compliance using smart groups

Jamf Pro evaluates whether the device is a member of the **compliance smart group** based on the inventory data received. This is the core compliance decision — if the device meets all criteria defined in the smart group, it is compliant. If not, it is non-compliant. No external system is involved in this evaluation; Jamf owns it entirely.

### 3. Jamf sends result to Intune via the connector

Jamf Pro calls the **Intune Partner Compliance API**, authenticated using the **Cloud Connector SP** token. It sends the compliance state (compliant or non-compliant) for each registered device in the applicable group.

### 4. Intune processes compliance

Intune receives the compliance payload from Jamf and records the state internally. Intune acts as the intermediary — it does not re-evaluate or override Jamf's compliance decision. It accepts and forwards it.

### 5. Intune updates the Entra device object

Intune writes the `isCompliant` attribute to the **device object** in Entra ID. The device object is updated to reflect `true` or `false`. This is the value that Conditional Access reads.

### 6. Conditional Access enforces policy at sign-in

When the user attempts to access a protected resource (e.g. Microsoft 365), Entra ID evaluates the Conditional Access policy. CA reads the `isCompliant` attribute from the device object and the user identity, then either grants access or blocks it with a remediation prompt.

> Devices managed by Jamf do not appear in the Intune device list. Compliance status is only visible in **Entra admin centre → Devices**.

---

## References

- [Jamf Pro: Device Compliance with Microsoft Entra and Jamf Pro](https://learn.jamf.com/r/en-US/technical-paper-microsoft-intune-current/Device_Compliance_with_Microsoft_Intune_and_Jamf_Pro)
- [Jamf Pro: Configuring the Microsoft Entra Integration](https://learn.jamf.com/r/en-US/technical-paper-microsoft-intune-current/Configuring_the_Microsoft_Intune_Integration)
- [Jamf Pro: Creating a Compliance Partner in Microsoft Intune](https://learn.jamf.com/r/en-US/technical-paper-microsoft-intune-current/Creating_a_Compliance_Partner_in_Microsoft_Intune)
- [Microsoft: Integrate Jamf Pro with Intune to report device compliance](https://learn.microsoft.com/en-us/intune/intune-service/protect/jamf-managed-device-compliance-with-entra-id)
- [Microsoft: Partner compliance management](https://learn.microsoft.com/en-us/mem/intune/protect/device-compliance-partners)
