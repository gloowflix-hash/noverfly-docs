# Live Streaming

Le service live exposé par Noverfly s'appuie sur le module `src/modules/live/` et sur le pipeline Flivex. Il est authentifié uniquement par une clé `gfk_` **secret** liée à un site.

Base URL HTTP : `https://api.noverfly.com`  
WebSocket live : `wss://api.noverfly.com/ws`

## Prérequis

- clé `gfk_` secret liée à un site
- feature plan `live_enabled`
- entitlement qualité 420p via `live_420p` si nécessaire
- limites utilisées par le service :
  - `max_concurrent_live_streams`
  - `max_live_duration_minutes`
  - `max_live_viewers`

## Activer / désactiver

### Activer le produit

Le module lu n'expose pas de route publique dédiée du type `/activate-live-feature`. L'accès produit dépend du plan et des métadonnées tenant/site.

En pratique :

- capacité live activée : le tenant possède `live_enabled`
- activation opérationnelle d'un live : créer un stream puis le démarrer

### Désactiver

- arrêt d'un live : `POST /v1/cloud/live/streams/:id/end`
- retrait du produit live : hors routes publiques du module, via plan / administration

## Cycle de vie

```text
CREATED
  -> FLIVEX_PREPARING
  -> WAITING_FOR_SIGNAL
  -> INGEST_CONNECTING
  -> TRANSCODING / HLS_READY / PLAYBACK_CHECKING
  -> LIVE
  -> STOPPING
  -> STOPPED
```

Le service peut aussi marquer un live en `FAILED`.

## Routes principales

| Méthode | Route | Usage |
|---|---|---|
| `GET` | `/v1/cloud/live/streams` | lister les streams |
| `POST` | `/v1/cloud/live/streams` | créer un stream |
| `GET` | `/v1/cloud/live/streams/:id` | détail |
| `POST` | `/v1/cloud/live/streams/:id/start` | démarrer |
| `POST` | `/v1/cloud/live/streams/:id/end` | terminer |
| `POST` | `/v1/cloud/live/streams/:id/uplink-connected` | signaler l'uplink côté client |
| `GET` | `/v1/cloud/live/streams/:id/playback` | URLs de lecture |
| `POST` | `/v1/cloud/live/streams/:id/viewer-heartbeat` | heartbeat viewer |
| `GET` | `/v1/cloud/live/streams/:id/diagnostics` | diagnostic synthétique |
| `POST` | `/v1/cloud/live/streams/:id/preflight` | check pré-broadcast |

Alias publics courts :

- `GET /api/live/:id/diagnostics`
- `POST /api/live/:id/preflight`

## Auth

Header requis :

```http
X-Api-Key: gfk_YOUR_SECRET_KEY
```

Le code refuse :

- les clés non présentes
- les clés invalides / expirées
- les clés non `SECRET`
- les `gfk_` non liées à un site

## Créer un live

```bash
curl -X POST https://api.noverfly.com/v1/cloud/live/streams \
  -H "X-Api-Key: gfk_YOUR_SECRET_KEY" \
  -H "Content-Type: application/json" \
  -H "Idempotency-Key: live-create-20260730-001" \
  -d '{
    "title": "Live shopping été",
    "description": "Présentation des nouveautés",
    "requestedQuality": "420p",
    "clientRequestId": "live-create-20260730-001",
    "creatorProfile": {
      "userId": "user_123",
      "displayName": "Alice",
      "avatarUrl": "https://cdn.example.com/alice.jpg",
      "sellerId": "seller_9"
    }
  }'
```

### Points opérationnels importants

- `requestedQuality` accepte `320p` ou `420p`
- si le plan n'autorise pas `420p`, le service retombe à `320p`
- la réponse de création contient le `streamKey` en clair une seule fois
- le Cloud ne conserve ensuite qu'un hash de cette clé

### Idempotence

Supportée à la création par :

- `Idempotency-Key`
- `X-Idempotency-Key`
- ou `clientRequestId`

En cas de replay, la réponse renvoie `_idempotent: true`.

## Démarrer le live

```bash
curl -X POST https://api.noverfly.com/v1/cloud/live/streams/STREAM_ID/start \
  -H "X-Api-Key: gfk_YOUR_SECRET_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "streamKey": "live_xxxxxxxxxxxxxxxxx"
  }'
```

Notes :

- si vous ne renvoyez pas la bonne `streamKey`, le service démarre quand même la session côté Cloud mais ne vous redonne pas les endpoints complets
- le start déclenche l'état `INGEST_CONNECTING`

## Signaler l'uplink client

