# Push Notifications

This page documents the native push-notification system already deployed in NoverFly. Notifications wake the application **even when it is closed**, on Android, iOS, React Native (Expo), and the web browser.

---

## Supported Providers

| Provider | Platform | Use case |
|---|---|---|
| `FCM` | Android (and iOS via Firebase) | Native Android apps using Firebase Cloud Messaging HTTP v1 |
| `APNS` | iOS | Native iOS apps using Apple Push Notification service (HTTP/2 + ES256 JWT) |
| `EXPO` | React Native | Expo-managed apps using Expo Push Service |
| `WEBPUSH` | Web (Chrome / Firefox / Edge / Safari 16+) | Browser notifications via VAPID Web Push |

You can check at any time which providers are currently configured on the deployed API:

```bash
curl https://api.noverfly.com/v1/push/status
# → { "fcm": true, "apns": true, "expo": true, "webpush": true }
```

A provider returns `false` when its credentials are not present in the API environment. In that case, sends to that provider are skipped (no error is raised for the other providers).

---

## Architecture Overview

```
Frontend (NoverFly app, web or native)
        │
        │ POST /v1/tenants/:tenantId/push/tokens
        ▼
NoverFly API (api.noverfly.com)
        │
        ├── PushToken         (one row per device + provider)
        ├── PushDelivery      (audit row per send attempt)
        ├── BullMQ worker     (sendPush.worker.ts, concurrency 20)
        │
        └── providers/
              ├─ fcm.ts        Firebase Cloud Messaging HTTP v1
              ├─ apns.ts       Apple Push (HTTP/2 + ES256 JWT)
              ├─ expo.ts       Expo Push Service
              └─ webpush.ts    Web Push VAPID (web-push)
```

Tokens reported as dead by the upstream provider (`UNREGISTERED`, `BadDeviceToken`, HTTP 404 / 410, `DeviceNotRegistered`) are automatically marked `isActive = false` by the worker. No manual cleanup is required.

---

## Public Routes (no auth)

| Method | Route | Description |
|---|---|---|
| `GET` | `/v1/push/status` | Returns which providers are configured |
| `GET` | `/v1/push/vapid-public-key` | Returns the public VAPID key used by the web browser to subscribe |

---

## Tenant Routes (Dashboard JWT)

These routes are scoped to a tenant and require a Dashboard JWT.

| Method | Route | Description |
|---|---|---|
| `POST` | `/v1/tenants/:tenantId/push/tokens` | Register a device token |
| `GET` | `/v1/tenants/:tenantId/push/tokens` | List registered tokens for the tenant |
| `DELETE` | `/v1/tenants/:tenantId/push/tokens/:tokenId` | Mark a token as inactive |
| `POST` | `/v1/tenants/:tenantId/push/send` | Send a push from the Dashboard |
| `GET` | `/v1/tenants/:tenantId/push/deliveries` | Audit trail of every send attempt |

### Register a token

```bash
curl -X POST https://api.noverfly.com/v1/tenants/YOUR_TENANT_ID/push/tokens \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "platform": "ANDROID",
    "provider": "FCM",
    "token": "FCM_DEVICE_TOKEN",
    "deviceId": "UNIQUE_DEVICE_ID",
    "appBundle": "com.noverfly.app",
    "appVersion": "1.0.0",
    "locale": "fr-FR",
    "timezone": "Africa/Abidjan"
  }'
```

`platform` accepts `ANDROID`, `IOS`, `EXPO`, `WEB`. `provider` accepts `FCM`, `APNS`, `EXPO`, `WEBPUSH`.

### Send from the Dashboard

```bash
curl -X POST https://api.noverfly.com/v1/tenants/YOUR_TENANT_ID/push/send \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "title": "Nouvelle commande",
    "body": "Vous avez reçu une commande de 12 500 FCFA",
    "link": "https://app.noverfly.com/orders/abc-123",
    "imageUrl": "https://cdn.noverfly.com/order-banner.png",
    "priority": "high",
    "userIds": ["uuid-1", "uuid-2"]
  }'
```

Targeting fields are mutually compatible. Choose one or several:

- `userIds` — list of dashboard user ids
- `siteUserIds` — list of site-user ids (end users of a published site)
- `tokenIds` — list of explicit `PushToken` ids
- `toAllTenant: true` — broadcast to every active token of the tenant

---

## Cloud API Route (`gfc_` key)

For server-to-server integrations or third-party apps, use the Cloud API. The `gfc_` key already resolves the tenant scope.

