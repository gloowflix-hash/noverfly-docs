# Guide opérationnel — Cloud Scripts, jobs & notifications

> **Important** : Noverfly **n'utilise pas systemd par script** ni un serveur Node.js séparé par application cliente.
> Tout passe par le conteneur API **`gloowflix-api`** (`https://api.noverfly.com`), Redis **BullMQ**, et PostgreSQL.

## Architecture réelle (prod)

```
Client APK / Web
    │
    ▼
api.noverfly.com  (Caddy → gloowflix-api Docker)
    │
    ├── Fastify REST  (/v1/api/data/*, /v1/devapi/scripts/*, /v1/cloud/*)
    ├── WebSocket     (wss://api.noverfly.com/ws — présence, messages temps réel)
    └── Workers BullMQ (même processus Node.js au démarrage API)
            │
            ├── devapi-automation   ← scripts triggers, cron DevAPI, workflows
            ├── send-push           ← FCM / APNs / Expo / Web Push
            ├── send-digest         ← digest notifications
            └── ai-cloud, publish-site, …
    │
    ▼
PostgreSQL (collections, scripts DevApiFunction, logs)
Redis      (cache scripts, présence, queues BullMQ)
```

| Composant | Nom technique | Rôle |
|-----------|---------------|------|
| API backend | Conteneur `gloowflix-api` | Exécute scripts, Data API, messenger |
| Planificateur | Queue BullMQ `devapi-automation` | Triggers collection, cron, workflows |
| Notifications | `notificationOrchestratorService` → queue `send-push` | Push si utilisateur hors conversation |
| Présence chat | `presenceService` (Redis + `user_presence_snapshots`) | online / offline / busy |
| Scripts cloud | Table `dev_api_functions` (`kind: 'script'`) | Code JS stocké en base, pas de fichier `.script` sur disque VPS |

Le VPS peut avoir **systemd uniquement pour Docker** (`docker compose up`). Ce n'est **pas** systemd qui appelle vos scripts — c'est **BullMQ**.

---

## Fichiers `.script.js` vs stockage cloud

En local, vous écrivez `feed.script.js`, `product.script.js`, etc.
En production, le **slug** est normalisé (`feed.script` → `feed`).

### Envoyer / créer un script (upsert)

```bash
API=https://api.noverfly.com
GFK=gfk_YOUR_SECRET_KEY

# Depuis un fichier local
CODE=$(node -e "console.log(JSON.stringify(require('fs').readFileSync('feed.script.js','utf8')))")

curl -X POST "$API/v1/devapi/scripts?upsert=true" \
  -H "X-Api-Key: $GFK" \
  -H "Content-Type: application/json" \
  -d "{
    \"name\": \"Feed\",
    \"slug\": \"feed\",
    \"status\": \"active\",
    \"code\": $CODE,
    \"cachePolicy\": { \"enabled\": true, \"ttl\": 30, \"tags\": [\"posts\", \"media_assets\"] },
    \"triggers\": [
      { \"type\": \"onUpdate\", \"collection\": \"posts\", \"enabled\": true },
      { \"type\": \"onUpdate\", \"collection\": \"media_assets\", \"enabled\": true }
    ]
  }"
```

### Mettre à jour le code

```bash
curl -X PATCH "$API/v1/devapi/scripts/feed" \
  -H "X-Api-Key: $GFK" \
  -H "Content-Type: application/json" \
  -d '{ "code": "export default async function main(ctx) { ... }", "changelog": "fix filtre" }'
```

### Publier / rollback

```bash
curl -X POST "$API/v1/devapi/scripts/feed/publish" -H "X-Api-Key: $GFK"
curl -X POST "$API/v1/devapi/scripts/feed/rollback" -H "X-Api-Key: $GFK" -d '{ "version": 2 }'
```

### Supprimer un script

```bash
curl -X DELETE "$API/v1/devapi/scripts/feed" -H "X-Api-Key: $GFK"
```

Le slug disparaît de la base ; les routes publiques liées doivent être désactivées séparément (`PATCH /v1/devapi/script-routes/:id`).

### Exécuter manuellement (test)

```bash
curl -X POST "$API/v1/devapi/scripts/run" \
  -H "X-Api-Key: $GFK" \
  -H "Content-Type: application/json" \
  -d '{ "script": "feed", "input": { "limit": 20, "page": 1 }, "mode": "test" }'
```

---

## Relier plusieurs collections → une seule réponse API

Pattern **feed agrégé** : lire plusieurs collections, joindre par ID, renvoyer **un JSON prêt client**.

Exemple réel : [`examples/scripts/feed.script.js`](examples/scripts/feed.script.js)

