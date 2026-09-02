# Client Apps Android — Push & Appels

Enregistrement des applications Android natives (APK) pour push FCM data-only, appels entrants et sécurité GFK.

## Modèle

| Entité | Rôle |
|--------|------|
| `client_apps` | App Android enregistrée (packageName, firebaseProjectId, flags) |
| `gfk_keys` | Clé SECRET liée à l'app pour signature push |
| `push_tokens` | Token FCM par device / utilisateur |

Flags importants :

- `pushEnabled` — notifications chat
- `callsEnabled` — appels entrants FCM `call_invite`

> Vocabulaire canonique : `appUserId` / `projectId` / `AppUser` / `Project`.
> `siteUserId` / `siteId` sont des alias legacy acceptés en entrée pour la rétro-compatibilité, mais `appUserId` est la forme à utiliser dans les intégrations nouvelles.

## Enregistrer un device

```http
POST /api/client/register-device
Content-Type: application/json
Authorization: Bearer SITE_USER_JWT   (optionnel)

{
  "appId": "11111111-1111-1111-1111-111111111111",
  "gfkPublicKey": "gpk_public_xxxxxxxx",
  "packageName": "com.streewi.app",
  "deviceId": "android-device-uuid",
  "fcmToken": "fcm-token-from-firebase",
  "platform": "android",
  "userId": "optional-dashboard-user-uuid",
  "appUserId": "optional-app-user-uuid",
  "appVersion": "1.2.0",
  "locale": "fr-FR",
  "timezone": "Europe/Paris"
}
```

Champs :

| Champ | Type | Requis | Description |
|-------|------|--------|-------------|
| `appId` | UUID | oui | Identifiant de la `client_app` |
| `gfkPublicKey` | string | oui | Clé publique GFK liée à l'app |
| `packageName` | string | oui | Package Android (ex. `com.streewi.app`) |
| `deviceId` | string | oui | UUID matériel Android |
| `fcmToken` | string | oui | Token FCM du device |
| `platform` | `'android'` | oui | Seul `android` est supporté |
| `userId` | UUID | non | Dashboard user propriétaire du device |
| `appUserId` | UUID | non | App user (end-user) propriétaire du device |
| `appVersion` | string | non | Version de l'APK |
| `locale` | string | non | Locale (ex. `fr-FR`) |
| `timezone` | string | non | Timezone IANA (ex. `Europe/Paris`) |

Résolution du propriétaire :

- Si `userId` ou `appUserId` est présent dans le body, il est utilisé directement.
- Sinon, si un `Authorization: Bearer` est fourni, le JWT est décodé :
  - un token `site_user` / `app_user` renseigne `appUserId`
  - un token dashboard renseigne `userId`

Réponse :

```json
{
  "ok": true,
  "deviceRegistered": true,
  "pushEnabled": true,
  "callsEnabled": true,
  "tokenId": "push-token-uuid",
  "clientAppId": "client-app-uuid",
  "projectId": "project-uuid",
  "packageName": "com.streewi.app",
  "firebaseProjectId": "streewi-6f78f",
  "registrationMode": "client_app"
}
```

Le token est upserté sur la clé unique `(provider, token)` : un ré-enregistrement met à jour le device existant au lieu d'en créer un doublon.

## Désenregistrer

```http
POST /api/client/unregister-device
Content-Type: application/json

{
  "appId": "...",
  "gfkPublicKey": "...",
  "packageName": "com.streewi.app",
  "platform": "android",
  "tokenId": "push-token-uuid"
}
```

Ou via le token FCM :

```json
{
  "appId": "...",
  "gfkPublicKey": "...",
  "packageName": "com.streewi.app",
  "platform": "android",
  "fcmToken": "fcm-token-from-firebase"
}
```

> `tokenId` **ou** `fcmToken` doit être fourni (la validation rejette un body sans aucun des deux).

Réponse (token trouvé et supprimé) :

```json
{
  "ok": true,
  "deviceUnregistered": true,
  "removed": 1,
  "tokenId": "push-token-uuid",
  "clientAppId": "client-app-uuid",
  "projectId": "project-uuid",
  "packageName": "com.streewi.app",
  "registrationMode": "client_app"
}
```

Réponse (token introuvable — idempotent) :

```json
{
  "ok": true,
  "deviceUnregistered": false,
  "removed": 0,
  "tokenId": null,
  "clientAppId": "client-app-uuid",
  "projectId": "project-uuid",
  "packageName": "com.streewi.app",
  "registrationMode": "client_app"
}
```