| Method | Route | Auth |
|---|---|---|
| `POST` | `/v1/cloud/push/send` | `X-API-Key: gfc_...` |

```bash
curl -X POST https://api.noverfly.com/v1/cloud/push/send \
  -H "X-API-Key: gfc_YOUR_CLOUD_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "title": "Promo flash",
    "body": "-50% pendant 1 heure",
    "link": "https://app.noverfly.com/promo",
    "userIds": ["uuid-1", "uuid-2"],
    "data": { "screen": "promo" }
  }'
```

---

## Frontend Integration (NoverFly SDK)

NoverFly ships a small TypeScript helper in `src/lib/push/index.ts` that wraps the public routes.

### Subscribe a browser

```ts
import { subscribeBrowserToPush, unsubscribeBrowserFromPush } from '@/lib/push';

// On a user click — required by browsers for permission prompts
const token = await subscribeBrowserToPush();
if (token) console.log('Subscribed. token id =', token.id);

// Later
await unsubscribeBrowserFromPush();
```

The helper handles all four steps: `Notification.requestPermission()`, registration of `/sw.js`, retrieval of the VAPID public key from `GET /v1/push/vapid-public-key`, and the `POST /v1/tenants/:id/push/tokens` call.

### Register a native device token

```ts
import { registerPushToken } from '@/lib/push';

// Android (FCM)
await registerPushToken({
  platform: 'ANDROID',
  provider: 'FCM',
  token: fcmTokenFromFirebase,
  deviceId: deviceUniqueId,
  appBundle: 'com.noverfly.app',
});

// iOS (APNs direct)
await registerPushToken({
  platform: 'IOS',
  provider: 'APNS',
  token: apnsDeviceTokenHexString,
  appBundle: 'com.noverfly.ios',
});

// Expo
import * as Notifications from 'expo-notifications';
const { data: expoToken } = await Notifications.getExpoPushTokenAsync();
await registerPushToken({
  platform: 'EXPO',
  provider: 'EXPO',
  token: expoToken,
  appBundle: 'com.noverfly.expo',
});
```

---

## Backend Integration

When you create a notification through the in-app notification service, pass `push: true` to fan out automatically to every active token of the recipient.

```ts
import { notificationService } from './modules/notifications/notification.service';

await notificationService.create({
  tenantId,
  userId,
  type: 'order',
  title: 'Nouvelle commande',
  body: 'Vous avez reçu une commande de 12 500 FCFA',
  link: 'https://app.noverfly.com/orders/abc-123',
  push: true, // ← triggers FCM / APNs / Expo / WebPush in parallel
});
```

For pure push (without an in-app notification record), call `pushService.send()` directly:

```ts
import { pushService } from './modules/push/push.service';

await pushService.send(tenantId, {
  title: 'Promo flash',
  body: '-50% pendant 1 heure',
  link: 'https://app.noverfly.com/promo',
  imageUrl: 'https://cdn.noverfly.com/promo.png',
  priority: 'high',
  collapseKey: 'promo',
  toAllTenant: true,
});
```

---

## Audit & Diagnostics

Every send attempt produces a `PushDelivery` row.

```bash
curl https://api.noverfly.com/v1/tenants/YOUR_TENANT_ID/push/deliveries \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN"
```

Status values:

- `PENDING` — queued in BullMQ
- `SENT` — accepted by the upstream provider
- `FAILED` — provider returned an error (the message is recorded for debugging)

If pushes do not arrive on a device:

1. Confirm the corresponding provider is `true` in `GET /v1/push/status`.
2. Confirm the user has at least one `isActive: true` token in `GET /v1/tenants/:id/push/tokens`.
3. Inspect `GET /v1/tenants/:id/push/deliveries` for the upstream error message.
4. On the web, confirm `/sw.js` is registered in DevTools → Application → Service Workers.
5. On native apps, confirm the OS-level notification permission is granted.

---

## Provider Activation (operations)

A provider stays inactive until its credentials are present in the API environment. The required environment variables are described in the operational notes of the deployment. Once added, the API must be restarted for the provider to load.

After a restart, `GET /v1/push/status` should report the provider as `true`.

---

## Notes for Integrators

- All push-token routes accept and return UUID identifiers consistent with the rest of the NoverFly API.
- `EXPO` tokens use the platform value `EXPO` and not `ANDROID` or `IOS`.
- The platform automatically deduplicates `(provider, token)` pairs and refreshes the `updatedAt` timestamp on re-registration.
- Tokens that the upstream provider rejects as permanently invalid are deactivated; the next send to the same user simply skips them.
