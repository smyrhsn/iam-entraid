### Implicit Grant & Hybrid Checkboxes
**Location:** Entra Admin Center → App Registrations → [App] → Authentication → *Implicit grant and hybrid flows*
 
| Checkbox | Enables | Recommendation | When acceptable |
|---|---|---|---|
| ☑ ID tokens | Hybrid flow — `id_token` from `/authorize` | ✅ Acceptable | Legacy web apps (ASP.NET) needing fast identity |
| ☑ Access tokens | Implicit flow — `access_token` from `/authorize` | ❌ Avoid | Never for new apps — `access_token` exposed in URL |
| ☑ Both | Full Implicit flow | ❌ Avoid | Most insecure — legacy only if no alternative |
| ☐ Neither | Auth Code / PKCE only | ✅ Recommended | All new apps |
 
> **Gap Analysis Note:** The Fail condition should trigger specifically on *Access tokens* being checked — not ID tokens alone. ID tokens only = Hybrid = acceptable for legacy apps.
 
