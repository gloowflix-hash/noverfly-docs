# Noverfly API Documentation

> Documentation développeur de référence pour le contrat actuellement déployé en production.

![API](https://img.shields.io/badge/API-v1-blue)
![Status](https://img.shields.io/badge/status-active-success)
[![Website](https://img.shields.io/badge/Website-noverfly.com-brightgreen)](https://noverfly.com)
[![API Base URL](https://img.shields.io/badge/API-api.noverfly.com-orange)](https://api.noverfly.com)

## Base URLs

- HTTP canonique : `https://api.noverfly.com`
- Alias compatible : `https://api.gloowflix.cloud`
- WebSocket canonique : `wss://api.noverfly.com/ws`

## Familles d'auth

| Auth | Usage |
|---|---|
| JWT dashboard | back-office, tenants, sites, gestion des clés |
| `gfk_` secret | Data API, auth applicative, live, Visual Search serveur |
| `gfc_` cloud | upload, assets, push cloud, messenger REST |
| `vst_` | Visual Search côté client |
| `ves_` | WebSocket Visual Events |
| `nfk_*` | Calls API moderne |

## Guides prioritaires

| Guide | Ce qu'il couvre |
|---|---|
| [Getting Started](docs/getting-started.md) | parcours de démarrage développeur |
| [Authentication](docs/authentication.md) | JWT, `gfk_`, `gfc_`, permissions |
| [API Reference](docs/api.md) | familles de routes principales |
| [Appels audio / vidéo](docs/calls-audio-video.md) | Messenger WebRTC + Calls API `nfk_*` |
| [NoverFly Translate](docs/noverfly-translate.md) | traduction IA multi-projets `nft_*`, 40 USD/mois |
| [Live Streaming](docs/live-streaming.md) | création, start, stop, playback, diagnostics |
| [Notifications Guide](docs/notifications-guide.md) | push, realtime WS, contrats |
| [Visual Search](docs/visual-search.md) | `vst_`, activation, recherche, Visual Events |
| [Hosting API](docs/hosting-api.md) | déploiement React/Vite/Next export sur S3 via `gfk_` ADMIN |

## Quick start

### 1. Se connecter au dashboard

```bash
curl -X POST https://api.noverfly.com/v1/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email":"you@example.com","password":"your-password"}'
```

### 2. Créer ou lister un site

```bash
curl https://api.noverfly.com/v1/tenants/YOUR_TENANT_ID/sites \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN"
```

### 3. Garantir les clés par défaut du site

```bash
curl -X POST https://api.noverfly.com/v1/sites/YOUR_SITE_ID/ensure-api-keys \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN"
```

### 4. Tester la Data API avec `gfk_`

```bash
curl https://api.noverfly.com/v1/api/data/collections \
  -H "X-Api-Key: gfk_YOUR_SECRET_KEY"
```

### 5. Tester la Cloud API avec `gfc_`

```bash
curl -X POST https://api.noverfly.com/v1/api/cloud/upload \
  -H "X-Api-Key: gfc_YOUR_CLOUD_KEY" \
  -H "Content-Type: application/json" \
  -d '{"filename":"photo.jpg","mime_type":"image/jpeg","size_bytes":245000}'
```

## Index documentation

| Document | Description |
|---|---|
| [Introduction](docs/introduction.md) | vue d'ensemble de la plateforme et des surfaces d'auth |
| [Getting Started](docs/getting-started.md) | onboarding développeur rapide |
| [Contract Clarification](docs/contract-clarification.md) | clarification `gfk_` / `gfc_` vs contrats legacy |
| [Authentication](docs/authentication.md) | JWT, clés, permissions, auth app |
| [Google & Firebase Auth](docs/google-firebase-auth.md) | BYOK Google login + FCM, routes et exemples |
| [API Reference](docs/api.md) | familles de routes principales |
| [Database / Data API](docs/database.md) | collections et records |
| [Cloud Scripts](docs/cloud-scripts.md) | scripts publics et cloud |
| [Hosting API](docs/hosting-api.md) | releases S3 immuables, domaines, rollback et quotas |
| [Cloud Scripts (opérationnel)](docs/cloud-scripts-operational-guide.md) | BullMQ, triggers, présence |
| [Mini-services Cloud Scripts](docs/mini-services-cloud-scripts.md) | pourquoi créer des scripts, automation, IA |
| [Fichiers Google → Noverfly](docs/cloud-files-google-upload.md) | upload signé, import, clés dans scripts |
| [DevAPI Automation](docs/devapi-automation.md) | automatisation et workflows |
| [Applications](docs/applications.md) | intégration frontend / backend |
| [Client Apps](docs/client-apps.md) | devices mobiles et tokens |
| [Notifications Guide](docs/notifications-guide.md) | guide unifié push + realtime |
| [Push Notifications](docs/push-notifications.md) | routes tenant JWT |
| [Push FCM Cloud](docs/push-fcm-cloud.md) | FCM cloud, chat et incoming calls |
| [Appels audio / vidéo](docs/calls-audio-video.md) | appels 1:1, groupe, live rooms |
| [Messenger & Realtime](docs/messenger-realtime.md) | guide Messenger historique |
| [Realtime Media](docs/realtime-media.md) | synthèse des surfaces realtime |
| [Live Streaming](docs/live-streaming.md) | live site à site |
| [Visual Search](docs/visual-search.md) | activation, recherche, events |
| [AI Cloud Service](docs/ai-cloud-service.md) | IA cloud |
| [Filter Kit](docs/filter-kit.md) | filtres AR |
| [Music Gateway](docs/music-gateway.md) | streaming musique |
| [Payments](docs/payments.md) | paiements |
| [Migrations & Jobs](docs/migrations-and-jobs.md) | migrations SQL et workers |
| [CMS](docs/cms.md) | concepts CMS |
| [GlowDesign](docs/glowdesign.md) | éditeur visuel |
| [E-Commerce](docs/ecommerce.md) | commerce |
| [Deployment](docs/deployment.md) | publication et livraison |
| [Security](docs/security.md) | principes sécurité |

## Familles de routes à connaître

| Famille | Routes | Auth |
|---|---|---|
| Dashboard | `/v1/auth/*`, `/v1/tenants/*`, `/v1/sites/*` | JWT |
| Data API | `/v1/api/data/*`, `/api/*` | `gfk_` |
| Hosting | `/v1/hosting/*` | `gfk_` (ADMIN pour déployer) |
| Cloud API | `/v1/api/cloud/*` | `gfc_` |
| Messenger REST | `/v1/cloud/messenger/*` | `gfk_` ou `gfc_` |
| Live Streaming | `/v1/cloud/live/streams/*` | `gfk_` secret |
| Push Cloud | `/v1/cloud/push/*` | `gfk_` ou `gfc_` |
| Notifications Cloud | `/v1/cloud/notifications/*` | `gfk_` ou `gfc_` |
| Visual Search | `/v1/api/visual-search/*` | `vst_` ou `gfk_` |
| Visual Events | `/v1/api/visual-events/*` | `vst_` puis `ves_` |
| Calls API | `/v1/calls/*`, `/v1/live/rooms/*`, `/v1/realtime` | `nfk_*` |

## Règles simples pour les intégrateurs

- utilisez `gfk_` pour les données structurées et les surfaces serveur liées au site
- utilisez `gfc_` pour le cloud media, les assets et le push cloud
- n'exposez jamais une `gfk_` ou une `gfc_` dans une app publique
- pour Visual Search côté client, échangez toujours un token court `vst_`
- pour la Calls API moderne, activez via `gfk_` puis exécutez via `nfk_*`

## Liens

- Website : [noverfly.com](https://noverfly.com)
- API host : [api.noverfly.com](https://api.noverfly.com)
- Repo docs : [gloowflix-hash/noverfly-docs](https://github.com/gloowflix-hash/noverfly-docs)
