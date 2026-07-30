# Appels audio / vidéo

Ce guide couvre les deux surfaces réellement présentes dans le code Noverfly pour l'audio, la vidéo et le realtime media :

1. `Messenger + /ws` pour le chat, les vocaux et les appels WebRTC historiques.
2. `Calls API` pour les appels 1:1, groupe et live rooms avec clés dédiées `nfk_*`.

Base URL HTTP : `https://api.noverfly.com`  
WebSocket Messenger : `wss://api.noverfly.com/ws`  
WebSocket Calls API : `wss://api.noverfly.com/v1/realtime?token=...`

## Quelle surface choisir ?

| Besoin | Surface recommandée | Auth principale |
|---|---|---|
| Chat, messages vocaux, présence, historique conversations | `Messenger` | `gfk_` / `gfc_` pour le REST, `gfk_` secret ou JWT pour `/ws` |
| Appel 1:1 simple déjà couplé à Messenger | `Messenger` | idem |
| Appel 1:1 ou groupe côté backend produit | `Calls API` | activation par `gfk_` admin, exécution par `nfk_*` |
| Live audio / live vidéo multi-participants | `Calls API` | `nfk_*` |
| Historique de call Messenger | `GET /v1/cloud/messenger/calls` | `gfk_` / `gfc_` |

## Surface 1 : Messenger historique

### Prérequis

- Feature flag plan / tenant : `messenger_enabled`
- Pour la vidéo : entitlement `messenger_video_calls`
- Quotas observés dans le code :
  - `max_messages_month`
  - `max_voice_messages_month`
  - `max_calls_month`
  - `max_call_minutes_month`
- REST DevAPI : `X-Api-Key: gfk_...` ou `X-Api-Key: gfc_...`
- WebSocket `/ws` : JWT dashboard ou `gfk_` secret uniquement. `gfc_` est refusée sur `/ws`.

### Activer / désactiver

Il n'existe pas de route DevAPI publique dédiée pour activer ou désactiver `messenger_enabled` dans `src/modules/messenger`. L'activation est pilotée par le plan et la configuration tenant/site.

En pratique :

- activation fonctionnelle : disposer d'un tenant/site avec `messenger_enabled`
- désactivation fonctionnelle : retirer le flag côté admin / plan, ou cesser d'ouvrir `/ws`

### Routes principales

| Méthode | Route | Usage |
|---|---|---|
| `GET` | `/v1/cloud/messenger/rtc-config` | Récupérer STUN/TURN et l'endpoint WS |
| `POST` | `/v1/cloud/messenger/conversations` | Créer une conversation directe |
| `GET` | `/v1/cloud/messenger/conversations` | Lister les conversations d'un utilisateur |
| `POST` | `/v1/cloud/messenger/conversations/:conversationId/messages` | Envoyer un message |
| `GET` | `/v1/cloud/messenger/conversations/:conversationId/messages` | Historique |
| `POST` | `/v1/cloud/messenger/messages/:messageId/delivered` | Receipt "delivered" |
| `POST` | `/v1/cloud/messenger/messages/:messageId/read` | Receipt "read" |
| `POST` | `/v1/cloud/messenger/messages/:messageId/listened` | Receipt vocal écouté |
| `GET` | `/v1/cloud/messenger/conversations/:conversationId/receipts` | Lister les receipts |
| `POST` | `/v1/cloud/messenger/conversations/:conversationId/focus` | Conversation active |
| `POST` | `/v1/cloud/messenger/conversations/:conversationId/blur` | Conversation inactive |
| `POST` | `/v1/cloud/messenger/conversations/:conversationId/presence` | Presence active / inactive |
| `GET` | `/v1/cloud/messenger/presence` | Présence utilisateurs |
| `POST` | `/v1/cloud/messenger/conversations/:conversationId/voice` | Créer un message vocal |
| `GET` | `/v1/cloud/messenger/calls` | Historique des appels |

### Créer un appel 1:1

