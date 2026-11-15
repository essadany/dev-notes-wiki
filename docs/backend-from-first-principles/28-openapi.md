# 28. OpenAPI (Le contrat universel entre clients et serveurs)

Lorsqu'on développe une application web ou mobile, l'un des plus grands pièges réside dans la désynchronisation entre ce que le serveur attend et ce que le client envoie. **OpenAPI** est le standard industriel mondial qui résout définitivement ce problème en établissant un contrat universel et exécutable à travers la frontière réseau.

---

## ⚡ 1. Le Problème : La Crise des Sources de Vérité Multiples

Dans une architecture sans contrat formel, la connaissance d'un endpoint HTTP (route, paramètres, corps de requête, codes de retour) est éparpillée à trois endroits distincts :

```
 ┌───────────────────────────┐      ┌───────────────────────────┐
 │     1. Backend Handler    │      │    2. Frontend (Fetch)    │
 │ (Code source qui lit le   │      │ (Appel de l'API dans      │
 │  body et valide les data) │      │  React, Vue, iOS, etc.)   │
 └─────────────┬─────────────┘      └─────────────┬─────────────┘
               │                                  │
               │      ┌────────────────────┐      │
               └────► │  3. Documentation  │ ◄────┘
                      │ (Markdown, Notion, │
                      │  Swagger obsolète) │
                      └────────────────────┘
```

