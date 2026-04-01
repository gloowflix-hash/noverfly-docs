# Build Applications

NoverFly lets you build data-driven sites and applications using the currently deployed stack: dashboard JWT management, per-site API keys, collections, records, end-user auth, realtime messaging, media workflows, and publish flows.

> All code examples on this page use placeholders only. Do not publish real API keys in frontend code.

---

## Practical App-building Flow

### 1. Create a tenant and a site

Use dashboard JWT routes:

- `POST /v1/tenants`
- `POST /v1/tenants/:tenantId/sites`
- `GET /v1/tenants/:tenantId/sites`

For API-only use cases, the deployed platform can also maintain a headless site for a tenant.

### 2. Ensure the site API keys

```bash
curl -X POST https://api.noverfly.com/v1/sites/YOUR_SITE_ID/ensure-api-keys \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN"
```

This gives the site its default:

- `gfk_` key for the Data API
- `gfc_` key for the Cloud API

### 3. Model your data

Use the Data API with `gfk_`:

```bash
curl -X POST https://api.noverfly.com/v1/api/data/collections \
  -H "X-Api-Key: gfk_YOUR_SECRET_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Tasks",
    "fields": [
      { "name": "Title", "slug": "title", "type": "TEXT", "required": true, "isPrimary": true },
      { "name": "Status", "slug": "status", "type": "TEXT" },
      { "name": "Priority", "slug": "priority", "type": "NUMBER" }
    ]
  }'
```

### 4. Add app auth

You can use either:

- app-scoped auth through `/v1/api/data/auth/*` with a `gfk_` key
- explicit site auth through `/v1/app/:siteId/auth/*`

### 5. Add assets and media

Use the Cloud API with `gfc_`:

```bash
curl -X POST https://api.noverfly.com/v1/api/cloud/upload \
  -H "X-Api-Key: gfc_YOUR_CLOUD_KEY" \
  -H "Content-Type: application/json" \
  -d '{"filename":"screenshot.png","mime_type":"image/png","size_bytes":120000}'
```

### 6. Add chat, voice, or calls

Use:

- REST routes under `/v1/cloud/messenger/*`
- WebSocket signaling under `/ws`

### 7. Publish the site when needed

```bash
curl -X POST https://api.noverfly.com/v1/sites/YOUR_SITE_ID/publish \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN"
```

---

## Vite Dev Proxy Example

This pattern is useful in local development to avoid browser CORS noise while keeping the production base URL unchanged.

```ts
import { defineConfig } from 'vite'
import react from '@vitejs/plugin-react'

export default defineConfig({
  plugins: [react()],
  server: {
    host: true,
    proxy: {
      '/api/cloud': {
        target: 'https://api.noverfly.com/v1',
        changeOrigin: true,
        secure: true,
      },
      '/api/data': {
        target: 'https://api.noverfly.com/v1',
        changeOrigin: true,
        secure: true,
      },
    },
  },
})
```

This means:

- `/api/data/...` becomes `https://api.noverfly.com/v1/api/data/...`
- `/api/cloud/...` becomes `https://api.noverfly.com/v1/api/cloud/...`

---

## Frontend Helper Example

This example mirrors a working application pattern, but uses placeholders and environment variables instead of real keys.

Use this only for local development or trusted internal builds. For public production apps, prefer calling your own backend.

