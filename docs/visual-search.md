# Visual Search

Le module Visual Search expose une surface client sécurisée par token court `vst_`, une surface serveur `gfk_`, et une extension temps réel `Visual Events` via token `ves_`.

Base URL HTTP : `https://api.noverfly.com`

## Auth et clés

| Usage | Auth | Remarques |
|---|---|---|
| Échange de token client | `appId` + `publicKey` | retourne un `vst_` |
| Recherche client (image / média / texte) | `Authorization: Bearer vst_...` ou `X-Visual-Search-Token` | token court, recommandé côté app |
| Recherche serveur | `X-Api-Key: gfk_...` | `gfc_` refusée |
| Activation / config / backfill | `gfk_` secret `ADMIN` | clé liée à un site |
| WebSocket Visual Events | `ves_` | délivré par création de session |

Le code rejette explicitement :

- `gfc_` sur les routes Visual Search
- les clients mobiles qui embarqueraient une clé serveur au lieu d'un `vst_`

## Prérequis plan / entitlements

L'activation est conditionnée par le produit Visual Search côté plan. Le code vérifie un accès de type `Pro (40 USD/30 jours) ou Enterprise`.

Entitlements observés :

- `visual_search.enabled`
- `visual_search.api_access`
- `visual_search.image_to_image`
- `visual_search.text_to_image`
- `visual_search.indexing`
- `visual_search.video_search`
- `visual_search.object_aware`
- `visual_search.visual_events`
- `visual_search.visual_effects`

Tant que le site n'est pas activé, les recherches sont bloquées avec `VISUAL_SEARCH_DISABLED_FOR_SITE`.

## Activer

### 1. État courant

```bash
curl https://api.noverfly.com/v1/api/visual-search/config \
  -H "X-Api-Key: gfk_YOUR_ADMIN_KEY"
```

### 2. Activation

```bash
curl -X POST https://api.noverfly.com/v1/api/visual-search/activate \
  -H "X-Api-Key: gfk_YOUR_ADMIN_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "appId": "YOUR_CLIENT_APP_ID",
    "overageMode": "block",
    "startBackfill": true,
    "collectionName": "assets",
    "videoSearchEnabled": true,
    "objectAwareEnabled": true
  }'
```

Cette route :

- active le site
- provisionne les scripts `visual-search`, `visual-index`, `visual-status`, `visual-usage`
- peut lancer un backfill initial

### 3. Mise à jour fine de la config

Tous les flags avancés ci-dessous restent **désactivés par défaut**.

```bash
curl -X PUT https://api.noverfly.com/v1/api/visual-search/config \
  -H "X-Api-Key: gfk_YOUR_ADMIN_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "enabled": true,
    "videoSearchEnabled": true,
    "objectAwareEnabled": true,
    "clothingDetectionEnabled": false,
    "logoDetectionEnabled": false,
    "brandMatchingEnabled": false,
    "ocrEnabled": false,
    "colorAnalysisEnabled": false,
    "poseDetectionEnabled": false,
    "visualStyleEnabled": false,
    "visualEventsEnabled": false,
    "visualEventsEnabled": true,
    "visualEffectsEnabled": false,
    "liveObjectTrackingEnabled": true,
    "maximumEventsPerSecond": 15,
    "eventRetentionSeconds": 600,
    "maximumConcurrentSessions": 20,
    "minimumDetectionConfidence": 0.55
  }'
```

## Désactiver

```bash
curl -X POST https://api.noverfly.com/v1/api/visual-search/deactivate \
  -H "X-Api-Key: gfk_YOUR_ADMIN_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "appId": "YOUR_CLIENT_APP_ID"
  }'
```

Effet observé dans le code : recherches et indexation sont bloquées pour le site.

## Routes principales

### Recherche et usage

| Méthode | Route | Usage |
|---|---|---|
| `POST` | `/v1/api/visual-search/token` | échanger `appId + publicKey` contre un `vst_` |
| `POST` | `/v1/api/visual-search/media` | recherche "media to media" |
| `POST` | `/v1/api/visual-search/image` | upload image puis recherche |
| `POST` | `/v1/api/visual-search/text` | recherche texte |
| `GET` | `/v1/api/visual-search/status/:jobId` | état job |
| `GET` | `/v1/api/visual-search/usage` | quotas / usage |

