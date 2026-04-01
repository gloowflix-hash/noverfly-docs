# Database / Data API

This page documents the current production data model exposed to developers.

In the deployed NoverFly stack, structured data is exposed through the Data API based on collections and records.

- Canonical host: `https://api.noverfly.com`
- Compatible alias: `https://api.gloowflix.cloud`
- Auth: `X-Api-Key: gfk_...`
- Route family: `/v1/api/data/*`
- Short alias: `/api/*`

There is no need to call a separate database host for the current deployment.

---

## Concept Model

| Concept | Meaning |
|---|---|
| Collection | A schema-like container for structured content |
| Record | A row / document inside a collection |
| Field | A typed property on a collection |
| Site scope | Each `gfk_` key is already linked to a site |

This model works well for headless CMS data, dynamic content, internal tools, and lightweight app backends.

---

## Authentication

Use a Secret API key:

```http
X-Api-Key: gfk_YOUR_SECRET_KEY
```

Important:

- Do not send `X-Tenant-Id` on Data API calls.
- Do not send `X-API-Secret`.
- The `gfk_` key already resolves tenant and site scope.

---

## Collections

### List collections

```bash
curl https://api.noverfly.com/v1/api/data/collections \
  -H "X-Api-Key: gfk_YOUR_SECRET_KEY"
```

### Get one collection

```bash
curl https://api.noverfly.com/v1/api/data/collections/articles \
  -H "X-Api-Key: gfk_YOUR_SECRET_KEY"
```

### Create a collection

Requires `ADMIN` permission.

```bash
curl -X POST https://api.noverfly.com/v1/api/data/collections \
  -H "X-Api-Key: gfk_YOUR_SECRET_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Articles",
    "description": "Blog posts",
    "fields": [
      { "name": "Title", "slug": "title", "type": "TEXT", "required": true },
      { "name": "Body", "slug": "body", "type": "RICHTEXT" },
      { "name": "Status", "slug": "status", "type": "TEXT" }
    ]
  }'
```

### Update a collection

Requires `ADMIN` permission.

```bash
curl -X PATCH https://api.noverfly.com/v1/api/data/collections/articles \
  -H "X-Api-Key: gfk_YOUR_SECRET_KEY" \
  -H "Content-Type: application/json" \
  -d '{"description":"Updated description"}'
```

### Delete a collection

Requires `ADMIN` permission.

```bash
curl -X DELETE https://api.noverfly.com/v1/api/data/collections/articles \
  -H "X-Api-Key: gfk_YOUR_SECRET_KEY"
```

---

## Records

### List records

```bash
curl "https://api.noverfly.com/v1/api/data/collections/articles/records?page=1&per_page=20&sort_by=createdAt&sort_dir=desc" \
  -H "X-Api-Key: gfk_YOUR_SECRET_KEY"
```

### Read one record

```bash
curl https://api.noverfly.com/v1/api/data/collections/articles/records/RECORD_ID \
  -H "X-Api-Key: gfk_YOUR_SECRET_KEY"
```

### Create a record

Requires `READ_WRITE` or `ADMIN`.

```bash
curl -X POST https://api.noverfly.com/v1/api/data/collections/articles/records \
  -H "X-Api-Key: gfk_YOUR_SECRET_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "title": "Hello world",
    "body": "My first record",
    "status": "PUBLISHED"
  }'
```

### Update a record

```bash
curl -X PATCH https://api.noverfly.com/v1/api/data/collections/articles/records/RECORD_ID \
  -H "X-Api-Key: gfk_YOUR_SECRET_KEY" \
  -H "Content-Type: application/json" \
  -d '{"status":"ARCHIVED"}'
```

### Delete a record

```bash
curl -X DELETE https://api.noverfly.com/v1/api/data/collections/articles/records/RECORD_ID \
  -H "X-Api-Key: gfk_YOUR_SECRET_KEY"
```

---

## Query Conventions

Common query parameters:

| Query param | Meaning |
|---|---|
| `page` | Page number |
| `per_page` | Items per page |
| `sort_by` | Field used for sorting |
| `sort_dir` | `asc` or `desc` |
| `status` | Record status filter |
| `search` | Full-text style search |
| `filter.FIELD` | Exact field filter |
| `filter.FIELD_gte` | Greater-than-or-equal filter |
| `filter.FIELD_lte` | Less-than-or-equal filter |
| `filter.FIELD_contains` | Text contains filter |

---

## Permissions

| Permission | What it can do |
|---|---|
| `READ` | Read collections and records |
| `READ_WRITE` | Read plus create/update/delete records |
| `ADMIN` | All of the above plus create/update/delete collections |

If your integration needs to create or delete collections, use a `gfk_` key with `ADMIN` permission.

---

## Public Read API

Published content can also be read without an API key through the public routes:

| Method | Route |
|---|---|
| `GET` | `/v1/public/sites/:siteId/collections/:slug/records` |
| `GET` | `/v1/public/sites/:siteId/collections/:slug/records/:recordSlug` |

Use these routes for public websites, blogs, and storefront reads.

---

## Alias

The Data API also supports a short alias:

- `/api/collections`
- `/api/collections/:slug`
- `/api/collections/:slug/records`

This is the same API as `/v1/api/data/*`.

---

## Summary

If you need structured data on NoverFly today:

- use `gfk_`
- call `https://api.noverfly.com`
- use `/v1/api/data/*` or `/api/*`
- use `ADMIN` when collection management is required