```ts
const isDev =
  typeof window !== 'undefined' &&
  window.location?.hostname === 'localhost' &&
  !window.Capacitor?.isNativePlatform?.()

const API_BASE = isDev ? '/api/data' : 'https://api.noverfly.com/v1/api/data'
const CLOUD_BASE = isDev ? '/api/cloud' : 'https://api.noverfly.com/v1/api/cloud'

const DATA_KEY = import.meta.env.VITE_NOVERFLY_DATA_KEY
const CLOUD_KEY = import.meta.env.VITE_NOVERFLY_CLOUD_KEY

const dataHeaders = () => ({
  'X-Api-Key': DATA_KEY,
  'Content-Type': 'application/json',
})

async function dataRequest(method: string, path: string, body?: unknown) {
  const res = await fetch(`${API_BASE}${path}`, {
    method,
    headers: dataHeaders(),
    body: body ? JSON.stringify(body) : undefined,
  })

  if (!res.ok) {
    const err = await res.json().catch(() => ({ error: res.statusText }))
    throw new Error(err.error?.message || err.error || err.message || res.statusText)
  }

  return res.json()
}

export function collection(slug: string) {
  return {
    list(params: Record<string, string> = {}) {
      const qs = new URLSearchParams(params).toString()
      return dataRequest('GET', `/collections/${slug}/records${qs ? `?${qs}` : ''}`)
    },
    get(id: string) {
      return dataRequest('GET', `/collections/${slug}/records/${id}`)
    },
    create(data: Record<string, unknown>) {
      return dataRequest('POST', `/collections/${slug}/records`, data)
    },
    update(id: string, data: Record<string, unknown>) {
      return dataRequest('PATCH', `/collections/${slug}/records/${id}`, data)
    },
    delete(id: string) {
      return dataRequest('DELETE', `/collections/${slug}/records/${id}`)
    },
  }
}

export const db = {
  tasks: collection('tasks'),
  messages: collection('messages'),
  products: collection('products'),
}

const TOKEN_KEY = 'noverfly_access_token'
const REFRESH_KEY = 'noverfly_refresh_token'

function authHeaders() {
  const token = localStorage.getItem(TOKEN_KEY)
  return {
    ...dataHeaders(),
    ...(token ? { Authorization: `Bearer ${token}` } : {}),
  }
}

async function authRequest(method: string, path: string, body?: unknown) {
  const res = await fetch(`${API_BASE}${path}`, {
    method,
    headers: authHeaders(),
    body: body ? JSON.stringify(body) : undefined,
  })

  if (!res.ok) {
    const err = await res.json().catch(() => ({ error: res.statusText }))
    throw new Error(err.error?.message || err.error || err.message || res.statusText)
  }

  return res.json()
}

export const auth = {
  async register(email: string, password: string, displayName?: string) {
    const result = await authRequest('POST', '/auth/register', { email, password, displayName })
    if (result.accessToken) localStorage.setItem(TOKEN_KEY, result.accessToken)
    if (result.refreshToken) localStorage.setItem(REFRESH_KEY, result.refreshToken)
    return result
  },
  async login(email: string, password: string) {
    const result = await authRequest('POST', '/auth/login', { email, password })
    localStorage.setItem(TOKEN_KEY, result.accessToken)
    localStorage.setItem(REFRESH_KEY, result.refreshToken)
    return result
  },
  me() {
    return authRequest('GET', '/auth/me')
  },
  refresh() {
    return authRequest('POST', '/auth/refresh', {
      refreshToken: localStorage.getItem(REFRESH_KEY),
    })
  },
  logout() {
    localStorage.removeItem(TOKEN_KEY)
    localStorage.removeItem(REFRESH_KEY)
  },
}

async function cloudRequest(method: string, path: string, body?: BodyInit | Record<string, unknown>) {
  const isFormData = typeof FormData !== 'undefined' && body instanceof FormData
  const res = await fetch(`${CLOUD_BASE}${path}`, {
    method,
    headers: isFormData
      ? { 'X-Api-Key': CLOUD_KEY }
      : { 'X-Api-Key': CLOUD_KEY, 'Content-Type': 'application/json' },
    body: body
      ? isFormData
        ? body
        : JSON.stringify(body)
      : undefined,
  })

  if (!res.ok) {
    const err = await res.json().catch(() => ({ error: res.statusText }))
    throw new Error(err.error?.message || err.error || err.message || res.statusText)
  }

  return res.json()
}

export const storage = {
  async upload(file: File, filename = file.name) {
    const form = new FormData()
    form.append('file', file, filename)
    const result = await cloudRequest('POST', '/upload/direct', form)
    return {
      asset: result.asset,
      url: result.asset.cdnUrl,
    }
  },
  list(type?: string) {
    const qs = type ? `?type=${encodeURIComponent(type)}` : ''
    return cloudRequest('GET', `/assets${qs}`)
  },
  delete(assetId: string) {
    return cloudRequest('DELETE', `/assets/${assetId}`)
  },
}
```