### Admin site

| Méthode | Route | Usage |
|---|---|---|
| `POST` | `/v1/api/visual-search/activate` | activer |
| `POST` | `/v1/api/visual-search/deactivate` | désactiver |
| `GET` | `/v1/api/visual-search/config` | lire la config |
| `PUT` | `/v1/api/visual-search/config` | mettre à jour la config |
| `POST` | `/v1/api/visual-search/backfill` | lancer / reprendre un backfill |

### Dashboard JWT

Les mêmes opérations existent côté dashboard :

- `PUT /v1/sites/:siteId/visual-search/config`
- `GET /v1/sites/:siteId/visual-search/config`
- `POST /v1/sites/:siteId/visual-search/backfill`
- `POST /v1/sites/:siteId/visual-search/provision-scripts`

## Démarrage client : obtenir un `vst_`

```bash
curl -X POST https://api.noverfly.com/v1/api/visual-search/token \
  -H "Content-Type: application/json" \
  -d '{
    "appId": "YOUR_CLIENT_APP_ID",
    "publicKey": "YOUR_CLIENT_PUBLIC_KEY",
    "packageName": "com.example.app"
  }'
```

Réponse typique :

```json
{
  "success": true,
  "token": "vst_...",
  "expiresIn": 900,
  "expiresAt": "2026-07-30T22:00:00.000Z"
}
```

## Rechercher par média indexé

Utilisez cette route quand vous connaissez déjà un `mediaId` indexé ou un asset lié.

```bash
curl -X POST https://api.noverfly.com/v1/api/visual-search/media \
  -H "Authorization: Bearer vst_YOUR_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "mediaId": "MEDIA_ID",
    "limit": 24,
    "includeVideos": true,
    "mediaTypes": ["IMAGE", "VIDEO"],
    "queryObjects": ["shoe", "bag"],
    "queryScenes": ["street"],
    "filters": {
      "brandId": "brand_123"
    }
  }'
```

## Rechercher par image uploadée

Cette route est `multipart/form-data` et attend un champ `image`.

```bash
curl -X POST https://api.noverfly.com/v1/api/visual-search/image \
  -H "Authorization: Bearer vst_YOUR_TOKEN" \
  -F "image=@C:\images\query.jpg" \
  -F "limit=12" \
  -F "filters={\"categoryId\":\"cat_001\"}"
```

## Rechercher par texte

```bash
curl -X POST https://api.noverfly.com/v1/api/visual-search/text \
  -H "Authorization: Bearer vst_YOUR_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "query": "robe rouge elegante",
    "limit": 24,
    "includeVideos": true,
    "mediaTypes": ["IMAGE", "VIDEO"],
    "queryObjects": ["dress"],
    "queryScenes": ["runway"]
  }'
```

## Réponse type

```json
{
  "requestId": "uuid",
  "mode": "text_to_image",
  "items": [
    {
      "mediaId": "asset_123",
      "postId": "post_456",
      "mediaType": "IMAGE",
      "thumbnailUrl": "https://cdn.example.com/thumb.jpg",
      "matchReason": "visually_similar",
      "score": 0.92,
      "similarityPercent": 92
    }
  ],
  "hasMore": false,
  "usage": {
    "limit": 1000,
    "used": 10,
    "remaining": 990
  }
}
```

## Indexation : ce qui est public, ce qui ne l'est pas

### Ce qui est public

- activer le site
- lancer un `backfill`
- suivre l'état du job via `GET /v1/api/visual-search/status/:jobId`

### Ce qui n'est pas exposé comme route publique dédiée

Dans le module lu, il n'existe pas de route publique `POST /index` ou `POST /delete-index`.

L'indexation réelle est déclenchée :

- par les services internes (`enqueueIndexMedia`, `enqueueDeleteMedia`)
- par les jobs de `backfill`
- par les webhooks / traitements Flivex (`visual.semantic.ready`)

