# NoverFly API Documentation

> Official developer documentation for the NoverFly platform API.
>
> This documentation reflects the API contract currently deployed in production.

![API](https://img.shields.io/badge/API-v1-blue)
![Status](https://img.shields.io/badge/status-active-success)
[![Website](https://img.shields.io/badge/Website-noverfly.com-brightgreen)](https://noverfly.com)
[![API Base URL](https://img.shields.io/badge/API-api.noverfly.com-orange)](https://api.noverfly.com)

---

## Production Model

NoverFly currently exposes one public API host:

- Main host: `https://api.noverfly.com`
- Compatible alias: `https://api.gloowflix.cloud`

On that same host, there are two developer API families:

| API family | Auth | Main routes | Purpose |
|---|---|---|---|
| Data API | `X-Api-Key: gfk_...` | `/v1/api/data/*` and `/api/*` | Collections, records, headless CMS data |
| Cloud API | `X-Api-Key: gfc_...` | `/v1/api/cloud/*` | Upload, assets, media search, import |

Important points:

- `gfk_` and `gfc_` are real production keys already used by existing applications.
- Both APIs go through `api.noverfly.com`.
- There is no separate public host you need to call for the current Cloud API.
- For `gfk_` and `gfc_` routes, the key already resolves the site and tenant scope.
- Do not add `X-Tenant-Id` on developer API requests authenticated by `gfk_` or `gfc_`.

---

## Authentication Overview

NoverFly currently uses three practical auth modes:

| Mode | Typical use | Headers |
|---|---|---|
| Dashboard JWT | Admin dashboard, tenant management, site management | `Authorization: Bearer ...` |
| Site-user JWT | End-user authentication inside a site/app | `Authorization: Bearer ...` |
| API keys | External backend integrations and automations | `X-Api-Key: gfk_...` or `X-Api-Key: gfc_...` |

API key permissions:

| Permission | Meaning |
|---|---|
| `READ` | Read-only |
| `READ_WRITE` | Read + create/update/delete records or assets |
| `ADMIN` | Includes collection management and other sensitive operations |

---

## Quick Start

### 1. Login to the dashboard

```bash
curl -X POST https://api.noverfly.com/v1/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email":"you@example.com","password":"your-password"}'
```

### 2. Create or list a site

```bash
curl https://api.noverfly.com/v1/tenants/YOUR_TENANT_ID/sites \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN"
```

### 3. Ensure default API keys for a site

```bash
curl -X POST https://api.noverfly.com/v1/sites/YOUR_SITE_ID/ensure-api-keys \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN"
```

### 4. Read collections with a `gfk_` key

```bash
curl https://api.noverfly.com/v1/api/data/collections \
  -H "X-Api-Key: gfk_YOUR_SECRET_KEY"
```

### 5. Upload with a `gfc_` key

```bash
curl -X POST https://api.noverfly.com/v1/api/cloud/upload \
  -H "X-Api-Key: gfc_YOUR_CLOUD_KEY" \
  -H "Content-Type: application/json" \
  -d '{"filename":"photo.jpg","mime_type":"image/jpeg","size_bytes":245000}'
```

---

## Documentation Index

| Document | Description |
|---|---|
| [Introduction](docs/introduction.md) | Platform overview and current deployment model |
| [Getting Started](docs/getting-started.md) | Step-by-step onboarding |
| [Authentication](docs/authentication.md) | Dashboard JWT, site-user JWT, `gfk_`, `gfc_`, permissions |
| [API Reference](docs/api.md) | Main route families and integration rules |
| [Database / Data API](docs/database.md) | Collections and records API with `gfk_` |
| [Build Applications](docs/applications.md) | Building apps with sites, collections, auth, and publish flow |
| [CMS](docs/cms.md) | Content modeling and CMS concepts |
| [E-Commerce](docs/ecommerce.md) | Commerce capabilities |
| [Deployment](docs/deployment.md) | Publish flow and delivery |
| [Security](docs/security.md) | Security principles |

---

## Core Route Families

| Category | Routes | Auth |
|---|---|---|
| Dashboard auth | `/v1/auth/*` | JWT |
| Tenants | `/v1/tenants/*` | JWT |
| Sites | `/v1/sites/*` and `/v1/tenants/:tenantId/sites` | JWT |
| API key management | `/v1/sites/:siteId/api-keys*` | JWT |
| Data API | `/v1/api/data/*` and `/api/*` | `gfk_` |
| Cloud API | `/v1/api/cloud/*` | `gfc_` |
| Site-user auth | `/v1/app/:siteId/auth/*` | Site-user JWT |
| Public content | `/v1/public/sites/:siteId/collections/*` | Public |

---

## Notes for Integrators

- Use `gfk_` for structured data.
- Use `gfc_` for files and media tools.
- `ADMIN` is a permission level, not a different key prefix.
- If a dashboard user needs API access quickly, create or ensure the site keys first.
- Existing integrations should prefer `https://api.noverfly.com` as the canonical base URL.

---

## Links

- Website: [noverfly.com](https://noverfly.com)
- API host: [api.noverfly.com](https://api.noverfly.com)
- Help: [help.noverfly.com](https://help.noverfly.com)
