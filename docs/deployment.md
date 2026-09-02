# Publication et déploiement Noverfly

Noverfly propose deux mécanismes distincts :

1. **Publication Site Builder** : pages/CMS enregistrés en base, publiés sous
   forme de `SiteRelease` avec un JWT dashboard.
2. **Noverfly Hosting** : artefacts HTML/React/Vite/Next export stockés sur S3,
   déployés avec une clé `gfk_` ADMIN.

Ne mélangez pas leurs endpoints ou leurs types de release.

## Publication Site Builder

Depuis GlowDesign, le bouton **Publier** crée un snapshot cohérent des pages
publiées et passe le site en `LIVE`.

```bash
curl -X POST "https://api.noverfly.com/v1/sites/SITE_ID/publish" \
  -H "Authorization: Bearer DASHBOARD_JWT"
```

Réponse :

```json
{
  "release": {
    "id": "uuid",
    "siteId": "uuid",
    "version": 4,
    "status": "LIVE",
    "snapshot": {}
  }
}
```

Historique réel :

```bash
curl "https://api.noverfly.com/v1/sites/SITE_ID/releases" \
  -H "Authorization: Bearer DASHBOARD_JWT"
```

Il n’existe pas de route
`GET /v1/sites/:siteId/deploys/:deployId` pour le Site Builder.

## Déploiement React/Vite/Next export

Pour publier un dossier compilé (`dist` ou `out`) :

```bash
npm run hosting:deploy -- \
  --dir dist \
  --framework vite \
  --key gfk_VOTRE_CLE_ADMIN
```

Cette commande crée un `HostingDeployment`, transfère directement les objets
vers S3 avec des URL signées, vérifie la taille et SHA-256, puis active la
release atomiquement.

Next.js doit être configuré avec `output: "export"` :

```bash
npm run hosting:deploy -- \
  --dir out \
  --framework next-export \
  --key gfk_VOTRE_CLE_ADMIN
```

Voir [Noverfly Hosting](hosting-api.md) pour le protocole API complet, les
quotas, le rollback et l’intégration CI/CD.

## Domaines

Chaque site reçoit automatiquement :

```text
https://SITE_SLUG.noverfly.com
```

Le wildcard DNS/TLS `*.noverfly.com` est géré par Noverfly et Cloudflare.

Pour un domaine personnalisé :

1. attachez le domaine au site depuis le dashboard ;
2. utilisez les enregistrements renvoyés par l’assistant DNS Noverfly ;
3. ne copiez pas une IP ou une cible CNAME depuis un ancien guide.

```bash
curl -X POST "https://api.noverfly.com/v1/sites/SITE_ID/domains" \
  -H "Authorization: Bearer DASHBOARD_JWT" \
  -H "Content-Type: application/json" \
  -d '{"domain":"www.example.com","isPrimary":true}'
```

Les domaines achetés par Noverfly utilisent aussi les routes registrar tenant :

```text
GET /v1/tenants/:tenantId/domains/:domainId/dns-records
```

La réponse de cette route est la source de vérité pour le CNAME/TXT à créer.

## Livraison publique

Le resolver Site Builder reste accessible sans clé :

```bash
curl -X POST "https://api.noverfly.com/v1/public/resolve" \
  -H "Content-Type: application/json" \
  -d '{"host":"www.example.com"}'
```

Le runtime Hosting S3 est résolu automatiquement par Caddy avant le frontend
Site Builder. Lorsqu’aucune release S3 active n’existe, Noverfly conserve le
rendu historique du Site Builder.

## Rollback

### Site Builder

Les versions sont visibles via `/v1/sites/:siteId/releases`. Le dashboard peut
restaurer leur snapshot.

### Hosting S3

Réactivez une ancienne release `SUPERSEDED` :

```bash
curl -X POST \
  "https://api.noverfly.com/v1/hosting/deployments/DEPLOYMENT_ID/activate" \
  -H "X-Api-Key: gfk_VOTRE_CLE_ADMIN"
```

L’ancienne et la nouvelle release sont permutées dans une transaction.

## Sécurité CI/CD

- Conservez `gfk_` dans le coffre de secrets CI, jamais dans le bundle web.
- Utilisez une clé ADMIN dédiée au déploiement.
- Une `gfk_` est liée à un site et ne peut pas déployer un autre site.
- Le Hosting mutualisé n’exécute aucun script ou `package.json` client.
- Pour le backend, utilisez Data API, Cloud Scripts, Auth, Calls ou votre BFF.

## Vérifications

```bash
curl -I "https://SITE_SLUG.noverfly.com/"

curl "https://api.noverfly.com/v1/hosting/deployments" \
  -H "X-Api-Key: gfk_VOTRE_CLE"
```

## Voir aussi

- [Hosting API](hosting-api.md)
- [API Reference](api.md)
- [Authentication](authentication.md)
- [Security](security.md)
