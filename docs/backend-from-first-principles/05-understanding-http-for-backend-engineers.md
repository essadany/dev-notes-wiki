# 5. Comprendre HTTP pour les ingénieurs backend

> *Objectif : Maîtriser le protocole HTTP, comprendre le CORS une bonne fois pour toutes, et retenir les codes d'état essentiels pour mes futurs débugs.*

---

## 🧭 1. Les Deux Piliers Fondamentaux de HTTP

### 🐠 A. Le "Statelessness" (Protocole Sans État)
Le serveur est un poisson rouge : **il n'a aucune mémoire des interactions passées**.
* **Auto-contenue (Self-contained)** : Chaque requête doit porter TOUTES ses infos (tokens, cookies, ID).
* **Nouveau départ** : Si je fais deux requêtes d'affilée, le serveur traite la deuxième comme si on ne s'était jamais vus.

```
┌──────────┐                 Requête 1 : "Je suis l'utilisateur X, donne-moi ma page"                  ┌──────────┐
│          ├──────────────────────────────────────────────────────────────────────────────────────────>│          │
│          │                 Réponse 1 : "Tiens, voilà tes données"                                    │          │
│  CLIENT  │<──────────────────────────────────────────────────────────────────────────────────────────┤ SERVEUR  │
│ (Browser)│                                                                                           │ (NodeJS) │
│          │                 Requête 2 : "Donne-moi mes messages" (SANS préciser qui je suis)          │          │
│          ├──────────────────────────────────────────────────────────────────────────────────────────>│          │
└──────────┘                 Réponse 2 : "Qui es-tu ? Erreur 401 Unauthorized" (Oubli de session !)    └──────────┘
```

> ⚙️ **Pourquoi ce choix ?**
> * **Simplicité** : Pas besoin de stocker des sessions en RAM côté serveur.
> * **Scalabilité de dingue** : On peut distribuer les requêtes sur plein de serveurs (Load Balancing). Si un serveur crash, un autre prend le relais sans devoir restaurer de session !

💡 **Astuce de dev :** Pour que l'utilisateur reste connecté (ex: panier d'achat, login), on rajoute une couche de gestion d'état par-dessus (Cookies, Sessions ou JWT Tokens).

---

### 🔌 B. Le Modèle Client-Serveur
* **La règle d'or** : C’est *toujours* le client (navigateur, appli, Postman) qui initie l'échange. Le serveur écoute passivement sur un port ouvert et répond.

---

## 📦 2. Anatomie d'un Message HTTP & La Métaphore du Colis

Un message HTTP, c'est uniquement du texte brut structuré.

### 🚚 La Métaphore du Colis Postal
Quand j'envoie un colis :
* J'écris l'adresse et le numéro du destinataire **sur le dessus du carton** → Ce sont les **Headers** (métadonnées).
* Je mets le cadeau **à l'intérieur du carton** → C'est le **Body** (données utiles).

Le livreur (proxies, routeurs, pare-feu) regarde juste l'étiquette extérieure pour acheminer le colis sans jamais l'ouvrir. C'est ultra-rapide !

```
┌──────────────────────────────────────────────────────────────┐
│  MÉTADONNÉES (L'étiquette sur le colis)                     │
│  • Requête : METHODE / URL / VERSION                         │
│  • En-têtes (Headers) : paires Clé-Valeur                    │
├──────────────────────────────────────────────────────────────┤
│  [ LIGNE VIDE ] (Indique la fin des en-têtes)                │
├──────────────────────────────────────────────────────────────┤
│  CORPS DU MESSAGE (L'intérieur du colis)                     │
│  • Payload / Body (JSON, XML, Fichiers, HTML...)             │
└──────────────────────────────────────────────────────────────┘
```

### 🔬 Comparaison Brute : Requête vs Réponse

#### 🔼 REQUÊTE CLIENT (HTTP/1.1)
```http
GET /api/user HTTP/1.1
Host: backend.demo.xyz
Authorization: Bearer token_xyz
Accept: application/json

{ "action": "read" }
```

#### 🔽 RÉPONSE SERVEUR (HTTP/1.1)
```http
HTTP/1.1 200 OK
Content-Type: application/json
Cache-Control: max-age=10
Content-Length: 42

{ "id": 1, "username": "junior_dev" }
```

---

## 🗂️ 3. Les 4 Grandes Familles de Headers (En-têtes)

Les headers sont des couples `Clé: Valeur`. On peut les regrouper ainsi :

