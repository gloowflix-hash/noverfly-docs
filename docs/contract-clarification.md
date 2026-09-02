# Noverfly API Clarification

Status date: `2026-04-01`

This note exists to remove a real confusion between:

1. the public GitHub docs for Noverfly, and
2. the API contract that is actually implemented in this backend repository.

## Executive summary

- In this repository, `gfk_` and `gfc_` are real implemented API key formats.
- `gfk_` is the developer data key for collections and records.
- `gfc_` is the developer cloud key for storage, upload, and partner media search.
- In the current VPS deployment, both developer APIs go through the same public host: `https://api.noverfly.com`.
- There is no need to use a separate `gfk.noverfly.com` host for the APIs implemented in this repository.
- The GitHub repo `gloowflix-hash/noverfly-docs` documents a different or newer public contract built around `nf_pk_` / `nf_sk_`, `X-Tenant-Id`, and `/v1/database/*`.
- This repository does not currently implement that `nf_pk_` / `nf_sk_` + `/v1/database/*` contract.

So a `gfk_` key is not "fake" in this codebase. It is valid for the developer API that exists here today.

## Verified source split

Public GitHub docs say:

- Main API base URL: `https://api.noverfly.com/v1`
- Main auth: JWT or `nf_pk_*` + `nf_sk_*`
- Multi-tenant header: `X-Tenant-Id`
- Database BaaS endpoints: `/database/tables`, `/database/tables/:name/rows`, `/database/query`
- Separate storage host: `https://gfk.noverfly.com`

Sources:

- `https://github.com/gloowflix-hash/noverfly-docs/blob/main/docs/api.md`
- `https://github.com/gloowflix-hash/noverfly-docs/blob/main/docs/authentication.md`
- `https://github.com/gloowflix-hash/noverfly-docs/blob/main/docs/database.md`

This repository says:

- Developer data API uses `X-Api-Key: gfk_*`
- Developer cloud API uses `X-Api-Key: gfc_*`
- Data endpoints are exposed under `/v1/api/data/*` and `/api/*`
- Cloud endpoints are exposed under `/v1/api/cloud/*`
- API keys are site-scoped and resolved from the key itself, not from `X-Tenant-Id`
- Public traffic is routed through `api.noverfly.com` to the API container on the VPS

Implementation evidence in this repo:

- `Caddyfile`
- `src/modules/collections/apikey.service.ts`
- `src/modules/collections/devapi.routes.ts`
- `src/modules/collections/cloudapi.routes.ts`
- `src/app.ts`
- `src/sdk/gloowflix-sdk.ts`
- `prisma/schema.prisma`

## What is implemented in this repo today

### 1. Dashboard and tenant/site management

Implemented with JWT auth:

- `POST /v1/auth/*`
- `POST /v1/tenants`
- `GET /v1/tenants/:tenantId`
- `POST /v1/tenants/:tenantId/sites`
- `GET /v1/tenants/:tenantId/sites`
- `GET /v1/sites/:siteId`
- `PATCH /v1/sites/:siteId`
- `POST /v1/sites/:siteId/publish`

Notes:

- Site creation is real and implemented.
- If a tenant has no site, the backend can auto-create a headless site for API-only usage when sites are listed.

### 2. Developer Data API

Implemented with `gfk_` keys:

- `GET /v1/api/data/collections`
- `GET /v1/api/data/collections/:slug`
- `POST /v1/api/data/collections`
- `PATCH /v1/api/data/collections/:slug`
- `DELETE /v1/api/data/collections/:slug`
- `GET /v1/api/data/collections/:slug/records`
- `POST /v1/api/data/collections/:slug/records`
- `PATCH /v1/api/data/collections/:slug/records/:id`
- `DELETE /v1/api/data/collections/:slug/records/:id`

Shortcut alias:

- `/api/*` exposes the same developer data API with a shorter prefix.

### 3. Developer Cloud API

