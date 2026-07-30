# Notifications : guide pratique

Ce guide unifie les briques réellement exposées dans le code pour :

- les notifications in-app
- les pushes FCM / APNS / Expo / Web Push
- le WebSocket temps réel `/ws`
- les événements Visual Events via `ves_`
- les notifications liées au live et aux appels

Base URL HTTP : `https://api.noverfly.com`  
WebSocket principal : `wss://api.noverfly.com/ws`

## Vue d'ensemble

| Besoin | Surface | Auth |
|---|---|---|
| Notification in-app pour un user ou site user | `/v1/cloud/notifications*` | `gfk_` ou `gfc_` |
| Push pur vers devices | `/v1/cloud/push/*` | `gfk_` ou `gfc_` |
| Configuration FCM / APNS / Expo / WebPush | `/v1/cloud/push/config/*` | `gfk_` ou `gfc_` |
| WebSocket notifications générales, présence, messenger, live | `/ws` | JWT dashboard ou `gfk_` secret + `userId` |
| Flux Visual Events | `/v1/api/visual-events/ws` | `ves_` |

## Contrats d'auth

### `/ws` principal

Le serveur `/ws` accepte :

- un JWT dashboard
- ou une `gfk_` secret avec un `userId`

Le serveur `/ws` refuse :

- les `gfc_`
- les site-user JWT

#### Auth `/ws` avec `gfk_`

```json
{
  "type": "auth",
  "payload": {
    "apiKey": "gfk_YOUR_SECRET_KEY",
    "userId": "USER_ID"
  }
}
```

#### Auth `/ws` avec JWT

```json
{
  "type": "auth",
  "payload": {
    "token": "YOUR_DASHBOARD_JWT",
    "tenantId": "YOUR_TENANT_ID"
  }
}
```

### WebSocket Visual Events

Visual Events n'utilise pas `/ws`. Il utilise :

```text
/v1/api/visual-events/ws?sessionId=...&token=ves_...
```

Le token `ves_` est émis lors de la création de session Visual Events.

## 1. Notifications in-app

### Prérequis

- clé `gfk_` ou `gfc_`
- cible connue : `userId`, `receiverId` ou `siteUserId`
- option `push: true` si vous voulez aussi réveiller les devices

### Routes principales

| Méthode | Route | Usage |
|---|---|---|
| `POST` | `/v1/cloud/notifications` | créer une notification |
| `GET` | `/v1/cloud/notifications` | lister |
| `GET` | `/v1/cloud/notifications/unread-count` | compteur unread |
| `POST` | `/v1/cloud/notifications/:notifId/read` | marquer lu |
| `POST` | `/v1/cloud/notifications/:notifId/seen` | marquer vu |
| `POST` | `/v1/cloud/notifications/read-all` | tout marquer lu |
| `DELETE` | `/v1/cloud/notifications/:notifId` | soft delete |
| `POST` | `/v1/cloud/notifications/email` | email SMTP, `gfk_` secret uniquement |

### Créer une notification in-app

```bash
curl -X POST https://api.noverfly.com/v1/cloud/notifications \
  -H "X-Api-Key: gfk_YOUR_SECRET_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "siteUserId": "SITE_USER_ID",
    "type": "ORDER_CREATED",
    "title": "Nouvelle commande",
    "body": "Commande #123 reçue",
    "link": "/orders/123",
    "push": true,
    "dedupeKey": "order-123-created"
  }'
```

### Réponse type

```json
{
  "notification": {
    "id": "uuid",
    "type": "ORDER_CREATED",
    "title": "Nouvelle commande",
    "message": "Commande #123 reçue"
  }
}
```

### Contrat temps réel associé

Quand l'orchestrateur est actif, le backend émet aussi :

- `notification:new`
- `notification:removed`
- `devapi:notification_new`
- `devapi:notification_removed`
- `devapi:notification_read`
- `devapi:notification_all_read`

Le contrat normalisé détaillé reste documenté dans `notifications-realtime-contract.md`.

## 2. Push devices

### Activer / désactiver

Il n'existe pas une unique route "activate notifications". En pratique, vous activez les capacités provider par provider :

- FCM : `PUT /v1/cloud/push/config/fcm`
- APNS : `PUT /v1/cloud/push/config/apns`
- WEBPUSH : `PUT /v1/cloud/push/config/webpush`
- EXPO : `PUT /v1/cloud/push/config/expo`

Désactivation :

- `DELETE /v1/cloud/push/config/:provider`

### Routes principales

| Méthode | Route | Usage |
|---|---|---|
| `POST` | `/v1/cloud/push/tokens` | enregistrer un token device |
| `GET` | `/v1/cloud/push/tokens` | lister les tokens |
| `DELETE` | `/v1/cloud/push/tokens/:tokenId` | supprimer un token |
| `POST` | `/v1/cloud/push/send` | envoyer un push |
| `GET` | `/v1/cloud/push/status` | statut providers côté tenant |
| `GET` | `/v1/cloud/push/config` | résumé config sans secrets |
| `POST` | `/v1/cloud/push/config/fcm/validate` | valider la config FCM |

### Enregistrer un token

```bash
curl -X POST https://api.noverfly.com/v1/cloud/push/tokens \
  -H "X-Api-Key: gfc_YOUR_CLOUD_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "platform": "ANDROID",
    "provider": "FCM",
    "token": "FCM_DEVICE_TOKEN",
    "deviceId": "device-001",
    "appBundle": "com.example.app",
    "siteUserId": "SITE_USER_ID",
    "pushEnabled": true,
    "callsEnabled": true
  }'
```

### Envoyer un push simple

