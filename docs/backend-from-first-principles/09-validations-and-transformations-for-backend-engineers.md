# 9. Les validations et les transformations pour les ingénieurs backend

**les validations et les transformations de données**. C'est littéralement la ligne de défense qui empêche notre serveur de planter à cause d'entrées utilisateur mal formées, corrompues ou malveillantes.

> *Objectif : Comprendre comment valider et transformer les données côté serveur pour assurer la sécurité, l'intégrité et la robustesse de notre application backend.*

---

## 🧭 1. L'Architecture en Couches : Où se place la validation ?

Dans une architecture backend classique, l'application est découpée en 3 couches distinctes. La requête traverse ces couches de haut en bas :

```
                  ┌────────────────────────────────────────┐
                  │          CLIENT / NAVIGATEUR           │
                  └───────────────────┬────────────────────┘
                                      │  (Requête HTTP / Payload JSON)
                                      ▼
┌──────────────────────────────────────────────────────────────────────────────────┐
│ 1. COUCHE CONTROLLER (HTTP & Portier de l'API)                                  │
│    • Gère l'HTTP (Codes d'état, headers, etc.)                                   │
│    • C'est ICI que tourne le pipeline de VALIDATION & TRANSFORMATION !           │
└─────────────────────────────────────┬────────────────────────────────────────────┘
                                      │  (Données propres et validées)
                                      ▼
┌──────────────────────────────────────────────────────────────────────────────────┐
│ 2. COUCHE SERVICE (Logique Métier)                                               │
│    • Exécute les algorithmes de l'application (calculs, logique spécifique)      │
│    • Déclenche des webhooks, envoie des emails, des notifications, etc.          │
└─────────────────────────────────────┬────────────────────────────────────────────┘
                                      │
                                      ▼
┌──────────────────────────────────────────────────────────────────────────────────┐
│ 3. COUCHE REPOSITORY (Accès aux Données / Base de Données)                       │
│    • Gère les connexions et requêtes vers la base de données (SQL, Redis, etc.)   │
└──────────────────────────────────────────────────────────────────────────────────┘
```

> ⚠️ **Règle d'or de sécurité** : On doit valider et transformer les données à la **première porte d'entrée (au niveau du Controller)**, avant même d'appeler la couche Service ou d'exécuter la moindre logique métier lourde !

---

## 💣 2. Pourquoi valider côté serveur ? (L'effet Domino)

Si on ne valide pas les données dès l'entrée, voici ce qui se passe en coulisses :

```
[ Client envoie : { "name": 0 } ] (un nombre au lieu d'une string)
                  │
                  ▼
[ Controller ] (Ne valide rien, laisse passer)
                  │
                  ▼
[ Service Layer ] (Transmet la donnée brute)
                  │
                  ▼
[ Repository / SQL Query ] (Tente d'insérer 0 dans une colonne PostgreSQL typée TEXT NOT NULL)
                  │
                  ▼
❌ [ Crash de la Base de Données ] ──> Le serveur lève une exception interne non gérée
                  │
                  ▼
[ Réponse finale ] ──> Erreur 500 Internal Server Error (Très mauvaise UX & faille potentielle !)
```

### Le bon réflexe avec un bouclier de validation :
En validant à l'entrée, le portier (middleware de validation) détecte immédiatement que `"name"` n'est pas une chaîne de caractères et coupe court à la requête en renvoyant une erreur **`400 Bad Request`**. La base de données et la logique métier ne sont jamais sollicitées pour rien !

---

## 🚦 3. Les 3 Familles de Validations à Maîtriser

On peut classer nos règles de validation en trois catégories principales pour structurer nos schémas :

| Type de Validation | Rôle concret | Exemples typiques |
| :--- | :--- | :--- |
| **1. Syntactique** | Vérifie si la structure du texte respecte un format ou un pattern précis. | • Structure d'un email (présence du `@`, d'un domaine et d'un TLD).<br>• Numéro de téléphone (format international, nombre de chiffres selon le pays).<br>• Format d'une date (ex: `YYYY-MM-DD`). |
| **2. Sémantique** | Vérifie si la donnée "fait sens" logiquement par rapport à la réalité ou au contexte métier. | • Date de naissance (ne peut pas être dans le futur !).<br>• Âge cohérent d'un humain (ex: compris entre 1 et 120 ans).<br>• Logique conditionnelle (ex: le champ `"nom_partenaire"` est obligatoire si `"est_marie"` est à `true`).<br>• Deux champs identiques (ex: `"mot_de_passe"` doit être identique à `"confirmation_mot_de_passe"`). |
| **3. De Type** | Vérifie la nature primitive de la donnée. | • Est-ce une chaîne de caractères (`string`) ?<br>• Un nombre (`number`) ?<br>• Un booléen (`boolean`) ?<br>• Un tableau (`array`) ? |

---

## 🔄 4. Le Rôle des Transformations (Casting & Nettoyage)

Souvent, on associe les validations et les transformations dans un même pipeline. Pourquoi ? Parce que le serveur reçoit parfois des données brutes inadaptées qu'il doit formater de manière transparente.

### A. Le casting de types (Exemple de la pagination)
Dans une route paginée (`/api/bookmarks?page=2&limit=20`), les paramètres `page` et `limit` arrivent **toujours sous forme de chaînes de caractères (`string`)** car ils proviennent de l'URL. 
* Si on valide directement que `page` doit être un nombre supérieur à 0, la validation échouera car `"2"` est une string.
* **La Transformation intervient ici** : Elle convertit (cast) `"2"` en `2` (nombre) *avant* d'exécuter la validation numérique.

### B. Le nettoyage et l'harmonisation
Le pipeline peut aussi modifier la donnée pour le confort de notre couche service :
* Mettre une adresse e-mail en minuscules (`TEST@GMAIL.COM` ──> `test@gmail.com`) pour éviter les doublons en base de données.
* Ajouter un préfixe international aux numéros de téléphone (ex: rajouter le `+` s'il est manquant).

---

## 🛡️ 5. Front-End vs Back-End Validations : La Véritable Règle d'Or

C'est une erreur classique de développeur junior : penser que la validation du formulaire faite en React / Vue suffit et qu'on peut s'en dispenser côté backend.

> 💡 **Leurs rôles sont fondamentalement différents et complémentaires :**

```
┌───────────────────────────────────────────────────────────────────────────────────────────┐
│                                   DUEL DES VALIDATIONS                                    │
├─────────────────────────────────────────────┬─────────────────────────────────────────────┤
│         VALIDATION FRONT-END (UX)           │         VALIDATION BACK-END (SÉCURITÉ)      │
├─────────────────────────────────────────────┼─────────────────────────────────────────────┤
│ • Sert uniquement à l'expérience utilisateur│ • Sert à la SÉCURITÉ et à l'INTÉGRITÉ       │
│   (User Experience).                        │   des données.                              │
│                                             │                                             │
│ • Donne un feedback visuel immédiat (ex:   │ • Obligatoire car n'importe qui peut appeler│
│   champ rouge, bouton désactivé).           │   notre API directement avec Postman ou     │
│                                             │   Insomnia en contournant le front-end !    │
│ • Évite d'envoyer des requêtes réseau       │                                             │
│   inutiles au serveur.                      │ • Est totalement indépendante du client     │
│                                             │   qui effectue l'appel.                     │
└─────────────────────────────────────────────┴─────────────────────────────────────────────┘
```

**Conclusion de mes révisions** : Quand je conçois une API, je dois toujours coder mes validations backend comme si le client front-end n'existait pas. C'est l'unique moyen d'assurer un système robuste et incassable !
