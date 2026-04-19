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

On that same host, there are two main developer API families:

| API family | Auth | Main routes | Purpose |
|---|---|---|---|
| Data API | `X-Api-Key: gfk_...` | `/v1/api/data/*` and `/api/*` | Collections, records, app auth bootstrap |
| Cloud API | `X-Api-Key: gfc_...` | `/v1/api/cloud/*` | Upload, assets, media search, import |

Important points:

- `gfk_` and `gfc_` are real production keys already used by existing applications.
- Both APIs go through `api.noverfly.com`.
- There is no separate public storage host required for the current deployed Cloud API.
- For `gfk_` and `gfc_` routes, the key already resolves the site and tenant scope.
- Do not add `X-Tenant-Id` on developer API requests authenticated by `gfk_` or `gfc_`.

---

## What You Can Build Today

The deployed stack already covers more than basic CRUD:

- data-first apps and headless backends with collections and records
- tenant, site, domain, publish, clone, import, and headless-site flows
- end-user authentication for apps and sites
- file upload, asset library, media search, and external image import
- Flivex media processing for image, video, and audio assets
- realtime messaging, voice messages, audio calls, and video calls
- hosted video workflows, public video access checks, and music streaming routes

---

## Safe Examples Only

All examples in this docs repository use placeholders only:

- `YOUR_ACCESS_TOKEN`
- `YOUR_SITE_ID`
- `YOUR_TENANT_ID`
- `gfk_YOUR_SECRET_KEY`
- `gfc_YOUR_CLOUD_KEY`

Do not paste real production keys into public frontend code, public repos, or shared documentation.

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

### 5. Start a media upload with a `gfc_` key

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
| [Authentication](docs/authentication.md) | Dashboard JWT, site-user JWT, `gfk_`, `gfc_`, permissions, app auth |
| [API Reference](docs/api.md) | Main route families and integration rules |
| [Database / Data API](docs/database.md) | Collections and records API with `gfk_` |
| [Build Applications](docs/applications.md) | Vite proxy example, frontend helper example, app-building flows |
| [Realtime and Media](docs/realtime-media.md) | Chat, voice, calls, WebSocket signaling, Flivex processing, streaming |
| [Push Notifications](docs/push-notifications.md) | Native push (FCM, APNs, Expo, Web Push VAPID) |
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
| Developer messenger | `/v1/cloud/messenger/*` | `gfk_` or `gfc_` |
| WebSocket realtime | `/ws` | JWT or API-key app auth |
| Push notifications | `/v1/push/*`, `/v1/tenants/:tenantId/push/*`, `/v1/cloud/push/*` | Public, JWT, or `gfc_` |
| Site-user auth | `/v1/app/:siteId/auth/*` and `/v1/public/sites/:siteId/auth/*` | Site-user JWT |
| Public content | `/v1/public/sites/:siteId/collections/*` | Public |
| Video and music | `/v1/sites/:siteId/videos/*`, `/v1/tenants/:tenantId/music/*` | JWT |

---

## Notes for Integrators

- Use `gfk_` for structured data and app-scoped auth bootstrap.
- Use `gfc_` for uploads, assets, media search, and import.
- `ADMIN` is a permission level, not a different key prefix.
- Existing integrations should prefer `https://api.noverfly.com` as the canonical base URL.
- Use `wss://api.noverfly.com/ws` as the canonical realtime endpoint. The `api.gloowflix.cloud` alias remains compatible.

---

## Links

- Website: [noverfly.com](https://noverfly.com)
- API host: [api.noverfly.com](https://api.noverfly.com)
- Help: [help.noverfly.com](https://help.noverfly.com)
