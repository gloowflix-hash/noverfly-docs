# Asset Folder Service — Noverfly

**Date :** 2026-08-06  
**API :** `https://api.noverfly.com`  
**Statut :** Lots 0–3 + Lot 6 partiel (embeddings réellement branchés côté API)

Les Asset Folders organisent le stockage cloud (S3) par dossier / sous-dossier, avec politique d’accès, URLs signées, et option embeddings.

---

## Ce que c’est

- Dossiers métier multi-tenant (`tenantId` + `siteId` dérivés de la clé API)
- Sous-dossiers via `parentFolderId`
- Stockage S3 :

```text
tenants/{tenantId}/sites/{siteId}/folders/{folderId}/assets/{assetId}/original.{ext}
```

- **Tous objets S3** : image, vidéo, audio, PDF, documents… (types sûrs)
- **Embeddings / recherche vectorielle** : optionnels, **images seulement**, activables **par dossier**
- **Lecture fichier** : URL S3 signée après ACL (pas de proxy public pour les assets liés à un dossier, sauf `PUBLIC` + `PUBLIC_READ`)

---

## Qui fait quoi

| Action | Permission clé `gfk_` |
|--------|------------------------|
| Créer / configurer / supprimer dossier | **ADMIN** |
| Activer / désactiver embeddings | **ADMIN** |
| Lister dossiers | READ |
| Uploader un fichier dans un dossier | READ_WRITE |
| Obtenir `read-url` / `download-url` | READ (+ `X-End-User-Id` selon politique) |

---

## Activer ou non les embeddings

L’admin choisit **par dossier** sur le `siteId` de sa clé. Les flags sont **appliqués** au moteur Visual Search (pas cosmétique) :

| Flag | Effet |
|------|--------|
| `allowEmbeddings=true` | Les nouvelles images du dossier peuvent être indexées |
| `allowEmbeddings=false` | Pas d’indexation ; désactivation → exclusion / delete vectorielle |
| `allowVectorSearch=true` | Peut apparaître en recherche visuelle |
| `allowVectorSearch=false` | Exclu des résultats de recherche |

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

### Dossier privé

```json
{ "visibility": "PRIVATE", "accessPolicy": "OWNER_ONLY" }
```

→ embeddings **impossibles**.  
Embeddings activés ≠ accès public au fichier.

---

## Lecture fichier (URL signée)

Assets liés à un dossier (`BOUND`) :

```http
POST /v1/assets/{assetId}/read-url
X-Api-Key: gfk_READ_...
X-End-User-Id: <uuid>   # requis si OWNER_ONLY / SITE_USERS / …

{
  "expiresSeconds": 300
}
```

Téléchargement :

```http
POST /v1/assets/{assetId}/download-url
```

| `accessPolicy` | Qui peut lire |
|----------------|---------------|
| `PUBLIC_READ` | Clé READ+ (proxy public OK si `visibility=PUBLIC`) |
| `API_KEY_SCOPED` | Clé site READ+ |
| `SITE_USERS` | `X-End-User-Id` requis |
| `OWNER_ONLY` | Propriétaire (ou ADMIN) |
| `RECORD_MEMBERS` | Owner pour l’instant (membership chat à venir) |

Le proxy historique `GET /v1/api/cloud/file/{id}` refuse les assets `BOUND` non publics (`BOUND_ASSET_READ_URL_REQUIRED`).

---

## Créer un dossier / sous-dossier

```http
POST /v1/asset-folders
X-Api-Key: gfk_ADMIN_...

{
  "name": "Médias",
  "slug": "medias",
  "visibility": "SITE",
  "accessPolicy": "SITE_USERS",
  "allowEmbeddings": false
}
```

Sous-dossier :

```json
{
  "name": "Archives",
  "parentFolderId": "<uuid-du-parent>"
}
```

---

## Upload (image, PDF, vidéo…)

Flags serveur requis :

```text
ASSET_FOLDER_V1_ENABLED=true
ASSET_FOLDER_UPLOAD_V1_ENABLED=true
```

```http
POST /v1/asset-folders/{folderId}/assets/presign
X-Api-Key: gfk_READ_WRITE_...

{
  "filename": "contrat.pdf",
  "mimeType": "application/pdf",
  "sizeBytes": 120000,
  "usage": "DOCUMENT"
}
```

Puis upload sur `uploadUrl`, puis :

```http
POST /v1/asset-folders/{folderId}/assets/commit
{
  "assetId": "...",
  "reservationId": "...",
  "uploadKey": "..."
}
```

- PDF / vidéo / audio → stockés S3, **pas** d’embedding
- Image + `allowEmbeddings=true` → index Visual Search

---

## Compatibilité legacy

Les routes historiques restent valides :

- `POST /v1/api/cloud/upload`
- `POST /v1/api/cloud/upload/direct`
- `POST /v1/api/cloud/upload/commit`

Sans `folderId` → asset `LEGACY_UNBOUND` (comportement inchangé).

---

## Voir aussi

- [Visual Search](visual-search.md)
- [Fichiers / upload signé](cloud-files-google-upload.md)
- [API Reference](api.md)