1. Créer ou retrouver la conversation directe.
2. Lire la config RTC.
3. Ouvrir `/ws`.
4. Authentifier la socket.
5. Envoyer `call:initiate`.
6. Échanger `call:offer`, `call:answer`, `call:ice`.
7. Terminer avec `call:hangup`, `call:reject` ou `call:busy`.

#### Exemple : config RTC

```bash
curl https://api.noverfly.com/v1/cloud/messenger/rtc-config \
  -H "X-Api-Key: gfk_YOUR_SECRET_KEY"
```

#### Exemple : créer la conversation

```bash
curl -X POST https://api.noverfly.com/v1/cloud/messenger/conversations \
  -H "X-Api-Key: gfk_YOUR_SECRET_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "userIdA": "USER_CALLER",
    "userIdB": "USER_CALLEE"
  }'
```

#### Auth WebSocket avec `gfk_`

```json
{
  "type": "auth",
  "payload": {
    "apiKey": "gfk_YOUR_SECRET_KEY",
    "userId": "USER_CALLER"
  }
}
```

#### Démarrer un appel audio

```json
{
  "type": "call:initiate",
  "payload": {
    "targetUserId": "USER_CALLEE",
    "callType": "AUDIO",
    "tenantId": "YOUR_TENANT_ID"
  }
}
```

#### Démarrer un appel vidéo

```json
{
  "type": "call:initiate",
  "payload": {
    "targetUserId": "USER_CALLEE",
    "callType": "VIDEO",
    "tenantId": "YOUR_TENANT_ID"
  }
}
```

### Appels de groupe

Le module `messenger` n'expose pas de route publique pour créer une room groupe. Dans l'état actuel du code, les groupes audio/vidéo passent par la `Calls API` et non par `Messenger`.

### Search / list

- conversations : `GET /v1/cloud/messenger/conversations`
- messages : `GET /v1/cloud/messenger/conversations/:conversationId/messages`
- receipts : `GET /v1/cloud/messenger/conversations/:conversationId/receipts`
- call history : `GET /v1/cloud/messenger/calls`

Il n'existe pas de route publique de recherche plein texte des appels dans le module lu.

### Notifications liées aux appels Messenger

Le code déclenche :

- événements WS : `call:created`, `call:incoming`, `call:offer`, `call:answer`, `call:ice`, `call:hangup`, `call:reject`, `call:busy`, `call:expired`
- push FCM data-only pour appels entrants / annulation
- notifications métier serveur `MESSENGER_CALL_INCOMING` et `MESSENGER_CALL_MISSED`

Pour le détail du payload push Android, voir `push-fcm-cloud.md`.

### Erreurs courantes

| Code | Cause probable |
|---|---|
| `MISSING_API_KEY` | header `X-Api-Key` absent |
| `INVALID_API_KEY` | clé invalide / inactive / expirée |
| `KEY_NOT_SECRET` | tentative d'auth `/ws` avec `gfc_` |
| `FEATURE_NOT_AVAILABLE` | `messenger_enabled` ou `messenger_video_calls` absent |
| `QUOTA_EXCEEDED` | quotas messages, vocaux, appels ou minutes atteints |
| `SENDER_NOT_PARTICIPANT` | l'émetteur n'appartient pas à la conversation |

## Surface 2 : Calls API moderne

La `Calls API` est un produit séparé. Elle expose ses propres clés `nfk_*`, ses limites, ses rooms et son WebSocket `/v1/realtime`.

### Prérequis

- plan tenant au moins `40 USD/mois` (message de code : `Pro or higher`)
- tenant `ACTIVE`
- activation initiale via une clé `gfk_` secret admin ou `Authorization: Bearer gfk_...`
- exécution avec une clé `nfk_live_*` ou `nfk_test_*`

### Activer

#### Vérifier le statut

```bash
curl https://api.noverfly.com/v1/cloud/calls/status \
  -H "X-Gfk-Key: gfk_YOUR_ADMIN_KEY"
```

