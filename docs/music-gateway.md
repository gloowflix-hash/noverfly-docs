# NoverFly Music API Gateway

## Objectif
Le `Music API Gateway` permet aux applications et sites crees sur NoverFly d'interroger une API audio professionnelle sans jamais exposer la cle fournisseur cote frontend. Les developpeurs externes utilisent uniquement leur cle `GFK` NoverFly.

Sources actuellement connectables:
- `freesound` pour effets, ambiances et loops
- `jamendo` pour catalogue musique

Architecture:

`Client app/site` -> `x-gfk-key` + `x-project-id` -> `NoverFly DevAPI` -> `auth GFK` -> `verification abonnement` -> `quota/rate limit` -> `cache Redis/PostgreSQL` -> `Freesound/Jamendo cote serveur`

## Prerequis
- Un projet NoverFly
- Une cle `GFK` active liee a ce projet
- Un abonnement actif a partir de `20 USD/mois`
- L'option Music API activee pour le projet

## Generer une cle GFK
Depuis le dashboard createur:

1. Ouvrir `Projet -> Music Gateway`
2. Generer une cle GFK dediee ou reutiliser une cle `SECRET` existante
3. Configurer les `allowedOrigins` si vous limitez les domaines autorises
4. Conserver la cle cote backend ou dans un environnement serveur securise si votre application a un backend

Endpoint dashboard:

```http
POST /v1/projects/:projectId/music-gateway/dashboard/keys
Authorization: Bearer <dashboard_jwt>
```

## En-tetes obligatoires
Chaque appel doit envoyer:

```http
x-gfk-key: GFK_PUBLIC_KEY
x-project-id: PROJECT_ID
```

## Endpoints DevAPI

### Recherche

```http
GET /api/dev/music/search?q=cinematic&type=effects&durationMax=60&page=1
GET /v1/music/search?q=piano
GET /v1/music/search?q=afro&source=jamendo
```

Notes:
- `pageSize` max: `50` par requete
- `page` permet de paginer dans tout le catalogue fournisseur
- `total` dans la reponse represente le nombre total de resultats cote fournisseur, pas seulement la taille de la page courante

Exemple:

```js
fetch("https://api.noverfly.com/api/dev/music/search?q=afro&source=jamendo", {
  headers: {
    "x-gfk-key": "GFK_PUBLIC_KEY",
    "x-project-id": "PROJECT_ID"
  }
});
```

### Trending

```http
GET /api/dev/music/trending
GET /api/dev/music/trending?source=jamendo
```

### Detail d'un son

```http
GET /api/dev/music/sound/:id
GET /api/dev/music/licenses/:id
```

### Declarer l'utilisation d'un son

```http
POST /api/dev/music/use
Content-Type: application/json

{
  "id": "12345",
  "source": "jamendo"
}
```

### Favoris

```http
POST /api/dev/music/favorite
DELETE /api/dev/music/favorite/:id
```

### Quota

```http
GET /api/dev/music/quota
```

## Reponse normalisee

```json
{
  "source": "jamendo",
  "externalId": "2166912",
  "title": "Afro Summer",
  "artist": "Janevo",
  "duration": 129,
  "previewUrl": "https://...",
  "originalUrl": "https://www.jamendo.com/track/2166912",
  "license": "Creative Commons Attribution NonCommercial NoDerivatives",
  "tags": ["african", "happy"],
  "allowedUse": false,
  "requiresAttribution": true,
  "attribution": "Music by Janevo on Jamendo",
  "provider": {
    "name": "Jamendo",
    "url": "https://www.jamendo.com"
  }
}
```

## Exemple React

```tsx
import { useEffect, useState } from "react";

type MusicItem = {
  externalId: string;
  title: string;
  artist: string | null;
  previewUrl: string | null;
  license: string | null;
};

export function MusicSearch() {
  const [items, setItems] = useState<MusicItem[]>([]);

  useEffect(() => {
    fetch("https://api.noverfly.com/v1/music/search?q=afro&source=jamendo", {
      headers: {
        "x-gfk-key": "GFK_PUBLIC_KEY",
        "x-project-id": "PROJECT_ID"
      }
    })
      .then((res) => res.json())
      .then((data) => setItems(data.items || []));
  }, []);

  return (
    <ul>
      {items.map((item) => (
        <li key={item.externalId}>
          {item.title} - {item.artist}
        </li>
      ))}
    </ul>
  );
}
```

## Exemple JavaScript simple

```js
async function searchMusic(query, source = "jamendo") {
  const res = await fetch(
    `https://api.noverfly.com/api/dev/music/search?q=${encodeURIComponent(query)}&source=${encodeURIComponent(source)}`,
    {
      headers: {
        "x-gfk-key": "GFK_PUBLIC_KEY",
        "x-project-id": "PROJECT_ID"
      }
    }
  );

  if (!res.ok) {
    throw await res.json();
  }

  return res.json();
}
```

## Erreurs possibles
- `MUSIC_API_DISABLED`
- `INVALID_GFK_KEY`
- `PROJECT_REQUIRED`
- `PROJECT_NOT_ALLOWED`
- `SUBSCRIPTION_REQUIRED`
- `PLAN_TOO_LOW`
- `MUSIC_QUOTA_EXCEEDED`
- `FREESOUND_PROVIDER_ERROR`
- `JAMENDO_PROVIDER_ERROR`
- `SOUND_NOT_FOUND`
- `LICENSE_NOT_ALLOWED`
- `RATE_LIMITED`

## Quotas et rate limit
- Plan `20 USD+` a `Pro`: `500` appels/jour par defaut
- Plan `Enterprise`: `5000` appels/jour par defaut
- Rate limit standard: `60 requetes/minute` par cle + IP
- Enterprise: `300 requetes/minute`
- Les abus repetes declenchent un blocage temporaire

## Attribution et licence
- `CC0`: generalement autorise sans attribution
- `CC Attribution`: attribution requise
- `CC Attribution NonCommercial`: bloque par defaut pour les usages commerciaux via NoverFly
- Les conditions Freesound, Jamendo et les licences des assets s'appliquent en plus des regles NoverFly

## Securite
- La cle Freesound reste exclusivement dans le backend NoverFly
- La configuration Jamendo reste exclusivement dans le backend NoverFly
- Aucune reponse DevAPI ne retourne la cle externe
- Seules les URLs `preview` autorisees et les metadonnees utiles sont renvoyees
- La validation d'origine, le quota et le rate limit s'executent avant l'appel fournisseur

## Variables d'environnement backend

```env
FREESOUND_API_KEY=
JAMENDO_CLIENT_ID=
JAMENDO_CLIENT_SECRET=
```

`JAMENDO_CLIENT_SECRET` n'est pas necessaire pour la recherche publique actuelle, mais il doit rester cote serveur si vous l'utilisez plus tard pour OAuth ou des appels prives.

## Pourquoi la cle fournisseur n'est jamais visible
Le navigateur ou l'application du client appelle uniquement NoverFly DevAPI. Le serveur NoverFly effectue ensuite l'appel Freesound ou Jamendo avec sa configuration privee stockee en `.env`. Le frontend ne recoit jamais ce secret, ce qui permet d'ajouter plus tard `stock interne`, `user generated` ou `AI music` sans changer l'API publique cote client.