Si vous avez besoin d'un "index now" manuel, il faut aujourd'hui passer par le backfill, le pipeline d'assets, ou un script interne provisionné.

## Visual Events

Visual Events est documentable dans le code actuel. Il s'appuie sur la config Visual Search du site.

### Prérequis

- `visualEventsEnabled = true`
- pour les règles d'effets : `visualEffectsEnabled = true`
- pour le tracking continu live : `liveObjectTrackingEnabled = true`
- entitlement plan `visual_search.visual_events`

### Sessions

| Méthode | Route | Auth |
|---|---|---|
| `POST` | `/v1/api/visual-events/sessions` | `vst_` ou `gfk_` |
| `GET` | `/v1/api/visual-events/sessions/:sessionId` | `vst_` ou `gfk_` |
| `POST` | `/v1/api/visual-events/sessions/:sessionId/close` | `vst_` ou `gfk_` |
| `GET` WS | `/v1/api/visual-events/ws?...` | token `ves_` |

### Créer une session

```bash
curl -X POST https://api.noverfly.com/v1/api/visual-events/sessions \
  -H "Authorization: Bearer vst_YOUR_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "mediaId": "MEDIA_ID",
    "mediaType": "video",
    "ttlSeconds": 900,
    "permissions": ["visual.events.subscribe"]
  }'
```

Réponse type :

```json
{
  "success": true,
  "sessionId": "vs_...",
  "token": "ves_...",
  "wsUrl": "/v1/api/visual-events/ws?sessionId=vs_...&token=ves_..."
}
```

### WebSocket Visual Events

Connexion :

```text
wss://api.noverfly.com/v1/api/visual-events/ws?sessionId=VS_ID&token=ves_...
```

Événements connus :

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

Limites par défaut observées dans le code :

- `maximumConcurrentSessions = 20`
- `maxWsConnectionsPerTenant = 50`
- `maxWsConnectionsPerApp = 20`
- `maxMessageBytes = 16384`

### Règles Visual Effects

| Méthode | Route |
|---|---|
| `POST` | `/v1/api/visual-effects/rules` |
| `GET` | `/v1/api/visual-effects/rules` |
| `PATCH` | `/v1/api/visual-effects/rules/:ruleId` |
| `DELETE` | `/v1/api/visual-effects/rules/:ruleId` |

## Erreurs courantes

| Code | Cause probable |
|---|---|
| `MISSING_VISUAL_AUTH` | aucun `vst_` ni `gfk_` fourni |
| `INVALID_VISUAL_TOKEN` | token `vst_` invalide ou expiré |
| `CLOUD_KEY_NOT_ALLOWED` | tentative avec `gfc_` |
| `ADMIN_PERMISSION_REQUIRED` | activation/config sans `gfk_` `ADMIN` |
| `SITE_REQUIRED` | clé `gfk_` non liée à un site |
| `VISUAL_SEARCH_PLAN_REQUIRED` | plan sans produit Visual Search |
| `VISUAL_SEARCH_DISABLED_FOR_SITE` | site pas encore activé |
| `VISUAL_SEARCH_VIDEO_NOT_ENTITLED` | vidéo non permise |
| `VISUAL_SEARCH_OBJECT_NOT_ENTITLED` | object-aware non permis |
| `VISUAL_EVENTS_NOT_ENTITLED` | Visual Events non permis |
| `VISUAL_EFFECTS_NOT_ENTITLED` | Visual Effects non permis |
| `VISUAL_SEARCH_UPLOAD_TOO_LARGE` | image trop lourde |
| `VISUAL_SEARCH_INVALID_MIME` | upload non image |
| `VISUAL_SEARCH_QUOTA_EXCEEDED` | quota atteint |
| `VISUAL_EVENTS_SESSION_LIMIT` | trop de sessions ouvertes |

## À retenir

- côté client, on échange `appId + publicKey` contre un `vst_`
- côté serveur, seules les `gfk_` secret sont acceptées
- l'index n'a pas de route publique dédiée "index now"
- `Visual Events` se consomme en `ves_` sur un WS séparé du `/ws` général
