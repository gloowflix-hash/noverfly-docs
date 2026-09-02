# API Reference

This page documents the API contract currently deployed on NoverFly production.

## Base URLs

- Canonical base URL: `https://api.noverfly.com`
- Compatible alias: `https://api.gloowflix.cloud`
- Canonical WebSocket endpoint: `wss://api.noverfly.com/ws`
- Compatible WebSocket alias: `wss://api.gloowflix.cloud/ws`

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
| API keys | `/v1/projects/:projectId/api-keys*`, `/v1/projects/:projectId/ensure-api-keys` |
| Publish | `/v1/sites/:siteId/publish` |
| Domains | `/v1/sites/:siteId/domains` |

Use `X-Tenant-Id` only where the dashboard route contract explicitly requires tenant scoping.

### 2. Data API

Used by external backends and automations to manage structured content and app-scoped auth.

Authentication:

```http
X-Api-Key: gfk_YOUR_SECRET_KEY
```

Main routes:

| Method | Route | Description | Permission |
|---|---|---|---|
| `GET` | `/v1/api/data/info` | Resolve current site and tenant from the key | `READ` |
| `GET` | `/v1/api/data/auth/config` | Read auth provider config for the key's site | `READ` |
| `PUT` | `/v1/api/data/auth/config/:provider` | Configure an auth provider | `ADMIN` |
| `POST` | `/v1/api/data/auth/config/google/credentials` | Import Google OAuth Web JSON (multipart or JSON) | `ADMIN` |
| `POST` | `/v1/api/data/auth/register` | Register an end user for the key's site | `READ` |
| `POST` | `/v1/api/data/auth/login` | Login an end user for the key's site | `READ` |
| `GET` | `/v1/api/data/auth/me` | Read the current end-user profile | App-user JWT |
| `PATCH` | `/v1/api/data/auth/me` | Update the current end-user profile | App-user JWT |
| `POST` | `/v1/api/data/auth/refresh` | Refresh the current end-user session | App-user JWT |
| `POST` | `/v1/api/data/auth/logout` | Logout the current end-user session | App-user JWT |
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
| `/api/auth/*` | Short alias for app auth on the same site resolved by the key |

Important:

- `gfk_` keys are site-scoped.
- Do not send `X-Tenant-Id` with `gfk_` requests.
- The key already resolves the tenant and site.

### 3. Hosting API

Publishes prebuilt React, Vite, Astro or Next.js static-export sites as
immutable S3 releases. Authentication uses the site-scoped `gfk_`; deployment
mutations require `ADMIN`.

| Method | Route | Description | Permission |
|---|---|---|---|
| `GET` / `PUT` / `DELETE` | `/v1/hosting/project` | Inspect, configure or remove Hosting/S3 | `READ` / `ADMIN` |
| `POST` | `/v1/hosting/deployments` | Create an immutable manifest | `ADMIN` |
| `POST` | `/v1/hosting/deployments/:id/upload-urls` | Sign S3 upload batches | `ADMIN` |
| `POST` | `/v1/hosting/deployments/:id/finalize` | Verify S3 files | `ADMIN` |
| `POST` | `/v1/hosting/deployments/:id/activate` | Publish or rollback atomically | `ADMIN` |
| `GET` | `/v1/hosting/deployments*` | List/inspect releases | `READ` |
| `DELETE` | `/v1/hosting/deployments/:id` | Delete a non-active release | `ADMIN` |

See [Noverfly Hosting](hosting-api.md).

### 4. Cloud API

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
| `POST` | `/v1/api/cloud/assets/make-public` | Repair public ACLs for a site | `ADMIN` |
| `GET` | `/v1/api/cloud/search/images` | Search images | `READ` |
| `GET` | `/v1/api/cloud/search/videos` | Search videos | `READ` |
| `GET` | `/v1/api/cloud/search/gifs` | Search GIFs | `READ` |
| `GET` | `/v1/api/cloud/search/icons` | Search icons | `READ` |
| `GET` | `/v1/api/cloud/search/3d` | Search 3D models | `READ` |
| `POST` | `/v1/api/cloud/import/image` | Import an external image into storage | `READ_WRITE` |

Important:

- `gfc_` keys also go through `https://api.noverfly.com`.
- Do not send `X-Tenant-Id` with `gfc_` requests.
- `gfc_` is a Cloud key family, not a JWT token.

### 5. Realtime and Messenger Developer API

Used by apps that need chat, voice messages, call history, and WebRTC signaling.

Authentication:

```http
X-Api-Key: gfk_YOUR_SECRET_KEY
```

or

```http
X-Api-Key: gfc_YOUR_CLOUD_KEY
```

Main routes:

| Method | Route | Description |
|---|---|---|
| `GET` | `/v1/cloud/messenger/rtc-config` | Get ICE server config and signal endpoint |
| `POST` | `/v1/cloud/messenger/conversations` | Create a direct conversation |
| `GET` | `/v1/cloud/messenger/conversations` | List a user's conversations |
| `POST` | `/v1/cloud/messenger/conversations/:conversationId/messages` | Send a text, image, or file message |
| `GET` | `/v1/cloud/messenger/conversations/:conversationId/messages` | Read message history |
| `POST` | `/v1/cloud/messenger/conversations/:conversationId/voice` | Create a voice message record |
| `GET` | `/v1/cloud/messenger/calls` | Read call history |
| `POST` | `/v1/cloud/messenger/webhooks/voice-processed` | Internal webhook for processed voice media |

