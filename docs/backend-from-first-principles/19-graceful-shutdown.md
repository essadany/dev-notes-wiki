# 19. Graceful Shutdown (L'Art de s'Éteindre Proprement)

C'est un sujet indispensable pour tout développeur backend : comment fermer un serveur en production **sans perdre de données, sans facturer deux fois un client et sans faire planter l'application**.

---

## 💣 1. Le Cauchemar du "Pull the Plug" (Pourquoi le Graceful Shutdown ?)

Imagine un scénario hyper classique en entreprise :
1. Un utilisateur est en train d'acheter un article sur Amazon ou Flipkart.
2. La requête de paiement est en train d'être traitée par l'API backend.
3. Au même moment, l'équipe DevOps lance un déploiement (`git push` en prod). Un nouveau conteneur/serveur démarre, et l'ancien serveur doit s'arrêter.

```
   SÉNARIO CATASTROPHIQUE (Arrêt Brutal) :
   
   [ Client / Achat ] ─── Requête de Paiement (En cours) ───> [ Ancien Serveur API ]
                                                                     │
                                                             💥 DEPLOIEMENT !
                                                             (Processus tué net)
                                                                     │
   ❌ Résultat : Transaction interrompue en plein milieu !
      - Client prélevé sans validation de commande ?
      - Double facturation en cas de retry automatique ?
      - Données corrompues en base de données ?
```

> 💡 **Le Concept Clé** : Le **Graceful Shutdown** consiste à enseigner de "bonnes manières" à notre serveur. Quand il doit s'arrêter, il ne claque pas la porte au nez des utilisateurs. Il refuse les nouveaux clients, finit poliment de répondre aux requêtes en cours, libère toutes ses ressources et ferme la porte proprement.

### 🍽️ La métaphore du restaurant :
Quand un restaurant ferme à 23h, le gérant ne coupe pas le disjoncteur général et ne jette pas les clients dehors !
1. **Étape 1** : Il met un panneau "Fermé" à l'entrée (ne laisse plus entrer de nouveaux clients).
2. **Étape 2** : Il laisse les tables actuelles finir leur repas et payer l'addition.
3. **Étape 3** : Le personnel nettoie les tables et lave la vaisselle.
4. **Étape 4** : Tout le monde sort et le gérant verrouille la porte.

---

## 🚦 2. Le Cycle de Vie d'un Processus & Les Signaux Unix (IPC)

Tout backend s'exécute sous forme de **processus** géré par le système d'exploitation (presque toujours un noyau **Linux** en production). Pour communiquer avec notre application et lui demander de s'arrêter, l'OS utilise le mécanisme d'**Inter-Process Communication (IPC)** appelé **Signaux**.

L'application enregistre des **Handlers** (des fonctions d'écoute) qui attendent ces signaux réseau/système pour réagir.

```
┌─────────────────────────────────────────────────────────────────────────────────────────┐
│                                 LES 3 SIGNAUX UNIX CLÉS                                 │
├──────────────┬───────────────────────────────┬───────────────┬──────────────────────────┤
│    Signal    │         Origine / Type        │ Traitable ?   │     Comportement         │
├──────────────┼───────────────────────────────┼───────────────┼──────────────────────────┤
│ **SIGTERM**  │ Orchestrateurs / Process Mgrs │ **OUI**       │ Demande polie d'arrêt.   │
│              │ (Kubernetes, Docker, PM2)     │ (Interceptable)│ Laisse le temps de tout  │
│              │                               │               │ nettoyer proprement.     │
├──────────────┼───────────────────────────────┼───────────────┼──────────────────────────┤
│ **SIGINT**   │ Développeur via le Terminal   │ **OUI**       │ Interruption clavier.    │
│              │ (`Ctrl + C` dans la console)  │ (Interceptable)│ À traiter exactement     │
│              │                               │               │ comme un SIGTERM !       │
├──────────────┼───────────────────────────────┼───────────────┼──────────────────────────┤
│ **SIGKILL**  │ OS / Admin en dernier recours │ **NON !**     │ L'option nucléaire !     │
│              │ (`kill -9 <pid>`)             │ (Impossible   │ Arrêt immédiat et brutal │
│              │                               │  à intercepter)│ sans nettoyage possible. │
└──────────────┴───────────────────────────────┴───────────────┴──────────────────────────┘
```

> ⚠️ **Règle d'or** : Si notre application n'écoute pas les signaux polis (`SIGTERM` / `SIGINT`), l'orchestrateur (Kubernetes) finira par lui envoyer un `SIGKILL` après un certain délai. C'est le comportement à éviter absolument !

---

## ⚙️ 3. Les Deux Piliers du Graceful Shutdown

Pour réussir l'extinction d'un serveur backend, il faut appliquer scrupuleusement deux phases :

