# Asset Folder Service — Noverfly

**API :** `https://api.noverfly.com`
**Auth :** clé `gfk_` (Secret) liée à un projet, header `X-Api-Key`
**Flags serveur requis :** `ASSET_FOLDER_V1_ENABLED=true`, `ASSET_FOLDER_UPLOAD_V1_ENABLED=true` (pour les routes upload)

Les Asset Folders organisent le stockage cloud (S3) par dossier / sous-dossier, avec politique d'accès, URLs signées, quotas, rétention, et option embeddings / recherche visuelle.

> Vocabulaire canonique : `projectId` / `appUserId`. `siteId` / `siteUserId` sont des alias legacy.

---

## Ce que c'est

- Dossiers métier multi-tenant (`tenantId` + `projectId` dérivés de la clé API)
- Sous-dossiers via `parentFolderId` (arborescence libre)
- Stockage S3 :

```text
tenants/{tenantId}/projects/{projectId}/folders/{folderId}/assets/{assetId}/original.{ext}
```

- **Tous objets S3** : image, vidéo, audio, PDF, documents… (types sûrs)
- **Embeddings / recherche vectorielle** : optionnels, **images seulement**, activables **par dossier**
- **Lecture fichier** : URL S3 signée après ACL (pas de proxy public pour les assets liés à un dossier, sauf `visibility=PUBLIC` + `accessPolicy=PUBLIC_READ`)

---

## Types de dossiers

Le système distingue deux catégories de dossiers :

### Dossiers GENERIC (créés par le client)

Créés via `POST /v1/asset-folders`. Le client organise **librement** son arborescence :

- dossiers de médias, archives, documents, etc.
- sous-dossiers via `parentFolderId`
- politique d'accès et flags embeddings configurables

### Dossiers système (créés via `/ensure`)

Créés via `POST /v1/asset-folders/ensure` — idempotents (clé `ensureKey` dérivée du type + bindings). Le backend les crée automatiquement pour les cas d'usage métier :

| `folderType` | Usage | Visibility défaut | AccessPolicy défaut | Embeddings défaut |
|---|---|---|---|---|
| `PROFILE` | Avatar / couverture / galerie d'un profil utilisateur | `PUBLIC` | `OWNER_ONLY` | `false` |
| `CHAT` | Images / vidéos / audio / documents d'une conversation | `PRIVATE` | `RECORD_MEMBERS` | `false` |
| `POST` | Médias attachés à un post (collection record) | `SITE` | `OWNER_ONLY` | `true` |
| `STORY` | Médias d'une story éphémère | `SITE` | `OWNER_ONLY` | `false` |
| `PRODUCT` | Galerie / couverture produit (e-commerce) | `SITE` | `OWNER_ONLY` | `true` |
| `DOCUMENT` | Documents privés (contrats, factures) | `PRIVATE` | `API_KEY_SCOPED` | `false` |
| `GENERATED` | Fichiers générés par IA / scripts | `PRIVATE` | `API_KEY_SCOPED` | `false` |
| `SYSTEM` | Dossiers internes (logs, exports) | `PRIVATE` | `API_KEY_SCOPED` | `false` |

> Les dossiers système ne peuvent pas être créés via `POST /v1/asset-folders` (seul `GENERIC` y est autorisé). Utilisez `/ensure` pour les autres types.

---

## Visibility et AccessPolicy (licences d'accès)

### `visibility` — qui peut voir l'existence du dossier

| Valeur | Portée |
|---|---|
| `PRIVATE` | Dossier privé (défaut GENERIC, CHAT, DOCUMENT) |
| `SITE` | Visible au niveau du projet (POST, STORY, PRODUCT) |
| `TENANT` | Visible au niveau du tenant |
| `PUBLIC` | Dossier public — l'asset peut être servi via proxy public si `accessPolicy=PUBLIC_READ` |

### `accessPolicy` — qui peut lire les fichiers du dossier

