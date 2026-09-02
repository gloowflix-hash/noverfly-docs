# Mini-services avec Cloud Scripts — pourquoi et comment

Ce guide explique **pourquoi** créer de petits services avec des fichiers scripts Noverfly, et **comment** automatiser des actions quelques secondes après un événement, à l’arrivée d’une nouvelle donnée, ou en chaînant IA / notifications / messagerie.

Base URL : `https://api.noverfly.com`  
Auth scripts : `X-Api-Key: gfk_YOUR_SECRET_KEY`  
Auth fichiers média : `X-Api-Key: gfc_YOUR_CLOUD_KEY`

---

## L’idée en une phrase

Vous n’hébergez **pas** un serveur Node.js par application.  
Vous déposez un fichier `*.script.js` dans le cloud Noverfly → la plateforme l’exécute pour vous (API, triggers, cron, IA).

C’est l’équivalent moderne de **Google Sheets + Apps Script**, mais branché sur vos collections, vos médias, vos push et votre IA.

| Chez Google | Chez Noverfly |
|-------------|----------------|
| Feuille / onglet | Collection (`orders`, `posts`, …) |
| Ligne | Record (`data` JSON) |
| Apps Script | Cloud Script (`feed.script.js` → `main(ctx)`) |
| Déclencheur onEdit | Triggers `onCreate` / `onUpdate` / `onDelete` |
| Déclencheur temporel | Job cron BullMQ |
| Web app déployée | Route publique `/r/api/feed` |

---

## Pourquoi c’est important

Sans mini-service cloud, chaque app (APK, web, TV) doit :

- recalculer les jointures posts + médias + auteurs
- gérer les règles métier côté client
- appeler plusieurs APIs et recomposer le JSON

Avec un script cloud :

1. **Une seule réponse prête client** — l’APK appelle `/r/api/feed` et affiche.
2. **Automatisation serveur** — nouvelle commande → push + email + IA, sans ouvrir l’app.
3. **Clés secrètes hors APK** — `gfk_` / providers restent côté serveur.
4. **Évolution sans rebuild** — vous patcher le script, pas republier l’APK.
5. **Composition intelligente** — un script appelle un autre (`ctx.scripts.run`), ou un workflow IA.

---

## Cycle de vie d’un mini-service

```
1. Écrire feed.script.js en local
2. Upsert vers Noverfly (gfk_)
3. Publier + (optionnel) route publique
4. Brancher triggers / cron
5. L’app consomme le JSON
6. Patcher le code sans toucher l’APK
```

### 1. Fichier local

```js
// feed.script.js
export default async function main(ctx) {
  const limit = Number(ctx.input.limit || 20);
  const page = Number(ctx.input.page || 1);

  const posts = await ctx.collections.posts.find({
    status: 'published',
    limit,
    page,
  });

  return { items: posts, page, limit };
}
```

### 2. Envoyer le script (upsert)

```bash
API=https://api.noverfly.com
GFK=gfk_YOUR_SECRET_KEY
CODE=$(node -e "console.log(JSON.stringify(require('fs').readFileSync('feed.script.js','utf8')))")

curl -X POST "$API/v1/devapi/scripts?upsert=true" \
  -H "X-Api-Key: $GFK" \
  -H "Content-Type: application/json" \
  -d "{
    \"name\": \"Feed\",
    \"slug\": \"feed\",
    \"status\": \"active\",
    \"code\": $CODE
  }"
```

### 3. Tester

```bash
curl -X POST "$API/v1/devapi/scripts/run" \
  -H "X-Api-Key: $GFK" \
  -H "Content-Type: application/json" \
  -d '{ "script": "feed", "input": { "limit": 10 }, "mode": "test" }'
```

### 4. Appeler depuis l’app (sans gfk_)

```bash
curl "$API/v1/public/sites/YOUR_PROJECT_ID/r/api/feed?limit=10"
```

---

## Automatiser : 4 façons concrètes

### A. Quelques secondes après une nouvelle donnée (`onCreate`)

Quand un record arrive dans une collection, BullMQ exécute vos scripts actifs.

```json
{
  "triggers": [
    { "type": "onCreate", "collection": "orders", "enabled": true },
    { "type": "onUpdate", "collection": "orders", "enabled": true }
  ]
}
```

```js
export default async function main(ctx) {
  const { action, collectionSlug, record, recordId } = ctx.input || {};

  if (collectionSlug === 'orders' && action === 'created') {
    await ctx.services.notifications.send({
      to: 'admin',
      title: 'Nouvelle commande',
      message: `Commande ${recordId}`,
      push: true,
    });

    // Enrichissement IA (si AI Cloud activé)
    if (ctx.services?.ai?.generateText) {
      const summary = await ctx.services.ai.generateText({
        prompt: `Résume cette commande en une phrase: ${JSON.stringify(record?.data || {})}`,
      });
      await ctx.collections.orders.update(recordId, {
        data: { ...(record.data || {}), aiSummary: summary?.text || summary },
      });
    }
  }

  return { ok: true };
}
```

Délai typique : **quelques secondes** (enqueue BullMQ → worker → exécution), pas une boucle synchrone dans l’APK.

### B. Toutes les nuits / toutes les heures (cron)

