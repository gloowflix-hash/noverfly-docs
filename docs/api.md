# API Reference

This page documents the API contract currently deployed on NoverFly production.

## Base URLs

- Canonical base URL: `https://api.noverfly.com`
- Compatible alias: `https://api.gloowflix.cloud`

All currently documented routes in this repo are served from the same public host.

---

## API Families

### 1. Dashboard API

Used by admins and internal management flows.

Authentication:

```http
Authorization: Bearer YOUR_ACCESS_TOKEN
```

Typical route groups:

| Group | Routes |
|---|---|
| Auth | `/v1/auth/*` |
| Tenants | `/v1/tenants/*` |
| Sites | `/v1/tenants/:tenantId/sites`, `/v1/sites/:siteId` |
| API keys | `/v1/sites/:siteId/api-keys*`, `/v1/sites/:siteId/ensure-api-keys` |
| Publish | `/v1/sites/:siteId/publish` |

Use `X-Tenant-Id` only where the dashboard route contract requires tenant scoping.

### 2. Data API

Used by external backends and automations to manage structured content.

Authentication:

```http
X-Api-Key: gfk_YOUR_SECRET_KEY
```

Main routes:

| Method | Route | Description | Permission |
|---|---|---|---|
| `GET` | `/v1/api/data/collections` | List collections | `READ` |
| `GET` | `/v1/api/data/collections/:slug` | Get collection schema | `READ` |
| `POST` | `/v1/api/data/collections` | Create collection | `ADMIN` |
| `PATCH` | `/v1/api/data/collections/:slug` | Update collection | `ADMIN` |
| `DELETE` | `/v1/api/data/collections/:slug` | Delete collection | `ADMIN` |
| `GET` | `/v1/api/data/collections/:slug/records` | List records | `READ` |
| `POST` | `/v1/api/data/collections/:slug/records` | Create record | `READ_WRITE` |
| `GET` | `/v1/api/data/collections/:slug/records/:id` | Read a record | `READ` |
| `PATCH` | `/v1/api/data/collections/:slug/records/:id` | Update a record | `READ_WRITE` |
| `DELETE` | `/v1/api/data/collections/:slug/records/:id` | Delete a record | `READ_WRITE` |

Shortcut alias:

| Route family | Notes |
|---|---|
| `/api/*` | Short alias for the same Data API |

Important:

- `gfk_` keys are site-scoped.
- Do not send `X-Tenant-Id` with `gfk_` requests.
- The key already resolves the tenant and site.

### 3. Cloud API

Used by external backends and automations for file workflows and media services.

Authentication:

```http
X-Api-Key: gfc_YOUR_CLOUD_KEY
```

Main routes:

| Method | Route | Description | Permission |
|---|---|---|---|
| `POST` | `/v1/api/cloud/upload` | Get a presigned upload URL | `READ_WRITE` |
| `POST` | `/v1/api/cloud/upload/commit` | Finalize the upload | `READ_WRITE` |
| `POST` | `/v1/api/cloud/upload/direct` | Direct multipart upload through the API | `READ_WRITE` |
| `GET` | `/v1/api/cloud/assets` | List assets | `READ` |
| `DELETE` | `/v1/api/cloud/assets/:assetId` | Delete an asset | `READ_WRITE` |
| `GET` | `/v1/api/cloud/search/images` | Search images | `READ` |
| `GET` | `/v1/api/cloud/search/videos` | Search videos | `READ` |
| `GET` | `/v1/api/cloud/search/gifs` | Search GIFs | `READ` |
| `GET` | `/v1/api/cloud/search/icons` | Search icons | `READ` |
| `GET` | `/v1/api/cloud/search/3d` | Search 3D models | `READ` |
| `POST` | `/v1/api/cloud/import/image` | Import an external image into storage | `READ_WRITE` |

Important:

- `gfc_` keys also go through `https://api.noverfly.com`.
- Do not send `X-Tenant-Id` with `gfc_` requests.
- `gfc_` is a Cloud key type, not a JWT token.

### 4. Site-user Auth API

Used for end users of a specific site or app.

| Method | Route |
|---|---|
| `POST` | `/v1/app/:siteId/auth/register` |
| `POST` | `/v1/app/:siteId/auth/login` |
| `GET` | `/v1/app/:siteId/auth/me` |
| `PATCH` | `/v1/app/:siteId/auth/me` |
| `POST` | `/v1/app/:siteId/auth/refresh` |
| `POST` | `/v1/app/:siteId/auth/logout` |

---

## Permissions

API keys use permission levels:

| Level | Meaning |
|---|---|
| `READ` | Read-only access |
| `READ_WRITE` | Read plus create/update/delete |
| `ADMIN` | Includes collection management and other sensitive operations |

`ADMIN` is not a different key family. It is a permission level applied to a `gfk_` or `gfc_` key depending on the route family.

---

## Request Conventions

| Area | Convention |
|---|---|
| Query params | `snake_case` |
| Upload body | `filename`, `mime_type`, `size_bytes` |
| Headers | `X-Api-Key`, `Authorization` |
| Data API alias | `/api/*` |

---

## Examples

### List collections

```bash
curl https://api.noverfly.com/v1/api/data/collections \
  -H "X-Api-Key: gfk_YOUR_SECRET_KEY"
```

### Create a record

```bash
curl -X POST https://api.noverfly.com/v1/api/data/collections/articles/records \
  -H "X-Api-Key: gfk_YOUR_SECRET_KEY" \
  -H "Content-Type: application/json" \
  -d '{"title":"Hello","status":"PUBLISHED"}'
```

### Start an upload

```bash
curl -X POST https://api.noverfly.com/v1/api/cloud/upload \
  -H "X-Api-Key: gfc_YOUR_CLOUD_KEY" \
  -H "Content-Type: application/json" \
  -d '{"filename":"photo.jpg","mime_type":"image/jpeg","size_bytes":245000}'
```

---

## What this documentation does not assume

This docs repo does not assume a separate production contract based on:

- `nf_pk_*`
- `nf_sk_*`
- `X-API-Secret`
- `/v1/database/tables`
- `https://gfk.noverfly.com` as the active public Cloud API host

If the platform evolves later, document that as a separate versioned contract.