| Valeur | Qui peut lire | Header requis |
|---|---|---|
| `OWNER_ONLY` | Propriétaire (`ownerUserId`) ou ADMIN | `X-End-User-Id` (si end-user) |
| `RECORD_MEMBERS` | Membres du record lié (chat conversation) | `X-End-User-Id` |
| `SITE_USERS` | Utilisateurs authentifiés du projet | `X-End-User-Id` |
| `TENANT_USERS` | Membres du tenant | — |
| `API_KEY_SCOPED` | Clé `gfk_` READ+ du projet | — |
| `PUBLIC_READ` | Tout le monde (clé READ+ ou proxy public si `visibility=PUBLIC`) | — |

### Combinaisons courantes

| Cas usage | `visibility` | `accessPolicy` | Public ? |
|---|---|---|---|
| Médias privés client | `PRIVATE` | `OWNER_ONLY` | Non |
| Galerie produit e-commerce | `SITE` | `OWNER_ONLY` | Non (owner via URL signée) |
| Images publiques (CDN) | `PUBLIC` | `PUBLIC_READ` | **Oui** (proxy public OK) |
| Documents internes | `PRIVATE` | `API_KEY_SCOPED` | Non (clé serveur uniquement) |
| Avatars profils | `PUBLIC` | `OWNER_ONLY` | Non (owner via URL signée) |
| Pièces jointes chat | `PRIVATE` | `RECORD_MEMBERS` | Non (membres conversation) |

---

## Qui fait quoi

| Action | Permission clé `gfk_` |
|---|---|
| Créer / configurer / supprimer dossier | **ADMIN** |
| Activer / désactiver embeddings | **ADMIN** |
| Lister dossiers | READ |
| Uploader un fichier dans un dossier | READ_WRITE |
| Obtenir `read-url` / `download-url` | READ (+ `X-End-User-Id` selon politique) |
| Lister les assets d'un dossier | READ |
| Supprimer un asset | READ_WRITE |

> `READ_WRITE` ne peut **pas** créer, configurer ou supprimer des dossiers. Seul `ADMIN` le peut.

---

## Routes API

### Gestion des dossiers

| Méthode | Route | Permission | Usage |
|---|---|---|---|
| `POST` | `/v1/asset-folders` | ADMIN | Créer un dossier GENERIC |
| `POST` | `/v1/asset-folders/ensure` | ADMIN | Créer/récupérer un dossier système (idempotent) |
| `GET` | `/v1/asset-folders` | READ | Lister les dossiers (filtres, pagination) |
| `GET` | `/v1/asset-folders/:folderId` | READ | Détail d'un dossier |
| `PATCH` | `/v1/asset-folders/:folderId` | ADMIN | Modifier (nom, slug, visibility, accessPolicy, embeddings, quotas) |
| `DELETE` | `/v1/asset-folders/:folderId` | ADMIN | Soft-delete (statut `DELETED`) |

### Upload d'assets

| Méthode | Route | Permission | Usage |
|---|---|---|---|
| `POST` | `/v1/asset-folders/:folderId/assets/presign` | READ_WRITE | URL signée S3 pour upload |
| `POST` | `/v1/asset-folders/:folderId/assets/upload` | READ_WRITE | Upload direct multipart (petits fichiers) |
| `POST` | `/v1/asset-folders/:folderId/assets/commit` | READ_WRITE | Finaliser un upload presign |
| `POST` | `/v1/asset-folders/:folderId/assets/abort` | READ_WRITE | Annuler un upload presign |
| `GET` | `/v1/asset-folders/:folderId/assets` | READ | Lister les assets du dossier |
| `GET` | `/v1/assets/:assetId` | READ | Métadonnées d'un asset |
| `DELETE` | `/v1/assets/:assetId` | READ_WRITE | Suppression logique d'un asset |

### Accès en lecture (URLs signées)

| Méthode | Route | Permission | Usage |
|---|---|---|---|
| `POST` | `/v1/assets/:assetId/read-url` | READ | URL signée lecture (streaming) |
| `POST` | `/v1/assets/:assetId/download-url` | READ | URL signée téléchargement |

