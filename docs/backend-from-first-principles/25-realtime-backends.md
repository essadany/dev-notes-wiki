# 25. Real-Time Backends (Short Polling, SSE, WebSockets & Pub/Sub)

Jusqu'à présent, toutes nos architectures respectaient le modèle classique : **Le client pose une question (requête HTTP), le serveur répond, puis la connexion se ferme.** 
Mais que faire lorsqu'on construit une application collaborative (ex: un Kanban, un chat en direct) où **deux utilisateurs doivent voir les modifications en direct sur leur écran**, sans rafraîchir la page ?

---

## 🛑 1. Le Problème Fondamental d'HTTP

Dans le modèle HTTP traditionnel (HTTP/1.1), la communication est à **l'initiative exclusive du client** :

```
┌──────────┐                               ┌──────────┐
│  Client  │ ────── GET /api/v1/board ────►│ Serveur  │
│ (Browser)│ ◄───── 200 OK (JSON) ─────────│          │
└──────────┘                               └──────────┘
```

> ⚠️ **Limitation** : Le serveur ne peut pas initier la conversation. Si un événement survient côté serveur, il est impossible de l'envoyer au navigateur sans que ce dernier ne l'ait demandé.

---

## 🔄 2. Short Polling (Polling HTTP Classique)

### Le Concept
Le navigateur interroge le serveur en boucle à intervalle régulier (ex: toutes les 3 secondes).

```
Browser                       Server
   ├─── GET /api/board/412 ─────► (Rien n'a changé)
   │◄── 200 OK (No change) ──────
   │  [ Pause 3s ]
   ├─── GET /api/board/412 ─────► (Tâche déplacée !)
   │◄── 200 OK (New Data) ───────
```

### Pourquoi c'est un Piège en Production ?
1. **Délai utilisateur (Latence moyenne)** :
   Latence Moyenne = Intervalle / 2. Pour 3s de polling, l'utilisateur attend en moyenne **1,5s**.
2. **Explosion du Coût & de la Charge** :
   Avec 10 000 utilisateurs simultanés à 3s d'intervalle, cela représente environ **3 333 requêtes / seconde**.
   Chaque requête traverse les middlewares d'auth, de logging, interroge la BDD et sérialise du JSON... **pour renvoyer une réponse vide 99% du temps**.
3. **Facturation aux Requêtes vs aux Événements** :
   Vous payez pour le nombre de requêtes transmises et non pour les vrais changements métier.
4. **Impact Mobile** :
   Le polling réveille constamment la puce radio du smartphone, répercutant le coût sur la batterie et la data de l'utilisateur.

---

## ⏳ 3. Long Polling (RFC 6202)

### Le Concept
Au lieu de répondre immédiatement, **le serveur retient la requête ouverte** tant qu'aucun événement n'est survenu (pendant 30s ou 60s). Dès qu'un changement a lieu, il répond et ferme la requête. Le navigateur reçoit la donnée et réouvre immédiatement une nouvelle requête de Long Polling.

```
Browser                                  Server
   ├─── GET /api/board/412/poll ───────────► (Garde la requête ouverte...)
   │                                        │ ⏳ Attente d'événement...
   │                                        │ ⚡ Événement déclenché !
   │◄── 200 OK (Event Data) ────────────────┤ (Connexion fermée)
   │
   ├─── GET /api/board/412/poll (Re-open) ─► (Nouvelle attente...)
```

### Limites du Long Polling
* **Intervalle de latence (Gap)** : Pendant la fraction de seconde où le client traite la réponse et réouvre la requête suivante, **le serveur n'a aucune connexion active vers le client**. Si un événement survient à cet instant, il est retardé.
* **Surcoût des En-têtes HTTP** : Chaque réouverture renvoie l'intégralité des en-têtes HTTP (~1 Ko de surcoût par cycle).

---

## 📡 4. Server-Sent Events (SSE)

### Le Concept
Le SSE résout le problème du Long Polling en gardant une **connexion HTTP unidirectionnelle ouverte indéfiniment** (du Serveur vers le Client).

```
Browser                                       Server
   ├─── GET /api/events (Accept: text/event-stream) ──►
   │◄── 200 OK (Content-Type: text/event-stream) ────── (Connexion ouverte)
   │
   │◄── event: task_moved 
 data: {...} ──────── (Événement 1)
   │◄── event: task_created 
 data: {...} ────── (Événement 2)
```

### Anatomie du Protocole SSE
Le serveur envoie du texte brut avec un formatage strict à 4 champs (`id`, `event`, `retry`, `data`) :
```http
id: 1042
event: task_moved
retry: 5000
data: {"taskId": 99, "status": "IN_PROGRESS"}
```

