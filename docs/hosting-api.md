# Noverfly Hosting — déployer React, Vite et Next.js statique

Noverfly Hosting publie des sites précompilés dans OVH Object Storage (S3), avec
releases immuables, vérification SHA-256, activation atomique, rollback,
quotas par offre et domaine automatique `slug.noverfly.com`.

Base URL : `https://api.noverfly.com`

Auth : clé secrète `gfk_` liée au site.

> Une `gfk_` est liée à un site existant : elle peut créer/configurer son
> **projet Hosting** et déployer ce site, mais ne peut pas créer un autre tenant
> ou site. La création initiale d’un site reste une opération dashboard JWT.

## Architectures acceptées

| Projet | Commande de build | Dossier à déployer | `framework` |
|---|---|---|---|
| HTML/CSS/JS | votre commande | dossier public | `static` |
| Vite / React | `npm run build` | `dist` | `vite` ou `react` |
| Next.js statique | `next build` avec `output: "export"` | `out` | `next-export` |
| Astro statique | `astro build` | `dist` | `astro` |

Le Hosting API n’exécute pas de `package.json`, de scripts shell ou de code
serveur fourni par un client. Cela protège l’infrastructure multi-tenant.
Pour un backend, utilisez Data API, Cloud Scripts, Calls, Auth ou une API
externe.

Next.js SSR, Server Actions et API Routes nécessitent un runtime conteneur
dédié ; ils ne sont pas acceptés dans le Hosting statique mutualisé.

## Modèle de sécurité

- `READ` : statut du projet et des releases ;
- `ADMIN` : configuration, upload, activation, rollback et suppression ;
- objets S3 privés, servis par le runtime Noverfly ;
- chemins normalisés, traversal et exécutables serveur bloqués ;
- taille, nombre de fichiers, stockage et rétention limités par le plan ;
- upload direct S3 signé pendant 15 minutes ;
- vérification de la taille et du SHA-256 avant activation ;
- une release ne devient visible qu’après activation atomique.

Ne placez jamais une `gfk_` dans le JavaScript envoyé au navigateur.
Déployez depuis CI/CD ou une machine d’administration.

## Déploiement rapide (CLI incluse)

Depuis le dépôt Cloud :

```bash
npm run hosting:deploy -- \
  --dir dist \
  --framework vite \
  --key gfk_VOTRE_CLE_ADMIN
```

Next.js export :

```bash
npm run hosting:deploy -- \
  --dir out \
  --framework next-export \
  --key gfk_VOTRE_CLE_ADMIN
```

La commande :

1. calcule la taille et SHA-256 de chaque fichier ;
2. crée une release immuable ;
3. demande les URL S3 signées par lots ;
4. transfère huit fichiers en parallèle ;
5. demande la vérification S3 ;
6. active atomiquement la release.

Variables possibles :

```bash
export NOVERFLY_GFK=gfk_VOTRE_CLE_ADMIN
npm run hosting:deploy -- --dir dist --framework vite
```

## API complète

### 1. Configurer le projet Hosting

```bash
curl -X PUT "https://api.noverfly.com/v1/hosting/project" \
  -H "X-Api-Key: gfk_VOTRE_CLE_ADMIN" \
  -H "Content-Type: application/json" \
  -d '{
    "framework": "vite",
    "output_directory": "dist",
    "settings": {
      "spa": true
    }
  }'
```

Cette opération enregistre aussi le domaine automatique
`SITE_SLUG.noverfly.com` dans le site.

### 2. Construire le manifeste

Chaque fichier doit fournir :

```json
{
  "path": "assets/app.2f91d8c1.js",
  "size_bytes": 184231,
  "sha256": "64_caracteres_hexadecimaux",
  "mime_type": "text/javascript; charset=utf-8"
}
```

Créer la release :

```bash
curl -X POST "https://api.noverfly.com/v1/hosting/deployments" \
  -H "X-Api-Key: gfk_VOTRE_CLE_ADMIN" \
  -H "Content-Type: application/json" \
  -d '{
    "framework": "vite",
    "entrypoint": "index.html",
    "source_type": "PREBUILT",
    "files": [
      {
        "path": "index.html",
        "size_bytes": 642,
        "sha256": "SHA256_INDEX",
        "mime_type": "text/html; charset=utf-8"
      },
      {
        "path": "assets/app.2f91d8c1.js",
        "size_bytes": 184231,
        "sha256": "SHA256_JS",
        "mime_type": "text/javascript; charset=utf-8"
      }
    ]
  }'
```

La réponse contient `deployment.id`.

### 3. Obtenir les URL d’upload

Maximum 100 chemins par appel :

