# Push FCM Cloud

Le backend ne lit pas un fichier Firebase local sur le serveur.

Le JSON `service account` doit etre envoye au backend via la DevAPI cloud, puis il est chiffre et stocke par tenant dans `tenant_push_configs`.

La gestion des credentials requiert une clé secrète `gfk_` de permission
**ADMIN**. Les clés `gfc_`, READ et READ_WRITE sont refusées.

## Routes utiles

- `PUT /v1/cloud/push/config/fcm`
- `POST /v1/cloud/push/config/fcm/validate`
- `GET /v1/cloud/push/config`
- `GET /v1/cloud/push/status`
- `POST /api/dev/test-incoming-call-push`

## Upload du JSON Firebase

```bash
curl -X PUT "https://api.noverfly.com/v1/cloud/push/config/fcm" \
  -H "X-Api-Key: gfk_VOTRE_CLE_ADMIN" \
  -H "Content-Type: application/json" \
  --data @firebase-service-account.json
```

Le body peut etre:

- le JSON Firebase complet a la racine
- `{ "serviceAccount": { ... } }`
- `{ "serviceAccountB64": "..." }`

## Validation immediate

```bash
curl -X POST "https://api.noverfly.com/v1/cloud/push/config/fcm/validate" \
  -H "X-Api-Key: gfk_VOTRE_CLE_ADMIN" \
  -H "Content-Type: application/json" \
  -d "{}"
```

Cette route verifie:

- que le tenant a bien une config FCM stockee
- que la cle privee peut signer le JWT OAuth
- que Google accepte l'echange OAuth pour `firebase.messaging`

## Messages chat (Android natif — data-only)

Pour `MESSENGER_MESSAGE` / `MESSENGER_VOICE_MESSAGE`, le cloud envoie un FCM **sans** bloc `notification` ni `android.notification`.

Le client Android construit la notif locale (`MessagingStyle`, avatar, bouton Répondre) via `onMessageReceived`.

Champs `data` (toutes valeurs string) :

- `type`: `chat_message`
- `eventType`: `message.created`
- `conversationId`, `messageId`, `fromId`, `senderName`, `senderAvatarUrl`, `text`, `sentAt`, `deepLink`, `clickAction`

Logs serveur :

- `[cloud:fcm:chat:data-only:payload]`
- `[cloud:fcm:chat:data-only:sent]`

Variables optionnelles :

- `CHAT_PUSH_DEEPLINK_SCHEME` (defaut `kinstore`)
- `CHAT_PUSH_APP_ID` (defaut `kinstore`)
- `API_BASE_URL` HTTPS pour resoudre `senderAvatarUrl` via `/v1/avatars/{userId}`

## Appels entrants (Android — data-only, TTL 30s)

Pipeline separe du chat. Jamais de bloc `notification` ni `android.notification`.

| `data.type` | `eventType` | Usage |
|-------------|-------------|--------|
| `call_invite` | `call.incoming` | Sonnerie + notification urgente cote APK |
| `call_cancel` | `call.expired` / `call.cancelled` / `call.declined` | Arreter sonnerie, retirer notification |

Champs `call_invite` : `callId`, `conversationId`, `fromId`, `callerName`, `callerAvatarUrl`, `callMode`, `callProvider`, `sentAt`, `expiresAt`, `deepLink`, `clickAction` (`OPEN_INCOMING_CALL`), `channelId` (`incoming_calls`).

Logs serveur :

- `[cloud:fcm:call:data-only:payload]`
- `[cloud:fcm:call:data-only:sent]`
- `[cloud:fcm:call:data-only:error]`

Variables optionnelles : `CALL_PUSH_APP_ID`, `CALL_PUSH_DEEPLINK_SCHEME` (sinon repli sur `CHAT_PUSH_*`).

Templates Kotlin : `docs/android-templates/INCOMING_CALL_NATIVE.md`.

## Script PowerShell

Utiliser:

```powershell
.\scripts\flivex\deploy-fcm-config.ps1 `
  -ApiKey "gfk_xxx" `
  -ServiceAccountPath "C:\secrets\firebase-service-account.json"
```

Optionnellement, pour verifier le signal d'appel entrant en dry-run:

```powershell
.\scripts\flivex\deploy-fcm-config.ps1 `
  -ApiKey "gfk_xxx" `
  -ServiceAccountPath "C:\secrets\firebase-service-account.json" `
  -CalleeSiteUserId "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
```
