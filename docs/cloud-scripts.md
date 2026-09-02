# NoverFly Cloud Scripts

La couche **Cloud Programmable Script Layer** s'appuie sur DevAPI Automation Cloud.
Chaque client définit **ses propres collections** (comme des feuilles Google Sheets) et **ses propres scripts** (comme Apps Script). La plateforme exécute le code et expose les résultats en API.

## Analogie développeur

| Google | Cloud Noverfly |
|--------|----------------|
| Google Sheets (onglets, colonnes) | **Collections** (`products`, `users`, …) |
| Lignes du tableur | **Records** (`data` JSON libre) |
| Apps Script | **Scripts cloud** (`feed.script.js` → `main(ctx)`) |
| Historique des versions | **`DevApiScriptVersion`** + rollback |
| Publier une web app | **Publish** + route `/r/api/feed` |

## Auth : deux portes distinctes

| Porte | Header | Routes |
|-------|--------|--------|
| **DevAPI CLI / clé secrète** | `X-Api-Key: gfk_…` ou `Authorization: Bearer gfk_…` | `/v1/devapi/scripts/*`, `/v1/api/data/*` |
| **Dashboard admin** | JWT session (`__session` ou Bearer JWT) | `/v1/projects/:projectId/scripts/*` |

Ne pas envoyer une clé `gfk_` sur les routes dashboard : elles attendent un JWT utilisateur.

## Qui peut modifier quoi (clé `gfk_`)

| Ressource | Créer | Lire | Modifier | Supprimer | Via script `ctx` |
|-----------|-------|------|----------|-----------|-------------------|
| **Collection** (schéma) | Oui `POST /v1/api/data/collections` (ADMIN) | Oui | Oui `PATCH` (ADMIN) | Oui (ADMIN) | `ctx.services.collections.createCollection` |
| **Record collection** | Oui `POST …/records` | Oui | Oui `PATCH …/records/:id` | Oui | `ctx.collections.{slug}.create/update/delete` |
| **Script cloud** | Oui `POST /v1/devapi/scripts` | Oui | Oui **`PATCH /v1/devapi/scripts/:slug`** | Oui **`DELETE`** | — |
| **Exécution script** | — | — | — | — | `ctx.scripts.run('autre-slug')` |

Les scripts modifient les **records** de n'importe quelle collection du site, sans recréer le schéma (sauf via `ctx.services.collections`).

## Gestion complète des scripts (DevAPI — clé `gfk_`)

### Créer

```http
POST /v1/devapi/scripts
X-Api-Key: gfk_xxxxx
```

```json
{
  "name": "Feed",
  "slug": "feed",
  "status": "active",
  "code": "export default async function main(ctx) { ... }"
}
```

### Mettre à jour un slug existant (plus besoin de `feed-mf2`)

```http
PATCH /v1/devapi/scripts/feed
X-Api-Key: gfk_xxxxx
```

```json
{
  "code": "export default async function main(ctx) { ... }",
  "changelog": "Fix filtre productStatus"
}
```

Alias upsert :

- `POST /v1/devapi/scripts?upsert=true` — crée ou met à jour si le slug existe
- `PUT /v1/devapi/scripts/feed` — idem (200 si mis à jour, 201 si créé)

### Lire / supprimer

```http
GET    /v1/devapi/scripts/feed
DELETE /v1/devapi/scripts/feed
GET    /v1/devapi/scripts/feed/versions?limit=50
```

### Historique et rollback

Chaque modification de `code` enregistre automatiquement une version dans `DevApiScriptVersion` (avant + après).

```http
POST /v1/devapi/scripts/feed/publish
{ "changelog": "Mise en prod v3" }

POST /v1/devapi/scripts/feed/rollback
{ "version": 2 }
```

Sans `version`, rollback = version publiée précédente.

### Exécuter (test ou live)

```http
POST /v1/devapi/scripts/run
```

```json
{
  "script": "feed",
  "input": { "limit": 20 },
  "mode": "test"
}
```

Alias : `POST /v1/devapi/scripts/feed/run`, header `X-NoverFly-Mode: test`.

## Contexte `ctx` dans un script

| Propriété | Description |
|-----------|-------------|
| `ctx.input` | Payload HTTP |
| `ctx.project` | `{ id: siteId, tenantId }` |
| `ctx.collections.{slug}` | CRUD sur **vos** collections |
| `ctx.services.collections` | CRUD schéma collection (admin) |
| `ctx.cache` | Cache Redis |
| `ctx.layout` / `ctx.performance` | Helpers layout feed |
| `ctx.scripts.run()` | Chaîner un autre script |

Exemple — le client choisit **son** schéma :

```js
export default async function main(ctx) {
  const products = await ctx.collections.products.find({ limit: 20 });
  const visible = products.filter((p) => p.data?.visibility === 'public');
  return { items: visible.map(/* mapping feed */) };
}
```

## Collections (données client)

Création schéma (API, clé ADMIN) :

```http
POST /v1/api/data/collections
```

Records :

```http
GET|POST|PATCH|DELETE /v1/api/data/collections/{slug}/records[/{id}]
```

Dans un script :

```js
await ctx.collections.products.find({ limit: 25 })
await ctx.collections.products.update(id, { data: { stock: 10 } })
```

## Routes publiques script

```http
POST /v1/devapi/script-routes
```

```json
{
  "script": "feed",
  "routePath": "/api/feed",
  "method": "GET",
  "authMode": "public-read",
  "status": "active"
}
```

→ `GET /v1/public/sites/{projectId}/r/api/feed`

## Dashboard (JWT — console web)

Mêmes opérations que DevAPI, plus `/test` avec brouillon :

- `GET|POST /v1/projects/:projectId/scripts`
- `GET|PATCH|DELETE /v1/projects/:projectId/scripts/:slug`
- `GET /v1/projects/:projectId/scripts/:slug/versions`
- `POST /v1/projects/:projectId/scripts/:slug/test|publish|rollback|logs`

## Workflow CLI recommandé

```
1. POST /v1/devapi/scripts?upsert=true     → upload feed.script.js
2. POST /v1/devapi/scripts/feed/publish    → version prod
3. POST /v1/devapi/scripts/run             → test { "script":"feed", "mode":"test" }
4. GET  /v1/devapi/scripts/feed/versions   → historique
5. POST /v1/devapi/scripts/feed/rollback   → revenir en arrière si besoin
```

## Sécurité

- Isolation stricte par `tenantId` + `siteId`
- Sandbox JavaScript (`node:vm`)
- Timeout, quotas plan, logs (`DevApiExecutionLog.scriptSlug`)
- Versioning + rollback sur chaque modification de code

## Exemples génériques

Templates dans [`docs/examples/scripts/`](examples/scripts/) — **non imposés** ; adaptez à vos collections.

## Guide opérationnel

Architecture Docker + BullMQ, envoi/suppression de `.script.js`, jointures multi-collections, triggers, présence messenger :

→ **[cloud-scripts-operational-guide.md](cloud-scripts-operational-guide.md)**