```bash
curl -X POST \
  "https://api.noverfly.com/v1/hosting/deployments/DEPLOYMENT_ID/upload-urls" \
  -H "X-Api-Key: gfk_VOTRE_CLE_ADMIN" \
  -H "Content-Type: application/json" \
  -d '{"paths":["index.html","assets/app.2f91d8c1.js"]}'
```

Pour chaque résultat, effectuer le `PUT` avec **tous** les
`required_headers` retournés :

```bash
curl -X PUT "URL_SIGNEE_S3" \
  -H "Content-Type: text/html; charset=utf-8" \
  -H "Cache-Control: public, max-age=0, must-revalidate" \
  -H "x-amz-acl: private" \
  -H "x-amz-meta-sha256: SHA256_INDEX" \
  --data-binary "@dist/index.html"
```

### 4. Vérifier puis activer

```bash
curl -X POST \
  "https://api.noverfly.com/v1/hosting/deployments/DEPLOYMENT_ID/finalize" \
  -H "X-Api-Key: gfk_VOTRE_CLE_ADMIN"

curl -X POST \
  "https://api.noverfly.com/v1/hosting/deployments/DEPLOYMENT_ID/activate" \
  -H "X-Api-Key: gfk_VOTRE_CLE_ADMIN"
```

L’activation remplace la release live en une transaction. Les visiteurs ne
voient donc jamais un site partiellement transféré.

### 5. Lister, inspecter, rollback

```bash
curl "https://api.noverfly.com/v1/hosting/deployments" \
  -H "X-Api-Key: gfk_VOTRE_CLE"

curl "https://api.noverfly.com/v1/hosting/deployments/DEPLOYMENT_ID" \
  -H "X-Api-Key: gfk_VOTRE_CLE"
```

Réactiver une ancienne release `SUPERSEDED` effectue le rollback :

```bash
curl -X POST \
  "https://api.noverfly.com/v1/hosting/deployments/ANCIENNE_RELEASE/activate" \
  -H "X-Api-Key: gfk_VOTRE_CLE_ADMIN"
```

Supprimer une release non active :

```bash
curl -X DELETE \
  "https://api.noverfly.com/v1/hosting/deployments/DEPLOYMENT_ID" \
  -H "X-Api-Key: gfk_VOTRE_CLE_ADMIN"
```

Supprimer tout le projet Hosting et ses objets S3, sans supprimer le site
Noverfly ni ses collections :

```bash
curl -X DELETE "https://api.noverfly.com/v1/hosting/project" \
  -H "X-Api-Key: gfk_VOTRE_CLE_ADMIN"
```

## Brancher le backend Noverfly

Le projet Hosting reste lié au même `siteId` et à la même `gfk_`.

| Fonction | Endpoint serveur |
|---|---|
| Données / collections | `https://api.noverfly.com/v1/api/data` |
| Cloud Scripts | `https://api.noverfly.com/v1/api/data/scripts` |
| Auth utilisateurs | `https://api.noverfly.com/v1/app/PROJECT_ID/auth` |
| Calls activation/admin | `https://api.noverfly.com/v1/cloud/calls` |
| Upload File Cloud | `https://api.noverfly.com/v1/files` |

Un navigateur ne doit pas appeler les routes admin avec `gfk_`. Pour les
opérations privilégiées, créez un Cloud Script public contrôlé ou votre propre
BFF. Pour l’auth et les appels, échangez les jetons courts prévus par leur API.

## Quotas

Les clés suivantes peuvent être définies dans `Plan.limits` :

- `hosting_storage_bytes`
- `hosting_max_deployment_bytes`
- `hosting_max_files`
- `hosting_max_file_bytes`
- `hosting_retained_deployments`

À défaut, Noverfly applique des limites adaptées aux offres Free, Starter,
Business et Pro. Les anciennes releases dépassant la rétention sont supprimées
de S3 après activation d’une nouvelle release.

## Codes d’erreur principaux

| Code | Signification |
|---|---|
| `INVALID_GFK_KEY` | clé absente, révoquée ou pas de type `gfk_` |
| `INSUFFICIENT_PERMISSION` | une clé ADMIN est requise |
| `HOSTING_STORAGE_QUOTA_EXCEEDED` | quota S3 du plan dépassé |
| `DEPLOYMENT_SIZE_LIMIT` | release trop volumineuse |
| `BLOCKED_FILE_TYPE` | fichier exécutable/serveur interdit |
| `OBJECT_SIZE_MISMATCH` | fichier S3 différent du manifeste |
| `OBJECT_CHECKSUM_MISMATCH` | métadonnée SHA-256 différente |
| `DEPLOYMENT_NOT_READY` | activation avant vérification |