> `expiresSeconds` : 60 à 3600 secondes (défaut 300).

---

## Créer un dossier GENERIC

```http
POST /v1/asset-folders
X-Api-Key: gfk_ADMIN_...
Content-Type: application/json

{
  "name": "Médias boutique",
  "slug": "medias-boutique",
  "visibility": "SITE",
  "accessPolicy": "OWNER_ONLY",
  "allowEmbeddings": false,
  "maxBytes": 5368709120,
  "maxAssetCount": 500,
  "retentionDays": null,
  "deleteOnRecordDelete": false
}
```

Réponse `201` :

```json
{
  "folder": {
    "id": "folder-uuid",
    "parentFolderId": null,
    "name": "Médias boutique",
    "slug": "medias-boutique",
    "folderType": "GENERIC",
    "visibility": "SITE",
    "accessPolicy": "OWNER_ONLY",
    "collectionSlug": null,
    "collectionId": null,
    "recordId": null,
    "ownerUserId": null,
    "maxBytes": 5368709120,
    "maxAssetCount": 500,
    "usedBytes": 0,
    "assetCount": 0,
    "allowEmbeddings": false,
    "allowVectorSearch": false,
    "allowPublicSearch": false,
    "retentionDays": null,
    "deleteOnRecordDelete": false,
    "status": "ACTIVE",
    "createdAt": "2026-09-02T10:00:00.000Z",
    "updatedAt": "2026-09-02T10:00:00.000Z"
  }
}
```

### Sous-dossier

```json
{
  "name": "Archives 2025",
  "parentFolderId": "folder-uuid-parent"
}
```

### Lier un dossier à une collection / record

```json
{
  "name": "Photos produit SKU-123",
  "collectionSlug": "products",
  "recordId": "record-uuid",
  "visibility": "SITE",
  "accessPolicy": "OWNER_ONLY",
  "deleteOnRecordDelete": true
}
```

> Si `recordId` est fourni sans `collectionId` / `collectionSlug` (sauf pour `PROFILE`, `CHAT`, `SYSTEM`), l'API renvoie `RECORD_REQUIRES_COLLECTION` (400).

---

## Créer un dossier système (ensure)

Idempotent : si le dossier existe déjà pour la même clé `ensureKey`, il est retourné sans erreur.

```http
POST /v1/asset-folders/ensure
X-Api-Key: gfk_ADMIN_...
Content-Type: application/json

{
  "folderType": "PROFILE",
  "ownerUserId": "user-uuid",
  "visibility": "PUBLIC",
  "accessPolicy": "OWNER_ONLY"
}
```

Exemples par type :

| Type | Bindings requis | Exemple body |
|---|---|---|
| `PROFILE` | `ownerUserId` | `{ "folderType": "PROFILE", "ownerUserId": "uuid" }` |
| `CHAT` | `recordId` (= conversationId) | `{ "folderType": "CHAT", "recordId": "conv-uuid" }` |
| `POST` | `collectionSlug` + `recordId` | `{ "folderType": "POST", "collectionSlug": "posts", "recordId": "post-uuid" }` |
| `PRODUCT` | `collectionSlug` + `recordId` | `{ "folderType": "PRODUCT", "collectionSlug": "products", "recordId": "product-uuid" }` |
| `STORY` | `collectionSlug` + `recordId` | `{ "folderType": "STORY", "collectionSlug": "stories", "recordId": "story-uuid" }` |
| `DOCUMENT` | — | `{ "folderType": "DOCUMENT", "name": "Contrats" }` |
| `GENERATED` | — | `{ "folderType": "GENERATED", "name": "Exports IA" }` |

---

## Uploader un fichier

### Méthode 1 — Upload signé (recommandé, gros fichiers)

```http
POST /v1/asset-folders/{folderId}/assets/presign
X-Api-Key: gfk_READ_WRITE_...
Content-Type: application/json

{
  "filename": "demo.mp4",
  "mimeType": "video/mp4",
  "sizeBytes": 52428800,
  "usage": "POST_MEDIA",
  "idempotencyKey": "upload-demo-001"
}
```

