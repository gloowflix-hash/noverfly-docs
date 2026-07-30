# Introduction

Noverfly expose aujourd'hui une seule API publique principale pour les intégrations développeur :

- HTTP : `https://api.noverfly.com`
- WebSocket principal : `wss://api.noverfly.com/ws`

L'alias `https://api.gloowflix.cloud` reste compatible, mais la documentation ci-dessous utilise toujours l'URL canonique.

## Les surfaces à connaître

| Surface | Auth | Usage |
|---|---|---|
| Dashboard API | JWT | gestion tenants, sites, publication, clés |
| Data API | `gfk_` | collections, records, auth applicative |
| Cloud API | `gfc_` | upload, assets, cloud media |
| Messenger REST + `/ws` | `gfk_` / `gfc_` + `gfk_` ou JWT sur `/ws` | chat, vocaux, présence, appels historiques |
| Live Streaming | `gfk_` secret | création, start, stop, playback live |
| Visual Search | `vst_` côté client, `gfk_` côté serveur | recherche image, texte, média |
| Visual Events | `ves_` | événements visuels en temps réel |
| Calls API | activation `gfk_`, exécution `nfk_*` | rooms 1:1, groupe, live rooms |

## Règles d'auth simples

- `gfk_` : clé serveur liée au site, pour les surfaces data et plusieurs modules site-centric
- `gfc_` : clé cloud pour uploads, assets, push cloud et certaines routes messenger REST
- `vst_` : token court Visual Search pour les apps clientes
- `ves_` : token de session Visual Events pour le WebSocket dédié
- `nfk_*` : clé Calls API pour les rooms modernes

## Ce qui est réellement temps réel

Le code actuel documente plusieurs couches realtime distinctes :

1. `/ws` pour les notifications, la présence, Messenger et les events live.
2. `/v1/realtime?token=...` pour la Calls API moderne.
3. `/v1/api/visual-events/ws?...&token=ves_...` pour les flux visuels.

Ces trois sockets n'ont pas le même protocole ni les mêmes tokens.

## Documentation recommandée

- [Getting Started](getting-started.md)
- [Authentication](authentication.md)
- [API Reference](api.md)
- [Mini-services Cloud Scripts](mini-services-cloud-scripts.md) — automatiser sans serveur Node dédié
- [Fichiers Google → upload signé](cloud-files-google-upload.md) — Drive/Files vers Noverfly + scripts
- [Appels audio / vidéo](calls-audio-video.md)
- [Live Streaming](live-streaming.md)
- [Notifications Guide](notifications-guide.md)
- [Visual Search](visual-search.md)

## Quand utiliser quoi ?

- besoin de données structurées : Data API `gfk_`
- besoin d'uploads et assets : Cloud API `gfc_`
- besoin d’un mini-backend sans VPS : Cloud Scripts + triggers / cron
- besoin d’importer un fichier Google / galerie : upload signé `gfc_` puis trigger script
- besoin de chat et d'appels 1:1 historiques : Messenger
- besoin de groupe / live rooms : Calls API
- besoin de diffusion live liée à un site : Live Streaming
- besoin de recherche visuelle et d'événements visuels : Visual Search + Visual Events