```bash
curl -X POST "$API/v1/devapi/jobs" \
  -H "X-Api-Key: $GFK" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Cleanup drafts",
    "functionId": "UUID_DU_SCRIPT",
    "cron": "0 */6 * * *",
    "payload": { "status": "draft" },
    "mode": "live",
    "status": "active"
  }'
```

### C. Chaîner des mini-services (composition)

```js
export default async function main(ctx) {
  const feed = await ctx.scripts.run('feed', { limit: 20 });
  const ranked = await ctx.scripts.run('ranking', { items: feed.items });
  return ranked;
}
```

Ou via **workflow** DevAPI (`execute_function`, `ai_generate_text`, `schedule_action`) — voir [devapi-automation.md](devapi-automation.md).

### D. Webhook / événement externe

```bash
curl -X POST "$API/v1/devapi/events" \
  -H "X-Api-Key: $GFK" \
  -H "Content-Type: application/json" \
  -H "Idempotency-Key: evt-unique-001" \
  -d '{ "event": "lead.qualified", "payload": { "email": "a@b.com" }, "mode": "live" }'
```

En `live`, la signature `X-NoverFly-Signature` est requise (voir [devapi-automation.md](devapi-automation.md)).

---

## Connecter des briques intelligentes

| Besoin | Comment depuis un script / workflow |
|--------|-------------------------------------|
| Lire / écrire données | `ctx.collections.{slug}.*` |
| Appeler un autre algo | `ctx.scripts.run('slug', input)` |
| Push / notif | `ctx.services.notifications.send({…})` |
| Email | `ctx.services.email.send({…})` |
| IA texte | `ctx.services.ai.generateText({ prompt })` (AI Cloud activé) |
| Messenger | `ctx.services.messenger.sendMessage({…})` |
| Temps réel | `ctx.realtime` (bridge site) |
| Cache réponse | `cachePolicy` sur le script (TTL + tags) |

Activation IA : [ai-cloud-service.md](ai-cloud-service.md).

---

## Clés et secrets : où les mettre

| Clé | Où | Jamais |
|-----|-----|--------|
| `gfk_` SECRET | Serveur CI, backend, upsert scripts | APK / React public |
| `gfc_` CLOUD | Upload médias serveur ou backend | APK si possible |
| `vst_` / `ves_` | Tokens courts remis au client | — |
| Clés OpenAI / Google | AI Cloud vault (`/v1/cloud/ai/keys/save`) ou collection privée serveur | Code script en clair dans un repo public |

Pattern recommandé pour config runtime :

```js
// Lire une config privée (collection non exposée publiquement)
const secrets = await ctx.collections.app_config.find({ key: 'integrations', limit: 1 });
const webhookUrl = secrets[0]?.data?.orderWebhookUrl;
```

Ne stockez **pas** de `gfk_` dans le code du script versionné publiquement. Le runtime Noverfly exécute déjà le script dans le contexte du site authentifié.

---

## Automatiser l’arrivée de nouveaux scripts / algos

Quand vous ajoutez un nouvel algo (ranking, modération, pricing) :

1. Créez `ranking.script.js` en local.
2. Upsert avec `?upsert=true` (même slug = mise à jour, pas de nouveau deploy VPS).
3. Branchez un trigger ou un appel depuis `feed` :
   ```js
   const items = await ctx.scripts.run('ranking', { items: rawItems });
   ```
4. Publiez / rollback si besoin :
   ```bash
   curl -X POST "$API/v1/devapi/scripts/ranking/publish" -H "X-Api-Key: $GFK"
   curl -X POST "$API/v1/devapi/scripts/ranking/rollback" \
     -H "X-Api-Key: $GFK" -d '{ "version": 2 }'
   ```

Pipeline CI typique :

```bash
# .github/workflows/deploy-scripts.yml (idée)
for f in scripts/*.script.js; do
  slug=$(basename "$f" .script.js)
  CODE=$(node -e "console.log(JSON.stringify(require('fs').readFileSync('$f','utf8')))")
  curl -sS -X POST "$API/v1/devapi/scripts?upsert=true" \
    -H "X-Api-Key: $GFK" \
    -H "Content-Type: application/json" \
    -d "{\"name\":\"$slug\",\"slug\":\"$slug\",\"status\":\"active\",\"code\":$CODE}"
done
```

---

## Ce que vous ne devez PAS faire

- Lancer `app.listen()` ou un worker infini dans le script
- Mettre `gfk_` / `gfc_` dans l’APK
- Compter sur un systemd par script (tout passe par BullMQ dans `gloowflix-api`)
- Réécrire des triggers en boucle depuis `ctx.collections.*` (les écritures script **ne re-déclenchent pas** les triggers — anti-boucle)

---

## Docs liées

- [cloud-scripts.md](cloud-scripts.md) — CRUD scripts
- [cloud-scripts-operational-guide.md](cloud-scripts-operational-guide.md) — BullMQ, présence, routes
- [devapi-automation.md](devapi-automation.md) — workflows, jobs, IA
- [cloud-files-google-upload.md](cloud-files-google-upload.md) — fichiers Google → URLs signées + automation
- [ai-cloud-service.md](ai-cloud-service.md) — vault clés + génération