Valeurs `usage` acceptées :

| Usage | Contexte |
|---|---|
| `PROFILE_AVATAR` | Avatar profil |
| `PROFILE_COVER` | Couverture profil |
| `PROFILE_GALLERY` | Galerie profil |
| `POST_MEDIA` | Média de post |
| `POST_COVER` | Couverture de post |
| `PRODUCT_GALLERY` | Galerie produit |
| `PRODUCT_COVER` | Couverture produit |
| `STORY_MEDIA` | Média story |
| `CHAT_IMAGE` | Image chat |
| `CHAT_VIDEO` | Vidéo chat |
| `CHAT_AUDIO` | Audio chat (message vocal) |
| `CHAT_DOCUMENT` | Document chat |
| `CHAT_THUMBNAIL` | Vignette chat |
| `GENERATED_IMAGE` | Image générée par IA |
| `DOCUMENT` | Document privé |
| `OTHER` | Autre |

Réponse :

```json
{
  "assetId": "asset-uuid",
  "reservationId": "reservation-uuid",
  "uploadUrl": "https://s3.../presigned?X-Amz-Signature=...",
  "uploadKey": "tenants/.../folders/.../assets/.../original.mp4",
  "expiresAt": "2026-09-02T10:15:00.000Z"
}
```

Puis `PUT` des bytes vers `uploadUrl`, puis :

```http
POST /v1/asset-folders/{folderId}/assets/commit
X-Api-Key: gfk_READ_WRITE_...
Content-Type: application/json

{
  "assetId": "asset-uuid",
  "reservationId": "reservation-uuid",
  "uploadKey": "tenants/.../original.mp4"
}
```

### Méthode 2 — Upload direct multipart (petits fichiers)

```http
POST /v1/asset-folders/{folderId}/assets/upload
X-Api-Key: gfk_READ_WRITE_...
Content-Type: multipart/form-data; boundary=...

--boundary
Content-Disposition: form-data; name="file"; filename="photo.jpg"
Content-Type: image/jpeg

<bytes>
--boundary
Content-Disposition: form-data; name="usage"

POST_MEDIA
--boundary--
```

Champs form attendus : `file` (requis), `usage` (requis), `filename`, `mimeType`, `sizeBytes`, `idempotencyKey`, `recordId`.

### Annuler un upload presign

```http
POST /v1/asset-folders/{folderId}/assets/abort
{
  "assetId": "asset-uuid",
  "uploadKey": "tenants/.../original.mp4"
}
```

---

## Lire un fichier (URL signée)

### URL de lecture (streaming / aperçu)

```http
POST /v1/assets/{assetId}/read-url
X-Api-Key: gfk_READ_...
X-End-User-Id: <uuid>   # requis si OWNER_ONLY / SITE_USERS / RECORD_MEMBERS
Content-Type: application/json

{
  "expiresSeconds": 300
}
```

### URL de téléchargement

```http
POST /v1/assets/{assetId}/download-url
X-Api-Key: gfk_READ_...
X-End-User-Id: <uuid>
```

Réponse :

```json
{
  "success": true,
  "url": "https://s3.../signed-download?...",
  "expiresAt": "2026-09-02T10:05:00.000Z"
}
```

| `accessPolicy` | Qui peut lire | `X-End-User-Id` requis ? |
|---|---|---|
| `PUBLIC_READ` | Clé READ+ (proxy public OK si `visibility=PUBLIC`) | Non |
| `API_KEY_SCOPED` | Clé projet READ+ | Non |
| `SITE_USERS` | Utilisateur du projet | **Oui** |
| `OWNER_ONLY` | Propriétaire (`ownerUserId`) ou ADMIN | **Oui** |
| `RECORD_MEMBERS` | Membre du record (chat) | **Oui** |
| `TENANT_USERS` | Membre du tenant | Non |