```
             ┌─────────────────────────────────────────────────────────┐
             │            PHASE 1 : CONNECTION DRAINING                │
             │  • Stopper l'acceptation de nouvelles requêtes         │
             │  • Laisser finir les requêtes en-vol (In-flight)        │
             └────────────────────────────┬────────────────────────────┘
                                          │
                                          ▼
             ┌─────────────────────────────────────────────────────────┐
             │            PHASE 2 : RESOURCE CLEANUP                   │
             │  • Fermer la pool de connexions BDD (Postgres, etc.)    │
             │  • Fermer les connexions Redis / Queues de tâches       │
             │  • Libérer les descripteurs de fichiers & Sockets       │
             └────────────────────────────┬────────────────────────────┘
                                          │
                                          ▼
                                     [ Exit 0 ]
```

### A. Connection Draining (La Vidange des Connexions)
* **Serveur HTTP** : Refuse immédiatement toute nouvelle connexion TCP/HTTP. Les requêtes en cours de traitement (*In-flight requests*) ont un temps imparti pour se terminer.
* **Le Compromis du Timeout (ex: 30 secondes)** :
  * Si le timeout est **trop court** (ex: 2s), on risque de couper des requêtes légitimes lourdes.
  * Si le timeout est **trop long** (ex: 10 minutes), les déploiements deviennent excessivement lents et léthargiques.
  * **La norme** : Définir un timeout dur de **30s à 60s** selon la nature de l'application.
* **Bases de Données & WebSockets** : 
  * Les requêtes SQL en cours doivent être terminées ou annulées proprement (Commit ou Rollback).
  * Les WebSockets doivent envoyer un message de fermeture au client avant de couper le socket.

### B. Nettoyage des Ressources (Resource Cleanup)
Chaque connexion réseau ou fichier ouvert consomme un descripteur de fichier et de la mémoire RAM. Ne pas les fermer avant l'extinction peut créer des fuites de mémoire et des blocages de verrous (*deadlocks*) en BDD.

> 🔄 **La Règle d'Or de l'Ordre Inverse** : On doit TOUJOURS nettoyer les ressources dans **l'ordre inverse de leur acquisition** !
> 1. Si au boot j'ai initialisé : `Redis` ──> `Postgres` ──> `Serveur HTTP`
> 2. Au shutdown je dois fermer : `Serveur HTTP` ──> `Postgres` ──> `Redis`
> 
> *Pourquoi ?* Pour éviter de fermer la base de données alors que le serveur HTTP traite encore une requête qui a besoin d'exécuter une requête SQL !

---

## 💻 4. Anatomie du Code & Flux d'Exécution Réel

Voici le déroulé pas-à-pas de ce qui se passe dans un serveur professionnel (exemple basé sur le code Go / Node.js montré par Sriniously) :

```
[ Système / Kubernetes / Dev ]                                    [ Application Backend ]
              │                                                             │
              │ ── ① Envoi du signal SIGTERM ou Ctrl+C (SIGINT) ──────────> │
              │                                                             │
              │                                                             ├─ ② Le Signal Handler intercepte le signal
              │                                                             ├─ ③ Arrêt du serveur HTTP (plus de nouvelles requêtes)
              │                                                             ├─ ④ Attente de la fin des requêtes en cours (In-flight)
              │                                                             ├─ ⑤ Fermeture du pool de connexions BDD (Postgres)
              │                                                             ├─ ⑥ Vidange & fermeture de la queue de tâches (Redis)
              │                                                             │
              │ <── ⑦ Le processus s'arrête proprement (Exit Code 0) ───────┤
```

### Extrait des Logs de Production d'un Graceful Shutdown :
```text
[INFO]  Starting server on port :8080...
[INFO]  Connected to PostgreSQL Database Pool.
[INFO]  Started Background Worker Queue (Redis).
... (Le serveur fonctionne normalement) ...
[WARN]  Signal received: SIGINT (Ctrl+C). Initiating Graceful Shutdown...
[INFO]  [1/3] Stopping HTTP Server... No longer accepting new connections.
[INFO]  [2/3] Waiting for in-flight requests to complete (Timeout: 30s)...
[INFO]  [2/3] All HTTP requests finished successfully.
[INFO]  [3/3] Closing PostgreSQL connection pool... Done.
[INFO]  [3/3] Stopping Async Worker Queue & Redis connections... Done.
[INFO]  Graceful shutdown complete. Exiting process with code 0.
```

---

## 📋 5. Checklist du Développeur Backend Junior

Pour garantir des déploiements zéro-downtime et des serveurs robustes en production :

- [x] **Enregistrer des écouteurs de signaux** pour `SIGTERM` (déploiements cloud/K8s) et `SIGINT` (`Ctrl+C` en local).
- [x] **Arrêter en premier le serveur HTTP** pour bloquer immédiatement l'entrée de tout nouveau trafic.
- [x] **Configurer un Timeout dur** (ex: 30 secondes) pour éviter qu'une requête bloquée n'empêche le serveur de s'éteindre indéfiniment.
- [x] **Respecter l'ordre inverse d'acquisition** lors de la fermeture des clients (HTTP ──> BDD ──> Caches/Queues).
- [x] **Gérer le Load Balancer / Health Check** : Faire échouer le endpoint `/health` dès la réception du signal pour que le Load Balancer retire le serveur du pool de routage.
- [x] **Ne jamais utiliser `SIGKILL` en interne** : Laisser l'OS ou l'orchestrateur gérer les cas désespérés via le timeout de grâce.
