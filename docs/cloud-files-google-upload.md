# Envoyer des fichiers (Google / local) vers Noverfly — URLs signées

Ce guide montre comment faire arriver un fichier (depuis **Google Drive / Google Files**, une galerie mobile, ou un PC) dans le stockage Noverfly, obtenir une **URL signée / publique**, puis **automatiser** la suite avec un Cloud Script.

Base URL : `https://api.noverfly.com`

| Action | Clé | Header |
|--------|-----|--------|
| Upload / import média | Cloud `gfc_` | `X-Api-Key: gfc_YOUR_CLOUD_KEY` |
| Scripts + collections | Secret `gfk_` | `X-Api-Key: gfk_YOUR_SECRET_KEY` |

> Ne mettez jamais `gfk_` ni `gfc_` dans une APK ou un front public.  
> Le client reçoit des **URLs** ou des tokens courts ; l’upload peut aussi passer par votre backend.

---

## Vue d’ensemble

```
Google Drive / Fichiers Google / Galerie
        │
        ▼
  (1) Obtenir les bytes du fichier côté client ou backend
        │
        ▼
  (2) POST /v1/api/cloud/upload     → upload_url signée S3  (gfc_)
        │
        ▼
  (3) PUT bytes vers upload_url
        │
        ▼
  (4) POST /v1/api/cloud/upload/commit → asset + public_url
        │
        ▼
  (5) Enregistrer l’asset dans une collection (gfk_)
        │
        ▼
  (6) Trigger onCreate → script (notif, IA, index Visual Search, …)
```

---

## Cas 1 — Upload standard (recommandé, y compris depuis Google)

### Étape A : récupérer le fichier depuis Google

Côté app (Android / Web) :

1. Utilisez le **Google Picker** / Files app / `ACTION_OPEN_DOCUMENT`.
2. Obtenez un `Uri` / `Blob` / `ArrayBuffer`.
3. **Ne passez pas** par une URL Drive privée non téléchargeable : téléchargez les bytes avec le token OAuth Google côté client, puis uploadez vers Noverfly.

Si le fichier Drive est **partagé publiquement** (lien « toute personne avec le lien »), vous pouvez parfois utiliser l’import par URL (cas 2). Sinon : bytes → upload signé.

### Étape B : URL signée Noverfly

```bash
curl -X POST "https://api.noverfly.com/v1/api/cloud/upload" \
  -H "X-Api-Key: gfc_YOUR_CLOUD_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "filename": "photo-google.jpg",
    "mime_type": "image/jpeg",
    "size_bytes": 245760
  }'
```

Réponse typique :

```json
{
  "upload_url": "https://s3.../presigned?X-Amz-Signature=...",
  "key": "tenants/.../assets/....jpg",
  "public_url": "https://cdn.../...."
}
```

`upload_url` est **signée** et à durée limitée : c’est elle qui autorise l’écriture S3 sans exposer vos credentials AWS.

### Étape C : envoyer les bytes

```bash
curl -X PUT "$UPLOAD_URL" \
  -H "Content-Type: image/jpeg" \
  --data-binary @photo-google.jpg
```

### Étape D : commit (enregistrer l’asset)

```bash
curl -X POST "https://api.noverfly.com/v1/api/cloud/upload/commit" \
  -H "X-Api-Key: gfc_YOUR_CLOUD_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "key": "tenants/.../assets/....jpg",
    "filename": "photo-google.jpg",
    "mime_type": "image/jpeg",
    "size_bytes": 245760
  }'
```

Vous obtenez un `asset` avec id + URL utilisable dans vos collections / feed.

### Variante mobile (évite CORS S3)

```bash
curl -X POST "https://api.noverfly.com/v1/api/cloud/upload/direct" \
  -H "X-Api-Key: gfc_YOUR_CLOUD_KEY" \
  -F "file=@photo-google.jpg"
```

Max ~20 MB pour l’upload direct.

---

## Cas 2 — Import depuis une URL publique (Drive partagé, CDN, …)

Si vous avez une **URL HTTP(S) téléchargeable** (image) :

```bash
curl -X POST "https://api.noverfly.com/v1/api/cloud/import/image" \
  -H "X-Api-Key: gfc_YOUR_CLOUD_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "url": "https://example.com/path/to/image.jpg",
    "alt_text": "Import Google / externe"
  }'
```

Limites :

- l’URL doit être accessible par les serveurs Noverfly (pas de lien Drive privé / cookie Google)
- usage principal : images ; pour vidéo/audio, préférez l’upload signé (cas 1)
- rate limit import : ~10/min

---

## Cas 3 — Backend qui signe pour l’app (meilleure sécurité)

```
APK  →  votre backend (détient gfc_)  →  Noverfly upload  →  APK PUT vers upload_url
```

1. L’APK demande à **votre** API : « je veux uploader `photo.jpg` (245760 bytes) ».
2. Votre backend appelle `POST /v1/api/cloud/upload` avec `gfc_`.
3. Il renvoie `upload_url` + `key` à l’APK.
4. L’APK fait le `PUT` des bytes (éventuellement déjà lus depuis Google Files).
5. Votre backend appelle `commit`, puis crée le record collection.

Ainsi **aucune clé Noverfly** n’est dans l’APK.

---

## Enregistrer le fichier dans une collection (Data API)

```bash
curl -X POST "https://api.noverfly.com/v1/api/data/collections/media_assets/records" \
  -H "X-Api-Key: gfk_YOUR_SECRET_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "data": {
      "assetId": "ASSET_UUID",
      "url": "https://cdn.../photo-google.jpg",
      "source": "google_files",
      "mimeType": "image/jpeg",
      "status": "ready"
    }
  }'
```

---

## Automatiser à l’arrivée d’un nouveau fichier