### Les Avantages Majeurs du SSE
1. **Basé à 100% sur HTTP** : Aucun protocole complexe. Traversée parfaite des proxies, pare-feu et Load Balancers sans configuration spéciale.
2. **Reconnexion Native Intégrée au Navigateur** : L'API JavaScript `EventSource` gère automatiquement les reconnexions en cas de coupure réseau.
3. **Le Header `Last-Event-ID`** : Lors de la reconnexion, le navigateur renvoie automatiquement le dernier ID reçu. Le serveur peut ainsi renvoyer immédiatement les événements manqués pendant la coupure !

### Cas d'Usage Réels du SSE
* **Streaming IA / LLM** : Le streaming token par token des modèles d'IA dans les interfaces web.
* **Applications de VTC / Transport** : Notification des chauffeurs pour les nouvelles courses.
* **Messageries & Réseaux Sociaux** : Notifications directes et indicateurs de saisie ("typing indicators").

> ⚠️ **Limitation du SSE** : C'est **unidirectionnel** (Serveur vers Client). Si le client veut interagir (ex: déplacer une carte), il doit envoyer une requête HTTP séparée.

---

## 🔌 5. WebSockets (WS / WSS) : La Communication Bi-directionnelle

Lorsque le système nécessite un flux bidirectionnel ultra-rapide (ex: Gaming multijoueur, Chat, Édition collaborative), les **WebSockets** s'imposent.

### A. Le Handshake WebSocket (HTTP vers WS)
Une connexion WebSocket commence **toujours par une requête HTTP GET standard**, ce qui garantit sa compatibilité avec l'infrastructure existante.

* **Requête Client** : Contient un en-tête d'upgrade réseau (`Upgrade: websocket`).
* **Réponse Serveur** : Le serveur prend la clé transmise par le client, calcule la valeur de confirmation avec la constante du protocole, et valide le changement de protocole.

> 🔒 **Pourquoi ce Handshake ?** Ce mécanisme ne sert pas à chiffrer la donnée, mais à **prouver au navigateur que le serveur comprend le protocole WebSocket** et qu'il ne s'agit pas d'un proxy HTTP réencapsulant un vieux cache.

---