```js
export default async function main(ctx) {
  const posts = await ctx.collections.posts.find({ status: 'published', limit: 20 });

  const mediaIds = posts.flatMap((p) => p.data?.mediaIds || p.mediaIds || []);
  const userIds = [...new Set(posts.map((p) => p.data?.authorId || p.authorId).filter(Boolean))];

  const [mediaMap, users] = await Promise.all([
    ctx.collections.media_assets.findMapByIds(mediaIds),
    ctx.collections.users.findByIds(userIds),
  ]);

  const items = posts.map((post) => ({
    id: post.id,
    title: post.data?.title || post.title,
    hour: post.data?.publishedAt || post.createdAt,
    author: users.find((u) => u.id === post.data?.authorId),
    media: (post.data?.mediaIds || []).map((id) => mediaMap[id]).filter(Boolean),
    layout: ctx.layout.computePostLayout(post, media),
  }));

  return { items, page: 1, limit: 20 };
}
```

| Méthode `ctx.collections.{slug}` | Usage |
|----------------------------------|-------|
| `find({ limit, page, status, … })` | Liste filtrée |
| `findByIds(ids[])` | Batch par ID |
| `findMapByIds(ids[])` | `{ [id]: record }` pour jointures |
| `create(data)` / `update(id, data)` / `delete(id)` | CRUD |
| `ctx.scripts.run('autre-slug', input)` | Appeler un autre script |

Les champs `id`, `title`, `hour`/`publishedAt` vivent dans **`data` JSON** de chaque record — vous définissez le schéma de vos collections.

---

## Appeler un script depuis un autre script

```js
export default async function main(ctx) {
  const feed = await ctx.scripts.run('feed', { limit: 10, page: 1 });
  const enriched = await ctx.scripts.run('marketing', { segment: 'vip', feedItems: feed.items });
  return enriched;
}
```

Équivalent DevAPI Automation (workflow) : action `execute_function` avec `functionName`.

---

## Automatisation : triggers collection (sans cron systemd)

Quand un record est créé / modifié / supprimé dans la Data API :

1. `dispatchScriptCollectionTrigger` enqueue un job BullMQ
2. Job name : `script_collection_trigger` sur queue **`devapi-automation`**
3. Le worker exécute tous les scripts `status: active` dont `triggers` matchent

Exemple de triggers sur un script :

```json
{
  "triggers": [
    { "type": "onCreate", "collection": "orders", "enabled": true },
    { "type": "onUpdate", "collection": "orders", "enabled": true }
  ]
}
```

Payload reçu dans `main(ctx)` :

```js
export default async function main(ctx) {
  const { record, previousRecord, action, collectionSlug, recordId } = ctx.input;
  if (action === 'created' && collectionSlug === 'orders') {
    await ctx.services.notifications.send({
      to: 'admin',
      title: 'Nouvelle commande',
      message: `Commande ${recordId} — ${record.data?.title}`,
      push: true,
    });
  }
  return { ok: true };
}
```

**Anti-boucle** : les écritures faites *depuis* un script (`ctx.collections.*`) ne re-déclenchent pas les triggers (`skipScriptTriggersOnWrite: true`).

---

## Jobs planifiés (cron) — BullMQ, pas systemd

```bash
curl -X POST "$API/v1/devapi/jobs" \
  -H "X-Api-Key: $GFK" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Sync feed nightly",
    "functionId": "UUID_DU_SCRIPT_OU_FONCTION",
    "cron": "0 3 * * *",
    "payload": { "limit": 100 },
    "mode": "live",
    "status": "active"
  }'
```

Supprimer : `DELETE /v1/devapi/jobs/JOB_ID`

---

## Notifications depuis un script

```js
await ctx.services.notifications.send({
  to: 'user:USER_UUID',
  title: 'Titre',
  message: 'Corps',
  push: true,
  link: '/orders/123',
});
```

Chaîne : orchestrateur → queue **`send-push`** → worker FCM.

Appels entrants : [`push-fcm-cloud.md`](push-fcm-cloud.md) (`call_invite`, `call_cancel`).

---

## Présence en ligne / hors ligne (messages)

| Statut | Condition |
|--------|-----------|
| `online` | WebSocket connecté |
| `offline` | Aucune connexion WS |
| `busy` | Appel actif |

À l'envoi d'un message :

```text
pushEnabled = !isUserActiveInConversation(recipient, conversationId)
```

```bash
curl "$API/v1/cloud/messenger/presence?userIds=uuid1,uuid2" -H "X-Api-Key: $GFK"
```

WebSocket : `wss://api.noverfly.com/ws`

---

## Routes publiques

```bash
curl "$API/v1/public/sites/YOUR_PROJECT_ID/scripts/feed?page=1&limit=15"
curl "$API/v1/public/sites/YOUR_PROJECT_ID/r/api/feed?page=1&limit=15"
```

---

## Docs liées

- [cloud-scripts.md](cloud-scripts.md) — CRUD scripts
- [devapi-automation.md](devapi-automation.md) — workflows
- [migrations-and-jobs.md](migrations-and-jobs.md) — queues BullMQ
- [push-fcm-cloud.md](push-fcm-cloud.md) — push tenant FCM