```bash
curl -X POST https://api.noverfly.com/v1/cloud/push/send \
  -H "X-Api-Key: gfc_YOUR_CLOUD_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "title": "Promo flash",
    "body": "-50% pendant 1 heure",
    "siteUserIds": ["SITE_USER_ID"],
    "priority": "high",
    "data": {
      "screen": "promo"
    }
  }'
```

### Configurer FCM

```bash
curl -X PUT https://api.noverfly.com/v1/cloud/push/config/fcm \
  -H "X-Api-Key: gfk_YOUR_SECRET_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "serviceAccount": {
      "project_id": "firebase-project",
      "client_email": "firebase-adminsdk@firebase-project.iam.gserviceaccount.com",
      "private_key": "-----BEGIN PRIVATE KEY-----\n...\n-----END PRIVATE KEY-----\n"
    }
  }'
```

### Valider FCM

```bash
curl -X POST https://api.noverfly.com/v1/cloud/push/config/fcm/validate \
  -H "X-Api-Key: gfk_YOUR_SECRET_KEY" \
  -H "Content-Type: application/json" \
  -d '{}'
```

## 3. Public push routes

Routes publiques utiles :

| Méthode | Route | Remarque |
|---|---|---|
| `GET` | `/v1/push/status` | check public minimal, pas le détail provider du tenant |
| `GET` | `/v1/push/vapid-public-key?tenantId=...` | clé VAPID publique web |

Le détail providers tenant-side se lit via `GET /v1/cloud/push/status` ou `GET /v1/tenants/:tenantId/push/config/status`.

## 4. Événements WebSocket métier

### Notifications génériques

- `notification:new`
- `notification:removed`

### Messenger / présence

- `messenger:message:new`
- `messenger:message:delivered`
- `messenger:message:read`
- `messenger:voice:new`
- `messenger:voice:listened`
- `messenger:conversation:active`
- `messenger:conversation:inactive`
- `messenger:typing:start`
- `messenger:typing:stop`
- `messenger:voice_recording:start`
- `messenger:voice_recording:stop`
- `presence:user:online`
- `presence:user:offline`
- `presence:user:last_seen`

### Appels Messenger

- `call:created`
- `call:incoming`
- `call:offer`
- `call:answer`
- `call:ice`
- `call:hangup`
- `call:reject`
- `call:busy`
- `call:expired`

### Live

- `live:status`
- `live:viewer_count`
- `live:chat`
- `live:comment`
- `live:reaction`

## 5. Live et appels : notifications liées

### Appels entrants Messenger

Le code a une pipeline dédiée FCM data-only pour :

- `call_invite` (`eventType = call.incoming`)
- `call_cancel` (`eventType = call.expired`, `call.cancelled` ou `call.declined`)

Champs principaux observés :

- `callId`
- `conversationId`
- `fromId`
- `callerName`
- `callerAvatarUrl`
- `callMode`
- `callProvider`
- `sentAt`
- `expiresAt`
- `deepLink`
- `clickAction = OPEN_INCOMING_CALL`
- `channelId = incoming_calls`

### Live

Le module live diffuse :

- notifications métier `LIVE_STARTED`, `LIVE_ENDED`, `LIVE_FAILED`
- fan-out push + websocket + in-app
- événements WS `live:status` et `live:viewer_count`

## 6. Visual Events

### Démarrage

1. créer une session Visual Events
2. récupérer le token `ves_`
3. ouvrir `/v1/api/visual-events/ws`

### Exemple session

```bash
curl -X POST https://api.noverfly.com/v1/api/visual-events/sessions \
  -H "Authorization: Bearer vst_YOUR_VISUAL_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "mediaId": "MEDIA_ID",
    "mediaType": "stream",
    "ttlSeconds": 900
  }'
```

### Exemple WS

```text
wss://api.noverfly.com/v1/api/visual-events/ws?sessionId=VS_ID&token=ves_...
```

Événements possibles :

- `visual.session.ready`
- `visual.processing.started`
- `visual.scene.changed`
- `visual.object.entered`
- `visual.object.updated`
- `visual.object.left`
- `visual.object.detected`
- `visual.concept.detected`
- `visual.effect.triggered`
- `visual.processing.progress`
- `visual.processing.completed`
- `visual.processing.failed`
- `visual.session.expired`

## 7. Quand passer par l'orchestrateur plutôt que du push brut ?

Passez par l'orchestrateur (`/v1/cloud/notifications`, `push: true`) quand vous voulez :

- une notification in-app persistée
- le fan-out WS + push
- le respect des préférences, quotas, quiet hours, digest
- la déduplication métier (`dedupeKey`)

Passez par `/v1/cloud/push/send` quand vous voulez :

- un push pur sans enregistrement in-app
- une campagne / action backend simple
- un payload data-only contrôlé

## Erreurs courantes

| Code | Cause probable |
|---|---|
| `MISSING_API_KEY` | clé API absente |
| `INVALID_API_KEY` | clé invalide / inactive |
| `FORBIDDEN_KEY_TYPE` | `/v1/cloud/notifications/email` appelé avec `gfc_` |
| `KEY_NOT_SECRET` | `/ws` essayé avec `gfc_` |
| `MISSING_USER_ID` | auth `/ws` avec clé mais sans `userId` |
| `SMTP_NOT_CONFIGURED` | email demandé sans profil SMTP |
| `FCM_NOT_CONFIGURED` | validation FCM sans config stockée |
| `FCM_CREDENTIAL_VALIDATION_FAILED` | JSON Firebase invalide |
| `VISUAL_EVENTS_DISABLED` | Visual Events désactivé sur le site |

## Guides complémentaires

- `push-notifications.md` pour les routes tenant JWT
- `push-fcm-cloud.md` pour les payloads FCM chat / appels Android
- `notifications-realtime-contract.md` pour le contrat WS détaillé
- `calls-audio-video.md` pour les flows appels
- `live-streaming.md` pour les notifications live
