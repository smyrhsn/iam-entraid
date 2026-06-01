### API Permissions — How Scopes Work in Entra
 
Two sides to scope configuration:
 
| Side | Location | Purpose |
|---|---|---|
| Define scopes (Resource/API app) | App Registrations → [API App] → **Expose an API** | Creates the scopes that exist |
| Grant scopes to client | App Registrations → [Client App] → **API Permissions** | Allows client to request those scopes |
 
| Permission type | Example | Meaning | Admin consent? |
|---|---|---|---|
| Delegated | `User.Read` | App acts on behalf of the signed-in user | Usually No |
| Application | `User.Read.All` | App acts as itself — no user (Client Credentials) | Always Yes |
 
**Standard OIDC scopes in Entra API Permissions:**
 
| Scope | Entra permission | Claims returned | Admin consent? |
|---|---|---|---|
| `openid` | openid (Microsoft Graph) | `sub`, `iss`, basic identity | No |
| `profile` | profile (Microsoft Graph) | `name`, `given_name`, `family_name` | No |
| `email` | email (Microsoft Graph) | `email` | No |
| `offline_access` | offline_access (Microsoft Graph) | Enables refresh token | No |
| `User.Read` | User.Read (Microsoft Graph) | Full profile via Graph `/me` | No |
 
**When you see `invalid_scope` — check in this order:**
1. Is the scope defined on the API app? → *API app → Expose an API*
2. Is the scope added to the client app? → *Client app → API Permissions*
3. Has admin consent been granted? → *Client app → API Permissions → Grant admin consent*
4. Does the scope in the URL exactly match what's registered? *(case-sensitive)*
---
