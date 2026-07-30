# Realtime media : carte d'orientation

Cette page sert de synthèse. Pour les détails opérationnels, utilisez les guides dédiés :

- `calls-audio-video.md`
- `live-streaming.md`
- `notifications-guide.md`
- `messenger-realtime.md`
- `visual-search.md`

Base URL HTTP : `https://api.noverfly.com`  
WebSocket principal : `wss://api.noverfly.com/ws`

## Les 4 surfaces à ne pas confondre

| Surface | Ce qu'elle fait | Auth | Guide |
|---|---|---|---|
| Messenger historique | chat, présence, vocaux, appels 1:1 | `gfk_` / `gfc_` + `/ws` | `messenger-realtime.md` et `calls-audio-video.md` |
| Calls API | 1:1, groupe, live rooms | activation `gfk_`, exécution `nfk_*` | `calls-audio-video.md` |
| Live Streaming | diffusion live pilotée côté site | `gfk_` secret | `live-streaming.md` |
| Visual Events | événements visuels temps réel séparés | `vst_` puis `ves_` | `visual-search.md` |

## WebSocket principal `/ws`

Le socket principal couvre :

- présence
- événements messenger
- signalisation d'appels Messenger
- rooms tenant / site / live
- notifications temps réel

Auth supportée :

- JWT dashboard
- `gfk_` secret + `userId`

Auth refusée :

- `gfc_`
- site-user JWT

## Audio / vidéo

### Messenger

À utiliser si vous êtes déjà dans le modèle conversation / message :

- `GET /v1/cloud/messenger/rtc-config`
- `POST /v1/cloud/messenger/conversations`
- `GET /v1/cloud/messenger/conversations`
- `POST /v1/cloud/messenger/conversations/:conversationId/messages`
- `POST /v1/cloud/messenger/conversations/:conversationId/voice`
- `GET /v1/cloud/messenger/calls`
- événements `/ws` : `call:initiate`, `call:offer`, `call:answer`, `call:ice`, `call:hangup`, `call:reject`, `call:busy`

### Calls API

À utiliser pour les rooms produit :

- `POST /v1/calls`
- `POST /v1/calls/:callId/join-token`
- `GET /v1/calls/:callId/participants`
- `POST /v1/live/rooms`
- `/v1/realtime?token=...`

## Live

Le module live est distinct du module calls :

- `POST /v1/cloud/live/streams`
- `POST /v1/cloud/live/streams/:id/start`
- `GET /v1/cloud/live/streams/:id/playback`
- `POST /v1/cloud/live/streams/:id/end`

Événements WS :

- `live:status`
- `live:viewer_count`
- `live:chat`
- `live:comment`
- `live:reaction`

## Uploads et assets

Les briques media d'entrée passent par la Cloud API :

- `POST /v1/api/cloud/upload`
- `POST /v1/api/cloud/upload/commit`
- `POST /v1/api/cloud/upload/direct`
- `GET /v1/api/cloud/assets`
- `DELETE /v1/api/cloud/assets/:assetId`

Elles servent ensuite :

- aux messages vocaux
- aux aperçus / playbacks vidéo
- au Visual Search
- aux pipelines Flivex

## Visual Search et Visual Events

Visual Search et Visual Events sont maintenant des surfaces à part entière :

- token court client `vst_`
- session temps réel `ves_`
- recherche image / texte / média
- événements `visual.object.*`, `visual.scene.changed`, `visual.effect.triggered`

Voir `visual-search.md`.

## Notifications liées au realtime media

Le backend envoie déjà :

- pushes d'appels entrants
- notifications métier live (`LIVE_STARTED`, `LIVE_ENDED`, `LIVE_FAILED`)
- notifications messenger (`MESSENGER_MESSAGE`, `MESSENGER_VOICE_MESSAGE`, appels manqués)
- contrats WS `notification:new`, `messenger:*`, `call:*`, `live:*`

Voir `notifications-guide.md`.

## Résumé

- si vous voulez des conversations et appels 1:1 couplés au chat, restez sur `Messenger`
- si vous voulez des rooms audio/vidéo/groupe/live, utilisez la `Calls API`
- si vous voulez de la diffusion live site, utilisez le module `live`
- si vous voulez du vision realtime, passez par `Visual Search` + `Visual Events`