#### Activer le produit

```bash
curl -X POST https://api.noverfly.com/v1/cloud/calls/activate \
  -H "X-Gfk-Key: gfk_YOUR_ADMIN_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Support Calls",
    "environment": "PRODUCTION"
  }'
```

Réponse attendue : abonnement Calls + application + `apiKey` en clair une seule fois.

#### Obtenir une clé serveur `nfk_*`

```bash
curl -X POST https://api.noverfly.com/v1/cloud/calls/ensure \
  -H "X-Gfk-Key: gfk_YOUR_ADMIN_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Backend Calls",
    "environment": "PRODUCTION",
    "rotate": true
  }'
```

### Désactiver

Le module ne publie pas de route `/deactivate`. Les options réelles vues dans le code sont :

- révoquer une clé `nfk_*` via `POST /v1/cloud/calls/keys/:keyId/revoke`
- faire une rotation via `POST /v1/cloud/calls/keys/rotate`
- terminer les rooms existantes avec `POST /v1/calls/:callId/end`

Pour une désactivation produit complète, il faut passer par l'administration / facturation hors routes publiques du module lu.

### Limites par défaut du plan `calls_api_40`

| Mode | Limite participants | Limite publishers | Durée max |
|---|---:|---:|---:|
| `audio_direct` | 2 | 2 | 3600 s |
| `video_direct` | 2 | 2 | 1800 s |
| `audio_group` | 8 | 8 | 3600 s |
| `video_group` | 6 | 6 | 1800 s |
| `live_audio` | viewers illimités par défaut | 8 | 3600 s |
| `live_video` | viewers illimités par défaut | 6 | 1800 s |

Par défaut :

- `allowRecording = false`
- `allowEgress = false`
- `allowSip = false`

### Routes principales

Toutes les routes ci-dessous utilisent `Authorization: Bearer nfk_...`.

| Méthode | Route | Usage |
|---|---|---|
| `POST` | `/v1/calls` | Créer une room audio/video 1:1 ou groupe |
| `GET` | `/v1/calls/:callId` | Lire une room |
| `POST` | `/v1/calls/:callId/invite` | Inviter des participants |
| `POST` | `/v1/calls/:callId/accept` | Accepter |
| `POST` | `/v1/calls/:callId/reject` | Refuser |
| `POST` | `/v1/calls/:callId/cancel` | Annuler avant connexion |
| `POST` | `/v1/calls/:callId/join-token` | Générer token LiveKit + signaling + ICE |
| `POST` | `/v1/calls/:callId/leave` | Quitter |
| `POST` | `/v1/calls/:callId/end` | Terminer |
| `GET` | `/v1/calls/:callId/participants` | Lister les participants |
| `POST` | `/v1/calls/:callId/participants/:participantId/remove` | Expulser |
| `POST` | `/v1/calls/:callId/participants/:participantId/mute` | Mute une track |
| `POST` | `/v1/calls/:callId/participants/:participantId/role` | Changer le rôle |
| `GET` | `/v1/calls/:callId/usage` | Usage par room |
| `POST` | `/v1/live/rooms` | Créer une live room audio/video |
| `GET` | `/v1/live/rooms/:callId` | Lire une live room |
| `POST` | `/v1/live/rooms/:callId/join-token` | Join token live room |
| `POST` | `/v1/live/rooms/:callId/end` | Terminer une live room |
| `POST` | `/v1/realtime/turn-credentials` | Délivrer credentials TURN |
| `GET` | `/v1/usage` | Usage agrégé |
| `GET` | `/v1/subscription` | Abonnement Calls |
| `GET` | `/v1/limits` | Limites du plan |

### Créer un appel 1:1

