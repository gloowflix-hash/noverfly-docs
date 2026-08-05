# NoverFly Translate

Produit de traduction IA multi-projets de la plateforme **Gloowflix / NoverFly** (comme Calls ou Effects) — pas toute la plateforme.

Prix V1 : **40 USD / mois** (`translate_api_40`, billing `manual`).

Base URL : `https://api.noverfly.com`

## Architecture

```text
Clients DevAPI (nft_test_ / nft_live_)
Streewi (JWT site_user → /api/noverfly/translate/*)
Portail NoverFly (JWT → /v1/translate/*)
        │
        ▼
GLOOWFLIX/V1  (API, Prisma, quotas, cache, BullMQ)
        │
        ▼
Runpod Serverless  (endpoint texte)
```

| Composant | Rôle |
|---|---|
| GLOOWFLIX/V1 | Backend central (routes, abonnements, clés, quotas, client Runpod) |
| NOVERFLY/V1 | Portail commercial / dashboard / admin UI |
| Streewi | App cliente — jamais de clé Runpod / `nft_live_` permanente dans le bundle |
| Runpod | Exécution GPU/CPU des modèles |
| Flivex | Moteur média existant — hors scope métier Translate |

## Types de clés

| Clé | Usage |
|---|---|
| `gfk_` | DevAPI : activer Translate, projets, émettre des `nft_` |
| `nft_test_` / `nft_live_` | Runtime : detect, text, batch, usage, media (si flag) |
| JWT dashboard | Portail `/v1/translate/*` |
| JWT `site_user` | Streewi `/api/noverfly/translate/*` |
| `RUNPOD_API_KEY` | Serveur Gloowflix uniquement — jamais frontend |
| `NOVERFLY_TRANSLATE_ADMIN_SECRET` | Optionnel serveur — ne remplace pas `isGloowAdmin` |

## Activation (GFK)

```bash
curl -X POST https://api.noverfly.com/v1/cloud/translate/activate \
  -H "x-gfk-key: gfk_xxxxxxxx" \
  -H "Content-Type: application/json" \
  -d '{"environment":"PRODUCTION","name":"Streewi","slug":"streewi"}'
```

La réponse contient `api_key` **une seule fois** (`nft_live_…` / `nft_test_…`). Seul le hash HMAC est stocké.

Prérequis : plan tenant ≥ 40 USD/mois.

## Runtime (NFT)

| Méthode | Route | Scope |
|---|---|---|
| `POST` | `/api/v1/translate/detect` | `translate:text` |
| `POST` | `/api/v1/translate/text` | `translate:text` |
| `POST` | `/api/v1/translate/batch` | `translate:batch` |
| `GET` | `/api/v1/translate/languages` | auth NFT |
| `GET` | `/api/v1/translate/usage` | `usage:read` |
| `POST` | `/api/v1/translate/media/jobs` | `translate:media` (flag off en V1) |
| `GET` | `/api/v1/translate/health` | public |

Exemple :

```bash
curl -X POST https://api.noverfly.com/api/v1/translate/text \
  -H "Authorization: Bearer nft_test_xxxxxxxx" \
  -H "Content-Type: application/json" \
  -d '{"text":"Bonjour","source_language":"fr","target_language":"en","context":"private_chat"}'
```

## Streewi

```text
JWT site_user → POST /api/noverfly/translate/text
```

Ne jamais embarquer `nft_`, `gfk_`, `RUNPOD_*` ou secret admin dans l’APK / le bundle Vite.

## Langues V1

Actives (après validation modèle) : `fr`, `en`, `es`, `pt`, `ar`, `sw` (+ paires documentées).

Expérimentales (avertissement qualité) : `ln`, `sw-CD`, `lua`, `kg`/`kt`.

Une langue catalogue ≠ paire opérationnelle. Les paires `disabled` / `planned` sont refusées.

## Feature flags

```env
NOVERFLY_TRANSLATE_ENABLED=true
NOVERFLY_TRANSLATE_TEXT_ENABLED=true
NOVERFLY_TRANSLATE_MEDIA_ENABLED=false
NOVERFLY_TRANSLATE_OCR_ENABLED=false
NOVERFLY_TRANSLATE_LIVE_ENABLED=false
```

## Runpod (ops)

Endpoint texte Serverless Flex recommandé :

- `workersMin=0`, `workersMax=1`, concurrency 1
- Variable Gloowflix : `RUNPOD_TRANSLATE_ENDPOINT_ID`
- Worker source : `GLOOWFLIX/V1/services/translate-worker/`
- Création : `python scripts/create-translate-runpod-endpoint.py`
- Image custom + `MODEL_PATH` requis pour une traduction réelle
- Sans modèle : erreur propre `TRANSLATE_MODEL_UNAVAILABLE` / provider — **jamais** de fausse traduction

Santé :

```bash
curl https://api.noverfly.com/ready
curl https://api.noverfly.com/api/v1/translate/health
```

## Dashboard JWT

| Méthode | Route |
|---|---|
| `POST` | `/v1/translate/activate` |
| `GET` | `/v1/translate/status` |
| `GET` | `/v1/translate/projects` |
| `GET/POST` | `/v1/translate/keys` |
| `GET` | `/v1/translate/usage` |

Portail : `https://noverfly.com/app/translate/dashboard`

## Admin

JWT administrateur + `requireGloowAdmin` → `/v1/admin/translate/*`

## Compatibilité

`POST /v1/global/translate` délègue au moteur Translate unique (pas de second pipeline).

## Erreurs stables

| Code | Sens |
|---|---|
| `TRANSLATE_PROVIDER_UNAVAILABLE` | Endpoint / clé Runpod absents ou circuit ouvert |
| `TRANSLATE_MODEL_UNAVAILABLE` | Modèle non chargé / licence non commerciale |
| `TRANSLATE_SUBSCRIPTION_SUSPENDED` | Abonnement suspendu |
| `TRANSLATE_LANGUAGE_NOT_ALLOWED` | Langue hors ACL projet |
| `TRANSLATE_QUOTA_EXCEEDED` | Quota période dépassé |

## Sécurité

- Cache isolé par `tenant_id` + `project_id` + hash texte + modèle
- Contenu chat privé : rétention minimale (`private_chat`)
- Idempotence facturable : `project_id` + `operation_type` + `idempotency_key`
- Streewi / clients ne contactent **jamais** Runpod directement