### 1. Script avec trigger

```json
{
  "name": "On media ready",
  "slug": "on-media-ready",
  "status": "active",
  "triggers": [
    { "type": "onCreate", "collection": "media_assets", "enabled": true }
  ],
  "code": "export default async function main(ctx) { /* ... */ }"
}
```

### 2. Exemple d’algo post-upload

```js
export default async function main(ctx) {
  const { action, record, recordId } = ctx.input || {};
  if (action !== 'created') return { skipped: true };

  const url = record?.data?.url;
  const source = record?.data?.source;

  // Notifier
  await ctx.services.notifications.send({
    to: 'admin',
    title: 'Nouveau média',
    message: `Fichier ${source || 'upload'} prêt`,
    push: true,
    link: `/media/${recordId}`,
  });

  // Générer un titre / tags via IA (si AI Cloud activé)
  if (url && ctx.services?.ai?.generateText) {
    const ai = await ctx.services.ai.generateText({
      prompt: `Propose un titre court et 5 tags pour un média importé depuis ${source}. URL: ${url}`,
    });
    await ctx.collections.media_assets.update(recordId, {
      data: {
        ...(record.data || {}),
        aiTitle: ai?.text || String(ai),
        processedAt: ctx.now,
      },
    });
  }

  return { ok: true, recordId };
}
```

### 3. Déployer le script

```bash
CODE=$(node -e "console.log(JSON.stringify(require('fs').readFileSync('on-media-ready.script.js','utf8')))")

curl -X POST "https://api.noverfly.com/v1/devapi/scripts?upsert=true" \
  -H "X-Api-Key: gfk_YOUR_SECRET_KEY" \
  -H "Content-Type: application/json" \
  -d "{
    \"name\": \"On media ready\",
    \"slug\": \"on-media-ready\",
    \"status\": \"active\",
    \"triggers\": [
      { \"type\": \"onCreate\", \"collection\": \"media_assets\", \"enabled\": true }
    ],
    \"code\": $CODE
  }"
```

À chaque nouveau record `media_assets`, le script tourne **quelques secondes plus tard** via BullMQ.

---

## Stocker des clés pour vos scripts (Google, webhooks, etc.)

### Option A — AI Cloud vault (providers IA)

```bash
curl -X POST "https://api.noverfly.com/v1/cloud/ai/keys/save" \
  -H "X-Api-Key: gfk_YOUR_SECRET_KEY" \
  -H "Content-Type: application/json" \
  -d '{ "appId": "my-app", "provider": "openai", "apiKey": "sk-..." }'
```

Voir [ai-cloud-service.md](ai-cloud-service.md).

### Option B — Collection privée `app_config` (intégrations Google, webhooks)

```bash
# Créer / mettre à jour une config (serveur uniquement)
curl -X POST "https://api.noverfly.com/v1/api/data/collections/app_config/records" \
  -H "X-Api-Key: gfk_YOUR_SECRET_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "data": {
      "key": "google_integration",
      "clientId": "....apps.googleusercontent.com",
      "webhookUrl": "https://hooks.example.com/media"
    }
  }'
```

Dans le script :

```js
const rows = await ctx.collections.app_config.find({ key: 'google_integration', limit: 1 });
const cfg = rows[0]?.data || {};
// utiliser cfg.webhookUrl, jamais exposer cette collection en route publique
```

### Option C — CI qui injecte les scripts + secrets

- Secrets GitHub Actions : `NOVERFLY_GFK`, `NOVERFLY_GFC`
- Job qui upsert tous les `*.script.js`
- Job qui met à jour `app_config` sans committer les valeurs

---

## Automatiser l’arrivée de nouveaux scripts / algorithmes

Quand vous ajoutez un nouvel algo (ex. `moderate-media.script.js`) :

1. Commit du fichier dans votre repo app.
2. CI upsert le slug.
3. Le script `on-media-ready` l’appelle :

```js
await ctx.scripts.run('moderate-media', {
  recordId,
  url: record.data?.url,
});
```

4. Pas de redéploiement Docker : le code vit en base (`dev_api_functions`).

Exemple CI minimal : voir [mini-services-cloud-scripts.md](mini-services-cloud-scripts.md).

---

## Checklist intégrateur

- [ ] Clé `gfc_` pour upload / import
- [ ] Clé `gfk_` pour collections + scripts
- [ ] Fichiers Google lus en **bytes** côté client (ou URL publique pour import image)
- [ ] Upload via URL **signée** puis `commit`
- [ ] Record créé dans une collection
- [ ] Trigger `onCreate` + script métier
- [ ] Secrets hors APK (vault AI ou `app_config` privée)
- [ ] Pipeline CI pour nouveaux `*.script.js`

---

## Erreurs fréquentes

| Symptôme | Cause | Action |
|----------|--------|--------|
| 403 sur `/v1/api/cloud/upload` | `gfk_` au lieu de `gfc_` | Utiliser une Cloud key |
| 403 sur Data API | `gfc_` au lieu de `gfk_` | Utiliser une Secret key |
| Import Drive échoue | Lien privé Google | Télécharger les bytes avec OAuth puis upload signé |
| Script ne part pas | Pas de trigger / script `inactive` | Vérifier `status: active` + `triggers` |
| Boucle infinie attendue | Écritures depuis script | Normal : anti-boucle désactive les re-triggers |

---

## Docs liées

- [mini-services-cloud-scripts.md](mini-services-cloud-scripts.md) — pourquoi les mini-services
- [cloud-scripts-operational-guide.md](cloud-scripts-operational-guide.md) — BullMQ & triggers
- [ai-cloud-service.md](ai-cloud-service.md) — vault clés IA
- [visual-search.md](visual-search.md) — indexer les images après upload