```bash
curl -X POST https://api.noverfly.com/v1/calls \
  -H "Authorization: Bearer nfk_live_YOUR_SERVER_KEY" \
  -H "Content-Type: application/json" \
  -H "Idempotency-Key: calls-audio-direct-001" \
  -d '{
    "mode": "audio_direct",
    "creator": {
      "externalUserId": "alice",
      "displayName": "Alice"
    },
    "participants": [
      {
        "externalUserId": "bob",
        "displayName": "Bob"
      }
    ],
    "recordingEnabled": false,
    "metadata": {
      "ticketId": "SUP-42"
    }
  }'
```

### Créer un appel groupe vidéo

```bash
curl -X POST https://api.noverfly.com/v1/calls \
  -H "Authorization: Bearer nfk_live_YOUR_SERVER_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "mode": "video_group",
    "creator": {
      "externalUserId": "host-1",
      "displayName": "Host"
    },
    "participants": [
      { "externalUserId": "u-2", "displayName": "U2" },
      { "externalUserId": "u-3", "displayName": "U3" }
    ]
  }'
```

### Créer une live room

```bash
curl -X POST https://api.noverfly.com/v1/live/rooms \
  -H "Authorization: Bearer nfk_live_YOUR_SERVER_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "mode": "live_video",
    "creator": {
      "externalUserId": "host-1",
      "displayName": "Host"
    },
    "participants": [
      { "externalUserId": "viewer-1", "displayName": "Viewer 1" }
    ]
  }'
```

### Rejoindre une room

```bash
curl -X POST https://api.noverfly.com/v1/calls/CALL_ID/join-token \
  -H "Authorization: Bearer nfk_live_YOUR_SERVER_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "externalUserId": "alice",
    "displayName": "Alice",
    "role": "participant"
  }'
```

Extrait de réponse :

```json
{
  "callId": "call_...",
  "serverUrl": "wss://...",
  "participantToken": "LIVEKIT_TOKEN",
  "signalingToken": "JWT_FOR_/v1/realtime",
  "iceServers": [
    {
      "urls": ["turn:..."],
      "username": "..."
    }
  ],
  "expiresAt": "2026-07-30T22:00:00.000Z"
}
```

Le `signalingToken` sert ensuite à ouvrir :

```text
wss://api.noverfly.com/v1/realtime?token=SIGNALING_TOKEN
```

### Search / list

Le module `src/modules/realtime-calls` n'expose pas de route publique de liste globale ou de recherche de rooms.

Ce qui existe réellement :

- `GET /v1/calls/:callId`
- `GET /v1/calls/:callId/participants`
- `GET /v1/calls/:callId/usage`
- `GET /v1/usage`

Si vous avez besoin d'une vue globale applicative, elle doit aujourd'hui être reconstruite côté backend appelant à partir des `callId` que vous créez ou des webhooks.

### Erreurs courantes

| Code | Cause probable |
|---|---|
| `INVALID_GFK_KEY` | activation Calls avec mauvaise clé admin |
| `PLAN_REQUIRED` | tenant sous le minimum Calls |
| `CALLS_NOT_ACTIVATED` | produit non activé avant émission de clé |
| `INVALID_API_KEY` | `nfk_*` invalide, expirée ou révoquée |
| `CONCURRENT_ROOM_LIMIT_REACHED` | limite de rooms concurrentes atteinte |
| `PARTICIPANT_LIMIT_REACHED` | limite de participants atteinte |
| `VIEWER_LIMIT_REACHED` | limite viewers atteinte |
| `PUBLISHER_LIMIT_REACHED` | limite publishers atteinte |
| `REGION_UNAVAILABLE` | région demandée indisponible |
| `ROOM_ALREADY_ENDED` | room déjà terminée |
| `CALL_MAX_DURATION_REACHED` | durée max atteinte |

## À retenir

- `Messenger` = surface historique chat + présence + appels 1:1.
- `Calls API` = surface moderne pour 1:1, groupe et live rooms.
- `gfc_` peut appeler le REST Messenger, mais jamais `/ws`.
- `Calls API` s'active avec `gfk_`, puis s'exécute avec `nfk_*`.
- Il n'existe pas de route publique de recherche globale des calls dans les modules lus.