| Catégorie | Rôle | Headers Clés | Explication |
| :--- | :--- | :--- | :--- |
| **1. Request Headers** | Décrivent le client et ses préférences | `User-Agent`<br>`Authorization`<br>`Accept` | - Identifie le navigateur/app.<br>- Transmet le token JWT.<br>- Dit quel format on attend (ex: JSON). |
| **2. General Headers** | Métadonnées globales hors contenu | `Date`<br>`Connection` | - Heure du message.<br>- `keep-alive` pour réutiliser la connexion TCP. |
| **3. Representation Headers** | Décrivent le format et la taille du Body | `Content-Type`<br>`Content-Length`<br>`Content-Encoding`<br>`ETag` | - Format (ex: `application/json`).<br>- Taille en octets.<br>- Compression (ex: `gzip`).<br>- Hash unique pour le cache. |
| **4. Security Headers** | Protègent le client contre les attaques | `HSTS`<br>`CSP`<br>`X-Frame-Options`<br>`Set-Cookie` | - Force le HTTPS.<br>- Bloque les XSS.<br>- Bloque le clickjacking.<br>- Sécurise les cookies (HttpOnly, Secure). |

> 🎮 **Le concept de "Remote Control" :**  
> Les headers fonctionnent comme une télécommande. Le client demande un format via `Accept` et le serveur s'adapte (ex: renvoie du JSON ou de l'HTML). Inversement, le serveur contrôle le comportement du cache du navigateur grâce à `Cache-Control`.

---

## ⚡ 4. Les Méthodes HTTP & L'Idempotence

Les méthodes indiquent la sémantique et **l'intention** de l'action.

| Méthode | Action | Body ? | Idempotente ? | En gros, ça veut dire quoi ? |
| :---: | :--- | :---: | :---: | :--- |
| **GET** | Récupérer des données | Non | **OUI** | Je peux l'appeler 100 fois, l'état du serveur ne bouge pas. |
| **POST** | Créer une ressource | **OUI** | **NON** | Si je double-clique sur "Envoyer", je crée 2 ressources en BDD. |
| **PATCH** | Modif partielle | **OUI** | **OUI** | Met à jour uniquement les champs envoyés (ex: juste le pseudo). |
| **PUT** | Remplacer complètement | **OUI** | **OUI** | Écrase la ressource entière. Même après 10 PUT identiques, le résultat reste le même. |
| **DELETE**| Supprimer | Non | **OUI** | Supprime une fois. Les appels suivants ne changent rien (la ressource est déjà supprimée). |
| **OPTIONS**| Demander les droits | Non | **OUI** | Permet au navigateur de vérifier les autorisations CORS. |

⚠️ **Règle de bonne pratique :** Toujours privilégier `PATCH` pour les modifications partielles. On utilise `PUT` uniquement pour un remplacement total de la ressource.

---

## 🛡️ 5. Le Mystère du CORS Enfin Clair !

Le **CORS (Cross-Origin Resource Sharing)** est une sécurité imposée par le **navigateur**. Il bloque le JavaScript qui tente de requêter un domaine/port différent de celui du site d'origine (ex: le front sur `localhost:5173` qui appelle l'API sur `localhost:3000`).

Il y a deux scénarios de requêtes :

### 🟢 A. La Requête Simple (Simple Request)
Se produit uniquement pour `GET`, `POST` ou `HEAD` avec des en-têtes basiques.
1. Le client envoie sa requête. Le navigateur y greffe automatiquement le header `Origin: localhost:5173`.
2. Le serveur traite la requête et renvoie la réponse avec le header `Access-Control-Allow-Origin: localhost:5173` (ou `*`).
3. Le navigateur vérifie si son origine est bien listée. Si le header est absent → **CORS Error** instantanée dans la console !

### 🟡 B. La Requête Pré-vérifiée (Preflight Request)
Se déclenche si la méthode est complexe (`PUT`, `DELETE`), si on envoie du JSON (`application/json`), ou si on utilise des headers spéciaux (comme `Authorization`).

```
[ Navigateur (Client) ]                                              [ Serveur API ]
         │                                                                  │
         │ ── ① OPTIONS /api/data (Preflight) ────────────────────────────> │  <- "Salut ! J'ai le droit de faire un PUT ?"
         │    Headers:                                                      │
         │      Access-Control-Request-Method: PUT                          │
         │      Access-Control-Request-Headers: Authorization               │
         │                                                                  │
         │ <── ② Réponse 204 No Content ─────────────────────────────────── │  <- "Pas de souci ! Voici mes conditions :"
         │    Headers CORS :                                                │
         │      Access-Control-Allow-Origin: http://localhost:5173          │
         │      Access-Control-Allow-Methods: GET, POST, PUT, DELETE        │
         │      Access-Control-Allow-Headers: Content-Type, Authorization   │
         │      Access-Control-Max-Age: 86400 (Évite de refaire OPTIONS)    │
         │                                                                  │
         │ ── ③ PUT /api/data (Requête réelle) ───────────────────────────> │  <- Tout est OK, j'envoie mon vrai PUT !
         │                                                                  │
         │ <── ④ Réponse 200 OK + Données ───────────────────────────────── │
```

---

## 🚦 6. L'Antisèche des Codes d'État (Status Codes)

Les codes d'état permettent de comprendre ce qui s'est passé en un clin d'œil, sans lire le Body.

```
┌──────────────────────────────────────────────────────────────────────────┐
│                           CODES D'ÉTAT HTTP                              │
├──────────────┬──────────────┬──────────────┬──────────────┬──────────────┤
│     1xx      │     2xx      │     3xx      │     4xx      │     5xx      │
│ Informational│   Success    │ Redirection  │ Client Error │ Server Error │
└──────────────┴──────────────┴──────────────┴──────────────┴──────────────┘
```

