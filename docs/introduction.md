# Introduction to NoverFly

NoverFly is a cloud-native SaaS platform that combines site building, application workflows, structured content, storage, publishing, and end-user auth behind one public API host.

## Current API Model

Production currently uses:

- Canonical host: `https://api.noverfly.com`
- Compatible alias: `https://api.gloowflix.cloud`

On this host, developers primarily use two API families:

| API family | Auth | Routes | Purpose |
|---|---|---|---|
| Data API | `gfk_` | `/v1/api/data/*`, `/api/*` | Collections and records |
| Cloud API | `gfc_` | `/v1/api/cloud/*` | Uploads, assets, media search |

There are also dashboard JWT routes and site-user auth routes.

---

## What NoverFly provides

- Site and app management
- Structured data with collections and records
- File upload and asset management
- Media search and import
- Site-user authentication per site
- Publish and deployment workflows
- Public content endpoints for published records

---

## Multi-tenant model

NoverFly is multi-tenant, but developer API keys are already site-scoped.

That means:

- dashboard JWT routes may need tenant context
- `gfk_` and `gfc_` routes do not need `X-Tenant-Id`
- the key itself already resolves tenant and site scope

---

## Key rule

- use `gfk_` for structured data
- use `gfc_` for cloud/media workflows
- use `Authorization: Bearer` for dashboard management flows

---

## Next steps

- [Authentication](authentication.md)
- [API Reference](api.md)
- [Database / Data API](database.md)
- [Build Applications](applications.md)