WebSocket transport:

- Endpoint: `/ws`
- Auth within 10 seconds after connect
- Supports JWT-based auth and API-key app auth

Example API-key app auth message:

```json
{
  "type": "auth",
  "payload": {
    "apiKey": "gfk_YOUR_SECRET_KEY",
    "userId": "USER_ID"
  }
}
```

Current call signaling message types include:

- `call:initiate`
- `call:offer`
- `call:answer`
- `call:ice`
- `call:hangup`
- `call:reject`
- `call:busy`

### 5. App-user Auth API

Used for end users of a specific project or app.

| Method | Route |
|---|---|
| `POST` | `/v1/app/:projectId/auth/register` |
| `POST` | `/v1/app/:projectId/auth/login` |
| `GET` | `/v1/app/:projectId/auth/me` |
| `PATCH` | `/v1/app/:projectId/auth/me` |
| `POST` | `/v1/app/:projectId/auth/refresh` |
| `POST` | `/v1/app/:projectId/auth/logout` |
| `POST` | `/v1/app/:projectId/auth/forgot-password` |
| `POST` | `/v1/app/:projectId/auth/reset-password` |
| `POST` | `/v1/app/:projectId/auth/verify-email` |
| `POST` | `/v1/app/:projectId/auth/resend-verification` |
| `POST` | `/v1/app/:projectId/auth/mfa/totp/setup` |
| `POST` | `/v1/app/:projectId/auth/mfa/totp/verify` |
| `POST` | `/v1/app/:projectId/auth/mfa/totp/disable` |
| `POST` | `/v1/app/:projectId/auth/me/avatar` |
| `GET` | `/v1/app/:projectId/auth/google` |
| `GET` | `/v1/app/:projectId/auth/google/callback` |

There is also a public mirror used by some site/social flows:

- `/v1/public/sites/:projectId/auth/register`
- `/v1/public/sites/:projectId/auth/login`
- `/v1/public/sites/:projectId/auth/refresh`
- `/v1/public/sites/:projectId/auth/me`
- `/v1/public/sites/:projectId/auth/logout`

### 6. Streaming and Media Routes

These routes are already present in the deployed platform for dashboard-managed products and media apps.

| Method | Route | Description |
|---|---|---|
| `POST` | `/v1/tenants/:tenantId/video/connect` | Connect a video provider |
| `POST` | `/v1/tenants/:tenantId/video/test` | Test the video provider connection |
| `GET` | `/v1/videos` | List project videos (DevAPI, scope `videos:read`) |
| `GET` | `/v1/videos/:videoId` | Get a video |
| `GET` | `/v1/videos/:videoId/status` | Flivex processing status |
| `GET` | `/v1/videos/:videoId/low` | Low-resolution variant |
| `GET` | `/v1/videos/:videoId/high` | High-resolution variant |
| `GET` | `/v1/videos/:videoId/cover` | Poster / thumbnail |
| `GET` | `/v1/videos/:videoId/play` | Playback URL |
| `GET` | `/v1/videos/:videoId/manifest` | HLS manifest |
| `POST` | `/v1/videos/:videoId/cover/from-frame` | Extract cover from frame |
| `PUT` | `/v1/videos/:videoId/cover/select` | Select cover |
| `POST` | `/v1/videos/:videoId/reprocess` | Reprocess via Flivex |
| `POST` | `/v1/videos/:videoId/publish` | Publish video |
| `POST` | `/v1/videos/:videoId/unpublish` | Unpublish video |
| `DELETE` | `/v1/videos/:videoId` | Delete video |
| `GET` | `/v1/videos/:videoId/events` | Processing events |
| `GET` | `/v1/tenants/:tenantId/music/tracks/:trackId/stream` | Return a music stream URL |
| `POST` | `/webhooks/flivex` | Receive processed media callbacks from Flivex |

Flivex-ready media callbacks can include:

- `poster_url`
- `preview_url`
- `playback_url`
- `hls_url`
- `renditions`
- `image_variants`

---

## Permissions

API keys use permission levels:

| Level | Meaning |
|---|---|
| `READ` | Read-only access |
| `READ_WRITE` | Read plus create/update/delete records or assets |
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
| Realtime auth | send a JSON `auth` message after opening `/ws` |

---

## Examples

### List collections

```bash
curl https://api.noverfly.com/v1/api/data/collections \
  -H "X-Api-Key: gfk_YOUR_SECRET_KEY"
```

### Register an app user through the Data API

```bash
curl -X POST https://api.noverfly.com/v1/api/data/auth/register \
  -H "X-Api-Key: gfk_YOUR_SECRET_KEY" \
  -H "Content-Type: application/json" \
  -d '{"email":"user@example.com","password":"StrongPass123","displayName":"User"}'
```

### Start an upload

```bash
curl -X POST https://api.noverfly.com/v1/api/cloud/upload \
  -H "X-Api-Key: gfc_YOUR_CLOUD_KEY" \
  -H "Content-Type: application/json" \
  -d '{"filename":"photo.jpg","mime_type":"image/jpeg","size_bytes":245000}'
```

### Get realtime config

```bash
curl https://api.noverfly.com/v1/cloud/messenger/rtc-config \
  -H "X-Api-Key: gfk_YOUR_SECRET_KEY"
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
