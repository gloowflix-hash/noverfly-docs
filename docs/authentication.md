# Authentication

This page documents the authentication model currently used by the deployed NoverFly stack.

## Auth Overview

NoverFly currently uses four practical auth surfaces.

| Surface | Used by | Main entrypoint |
|---|---|---|
| Dashboard JWT | Admin dashboard, tenants, sites, API key management | `/v1/auth/*` |
| Developer API keys | External integrations and automations | `X-Api-Key: gfk_...` or `X-Api-Key: gfc_...` |
| App auth through Data API | Headless apps where the site is resolved by the `gfk_` key | `/v1/api/data/auth/*` or `/api/auth/*` |
| Site-user JWT | End users inside a specific site or app | `/v1/app/:siteId/auth/*` |

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

Use `X-Tenant-Id` only when the dashboard route contract explicitly requires tenant context.

---

## 2. Developer API Keys

API keys are the standard way to access the developer APIs from external code.

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

---

## 3. App Auth Through the Data API

This is the auth surface many headless apps use in practice.

The site is resolved by the `gfk_` key, so the app does not need to send a `siteId` in the route path.

Bootstrap requests use the key:

```http
X-Api-Key: gfk_YOUR_SECRET_KEY
```

Main routes:

| Method | Route | Notes |
|---|---|---|
| `GET` | `/v1/api/data/auth/config` | Read enabled auth providers for the key's site |
| `POST` | `/v1/api/data/auth/register` | Register a site user |
| `POST` | `/v1/api/data/auth/login` | Login a site user |
| `GET` | `/v1/api/data/auth/me` | Read current profile using bearer token |
| `PATCH` | `/v1/api/data/auth/me` | Update current profile using bearer token |
| `POST` | `/v1/api/data/auth/refresh` | Refresh a site-user session |
| `POST` | `/v1/api/data/auth/logout` | Logout a site-user session |

Short alias:

- `/api/auth/config`
- `/api/auth/register`
- `/api/auth/login`
- `/api/auth/me`
- `/api/auth/refresh`
- `/api/auth/logout`

### Register a user through the Data API

```bash
curl -X POST https://api.noverfly.com/v1/api/data/auth/register \
  -H "X-Api-Key: gfk_YOUR_SECRET_KEY" \
  -H "Content-Type: application/json" \
  -d '{"email":"user@example.com","password":"StrongPass123","displayName":"User"}'
```

### Login through the Data API

```bash
curl -X POST https://api.noverfly.com/v1/api/data/auth/login \
  -H "X-Api-Key: gfk_YOUR_SECRET_KEY" \
  -H "Content-Type: application/json" \
  -d '{"email":"user@example.com","password":"StrongPass123"}'
```

After login, use the returned access token as a bearer token for `/auth/me`, `/auth/refresh`, and `/auth/logout`.

---

## 4. Site-user JWT API

Use site-user JWT for end users inside a site or application when you already know the site ID.

Main routes:

| Method | Route |
|---|---|
| `POST` | `/v1/app/:siteId/auth/register` |
| `POST` | `/v1/app/:siteId/auth/login` |
| `GET` | `/v1/app/:siteId/auth/me` |
| `PATCH` | `/v1/app/:siteId/auth/me` |
| `POST` | `/v1/app/:siteId/auth/refresh` |
| `POST` | `/v1/app/:siteId/auth/logout` |
| `POST` | `/v1/app/:siteId/auth/forgot-password` |
| `POST` | `/v1/app/:siteId/auth/reset-password` |
| `POST` | `/v1/app/:siteId/auth/verify-email` |
| `POST` | `/v1/app/:siteId/auth/resend-verification` |
| `POST` | `/v1/app/:siteId/auth/mfa/totp/setup` |
| `POST` | `/v1/app/:siteId/auth/mfa/totp/verify` |
| `POST` | `/v1/app/:siteId/auth/mfa/totp/disable` |
| `POST` | `/v1/app/:siteId/auth/me/avatar` |
| `GET` | `/v1/app/:siteId/auth/google` |
| `GET` | `/v1/app/:siteId/auth/google/callback` |

There is also a public mirror used by some public/social site flows:

- `/v1/public/sites/:siteId/auth/register`
- `/v1/public/sites/:siteId/auth/login`
- `/v1/public/sites/:siteId/auth/refresh`
- `/v1/public/sites/:siteId/auth/me`
- `/v1/public/sites/:siteId/auth/logout`

This auth mode is scoped to a single site.

---

## Security Notes

- Never expose dashboard JWT tokens in frontend bundles.
- Never publish real `gfk_` or `gfc_` keys in repos or public docs.
- For public production apps, prefer calling your own backend instead of exposing developer API keys directly.
- If you use a frontend-only dev setup, use environment variables and short-lived test keys only.
- Rotate keys when they are no longer needed.
- Prefer the minimum permission level required.

---

## Summary Rule

- Use dashboard JWT for admin and management routes.
- Use `gfk_` for structured data and app-scoped auth bootstrap.
- Use `gfc_` for storage and media services.
- Use site-user bearer tokens for `/auth/me`, `/auth/refresh`, and user-session flows.