Implemented with `gfc_` keys:

- `POST /v1/api/cloud/upload`
- `POST /v1/api/cloud/upload/commit`
- `POST /v1/api/cloud/upload/direct`
- `GET /v1/api/cloud/assets`
- `DELETE /v1/api/cloud/assets/:assetId`
- `GET /v1/api/cloud/search/images`
- `GET /v1/api/cloud/search/videos`
- `GET /v1/api/cloud/search/gifs`
- `GET /v1/api/cloud/search/icons`
- `GET /v1/api/cloud/search/3d`
- `POST /v1/api/cloud/import/image`

### 4. API key lifecycle

Implemented per site:

- `POST /v1/projects/:projectId/ensure-api-keys`
- `POST /v1/projects/:projectId/api-keys`
- `GET /v1/projects/:projectId/api-keys`
- `GET /v1/projects/:projectId/api-keys/:keyId/reveal`
- `PATCH /v1/projects/:projectId/api-keys/:keyId`
- `DELETE /v1/projects/:projectId/api-keys/:keyId`
- `GET /v1/projects/:projectId/api-docs`

## What is not implemented in this repo today

Not found in the current backend code:

- `nf_pk_*` / `nf_sk_*` key pair auth
- `X-API-Secret` header handling
- `/v1/database/tables`
- `/v1/database/tables/:name/rows`
- `/v1/database/query`
- a separate `gfk.noverfly.com` host contract

That means an integration that posts to `POST /v1/database/tables/{table}/rows` is not targeting the API implemented in this repository.

## Practical mapping

| Public GitHub docs concept | Current repo equivalent | Recommendation |
| --- | --- | --- |
| `nf_pk_` + `nf_sk_` | `gfk_` or `gfc_` only | Do not mix formats |
| `/v1/database/tables` | `/v1/api/data/collections` | Use collections/records in this repo |
| `/v1/database/tables/:name/rows` | `/v1/api/data/collections/:slug/records` | Use records endpoints in this repo |
| `X-Tenant-Id` on developer API | tenant/site inferred from the API key | Do not add tenant headers for `gfk_` / `gfc_` routes |
| separate `gfk.noverfly.com` storage host | `/v1/api/cloud/upload` on `https://api.noverfly.com` | Use the current cloud API routes on the main API host |

## Feasibility today

### Feasible now in this backend

- create tenants
- create sites
- run a headless site for API-only data
- create collections
- create, read, update, and delete records
- generate and reveal site API keys
- upload files and manage assets
- search partner images, videos, gifs, icons, and 3D models
- use the provided TypeScript and Python SDKs for `gfk_` / `gfc_`

### Not feasible without new implementation

- calling a native `/v1/database/*` BaaS API
- authenticating with `nf_pk_` / `nf_sk_`
- using `X-API-Secret` against this backend
- treating this repo as if it already exposed the GitHub `database.md` contract

## Decision rule for integrations

Use this rule to avoid future confusion:

- If the client gives you `gfk_*` or `gfc_*`, integrate against the API implemented in this repo.
- If the client gives you `nf_pk_*` and `nf_sk_*`, integrate against the public Noverfly contract from `noverfly-docs`.
- Do not claim that a `gfk_*` key is invalid just because the GitHub docs describe a different contract.

## Recommended wording for internal teams

Use this sentence when explaining the situation:

> The current Gloowflix backend already has a real developer API based on `gfk_` and `gfc_`. The GitHub `noverfly-docs` repository describes a different or newer public API contract based on `nf_pk_` / `nf_sk_` and `/v1/database/*`. These two contracts should not be mixed.

## Next step if contract alignment is required

If you want this backend to match the public GitHub contract, do it explicitly as a new feature:

1. add native `/v1/database/*` routes, or
2. add a compatibility adapter that maps `tables/rows` to `collections/records`, or
3. split the documentation so "current backend API" and "target public API" are clearly separated.
