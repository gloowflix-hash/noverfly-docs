# Notifications & Realtime Contract

Ce document fixe le contrat minimal entre le backend Gloowflix/NoverFly et le frontend client.

## Source d'autorité

- Toute logique métier de notification, quota, quiet hours, digest, présence, delivery/read/listened et push est calculée côté serveur.
- Le frontend consomme seulement :
  - les endpoints HTTP
  - les événements WebSocket
  - les états persistés renvoyés par l'API

## WebSocket Events

Chaque événement métier normalisé suit cette forme :

```json
{
  "event": "notification:new",
  "tenantId": "uuid",
  "recipientUserId": "uuid|null",
  "actorUserId": "uuid|null",
  "entityType": "string|null",
  "entityId": "uuid|string|null",
  "payload": {},
  "createdAt": "2026-05-06T12:00:00.000Z"
}
```

Événements disponibles :

- `notification:new`
- `notification:removed`
- `messenger:message:new`
- `messenger:message:delivered`
- `messenger:message:read`
- `messenger:voice:new`
- `messenger:voice:listened`
- `messenger:typing:start`
- `messenger:typing:stop`
- `messenger:voice_recording:start`
- `messenger:voice_recording:stop`
- `messenger:conversation:active`
- `messenger:conversation:inactive`
- `presence:user:online`
- `presence:user:offline`
- `presence:user:last_seen`
- `presence:call_status`
- `call:incoming`
- `call:ringing`
- `call:accepted`
- `call:declined`
- `call:missed`
- `call:ended`
- `commerce:order:created`
- `commerce:payment:received`
- `commerce:payment:failed`
- `social:like:new`
- `social:comment:new`
- `social:follow:new`
- `publication:approved`
- `publication:rejected`

## Messenger HTTP

Routes tenant :

- `POST /v1/tenants/:tenantId/messenger/messages/:messageId/delivered`
- `POST /v1/tenants/:tenantId/messenger/messages/:messageId/read`
- `POST /v1/tenants/:tenantId/messenger/messages/:messageId/listened`
- `GET /v1/tenants/:tenantId/messenger/conversations/:conversationId/receipts`
- `GET /v1/tenants/:tenantId/messenger/presence?userIds=id1,id2`

Routes cloud DevAPI :

- `POST /v1/cloud/messenger/messages/:messageId/delivered`
- `POST /v1/cloud/messenger/messages/:messageId/read`
- `POST /v1/cloud/messenger/messages/:messageId/listened`
- `GET /v1/cloud/messenger/conversations/:conversationId/receipts`
- `GET /v1/cloud/messenger/presence?userIds=id1,id2`

## Receipts

Le frontend doit appeler :

- `delivered` quand le message atteint réellement le client.
- `read` quand la conversation/message a été vu.
- `listened` pour un vocal quand l'écoute est confirmée.

Réponse attendue :

```json
{
  "receipt": {
    "messageId": "uuid",
    "userId": "uuid",
    "deliveredAt": "date|null",
    "readAt": "date|null",
    "listenedAt": "date|null",
    "playedDurationMs": 0
  }
}
```

## Présence

Le frontend peut :

- envoyer `presence:heartbeat`
- envoyer `messenger:conversation:active`
- envoyer `messenger:conversation:inactive`
- envoyer `messenger:typing:start`
- envoyer `messenger:typing:stop`
- envoyer `messenger:voice_recording:start`
- envoyer `messenger:voice_recording:stop`

Le backend retourne l'état autoritaire :

```json
{
  "userId": "uuid",
  "status": "online|offline|away|busy",
  "lastSeenAt": "date|null",
  "activeConversationId": "uuid|null",
  "typingConversationIds": [],
  "recordingConversationIds": [],
  "callStatus": "ringing|in_call|ended|missed|null"
}
```

## Notifications

- Le frontend ne doit pas recalculer les quotas.
- Le frontend ne doit pas recalculer quiet hours ou digest.
- Le frontend ne doit pas dédupliquer localement autre que sur l'id de notification reçu.

Payload notification recommandé :

```json
{
  "type": "ORDER_CREATED",
  "title": "Nouvelle commande",
  "body": "Commande #123 reçue",
  "metadata": {
    "entityType": "order",
    "entityId": "uuid",
    "deepLink": "/orders/uuid"
  }
}
```

## Push Token Registration

Le frontend doit enregistrer les tokens push via les routes push existantes avec :

- `tenantId`
- `userId` ou `appUserId` (`siteUserId` accepté comme alias legacy)
- `provider`
- `platform`
- `token`
- `deviceId`
- `appVersion`
- `locale`
- `timezone`

Le backend décide ensuite :

- si le push est autorisé
- si le push est retardé
- si le push part en digest
- si le push est bloqué par quota

## Règles d'affichage frontend

- Messages/chat : badge messenger, bannière dédiée, ouverture directe du chat.
- Notifications métier : centre global.
- Appels : UI dédiée + push prioritaire si nécessaire.
- Le frontend ne doit jamais exposer de secrets, tokens provider ou données paiement sensibles.