## Push appel entrant (cloud)

Le cloud envoie un FCM **100 % data-only** :

- `type: call_invite` — sonnerie gérée côté APK
- `type: call_cancel` — expiration / annulation

Voir [push-fcm-cloud.md](./push-fcm-cloud.md) pour `call_invite` / `call_cancel` (payload data-only FCM).

## Test DevAPI

```http
POST /api/dev/test-incoming-call-push
X-API-Key: gfk_YOUR_KEY

{
  "calleeUserId": "uuid",
  "callerName": "Test Appel",
  "callerAvatar": "https://cdn.example.com/avatar.png",
  "callMode": "audio",
  "clientAppId": "optional-client-app-uuid",
  "dryRun": true
}
```

Champs :

| Champ | Type | Requis | Défaut | Description |
|-------|------|--------|--------|-------------|
| `calleeUserId` | UUID | un des deux | — | Dashboard user destinataire |
| `calleeAppUserId` | UUID | un des deux | — | App user destinataire |
| `callerId` | string | non | `dev-test-caller` | Identifiant appelant |
| `callerName` | string | non | `Test Gloowflix` | Nom affiché appelant |
| `callerAvatar` | URL | non | — | Avatar appelant |
| `callMode` | `'audio'` \| `'video'` | non | `audio` | Mode d'appel |
| `clientAppId` | UUID | non | — | Force une client app spécifique |
| `dryRun` | boolean | non | `true` | Validation FCM sans envoi réel |

> `calleeUserId` **ou** `calleeAppUserId` doit être fourni.

Réponse :

```json
{
  "ok": true,
  "traceId": "trace-uuid",
  "callId": "call-uuid",
  "clientAppId": "client-app-uuid",
  "gfkStatus": "active",
  "packageName": "com.streewi.app",
  "fcm": {
    "sent": true,
    "dryRun": true,
    "messageId": "projects/streewi-6f78f/messages/..."
  },
  "payloadCheck": {
    "dataOnly": true,
    "hasNotificationPayload": false,
    "androidPriority": "high",
    "ttlMs": 30000,
    "allDataValuesAreString": true
  },
  "attempts": 1
}
```

## Validation côté cloud

Avant envoi push, le backend valide la `client_app` :

- `clientApp.status === ACTIVE` — sinon `CLIENT_APP_NOT_ACTIVE` (403)
- `clientApp.packageName === payload.packageName` — sinon `PACKAGE_NAME_MISMATCH` (403)
- `clientApp.pushEnabled === true` — sinon `CLIENT_APP_PUSH_DISABLED` (403)
- `clientApp.callsEnabled === true` (pour les appels entrants) — sinon `CLIENT_APP_CALLS_DISABLED` (403)
- Clé GFK liée : `keyType === SECRET`, `isActive === true`, non expirée — sinon `GFK_INACTIVE` (403)
- Config push FCM présente pour le tenant — sinon `FCM_NOT_CONFIGURED` (503)
- `clientApp.firebaseProjectId` aligné avec le `project_id` du service account FCM du tenant — sinon `FIREBASE_PROJECT_MISMATCH` (409)

Codes d'erreur de résolution :

| Code | Statut | Cause |
|------|--------|-------|
| `UNSUPPORTED_PLATFORM` | 400 | `platform` ≠ `android` |
| `CLIENT_APP_NOT_FOUND` | 404 | Aucune app pour `(appId, publicKey, ANDROID)` |
| `CLIENT_APP_NOT_ACTIVE` | 403 | App non `ACTIVE` |
| `PACKAGE_NAME_MISMATCH` | 403 | Package body ≠ package app |
| `CLIENT_APP_PUSH_DISABLED` | 403 | `pushEnabled` désactivé |
| `CLIENT_APP_CALLS_DISABLED` | 403 | `callsEnabled` désactivé |
| `GFK_INACTIVE` | 403 | Clé GFK inactive / expirée / non SECRET |
| `FCM_NOT_CONFIGURED` | 503 | Aucune config FCM pour le tenant |
| `FIREBASE_PROJECT_MISMATCH` | 409 | Project ID FCM ≠ `firebaseProjectId` app |

## Migration SQL

`prisma/migrations/20260518110000_client_apps_incoming_call_push/migration.sql`

Manual fallback : `prisma/migrations_manual/20260518_client_apps_incoming_call_push.sql`