Cette route ne rend pas le live public à elle seule ; elle indique seulement que le client pense avoir commencé à pousser de la vidéo.

```bash
curl -X POST https://api.noverfly.com/v1/cloud/live/streams/STREAM_ID/uplink-connected \
  -H "X-Api-Key: gfk_YOUR_SECRET_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "source": "mobile_app",
    "details": {
      "network": "wifi"
    }
  }'
```

## Playback

```bash
curl https://api.noverfly.com/v1/cloud/live/streams/STREAM_ID/playback \
  -H "X-Api-Key: gfk_YOUR_SECRET_KEY"
```

Réponse typique :

```json
{
  "liveStreamId": "STREAM_ID",
  "status": "LIVE",
  "quality": "420p",
  "playable": true,
  "playbackUrlPrimary": "https://...",
  "playbackUrlOrigin": "https://...",
  "hlsPlaylistUrl": "https://.../index.m3u8"
}
```

## Search / list

### Lister les streams

```bash
curl "https://api.noverfly.com/v1/cloud/live/streams?status=LIVE,WAITING_FOR_SIGNAL&limit=20" \
  -H "X-Api-Key: gfk_YOUR_SECRET_KEY"
```

Filtres observés :

- `status` : liste CSV de statuts
- `limit` : max `60`

### Lire un stream

```bash
curl https://api.noverfly.com/v1/cloud/live/streams/STREAM_ID \
  -H "X-Api-Key: gfk_YOUR_SECRET_KEY"
```

Il n'existe pas de route publique de recherche textuelle des lives dans le module lu.

## Viewers et diagnostics

### Heartbeat spectateur

```bash
curl -X POST https://api.noverfly.com/v1/cloud/live/streams/STREAM_ID/viewer-heartbeat \
  -H "X-Api-Key: gfk_YOUR_SECRET_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "viewerId": "viewer_1",
    "sessionId": "session_1"
  }'
```

### Diagnostic

```bash
curl https://api.noverfly.com/v1/cloud/live/streams/STREAM_ID/diagnostics \
  -H "X-Api-Key: gfk_YOUR_SECRET_KEY"
```

Le diagnostic retourne notamment :

- `cloudStatus`
- `flivexStatus`
- `ingestReceived`
- `hlsManifestExists`
- `hlsHttpsProbeOk`
- `recommendedAction`

### Préflight

```bash
curl -X POST https://api.noverfly.com/v1/cloud/live/streams/STREAM_ID/preflight \
  -H "X-Api-Key: gfk_YOUR_SECRET_KEY"
```

Le préflight vérifie côté Cloud :

- accessibilité API
- reachability Flivex
- domaine HLS public
- présence des endpoints ingest
- présence d'une stream key
- disponibilité RTMPS / WHIP si configurés

## Notifications et temps réel liés au live

### Notifications métier

Le service déclenche des notifications serveur :

- `LIVE_STARTED`
- `LIVE_ENDED`
- `LIVE_FAILED`

Elles sont diffusées vers :

- les membres du tenant
- les abonnés / followers de la source live

Canaux activés dans le code :

- push
- websocket
- in-app

### Événements WebSocket

Le live émet :

- `live:status`
- `live:viewer_count`
- `live:chat`
- `live:comment`
- `live:reaction`

Pour recevoir les événements live via `/ws`, le client doit s'authentifier puis souscrire à la room :

```json
{
  "type": "subscribe",
  "payload": {
    "room": "live:STREAM_ID"
  }
}
```

Le contrôle d'accès est vérifié côté serveur :

- en JWT : appartenance au tenant du live
- en `gfk_` : même `tenantId` et même `siteId` que le live

## Erreurs courantes

| Code | Cause probable |
|---|---|
| `MISSING_API_KEY` | header absent |
| `INVALID_API_KEY` | clé invalide / inactive / expirée |
| `GFK_REQUIRED` | clé non secret ou mauvais type |
| `SITE_CONTEXT_REQUIRED` | clé non liée à un site |
| `FEATURE_NOT_AVAILABLE` | `live_enabled` absent |
| `REQUEST_IN_PROGRESS` | idempotency key encore en cours |
| `403` | limite de lives concurrents ou qualité non autorisée |
| `404` | stream inexistant |
| `409` | start depuis un état invalide |

## À retenir

- le live public se pilote exclusivement avec une `gfk_` secret liée au site
- il n'y a pas de route publique d'activation produit, seulement des routes de session live
- la `streamKey` n'est visible qu'à la création
- les notifications live et les événements WS sont déjà branchés dans le backend