> Le proxy historique `GET /v1/api/cloud/file/{id}` refuse les assets `BOUND` non publics (`BOUND_ASSET_READ_URL_REQUIRED`).

---

## Activer / désactiver les embeddings

L'admin choisit **par dossier**. Les flags sont **appliqués** au moteur Visual Search (pas cosmétique) :

| Flag | Effet |
|---|---|
| `allowEmbeddings=true` | Les nouvelles images du dossier peuvent être indexées |
| `allowEmbeddings=false` | Pas d'indexation ; désactivation → exclusion / delete vectorielle |
| `allowVectorSearch=true` | Peut apparaître en recherche visuelle |
| `allowVectorSearch=false` | Exclu des résultats de recherche |
| `allowPublicSearch=true` | Visible dans la recherche publique |
| `allowPublicSearch=false` | Exclu de la recherche publique |

```http
PATCH /v1/asset-folders/{folderId}
X-Api-Key: gfk_ADMIN_...
Content-Type: application/json

{
  "visibility": "SITE",
  "accessPolicy": "SITE_USERS",
  "allowEmbeddings": true,
  "allowVectorSearch": true
}
```

Décrocher :

```json
{
  "allowEmbeddings": false,
  "allowVectorSearch": false
}
```

> Un dossier `PRIVATE` + `OWNER_ONLY` ne peut **pas** avoir d'embeddings. Embeddings activés ≠ accès public au fichier.

---

## Lister les dossiers

```http
GET /v1/asset-folders?parentFolderId=null&folderType=GENERIC&status=ACTIVE&page=1&perPage=20
X-Api-Key: gfk_READ_...
```

Filtres :

| Param | Type | Description |
|---|---|---|
| `parentFolderId` | UUID ou `null` | Dossiers racine (`null`) ou sous-dossiers d'un parent |
| `folderType` | enum | Filtrer par type (`GENERIC`, `PROFILE`, `CHAT`, …) |
| `status` | enum | `ACTIVE`, `ARCHIVED`, `DELETED` |
| `recordId` | UUID | Dossiers liés à un record |
| `ownerUserId` | UUID | Dossiers d'un propriétaire |
| `q` | string | Recherche texte sur nom/slug |
| `page` | int | Page (défaut 1) |
| `perPage` | int | Par page (défaut 20, max 100) |
| `includeDeleting` | `true` | Inclure `DELETING` (ADMIN uniquement) |

---

## Lister les assets d'un dossier

```http
GET /v1/asset-folders/{folderId}/assets?page=1&perPage=20
X-Api-Key: gfk_READ_...
```

---

## Quotas et rétention

Par dossier :

| Champ | Type | Description |
|---|---|---|
| `maxBytes` | int \| null | Quota stockage du dossier (bytes) |
| `maxAssetCount` | int \| null | Nombre max d'assets |
| `usedBytes` | int | Bytes utilisés (calculé) |
| `assetCount` | int | Nombre d'assets (calculé) |
| `retentionDays` | int \| null | Rétention automatique (delete après N jours) |
| `deleteOnRecordDelete` | boolean | Supprimer le dossier quand le record lié est supprimé |

---

## Vidéos et hosting média

Les vidéos uploadées dans un dossier sont automatiquement traitées par le pipeline Flivex (si activé sur le tenant) :

- `poster_url` / `preview_url` — vignettes générées
- `playback_url` / `hls_url` — URLs de lecture HLS / MP4
- `renditions` — versions transcodées (low/high)
- `image_variants` — variantes d'images

Routes vidéo dédiées (DevAPI — clé `gfk_` ou Bearer, scope `videos:read`) :

