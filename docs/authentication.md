# Authentication

This page documents the authentication model currently used by the deployed NoverFly stack.

## Auth Modes

NoverFly currently uses three practical authentication modes.

| Mode | Used by | Header |
|---|---|---|
| Dashboard JWT | Admin dashboard, tenants, sites, API key management | `Authorization: Bearer ...` |
| Site-user JWT | End users of a specific site/app | `Authorization: Bearer ...` |
| API key | External integrations and automations | `X-Api-Key: gfk_...` or `X-Api-Key: gfc_...` |

---

## 1. Dashboard JWT

Use dashboard JWT for tenant and site management.

### Login

```http
POST /v1/auth/login
Content-Type: application/json

{
  "email": "user@example.com",
  "password": "SecureP@ss123"
}
```

### Typical dashboard routes

| Route family | Notes |
|---|---|
| `/v1/auth/*` | Dashboard auth |
| `/v1/tenants/*` | Tenant management |
| `/v1/tenants/:tenantId/sites` | Site creation and listing |
| `/v1/sites/:siteId` | Site details and updates |
| `/v1/sites/:siteId/api-keys*` | API key management |
| `/v1/sites/:siteId/publish` | Publish flow |

Use `X-Tenant-Id` only when the dashboard route contract requires tenant context.

---

## 2. Site-user JWT

Use site-user JWT for end users inside a site or application.

Main routes:

| Method | Route |
|---|---|
| `POST` | `/v1/app/:siteId/auth/register` |
| `POST` | `/v1/app/:siteId/auth/login` |
| `GET` | `/v1/app/:siteId/auth/me` |
| `PATCH` | `/v1/app/:siteId/auth/me` |
| `POST` | `/v1/app/:siteId/auth/refresh` |
| `POST` | `/v1/app/:siteId/auth/logout` |

This auth mode is scoped to a single site.

---

## 3. API Keys

API keys are the standard way to access the developer APIs from external backend code.

There are two key families:

| Key family | Prefix | API family | Main routes |
|---|---|---|---|
| Secret key | `gfk_` | Data API | `/v1/api/data/*` and `/api/*` |
| Cloud key | `gfc_` | Cloud API | `/v1/api/cloud/*` |

Important:

- `gfk_` and `gfc_` are already used by real production integrations.
- Both are served from `https://api.noverfly.com`.
- Do not send `X-Tenant-Id` with `gfk_` or `gfc_` requests.
- Do not send `X-API-Secret` for the current deployed developer APIs.

### Permissions

| Level | Meaning |
|---|---|
| `READ` | Read-only |
| `READ_WRITE` | Read plus create/update/delete |
| `ADMIN` | Includes collection management and other sensitive actions |

`ADMIN` is a permission level, not a different key format.

### Managing keys from the dashboard

Key management is done with dashboard JWT auth.

| Method | Route | Purpose |
|---|---|---|
| `POST` | `/v1/sites/:siteId/ensure-api-keys` | Create default `gfk_` and `gfc_` keys if missing |
| `POST` | `/v1/sites/:siteId/api-keys` | Create a key |
| `GET` | `/v1/sites/:siteId/api-keys` | List keys |
| `GET` | `/v1/sites/:siteId/api-keys/:keyId/reveal` | Reveal full key |
| `PATCH` | `/v1/sites/:siteId/api-keys/:keyId` | Activate or deactivate |
| `DELETE` | `/v1/sites/:siteId/api-keys/:keyId` | Revoke |

### Create a key

```http
POST /v1/sites/:siteId/api-keys
Authorization: Bearer YOUR_ACCESS_TOKEN
Content-Type: application/json

{
  "name": "Backend production",
  "keyType": "SECRET",
  "permission": "ADMIN",
  "rateLimit": 60
}
```

### Example: use `gfk_` on the Data API

```bash
curl https://api.noverfly.com/v1/api/data/collections \
  -H "X-Api-Key: gfk_YOUR_SECRET_KEY"
```

### Example: use `gfc_` on the Cloud API

```bash
curl -X POST https://api.noverfly.com/v1/api/cloud/upload \
  -H "X-Api-Key: gfc_YOUR_CLOUD_KEY" \
  -H "Content-Type: application/json" \
  -d '{"filename":"photo.jpg","mime_type":"image/jpeg","size_bytes":245000}'
```

---

## Security Notes

- Never expose dashboard JWT tokens in frontend bundles.
- Never expose `gfk_` or `gfc_` keys in frontend bundles.
- Use server-side code for all developer API calls.
- Rotate keys when they are no longer needed.
- Prefer the minimum permission level required.

---

## Summary Rule

- Use dashboard JWT for admin and management routes.
- Use site-user JWT for end users of a site.
- Use `gfk_` for structured data.
- Use `gfc_` for storage and media services.
