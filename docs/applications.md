# Build Applications

NoverFly lets you build data-driven sites and applications using the currently deployed stack: dashboard JWT management, per-site API keys, collections/records, end-user auth, and publish workflows.

---

## Practical app building flow

### 1. Create a tenant and a site

Use dashboard JWT routes:

- `POST /v1/tenants`
- `POST /v1/tenants/:tenantId/sites`
- `GET /v1/tenants/:tenantId/sites`

### 2. Ensure the site API keys

```bash
curl -X POST https://api.noverfly.com/v1/sites/YOUR_SITE_ID/ensure-api-keys \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN"
```

This gives the site its default:

- `gfk_` Secret key for the Data API
- `gfc_` Cloud key for the Cloud API

### 3. Model your data

Use the Data API with `gfk_`:

```bash
curl -X POST https://api.noverfly.com/v1/api/data/collections \
  -H "X-Api-Key: gfk_YOUR_SECRET_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Tasks",
    "fields": [
      { "name": "Title", "slug": "title", "type": "TEXT", "required": true },
      { "name": "Status", "slug": "status", "type": "TEXT" },
      { "name": "Priority", "slug": "priority", "type": "NUMBER" }
    ]
  }'
```

### 4. Store files and media

Use the Cloud API with `gfc_`:

```bash
curl -X POST https://api.noverfly.com/v1/api/cloud/upload \
  -H "X-Api-Key: gfc_YOUR_CLOUD_KEY" \
  -H "Content-Type: application/json" \
  -d '{"filename":"screenshot.png","mime_type":"image/png","size_bytes":120000}'
```

### 5. Add end-user auth

For site users, use:

- `POST /v1/app/:siteId/auth/register`
- `POST /v1/app/:siteId/auth/login`
- `GET /v1/app/:siteId/auth/me`

### 6. Publish the site

```bash
curl -X POST https://api.noverfly.com/v1/sites/YOUR_SITE_ID/publish \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN"
```

---

## Current app stack

| Area | Current route family | Auth |
|---|---|---|
| Tenant and site management | `/v1/tenants/*`, `/v1/sites/*` | Dashboard JWT |
| Structured data | `/v1/api/data/*`, `/api/*` | `gfk_` |
| Media and files | `/v1/api/cloud/*` | `gfc_` |
| End-user auth | `/v1/app/:siteId/auth/*` | Site-user JWT |
| Public content | `/v1/public/sites/:siteId/collections/*` | Public |

---

## Key integration rules

- Both Data API and Cloud API already run on `https://api.noverfly.com`.
- Do not use a separate storage host for the current Cloud API contract.
- Do not send `X-Tenant-Id` with `gfk_` or `gfc_` requests.
- Use `ADMIN` permission on a `gfk_` key if the app needs to create or delete collections.

---

## Recommended architecture

For most external applications:

1. use dashboard JWT only for setup and administration
2. use `gfk_` in the backend for collections and records
3. use `gfc_` in the backend for assets and media workflows
4. expose only your own backend to the frontend

That keeps the app secure and aligned with the current production model.
