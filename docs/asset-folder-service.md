# Asset Folder Service — Noverfly

**Date :** 2026-08-06  
**API :** `https://api.noverfly.com`

Les Asset Folders organisent le stockage cloud (S3) par dossier / sous-dossier, avec politique d’accès et option embeddings.

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

---

## Qui fait quoi

| Action | Permission clé `gfk_` |
|--------|------------------------|
| Créer / configurer / supprimer dossier | **ADMIN** |
| Activer / désactiver embeddings | **ADMIN** |
| Lister dossiers | READ |
| Uploader un fichier dans un dossier | READ_WRITE |

---

## Activer ou non les embeddings

L’admin choisit **par dossier** :

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

Décrocher (plus d’index / recherche vectorielle sur ce dossier) :

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

→ embeddings **impossibles** (sécurité).  
Le service Visual Search refuse d’indexer ces assets.

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
- Image + `allowEmbeddings=true` → index Visual Search possible

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
