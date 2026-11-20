# Notes séance 1 — TP guidé

## Étape 1 — Anatomie d'un échange

Requête : `GET /repos/nodejs/node HTTP/2`
Statut : `HTTP/2 200`

Version utilisée : **HTTP/2**.

Quatre en-têtes :

| En-tête                                         | Rôle                                            |
| ----------------------------------------------- | ----------------------------------------------- |
| `content-type: application/json; charset=utf-8` | Le corps est du JSON.                           |
| `etag: W/"a4b8…"`                               | Identifiant de version, sert au cache.          |
| `cache-control: public, max-age=60`             | La réponse peut être mise en cache 60 secondes. |
| `x-ratelimit-remaining: 59`                     | Nombre de requêtes encore autorisées.           |

`x-ratelimit-remaining` = requêtes restantes sur le quota (60/h sans token). À 0, GitHub refuse avec **403** (rate limit exceeded).

## Étape 2 — Cache conditionnel

Statut obtenu : **304 Not Modified**.

Le corps est vide parce que l'ETag envoyé dans `If-None-Match` est le même : rien n'a changé, le client garde sa copie.

- Mobile : on ne retélécharge pas le JSON, donc moins de data.
- Quota GitHub : un 304 évite de renvoyer tout le corps. La doc GitHub indique qu'un 304 ne compte pas dans la limite principale.

## Étape 3 — Les erreurs

| API             | Requête                                   | Statut | Corps                                         | Utile pour corriger                       |
| --------------- | ----------------------------------------- | ------ | --------------------------------------------- | ----------------------------------------- |
| GitHub          | `GET /repos/nodejs/ce-depot-n-existe-pas` | 404    | JSON `message`, `documentation_url`, `status` | le message + le lien vers la doc          |
| GitHub          | `DELETE /repos/nodejs/node`               | 401    | JSON `message`: "Requires authentication"     | il faut un token                          |
| geo.api.gouv.fr | `GET /communes?codePostal=abc`            | 200    | `[]`                                          | rien : on dirait juste « aucun résultat » |

Pas de format commun. Le JSON GitHub est le plus utile (message clair + doc). geo renvoie une liste vide.

L'erreur qui ne respecte pas la sémantique HTTP : **geo.api.gouv.fr**. `codePostal=abc` est invalide, le serveur devrait répondre **400**, pas **200**. Un 200 veut dire que la requête a réussi.

Réécriture RFC 9457 du 404 GitHub :

```json
{
  "type": "https://api.github.com/problems/not-found",
  "title": "Ressource introuvable",
  "status": 404,
  "detail": "Le dépôt nodejs/ce-depot-n-existe-pas n'existe pas."
}
```

## Étape 4 — Collections et pagination

1. geo.api.gouv.fr filtre avec des paramètres d'URL : `?nom=Lyon`. Les champs renvoyés se choisissent avec `fields=nom,code,population`.
2. Chez GitHub, la page suivante est dans l'en-tête **`Link`**.
3. Avantage : le corps ne contient que les données. Le client suit le lien sans deviner l'URL de la page suivante.

## Étape 5 — Négociation de contenu

C'est l'en-tête **`Accept`** qui choisit le format.

- `Accept: application/vnd.github.raw+json` → Markdown brut (`content-type: application/vnd.github.raw+json`)
- `Accept: application/vnd.github.html+json` → HTML (`content-type: application/vnd.github.html+json`)

Si aucun format demandé n'existe, le serveur doit répondre **406 Not Acceptable**.

## Étape 6 — Échelle de Richardson

| API             | Niveau            | Exemple observé                                                                                                                                             |
| --------------- | ----------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------- |
| GitHub REST     | **2** (presque 3) | `GET /repos/nodejs/node` → 200, `DELETE` sans token → 401. L'en-tête `Link; rel="next"` est un lien hypermédia (début de niveau 3).                         |
| geo.api.gouv.fr | **2**             | `GET /communes?nom=Lyon` → 200 JSON. Plusieurs URI, verbe GET, mais pas de liens hypermédias. Le 200 sur `codePostal=abc` est un mauvais usage des statuts. |
| JSONPlaceholder | **2**             | `GET /posts/1` → 200 JSON, `GET /posts/999999` → 404. Ressources + verbes + statuts, pas de lien `Link`.                                                    |
