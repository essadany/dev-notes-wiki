# 22. Concurrence & Parallélisme (IO-Bound vs CPU-Bound)

Tous les systèmes backend sans exception partagent une exigence fondamentale : **pouvoir traiter des milliers de requêtes en même temps**. Si notre serveur web était purement synchrone et ne pouvait traiter qu'une seule requête à la fois, chaque utilisateur devrait attendre son tour ou recevrait une erreur serveur saturé.

---

## 🛑 1. Le Problème : Le Gaspillage Massif du CPU (IO-Bound)

Dans une application backend typique, une requête API réalise en moyenne **3 à 5 requêtes BDD** et **1 à 2 appels réseau externes** (cache Redis, envoi d'emails via API tierce).

### Le Chiffre Choc des Temps d'Attente IO :
* **En local (Localhost)** : Une requête BDD prend ~1 à 2 ms.
* **En production (Même Zone Cloud)** : Une requête BDD prend ~20 à 30 ms.
* **Inter-régions Cloud** : Une requête BDD prend ~90 à 100 ms.

```
┌────────────────────────────────────────────────────────────────────────────────────────┐
│               ANATOMIE D'UNE REQUÊTE BACKEND TYPIQUE (~260 ms au total)                │
├────────────────────────────────────────────────────────────────────────────────────────┤
│ Temps CPU réel (parsing JSON, validations, logique métier) :  ██ 10 ms (4%)            │
│ Temps d'attente IO (BDD, Redis, API externe Resend/Clerk)  :  ██████████████████ 250 ms (96%)│
└────────────────────────────────────────────────────────────────────────────────────────┘
```

> ⚠️ **Le Gaspillage Monstrueux** : Un CPU moderne tourne à environ **3 GHz** (exécute 3 millions d'instructions par milliseconde). Attendre 100 ms le retour d'une requête BDD sans rien faire d'autre représente **300 MILLIONS d'instructions CPU inutilisées** ! Le CPU reste oisif (idle) **95% du temps**.

---

## ⚖️ 2. Concurrence vs Parallélisme : La Différence Clé

La confusion entre concurrence et parallélisme est fréquente. Voici la distinction mécanique exacte :

```
┌────────────────────────────────────────────────────────────────────────────────────────┐
│ 1. PARALLÉLISME (Hardware Required)                                                    │
│    • Exécuter 2 instructions STRICTEMENT EN MÊME TEMPS à l'instant T.                   │
│    • Nécessite au moins 2 cœurs CPU physiques distincts.                              │
│                                                                                        │
│ 2. CONCURRENCE (Software & Structure Required)                                         │
│    • Gérer (dealing with) plusieurs tâches en même temps, même avec 1 SEUL cœur CPU.   │
│    • Repose sur l'entrelacement : démarrer, mettre en pause, basculer et reprendre.     │
└────────────────────────────────────────────────────────────────────────────────────────┘
```

* **Citation de référence** : *"Parallelism is about doing multiple things at once. Concurrency is about dealing with multiple things at once."*

---

## 🎯 3. IO-Bound vs CPU-Bound

| Caractéristique | Workloads IO-Bound | Workloads CPU-Bound |
| :--- | :--- | :--- |
| **Définition** | Le temps d'exécution est limité par les entrées/sorties (Réseau, BDD, Fichiers, Logs). | Le temps d'exécution est limité par la puissance brute de calcul du processeur. |
| **Exemples Backend** | API REST/GraphQL, requêtes SQL, appels d'APIs externes, lecture/écriture de fichiers. | Chiffrement JWT/BCrypt, compression, traitement d'images/vidéos, ML, parsing lourd. |
| **Modèle idéal** | **Concurrence** (Event Loop / Async-Await / Goroutines) pour ne pas bloquer le CPU. | **Parallélisme** (Multithreading / Multi-processus sur plusieurs cœurs CPU). |
| **Temps CPU typique** | 1 à 10 ms de CPU pour 200+ ms d'attente réseau. | 100% d'utilisation CPU pendant tout le traitement. |

---

## ⚙️ 4. Les 3 Modèles de Concurrence en Production

### 🅰️ Modèle 1 : Les Threads OS (Threading Model classique - C, Java ancien, Python OS Threads)
Chaque thread est géré nativement par le Kernel de l'OS.

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        THREAD OS NATIVE (Kernel)                        │
├─────────────────────────────────────────────────────────────────────────┤
│ • Stack Memory : 512 KB à 8 MB alloués par thread                       │
│ • Instruction Pointer & Registres CPU sauvegardés par l'OS              │
│ • Ordonnancement Préemptif (Preemptive Scheduling par le Kernel)        │
└─────────────────────────────────────────────────────────────────────────┘
```

* **Avantage** : Partage direct de mémoire en RAM au sein d'un même processus (sans copie de données).
* **Les 3 Défauts Majeurs** :
  1. **Coût Mémoire Élevé** : 10 000 requêtes simultanées = 10 000 threads $
ightarrow$ **~8 Go de RAM** consommés uniquement pour maintenir les stacks !
  2. **Coût de Création** : Appels système Kernel (`sys_clone` / `pthread_create`) coûteux.
  3. **Context Switch (Changement de contexte)** : Sauvegarder les registres et recharger le stack prend 1 à 10 $\mu s$. Multiplié par des milliers de threads, le CPU passe son temps à gérer le basculement plutôt qu'à exécuter du code.

---

### 🅱️ Modèle 2 : L'Event Loop (Single-Threaded Async - JavaScript / Node.js)
Un seul thread principal s'exécute en boucle. **Règle d'or : NE JAMAIS BLOQUER L'EVENT LOOP !**

```
 [ Thread Unique Node.js ] ──> [ Exécute le code JS ]
                                      │
                                      ▼ (Détecte une opération IO : DB / HTTP)
                              [ Délègue à l'OS via epoll (Linux) / kqueue (macOS) ]
                                      │
                                      ▼ (Libère immédiatement le thread JS)
 [ Traite d'autres requêtes ] <───────┤
                                      │ (Une fois l'IO terminée)
                              [ Dépose le Callback dans la Queue ] ──> [ Exécute le Callback ]
```

* **Comment fonctionne Async/Await sous le capot ?**
Le compilateur/runtime transforme une fonction `async/await` en une **Machine à États (State Machine)** :
```javascript
// Votre code :
async function fetchUser(id) {
  const user = await db.getUser(id);  // <-- Point de coupure (State 0 -> State 1)
  const orders = await db.getOrders(user.id); // <-- Point de coupure (State 1 -> State 2)
  return { user, orders };
}
```
Au moment de l'__`await`__, le contrôle est immédiatement rendu à l'Event Loop. La suite du code est encapsulée dans le callback d'état qui s'exécutera à la fin de l'IO.

---

### 🅲 Modèle 3 : Les Virtual Threads & Goroutines (Go Runtime)
Go utilise un modèle hybride **M:N** (M Goroutines réparties sur N Threads OS).

```
┌─────────────────────────────────────────────────────────────────────────┐
│                      GOROUTINES (Virtual Threads)                       │
├─────────────────────────────────────────────────────────────────────────┤
│ • Empreinte mémoire initiale minuscule : ~2 KB par Goroutine !          │
│ • Ordonnanceur interne Go (Go Runtime Scheduler) en Espace Utilisateur  │
│ • Un simple switch de pointeur (Context Switch en nanosecondes)         │
└─────────────────────────────────────────────────────────────────────────┘

[ Goroutine 1 ] ──┐
[ Goroutine 2 ] ──┼──> [ Go Runtime Scheduler ] ──> [ Thread OS 1 (Cœur 1) ]
[ Goroutine 3 ] ──┤                                 [ Thread OS 2 (Cœur 2) ]
[ Goroutine 4 ] ──┘
```
Dans Go, le package HTTP standard crée **automatiquement une nouvelle Goroutine par requête reçue** (`go c.serve(conn)`). Comme une Goroutine ne coûte que 2 Ko, on peut en faire tourner des centaines de milliers sans s'épuiser en RAM.

---

## ⚡ 5. Les Pièges de Concurrence & Les Race Conditions

Dès que deux threads ou deux tâches concourantes accèdent à un **état partagé (Shared Memory)**, des bugs critiques surviennent.

### A. Le Problème du "Lost Update" (Mise à jour perdue sur compteur)
Pour incrémenter une variable `counter++`, un processeur effectue 3 étapes : **Read** $
ightarrow$ **Increment** $
ightarrow$ **Write**.

```
Thread A : Read (0) ───────> Increment (1) ─────────────────> Write (1)
Thread B : ────> Read (0) ───────────> Increment (1) ───────────────> Write (1) (ÉCRASE !)

Résultat attendu : 2  │  Résultat réel obtenu : 1 (1 incrémentation perdue !)
```

### B. Le Piège de la Race Condition en Async/Await (Single-Threaded JS !)
Même dans un environnement mono-threadé comme Node.js, l'asynchronisme crée des Race Conditions au niveau logique métier !

```javascript
// CODE VULNÉRABLE :
async function withdraw(userId, amount) {
  const balance = await db.getBalance(userId); // Ex: 100€
  
  if (balance >= amount) { // 100 >= 100 -> TRUE pour les 2 requêtes simultanées !
    await processPayment(amount); // IO Async : Rend le contrôle à l'Event Loop !
    await db.setBalance(userId, balance - amount);
  }
}
```
Si deux requêtes de retrait de 100€ arrivent au même instant :
1. Requête 1 lit le solde (100€) $
ightarrow$ OK $
ightarrow$ démarre `processPayment()` et cède le contrôle.
2. Requête 2 lit le solde (100€) $
ightarrow$ OK $
ightarrow$ démarre `processPayment()` et cède le contrôle.
3. Les deux requêtes débitent 100€ $
ightarrow$ **Solde final = -100€ !**

### C. Solutions de Sécurisation :
* **Mutuelle Exclusion (Locks / Mutex)** : Bloquer l'accès concurrent aux sections critiques.
* **Passage de Messages (Channels en Go)** : *"Do not communicate by sharing memory; instead, share memory by communicating."*
* **Transactions SQL & Atomic Updates** : `UPDATE accounts SET balance = balance - 100 WHERE id = 1 AND balance >= 100;`.

---

## 📋 6. Matrice Récapitulative pour le Développeur Backend

```
┌────────────────────────┬──────────────────────────┬──────────────────────────┐
│ Critère                │ Workloads IO-Bound       │ Workloads CPU-Bound      │
├────────────────────────┼──────────────────────────┼──────────────────────────┤
│ Nature du goulot       │ Attente Réseau / BDD     │ Calcul processeur pure   │
│ Stratégie              │ Concurrence (Non-bloquant)│ Parallélisme (Multi-core)│
│ Modèle recommandé      │ Event Loop / Goroutines  │ Worker Threads / Worker Pool
│ Impact du Multithread  │ Négatif (Context switch) │ Trè très positif         │
└────────────────────────┴──────────────────────────┴──────────────────────────┘
```

---

🎓 **Bilan** : Comprendre si notre code est **IO-Bound** ou **CPU-Bound** permet de choisir l'architecture applicative optimale : utiliser l'asynchronisme et la concurrence pour maximiser l'utilisation du CPU lors des attentes BDD, et déporter les lourds calculs CPU sur des Workers dédiés !
