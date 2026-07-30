# Auth Google & Firebase — intégration Noverfly

Guide complet pour brancher **vos propres clés** Google (inscription / login) et Firebase (push FCM) sur un site Noverfly.

Base URL : `https://api.noverfly.com`

> Noverfly **n’héberge pas** votre projet Firebase.  
> Chaque site / tenant apporte ses clés (**BYOK**). Les utilisateurs finaux sont stockés dans Noverfly (`SiteUser`).

---

## Vue d’ensemble

| Besoin | Où créer les clés | Où les envoyer à Noverfly | Qui appelle |
|--------|-------------------|---------------------------|-------------|
| Login / inscription Google | [Google Cloud Console](https://console.cloud.google.com/) → OAuth Client | `POST /v1/sites/:siteId/auth/providers` | Dashboard JWT |
| Push FCM Android / iOS | [Firebase Console](https://console.firebase.google.com/) → Service account | `PUT /v1/cloud/push/config/fcm` | `gfk_` ou `gfc_` |
| Login mobile déjà authentifié Google | SDK Google / Firebase Auth **chez vous** | `POST /v1/app/:siteId/auth/oauth/google` | App (sans clé secrète) |

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
https://api.noverfly.com/v1/app/YOUR_SITE_ID/auth/google/callback
```

Vous obtenez :
- `clientId` → `xxxxx.apps.googleusercontent.com`
- `clientSecret` → `GOCSPX-...`

### A2. Envoyer les clés à Noverfly (dashboard)

Auth : **JWT dashboard** (compte propriétaire / admin du site).

```bash
curl -X POST "https://api.noverfly.com/v1/sites/YOUR_SITE_ID/auth/providers" \
  -H "Authorization: Bearer YOUR_DASHBOARD_JWT" \
  -H "Content-Type: application/json" \
  -d '{
    "provider": "google",
    "enabled": true,
    "clientId": "xxxxx.apps.googleusercontent.com",
    "clientSecret": "GOCSPX-xxxxx",
    "callbackUrl": "https://api.noverfly.com/v1/app/YOUR_SITE_ID/auth/google/callback",
    "allowSignups": true
  }'
```

Mettre à jour :

```bash
curl -X PATCH "https://api.noverfly.com/v1/sites/YOUR_SITE_ID/auth/providers/google" \
  -H "Authorization: Bearer YOUR_DASHBOARD_JWT" \
  -H "Content-Type: application/json" \
  -d '{ "enabled": true, "clientSecret": "GOCSPX-nouveau" }'
```

Lister / désactiver :

```bash
curl "https://api.noverfly.com/v1/sites/YOUR_SITE_ID/auth/providers" \
  -H "Authorization: Bearer YOUR_DASHBOARD_JWT"

curl -X DELETE "https://api.noverfly.com/v1/sites/YOUR_SITE_ID/auth/providers/google" \
  -H "Authorization: Bearer YOUR_DASHBOARD_JWT"
```

### A3. Utiliser — flux Web (redirect)

1. L’utilisateur clique « Continuer avec Google ».
2. L’app ouvre :

```text
GET https://api.noverfly.com/v1/app/YOUR_SITE_ID/auth/google
```

3. Google authentifie → callback Noverfly :

```text
GET https://api.noverfly.com/v1/app/YOUR_SITE_ID/auth/google/callback?code=...&state=...
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
curl -X POST "https://api.noverfly.com/v1/app/YOUR_SITE_ID/auth/oauth/google" \
  -H "Content-Type: application/json" \
  -d '{
    "profile": {
      "googleId": "1234567890",
      "email": "user@gmail.com",
      "displayName": "Ada Lovelace",
      "avatarUrl": "https://lh3.googleusercontent.com/..."
    }
  }'
```

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

ex. `GET /v1/app/YOUR_SITE_ID/auth/me`

### A5. Ce que Noverfly fait avec le profil Google

1. Cherche un utilisateur déjà lié (`googleId`).
2. Sinon, rattache un compte existant même email.
3. Sinon, crée un compte (si `allowSignups: true`).
4. Émet les tokens de session du **site** (pas un token Google).

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
- [ ] Test navigateur : `/v1/app/SITE_ID/auth/google`
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
| GET | `/v1/sites/:siteId/auth/providers` | JWT |
| POST | `/v1/sites/:siteId/auth/providers` | JWT |
| PATCH | `/v1/sites/:siteId/auth/providers/:provider` | JWT |
| DELETE | `/v1/sites/:siteId/auth/providers/:provider` | JWT |

### Login Google (app)
| Méthode | Route | Auth |
|---------|-------|------|
| GET | `/v1/app/:siteId/auth/google` | public (redirect) |
| GET | `/v1/app/:siteId/auth/google/callback` | public (redirect) |
| POST | `/v1/app/:siteId/auth/oauth/google` | public (profil Google) |
| GET | `/v1/app/:siteId/auth/me` | Bearer site-user |

### Firebase Push
| Méthode | Route | Auth |
|---------|-------|------|
| PUT | `/v1/cloud/push/config/fcm` | `gfk_` / `gfc_` |
| POST | `/v1/cloud/push/config/fcm/validate` | `gfk_` / `gfc_` |
| GET | `/v1/cloud/push/config` | `gfk_` / `gfc_` |

---

## Docs liées

- [authentication.md](authentication.md) — JWT, `gfk_`, auth site
- [push-fcm-cloud.md](push-fcm-cloud.md) — détail FCM, appels entrants
- [client-apps.md](client-apps.md) — enregistrement devices / package
- [notifications-guide.md](notifications-guide.md) — push + realtime
- [security.md](security.md) — bonnes pratiques clés