- `GET /v1/videos` — lister les vidéos du projet
- `GET /v1/videos/:videoId` — détail d'une vidéo
- `GET /v1/videos/:videoId/status` — statut de traitement Flivex
- `GET /v1/videos/:videoId/low` — variante basse résolution
- `GET /v1/videos/:videoId/high` — variante haute résolution
- `GET /v1/videos/:videoId/cover` — poster / vignette
- `GET /v1/videos/:videoId/play` — URL de lecture
- `GET /v1/videos/:videoId/manifest` — manifest HLS
- `GET /v1/videos/:videoId/cover/candidates` — vignettes candidates
- `POST /v1/videos/:videoId/cover/from-frame` — extraire une vignette d'une frame
- `PUT /v1/videos/:videoId/cover/select` — sélectionner la vignette
- `POST /v1/videos/:videoId/reprocess` — retraiter via Flivex
- `DELETE /v1/videos/:videoId` — supprimer
- `GET /v1/videos/:videoId/events` — événements de traitement

Routes musicales (Dashboard JWT + permission RBAC) :

- `GET /v1/tenants/:tenantId/music/albums` — lister albums (`music.view`)
- `POST /v1/tenants/:tenantId/music/albums` — créer album (`music.manage`)
- `GET /v1/tenants/:tenantId/music/tracks/:trackId/stream` — stream musical

Voir [realtime-media.md](realtime-media.md) et [applications.md](applications.md) pour l'intégration complète.

---

## Messages vocaux (Messenger)

Les messages vocaux passent par le module Messenger avec un dossier `CHAT` :

1. Enregistrer l'audio côté client (WebM/Opus)
2. Uploader le blob vers un dossier `CHAT` du conversationId
3. `POST /v1/cloud/messenger/conversations/:conversationId/voice` avec `originalKey` + `durationMs`
4. Le backend traite : normalisation loudness + conversion AAC + waveform
5. WebSocket event `messenger:voice_ready` → waveform + URL audio traité livré au destinataire

Voir [messenger-realtime.md](messenger-realtime.md) section « Voice Messages ».

---

## Compatibilité legacy

Les routes historiques restent valides :

- `POST /v1/api/cloud/upload`
- `POST /v1/api/cloud/upload/direct`
- `POST /v1/api/cloud/upload/commit`

Sans `folderId` → asset `LEGACY_UNBOUND` (comportement inchangé).

Voir [cloud-files-google-upload.md](cloud-files-google-upload.md) pour le guide d'upload depuis Google Drive / mobile.

---

## Codes d'erreur

| Code | Statut | Cause |
|---|---|---|
| `FOLDER_ADMIN_REQUIRED` | 403 | Opération nécessite une clé ADMIN |
| `FORBIDDEN` | 403 | Permission insuffisante (READ / READ_WRITE) |
| `ASSET_FOLDER_NOT_FOUND` | 404 | Dossier introuvable |
| `PARENT_FOLDER_NOT_FOUND` | 404 | Dossier parent introuvable |
| `ASSET_FOLDER_SLUG_CONFLICT` | 409 | Slug déjà utilisé |
| `SYSTEM_FOLDER_CREATE_FORBIDDEN` | 403 | Création d'un dossier système via `POST` au lieu de `/ensure` |
| `INVALID_ENSURE_TYPE` | 400 | `ensure` avec `folderType=GENERIC` |
| `INVALID_FOLDER_SLUG` | 400 | Slug invalide (caractères interdits) |
| `INVALID_FOLDER_NAME` | 400 | Nom vide |
| `RECORD_REQUIRES_COLLECTION` | 400 | `recordId` sans `collectionId` / `collectionSlug` |
| `VALIDATION_ERROR` | 400 | Body invalide (Zod) |
| `BOUND_ASSET_READ_URL_REQUIRED` | 403 | Asset `BOUND` non public — utiliser `/read-url` |
| `FOLDER_QUOTA_EXCEEDED` | 403 | Quota `maxBytes` ou `maxAssetCount` atteint |

---

## Voir aussi

- [Visual Search](visual-search.md) — indexation et recherche d'images
- [Fichiers / upload signé](cloud-files-google-upload.md) — guide upload Google Drive / mobile
- [Messenger](messenger-realtime.md) — messages vocaux et pièces jointes chat
- [Applications](applications.md) — flux complet de construction d'app
- [API Reference](api.md) — référence globale des routes