### La frontière réseau (Network Boundary) vs La frontière du code
* **À l'intérieur du codebase backend (Codebase Boundary)** : Si un développeur modifie la signature d'une fonction interne (ex: changement du type d'un argument), le compilateur (TypeScript, Go, Java, Rust) lève **immédiatement une erreur à la compilation**.
* **À travers le réseau (Network Boundary)** : Le protocole HTTP est par nature dépourvu de signature statique. Si quelqu'un renomme un champ dans le handler backend (ex: `two` -> `column`), le frontend continue d'envoyer `two` et l'erreur n'apparaît qu'en **runtime** (erreur 400 ou 500 en production).

> 💡 **Le rôle d'OpenAPI** : Donner aux appels réseaux distants la même protection par signature stéréotypée et compilée que celle dont bénéficient nos fonctions locales dans le code.

---

## 📜 2. Origine et Clarification des Termes

### L'histoire (Wordnik, 2010)
En 2010, **Tony Tam** et l'équipe de la startup Wordnik ont un problème : leurs clients exigent des SDKs et des documentations dans plusieurs langages (Python, PHP, JavaScript). Face à ce constat (*"We often code faster than we can document"*), ils créent une spécification JSON permettant au serveur de décrire lui-même ses capacités. De là est né **Swagger**.

* **2011** : Swagger 1.0 est rendu open-source.
* **2015** : La société SmartBear rachete Swagger et fait don de la spécification à la **Linux Foundation** (avec le soutien de Google, Microsoft, IBM, PayPal).
* **2016** : La spécification est officiellement renommée **OpenAPI Specification (OAS)**.

### Ne plus confondre les 4 termes :
| Terme | Signification |
| :--- | :--- |
| **OAS (OpenAPI Specification)** | L'ensemble des règles standardisées qui définissent le format. |
| **OAD (OpenAPI Description)** | Le fichier physique (`openapi.json` ou `openapi.yaml`) décrivant votre API. |
| **OAI (OpenAPI Initiative)** | L'organisation sous l'égide de la Linux Foundation qui maintient la spec. |
| **Swagger** | La marque et les outils historiques (ex: Swagger UI, Swagger Editor). |

---

## 🏗️ 3. Anatomie d'un Fichier OpenAPI (`openapi.yaml`)

Pourquoi utiliser du **YAML** ou du **JSON** plutôt que du Markdown ? Parce qu'un format structuré permet d'être parsé de manière **programmatique** par des ordinateurs tout en restant lisible par les humains.

### La structure minimale d'un document OpenAPI 3.1
```yaml
openapi: 3.1.0          # Version de la spécification OpenAPI
info:
  title: Tasker API     # Nom de votre API
  version: 1.0.0        # Version sémantique de VOTRE backend
servers:
  - url: http://localhost:8080
    description: Serveur de Développement Local
paths: {}               # Liste des endpoints HTTP
```

### Détail d'une Opération (`paths`)

```yaml
paths:
  /api/tasks/{taskId}/move:
    post:
      summary: Déplacer une tâche vers une autre colonne
      operationId: moveTask    # IMPORTANT : deviendra le nom de la fonction générée
      tags:
        - Tasks
      parameters:
        - name: taskId
          in: path             # path, query, header, ou cookie
          required: true
          schema:
            type: string
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              required:
                - targetColumnId
              properties:
                targetColumnId:
                  type: string
                actor:
                  type: string
                  default: guest
      responses:
        '200':
          description: Tâche déplacée avec succès
        '400':
          description: Erreur de validation des entrées
        '404':
          description: Tâche introuvable
```

---

## 🔀 4. Code-First vs Design-First (Contract-First)

Il existe deux approches pour intégrer OpenAPI dans une équipe de développement :

```
                      ┌──────────────────────────────────────┐
                      │    L'APPROCHE CODE-FIRST (Legacy)    │
                      │  On code les handlers -> On génère   │
                      │  la spec via des annotations/code    │
                      └──────────────────┬───────────────────┘
                                         │
                                         ▼
                      ┌──────────────────────────────────────┐
                      │   L'APPROCHE DESIGN-FIRST (Recommandée)│
                      │  On écrit le contrat OpenAPI D'ABORD │
                      │  -> On génère serveurs & clients !   │
                      └──────────────────────────────────────┘
```

### 1. Code-First
* **Principe** : On écrit le code des controllers/handlers backend d'abord. La spec OpenAPI est générée automatiquement à l'aide d'annotations dans le code (ex: FastAPI en Python, annotations Swagger en Java, ou schémas Zod/ts-rest en TypeScript).
* **Inconvénient** : Le frontend doit attendre que le backend ait terminé l'implémentation avant de pouvoir commencer le travail d'intégration.

### 2. Design-First (Contract-First) — La méthode moderne
* **Principe** : On rédige le contrat OpenAPI (`openapi.yaml`) **avant** toute ligne de code.
* **Avantages majeurs** :
  1. **Développement Parallèle** : Le frontend et le backend travaillent simultanément dès le jour 1.
  2. **Intégration d'Agents IA** : Permet de donner le contrat à des agents IA autonomes (un agent côté serveur, un agent côté client) sans blocage.
  3. **Vérité Unique** : Aucune dérive entre la doc et le code réel.

---

## 🛠️ 5. Le Pipeline de Production Design-First en 5 Étapes

Une fois le contrat `openapi.yaml` écrit, voici la chaîne d'outils automatisée à mettre en place :

```
  ┌─────────────────┐
  │  openapi.yaml   │
  └────────┬────────┘
           │
           ├────────► 1. LINTER (Spectral)
           │          └─► Vérifie la syntaxe et la complétude
           │
           ├────────► 2. MOCK SERVER (Prism)
           │          └─► Fournit une API factice immédiate au Frontend
           │
           ├────────► 3. CLIENT GEN (openapi-typescript / openapi-fetch)
           │          └─► Génère le SDK et les types pour le Frontend
           │
           ├────────► 4. SERVER GEN (oapi-codegen pour Go)
           │          └─► Génère les interfaces strictes pour le Backend
           │
           ├────────► 5. INTERACTIVE DOCS (Scalar / Swagger UI / ReDoc)
           │          └─► Génère une interface web de test d'API
           │
           └────────► 6. CONTRACT TESTING (Schemathesis)
                      └─► Valide que le serveur réel respecte à 100% la spec
```

1. **Étape 1 - Linter (`Spectral`)** : Analyse le fichier YAML et remonte les avertissements (ex: opération sans `operationId` ou réponse 200 sans schéma).
2. **Étape 2 - Mock Server (`Prism`)** : Lance un serveur HTTP factice basé sur le fichier OpenAPI. Le frontend peut effectuer des requêtes réelles et recevoir les données d'exemple documentées avant même que le backend n'existe.
3. **Étape 3 - Génération de Code Client (`openapi-typescript`)** : Génère les types TypeScript et les fonctions d'appel API. Toute erreur de frappe sur un champ provoque une erreur à la compilation frontend.
4. **Étape 4 - Génération d'Interfaces Serveur (`oapi-codegen`)** : Génère des interfaces dans le langage du serveur (ex: Go). Si le handler backend ne respecte pas l'interface, le projet **refuse de compiler**.
5. **Étape 5 - Documentation Interactive (`Scalar` / `Swagger UI`)** : Offre une interface Web moderne pour consulter et tester les endpoints avec un bouton *"Try it out"*.
6. **Étape Bonus - Contract Testing (`Schemathesis`)** : Outil qui lit la spec, génère des requêtes de test pour vérifier que le serveur réel ne dévie jamais du contrat.

---

## 🌐 6. OpenAPI dans l'Écosystème Réel & L'Ére des Agents IA

* **Specs Géantes en Production** : Les géants du Web publient leurs spécifications OpenAPI complètes. Celle de **GitHub** dépasse les 344 000 lignes YAML, et celle de **Stripe** fait plus de 171 000 lignes.
* **Le Standard pour les Agents IA** : Pour qu'un agent d'IA (LLM) puisse interagir avec vos systèmes backend (Tool Use / Function Calling), il suffit de lui fournir la spécification OpenAPI de vos services. L'IA comprend instantanément les routes, les paramètres et les formats d'entrées/sorties sans aucun code de colle supplémentaire.

---

> 💡 **Le conseil de l'ingénieur backend junior** : OpenAPI n'est pas juste de la documentation passive, c'est le **noyau exécutable** de l'architecture API. En adoptant l'approche *Design-First*, on élimine 90% des bugs d'intégration réseau avant même le premier déploiement !