---

## Realtime, Chat, Voice, and Calls

The deployed platform already includes Flivex Messenger and realtime transport.

Developer routes:

- `GET /v1/cloud/messenger/rtc-config`
- `POST /v1/cloud/messenger/conversations`
- `GET /v1/cloud/messenger/conversations`
- `POST /v1/cloud/messenger/conversations/:conversationId/messages`
- `GET /v1/cloud/messenger/conversations/:conversationId/messages`
- `POST /v1/cloud/messenger/conversations/:conversationId/voice`
- `GET /v1/cloud/messenger/calls`
- `wss://api.noverfly.com/ws`

Minimal WebSocket example:

```ts
const ws = new WebSocket('wss://api.noverfly.com/ws')

ws.onopen = () => {
  ws.send(JSON.stringify({
    type: 'auth',
    payload: {
      apiKey: DATA_KEY,
      userId: 'USER_ID',
    },
  }))
}
```

Current call signaling message types:

- `call:initiate`
- `call:offer`
- `call:answer`
- `call:ice`
- `call:hangup`
- `call:reject`
- `call:busy`

Video calls are feature-gated by plan and tenant activation. Audio/video call usage and voice messages are also quota-aware.

---

## Flivex Media Processing, Streaming, and Hosted Media

When an image, video, or audio asset is uploaded through the Cloud API, the deployed stack can notify the Flivex media engine for asynchronous processing.

That processing flow can enrich assets with:

- `poster_url`
- `preview_url`
- `playback_url`
- `hls_url`
- `renditions`
- `image_variants`

Other already deployed media capabilities include:

- site video library routes under `/v1/sites/:siteId/videos`
- MP4 URL validation and ingestion under `/v1/sites/:siteId/videos/mp4-url*`
- public video access checks under `/v1/public/sites/:siteId/videos/:videoId/access`
- music track streaming under `/v1/tenants/:tenantId/music/tracks/:trackId/stream`

See [Realtime and Media](realtime-media.md) for the dedicated route summary.

---

## Current App Stack

| Area | Current route family | Auth |
|---|---|---|
| Tenant and site management | `/v1/tenants/*`, `/v1/sites/*` | Dashboard JWT |
| Structured data | `/v1/api/data/*`, `/api/*` | `gfk_` |
| App auth through key-scoped site | `/v1/api/data/auth/*`, `/api/auth/*` | `gfk_` and site-user JWT |
| Media and files | `/v1/api/cloud/*` | `gfc_` |
| Realtime and messenger | `/v1/cloud/messenger/*`, `/ws` | `gfk_` or `gfc_` plus app user context |
| End-user auth by site ID | `/v1/app/:siteId/auth/*` | Site-user JWT |
| Public content | `/v1/public/sites/:siteId/collections/*` | Public |
| Video and music | `/v1/sites/:siteId/videos/*`, `/v1/tenants/:tenantId/music/*` | Dashboard JWT |

---

## Key Integration Rules

- Both Data API and Cloud API already run on `https://api.noverfly.com`.
- Do not use a separate storage host for the current Cloud API contract.
- Do not send `X-Tenant-Id` with `gfk_` or `gfc_` requests.
- Use `ADMIN` permission on a `gfk_` key if the app needs to create or delete collections.
- Keep real developer API keys out of public repositories.

---

## Recommended Architecture

For most external applications:

1. use dashboard JWT only for setup and administration
2. use `gfk_` in the backend for collections, records, and app bootstrap auth
3. use `gfc_` in the backend for assets and media workflows
4. use `/v1/cloud/messenger/*` plus `/ws` for realtime messaging and calls
5. expose only your own backend to the public frontend in production

That keeps the app secure and aligned with the current production model.