### B. Le Framing Protocol & Le Masking Client
Une fois le Handshake validé, HTTP s'efface. La connexion devient un flux binaire continu découpé en **Frames** (trames) avec un surcoût d'en-tête très réduit (**2 à 10 octets par frame**, contre des centaines d'octets en HTTP).

```
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-------+-+-------------+-------------------------------+
|F|R|R|R| opcode|M| Payload len |    Extended payload length    |
|I|S|S|S| (4b)  |A|    (7b)     |             (16/64b)          |
|N|1|2|3|       |S|             |                               |
+-+-+-+-+-------+-+-------------+-------------------------------+
|     Masking-key (4 bytes, client-to-server only)             |
+---------------------------------------------------------------+
|     Payload Data ...                                          |
+---------------------------------------------------------------+
```

* **Opcodes clés** : Texte (`0x1`), Binaire (`0x2`), Close (`0x8`), Ping (`0x9`), Pong (`0xA`).
* **Masking (XOR)** : Tout message envoyé du navigateur vers le serveur **DOIT être masqué** avec une clé aléatoire de 4 octets pour éviter les erreurs d'interprétation par les proxies réseau intermédiaires.

---

### C. Heartbeats (Ping / Pong)
Comme une connexion TCP inactive ne consomme pas de trafic, le serveur ne peut pas savoir si un client a perdu sa connexion (passage Wi-Fi vers 4G, coupure tunnel/ascenseur).
* Le serveur envoie régulièrement une frame **Ping** (`Opcode 9`).
* Le client doit répondre par un **Pong** (`Opcode 10`).
* **Règle de prod** : Si aucun Pong n'est reçu sous 30 secondes, la connexion est déclarée morte et le socket est fermé pour libérer la mémoire.

---

## 💻 6. Scalabilité au Niveau Kernel & Matériel (Linux Benchmarks)

Sriniously a réalisé un benchmark réel sur un serveur Linux 16 Cœurs / 61 Go de RAM avec un serveur Go pour mesurer les limites physiques des connexions persistantes :

### Limite 1 : File Descriptors (`ulimit -n`)
Sous Unix, *"Tout est fichier"*. Un socket réseau actif consomme un **File Descriptor (FD)**.
* La limite par défaut du système est souvent fixée à **1 024 FD**.
* Au-delà de 1 017 connexions, le serveur rejette les requêtes avec l'erreur : `too many open files`.
* **Spécificité Go** : Au démarrage, le runtime de Go augmente automatiquement sa propre soft-limit à la hard-limit maximale de l'OS (`1 048 575`), alors que Node.js, Python ou Java restent bloqués par défaut par la limite utilisateur si elle n'est pas configurée dans l'OS.

### Limite 2 : Épuisement des Ports Éphémères Client
Lors des tests de charge, l'échec est survenu à **28 190 connexions** avec l'erreur : `cannot assign requested address`.
* **L'erreur classique** : Penser qu'un serveur est limité à 65 535 connexions au total.
* **La réalité TCP** : Une connexion TCP est identifiée par un **4-tuple unique** : (IP Source, Port Source, IP Dest, Port Dest).
* Comme le serveur a 1 IP et 1 Port fixe, et que la machine de test client avait 1 seule IP, le client a épuisé sa plage de ports éphémères (32 768 à 60 999). 
* **La solution** : Configurer la machine de test avec plusieurs IPs virtuelles de sortie (dial-out IPs).

### Limite 3 : Consommation Mémoire RAM
Une fois les barrières réseau levées, la vraie limite est **la RAM** :
* **Empreinte mémoire par connexion WebSocket en Go** : **~9,6 Ko** de Heap par socket (stack goroutine + buffers d'E/S).
* **Calcul d'échelle** : **1 Million de connexions WebSockets inactives** correspond à environ **~10 Go de RAM**.

---

## 🏗️ 7. Architecture Multi-Instances & Pattern Pub/Sub

### Le Problème du Stateful dans un Monde Multi-Serveurs
En acceptant des WebSockets, **notre backend redevient Stateful**. La connexion physique vit dans la mémoire RAM d'une instance spécifique (ex: Serveur A).

```
Utilisateur A ──────(WebSocket)──────► [ Instance 1 ]
                                            │ (Ne se parlent pas !)
Utilisateur B ──────(WebSocket)──────► [ Instance 2 ]
```
Si l'Utilisateur A déplace une carte sur l'Instance 1, l'Instance 2 n'en sait rien et l'Utilisateur B ne voit rien ! Les **Sticky Sessions** ne résolvent pas ce problème car elles ne font qu'ancrer un utilisateur sur un serveur, sans connecter les utilisateurs entre eux.

### La Solution : Pub/Sub (Redis / NATS / Kafka)
Pour interconnecter les instances backend, on utilise un bus de messages centralisé :

```
Utilisateur A ──► [ Instance 1 ] ── Publish("board:412") ──► ┌─────────────┐
                                                            │ Redis PubSub│
Utilisateur B ◄── [ Instance 2 ] ◄── Subscribe("board:412") ─└─────────────┘
```

1. L'Instance 1 reçoit le mouvement de carte et **publie** l'événement sur le canal `board:412` dans Redis.
2. L'Instance 2 (abonnée au canal `board:412`) reçoit l'événement depuis Redis et **transmet la frame WebSocket** à l'Utilisateur B.

> ⚠️ **At-Most-Once vs At-Least-Once** :
> * **Redis Pub/Sub classique** est en *"Fire and Forget"* (At-most-once). Si l'Instance 2 redémarre au moment de la publication, le message est perdu !
> * Si vos événements sont critiques, utilisez **Redis Streams** ou **Kafka** qui permettent de stocker et rejouer les événements manqués avec accusé de réception (ACK).

---

## ⚡ 8. Problèmes Avancés de Production

### 1. Fan-Out Explosion
Si **30 000 personnes** sont en ligne sur le même salon, 1 seul message publié implique que le backend doit effectuer **30 000 écritures réseau simultanées**. 
* Les architectures distribuées gèrent cela en répartissant le Fan-out sur plusieurs machines pour éviter un goulot d'étranglement de 1 à 2 secondes.
* **Solution** : Distribuer le Fan-out sur des Workers dédiés par sous-groupes d'abonnés.

### 2. Reconnection Storms (Tempêtes de Reconnexion)
Si une instance hébergeant 50 000 WebSockets plante, 50 000 clients vont tenter de se reconnecter **exactement à la même seconde** et demander leur état initial (tableau, tâches, profils).
* Cela crée une surcharge simultanée dévastatrice sur la base de données SQL.
* **La solution** : Placer un **Cache Edge** à l'entrée du réseau qui sert l'état initial des clients sans impacter la BDD principale lors des tempêtes de reconnexion.

---

## 📊 Tableau Récapitulatif des Technologies Temps Réel

| Technologie | Direction | Protocole | Complexité Infra | Cas d'Usage Idéal |
| :--- | :--- | :--- | :--- | :--- |
| **Short Polling** | Client vers Server | HTTP standard | Ultra-simple | Dashboards internes, peu d'utilisateurs, mises à jour lentes (> 30s) |
| **Long Polling** | Client vers Server (retardé) | HTTP standard | Moyenne | Legacy fallback quand WS est bloqué par des proxies entreprise stricts |
| **SSE (Server-Sent)** | Server vers Client | HTTP (`text/event-stream`) | Simple | Streaming LLM / IA, Fil d'actualité, Notifications push, Live Scores |
| **WebSockets** | Bi-directionnel | TCP dédié (WS/WSS) | Élevée (Pub/Sub requise) | Apps collaboratives (Kanban), Chat live, Jeux multijoueurs |

---