### 🎯 Les indispensables à retenir absolument :

#### ✅ 2xx : Succès
* `200 OK` : Tout s'est bien passé.
* `201 Created` : Ressource créée avec succès (idéal après un `POST`).
* `204 No Content` : Action réussie mais pas de contenu à renvoyer (ex: requête `OPTIONS` ou un `DELETE`).

#### 🔀 3xx : Redirection
* `301 Moved Permanently` : Redirection définitive. Le navigateur va directement à la nouvelle adresse à l'avenir.
* `302 Found` : Redirection temporaire.
* `304 Not Modified` : Dit au navigateur : "Le fichier n'a pas bougé, recharge-le depuis ton cache local !" (Économise de la bande passante).

#### ❌ 4xx : Erreurs Client (C'est ma faute / celle de l'utilisateur !)
* `400 Bad Request` : Données invalides (ex: format d'email incorrect).
* `401 Unauthorized` : Pas authentifié ou token expiré.
* `403 Forbidden` : Authentifié mais pas les droits d'accès (ex: modifier le compte d'un autre utilisateur).
* `404 Not Found` : La ressource demandée n'existe pas.
* `405 Method Not Allowed` : Méthode non supportée par cette route.
* `409 Conflict` : Conflit de données (ex: créer un utilisateur avec un email déjà pris).
* `429 Too Many Requests` : Rate limiting déclenché (trop de requêtes d'un coup).

#### 💥 5xx : Erreurs Serveur (Le backend a planté !)
* `500 Internal Server Error` : Crash inattendu de l'application (exception non gérée).
* `502 Bad Gateway` : Le reverse proxy (Nginx) ne reçoit pas de réponse de l'application locale (ex: Node.js est arrêté).
* `503 Service Unavailable` : Serveur surchargé ou en cours de maintenance.
* `504 Gateway Timeout` : L'application backend a mis trop de temps à répondre à Nginx.

---

## 💾 7. Fonctionnement de la Mise en Cache HTTP

Le cache évite de re-télécharger des fichiers ou des données inchangés.

```
[ Navigateur ]                                                        [ Serveur ]
      │                                                                    │
      │ ── ① GET /resource ──────────────────────────────────────────────> │
      │                                                                    │
      │ <── ② Réponse 200 OK + Données ─────────────────────────────────── │
      │     Headers :                                                      │
      │       Cache-Control: max-age=10 (Valide pendant 10s)               │
      │       ETag: "hash3141" (Empreinte numérique du contenu)            │
      │       Last-Modified: Sat, 22 Aug 2026 12:00:00 GMT                 │
      │                                                                    │
      │ ── [ Plus tard : Nouvelle demande ] ─────────────────────────────── │
      │                                                                    │
      │ ── ③ GET /resource ──────────────────────────────────────────────> │
      │     Headers :                                                      │
      │       If-None-Match: "hash3141"                                    │
      │       If-Modified-Since: Sat, 22 Aug 2026 12:00:00 GMT             │
      │                                                                    │
      │ <── ④ Réponse 304 Not Modified ─────────────────────────────────── │
      │     (Pas de body renvoyé ! Le navigateur réutilise son cache)      │
```

📌 **Note de dev moderne :** En production, on utilise souvent des bibliothèques comme **React Query** côté client. Elles permettent de gérer le cache de façon ultra fine sans subir les comportements parfois imprévisibles du cache HTTP natif des navigateurs.

---

## 📁 8. Gestion des Fichiers Volumineux

### 📤 Multi-part Request (Envoyer de gros fichiers)
Pour envoyer des images/vidéos du client vers le serveur :
* Utiliser `Content-Type: multipart/form-data`.
* Le body est découpé en plusieurs morceaux séparés par une chaîne unique appelée **`boundary`** (définie dans le header).
* Le serveur peut ainsi lire et traiter le fichier au fur et à mesure de l'envoi.

### 📥 Chunked / Event Stream (Recevoir de gros fichiers)
Pour renvoyer des flux de données ou de très gros fichiers du serveur vers le client :
* Headers requis : `Content-Type: text/event-stream` et `Connection: keep-alive`.
* Le serveur envoie les données par morceaux ("chunks") en continu sur la même connexion persistante. Le client les assemble dynamiquement.

---

## 🔐 9. Sécurité : SSL, TLS et HTTPS en bref

Pas besoin de devenir ingénieur réseau, mais voici la base :
* **SSL** : Le protocole historique de sécurisation. Aujourd'hui déprécié car vulnérable.
* **TLS** : Le remplaçant moderne et hautement sécurisé de SSL. Il gère l'authentification du serveur et le chiffrement des données de bout en bout.
* **HTTPS** : C’est tout simplement le protocole standard **HTTP encapsulé dans une couche de chiffrement TLS**. Indispensable pour sécuriser les données utilisateur en transit !
