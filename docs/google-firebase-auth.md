# Auth Google & Firebase — intégration Noverfly

Guide complet pour brancher **vos propres clés** Google (inscription / login) et Firebase (push FCM) sur un site Noverfly.

Base URL : `https://api.noverfly.com`

> Noverfly **n’héberge pas** votre projet Firebase.  
> Chaque site / tenant apporte ses clés (**BYOK**). Les utilisateurs finaux sont stockés dans Noverfly (`SiteUser`).

---

## Vue d’ensemble

| Besoin | Où créer les clés | Où les envoyer à Noverfly | Qui appelle |
|--------|-------------------|---------------------------|-------------|
| Login / inscription Google | [Google Cloud Console](https://console.cloud.google.com/) → OAuth Web Client | `POST /v1/api/data/auth/config/google/credentials` | `gfk_` ADMIN (recommandé) |
| Push FCM Android / iOS | [Firebase Console](https://console.firebase.google.com/) → Service account | `PUT /v1/cloud/push/config/fcm` | `gfk_` ADMIN |
| Login mobile déjà authentifié Google | SDK Google / Firebase Auth **chez vous** | `POST /v1/app/:projectId/auth/oauth/google` | App (sans clé secrète) |

```
┌─────────────────┐     OAuth Client ID/Secret      ┌──────────────────┐
│ Google Cloud    │ ─────────────────────────────►  │ Noverfly site    │
│ (votre projet)  │                                 │ auth/providers   │
└─────────────────┘                                 └────────┬─────────┘
                                                             │
┌─────────────────┐     Service Account JSON         ┌───────▼─────────┐
│ Firebase        │ ─────────────────────────────►  │ Push FCM tenant │
│ (votre projet)  │                                 │ (chiffré)       │
└─────────────────┘                                 └─────────────────┘
```

---

## Partie A — Google Sign-In (auth utilisateurs)

### A1. Créer les clés Google (côté client)

1. Ouvrir [Google Cloud Console](https://console.cloud.google.com/).
2. Créer ou sélectionner un projet.
3. **APIs & Services** → **OAuth consent screen** → configurer (External / Internal).
4. **Credentials** → **Create credentials** → **OAuth client ID**.
5. Types utiles :
   - **Web application** pour le flux redirect Noverfly
   - **Android** / **iOS** si login natif dans l’APK
6. Pour le client **Web**, ajouter l’URI de redirection :

```text
https://api.noverfly.com/v1/app/YOUR_PROJECT_ID/auth/google/callback
```

Vous obtenez :
- `clientId` → `xxxxx.apps.googleusercontent.com`
- `clientSecret` → `GOCSPX-...`

### A2. Envoyer le fichier OAuth à Noverfly (`gfk_` ADMIN, recommandé)

Dans Google Cloud, ouvrez le client **OAuth 2.0 / Application Web**, puis
cliquez **Télécharger le JSON**. Envoyez ce fichier directement :

```bash
curl -X POST \
  "https://api.noverfly.com/v1/api/data/auth/config/google/credentials" \
  -H "X-Api-Key: gfk_VOTRE_CLE_ADMIN" \
  -F "file=@client_secret_xxxxx.apps.googleusercontent.com.json;type=application/json" \
  -F "enabled=true" \
  -F "allowSignups=true" \
  -F "appName=Streewi"
```

La `gfk_` doit :

- être de type Secret (`gfk_`, pas `gfc_`) ;
- appartenir au site concerné ;
- avoir la permission `ADMIN`.

Il ne faut **pas** ajouter `NOVERFLY_DASHBOARD_JWT` pour cette route.
Une `401` sur `/v1/projects/:projectId/auth/providers` signifie généralement que
vous avez appelé la route dashboard JWT au lieu de la route DevAPI ci-dessus.

Noverfly :

- vérifie qu’il s’agit d’un client OAuth **Web** ;
- refuse `google-services.json`, un client Desktop/Installed et un service
  account Firebase ;
- chiffre `client_secret` en AES-256-GCM avant stockage ;
- ne renvoie jamais le secret dans les réponses ;
- indique si l’URI de callback manque dans le JSON téléchargé.

Le fichier doit contenir une section `web` :

```json
{
  "web": {
    "client_id": "xxxxx.apps.googleusercontent.com",
    "client_secret": "GOCSPX-xxxxx",
    "redirect_uris": [
      "https://api.noverfly.com/v1/app/YOUR_PROJECT_ID/auth/google/callback"
    ],
    "javascript_origins": [
      "https://votre-site.noverfly.com"
    ]
  }
}
```

Vérifier sans exposer le secret :

```bash
curl "https://api.noverfly.com/v1/api/data/auth/config" \
  -H "X-Api-Key: gfk_VOTRE_CLE_ADMIN"
```

La réponse contient `clientIdConfigured` et `clientSecretConfigured`, jamais
la valeur de `clientSecret`.

Configuration JSON directe (utile en CI, moins recommandée qu’un secret
manager) :

```bash
curl -X PUT \
  "https://api.noverfly.com/v1/api/data/auth/config/google" \
  -H "X-Api-Key: gfk_VOTRE_CLE_ADMIN" \
  -H "Content-Type: application/json" \
  -d '{
    "enabled": true,
    "clientId": "xxxxx.apps.googleusercontent.com",
    "clientSecret": "GOCSPX-xxxxx",
    "callbackUrl": "https://api.noverfly.com/v1/app/YOUR_PROJECT_ID/auth/google/callback",
    "allowSignups": true
  }'
```

### A2 bis. Route dashboard JWT

Auth : **JWT dashboard** (compte propriétaire / admin du site).

```bash
curl -X POST "https://api.noverfly.com/v1/projects/YOUR_PROJECT_ID/auth/providers" \
  -H "Authorization: Bearer YOUR_DASHBOARD_JWT" \
  -H "Content-Type: application/json" \
  -d '{
    "provider": "google",
    "enabled": true,
    "clientId": "xxxxx.apps.googleusercontent.com",
    "clientSecret": "GOCSPX-xxxxx",
    "callbackUrl": "https://api.noverfly.com/v1/app/YOUR_PROJECT_ID/auth/google/callback",
    "allowSignups": true
  }'
```

Mettre à jour :

```bash
curl -X PATCH "https://api.noverfly.com/v1/projects/YOUR_PROJECT_ID/auth/providers/google" \
  -H "Authorization: Bearer YOUR_DASHBOARD_JWT" \
  -H "Content-Type: application/json" \
  -d '{ "enabled": true, "clientSecret": "GOCSPX-nouveau" }'
```

Lister / désactiver :

```bash
curl "https://api.noverfly.com/v1/projects/YOUR_PROJECT_ID/auth/providers" \
  -H "Authorization: Bearer YOUR_DASHBOARD_JWT"

curl -X DELETE "https://api.noverfly.com/v1/projects/YOUR_PROJECT_ID/auth/providers/google" \
  -H "Authorization: Bearer YOUR_DASHBOARD_JWT"
```

### A3. Utiliser — flux Web (redirect)

1. L’utilisateur clique « Continuer avec Google ».
2. L’app ouvre :

```text
GET https://api.noverfly.com/v1/app/YOUR_PROJECT_ID/auth/google
```

3. Google authentifie → callback Noverfly :

```text
GET https://api.noverfly.com/v1/app/YOUR_PROJECT_ID/auth/google/callback?code=...&state=...
```

4. Noverfly crée / retrouve le `SiteUser`, émet `accessToken` + `refreshToken`.
5. Redirection vers votre front :

```text
https://VOTRE_DOMAINE/auth/callback?accessToken=...&refreshToken=...&sessionId=...
```

Erreurs possibles (query `?error=`) : `google_denied`, `invalid_state`, `google_not_configured`, `email_not_verified`, `account_suspended`.

### A4. Utiliser — flux Mobile (Google / Firebase Auth chez vous)

L’APK utilise le SDK Google Sign-In ou Firebase Auth **avec vos clés Firebase**.  
Après succès côté Google, l’app appelle Noverfly pour créer la session site :

```bash
curl -X POST "https://api.noverfly.com/v1/app/YOUR_PROJECT_ID/auth/oauth/google" \
  -H "Content-Type: application/json" \
  -d '{
    "idToken": "TOKEN_ID_SIGNE_RETOURNE_PAR_GOOGLE"
  }'
```

Noverfly vérifie auprès de Google la signature/validité, l’expiration,
l’émetteur, l’email vérifié et surtout `aud === clientId` du site. Un profil
fourni directement par le client n’est jamais accepté comme preuve d’identité.

Réponse typique :

```json
{
  "accessToken": "eyJ...",
  "refreshToken": "...",
  "sessionId": "...",
  "user": {
    "id": "...",
    "email": "user@gmail.com",
    "provider": "google"
  }
}
```

Ensuite, les appels authentifiés utilisateur utilisent :

```http
Authorization: Bearer ACCESS_TOKEN
```

ex. `GET /v1/app/YOUR_PROJECT_ID/auth/me`

### A5. Ce que Noverfly fait avec l’identité Google vérifiée

1. Vérifie le `idToken` et extrait `sub`, `email`, `name`, `picture`.
2. Cherche un utilisateur déjà lié (`googleId = sub`).
3. Sinon, rattache un compte existant même email.
4. Sinon, crée un compte (si `allowSignups: true`).
5. Émet les tokens de session du **site** (pas un token Google).

---

## Partie B — Firebase pour les push (FCM)

Firebase sert ici à **envoyer des notifications**, pas à stocker vos users Noverfly.

### B1. Créer le service account Firebase

1. [Firebase Console](https://console.firebase.google.com/) → votre projet.
2. Paramètres du projet → **Comptes de service**.
3. **Générer une nouvelle clé privée** → fichier JSON.

Exemple de champs (ne jamais committer ce fichier) :

```json
{
  "type": "service_account",
  "project_id": "mon-app-123",
  "private_key_id": "...",
  "private_key": "-----BEGIN PRIVATE KEY-----\n...\n-----END PRIVATE KEY-----\n",
  "client_email": "firebase-adminsdk@mon-app-123.iam.gserviceaccount.com"
}
```

### B2. Envoyer le JSON à Noverfly

Auth requise : une clé secrète `gfk_` avec permission **ADMIN**. Une `gfc_`
ou une clé READ/READ_WRITE ne peut pas remplacer les credentials push.

```bash
curl -X PUT "https://api.noverfly.com/v1/cloud/push/config/fcm" \
  -H "X-Api-Key: gfk_YOUR_SECRET_KEY" \
  -H "Content-Type: application/json" \
  --data @firebase-service-account.json
```

Formats acceptés :
- JSON Firebase à la racine
- `{ "serviceAccount": { ... } }`
- `{ "serviceAccountB64": "..." }`

Valider :

```bash
curl -X POST "https://api.noverfly.com/v1/cloud/push/config/fcm/validate" \
  -H "X-Api-Key: gfk_YOUR_SECRET_KEY" \
  -H "Content-Type: application/json" \
  -d '{}'
```

Résumé (sans secrets) :

```bash
curl "https://api.noverfly.com/v1/cloud/push/config" \
  -H "X-Api-Key: gfk_YOUR_SECRET_KEY"
```

Le JSON est **chiffré** et stocké par tenant (`tenant_push_configs`). Voir aussi [push-fcm-cloud.md](push-fcm-cloud.md).

### B3. Enregistrer un device (token FCM)

Côté app, après `FirebaseMessaging.getToken()` :

```bash
curl -X POST "https://api.noverfly.com/v1/cloud/push/devices/register" \
  -H "X-Api-Key: gfk_YOUR_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "userId": "SITE_USER_UUID",
    "platform": "android",
    "fcmToken": "fcm-token-from-firebase",
    "packageName": "com.example.app"
  }'
```

Aligner `firebaseProjectId` / package avec la config FCM du tenant ([client-apps.md](client-apps.md)).

---

## Checklist intégrateur

### Google Auth
- [ ] Projet Google Cloud + écran de consentement
- [ ] OAuth Client Web + redirect URI Noverfly
- [ ] Clés envoyées via `POST .../auth/providers` (`provider: google`)
- [ ] `enabled: true`, `allowSignups` selon le besoin
- [ ] Test navigateur : `/v1/app/PROJECT_ID/auth/google`
- [ ] Test mobile : `POST .../auth/oauth/google` avec profil

### Firebase Push
- [ ] Projet Firebase + app Android/iOS
- [ ] Service account JSON téléchargé
- [ ] `PUT /v1/cloud/push/config/fcm`
- [ ] `POST .../fcm/validate` OK
- [ ] Device enregistré avec token FCM
- [ ] Test push (chat / incoming call selon [push-fcm-cloud.md](push-fcm-cloud.md))

---

## Sécurité

| À faire | À ne pas faire |
|---------|----------------|
| Garder `clientSecret` et service account côté serveur / dashboard | Mettre `clientSecret` ou JSON Firebase dans l’APK |
| Utiliser JWT dashboard pour configurer Google | Exposer `gfk_` dans le front public |
| Tourner les clés si fuite | Committer les JSON dans Git |
| Un projet Google/Firebase **par client / marque** si isolation forte | Partager un seul Firebase pour tous les sites clients |

---

## Erreurs fréquentes

| Symptôme | Cause | Action |
|----------|--------|--------|
| `Google OAuth not configured` | Pas de provider ou `enabled: false` | `POST/PATCH .../auth/providers` |
| `redirect_uri_mismatch` | URI Google ≠ callback Noverfly | Ajouter l’URI exacte dans Google Cloud |
| `email_not_verified` | Compte Google sans email vérifié | Demander un autre compte |
| FCM validate échoue | Mauvais JSON / projet | Re-uploader le service account |
| Push OK auth KO | Firebase ≠ Google OAuth Client | Configurer les **deux** : providers + FCM |

---

## Routes récap

### Config (admin site)
| Méthode | Route | Auth |
|---------|-------|------|
| GET | `/v1/projects/:projectId/auth/providers` | JWT |
| POST | `/v1/projects/:projectId/auth/providers` | JWT |
| PATCH | `/v1/projects/:projectId/auth/providers/:provider` | JWT |
| DELETE | `/v1/projects/:projectId/auth/providers/:provider` | JWT |

### Login Google (app)
| Méthode | Route | Auth |
|---------|-------|------|
| GET | `/v1/app/:projectId/auth/google` | public (redirect) |
| GET | `/v1/app/:projectId/auth/google/callback` | public (redirect) |
| POST | `/v1/app/:projectId/auth/oauth/google` | public (`idToken` Google vérifié) |
| GET | `/v1/app/:projectId/auth/me` | Bearer app-user |

### Firebase Push
| Méthode | Route | Auth |
|---------|-------|------|
| PUT | `/v1/cloud/push/config/fcm` | `gfk_` ADMIN |
| POST | `/v1/cloud/push/config/fcm/validate` | `gfk_` ADMIN |
| GET | `/v1/cloud/push/config` | `gfk_` / `gfc_` |

---

## Docs liées

- [authentication.md](authentication.md) — JWT, `gfk_`, auth site
- [push-fcm-cloud.md](push-fcm-cloud.md) — détail FCM, appels entrants
- [client-apps.md](client-apps.md) — enregistrement devices / package
- [notifications-guide.md](notifications-guide.md) — push + realtime
- [security.md](security.md) — bonnes pratiques clés
