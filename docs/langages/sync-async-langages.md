# Synchrone vs Asynchrone selon les langages

> Référence de mastery — comparaison JS/TS, Python, C, C++, Java, PHP, et leur relation avec le navigateur.

## Rappel express : les 3 couches (JS/TS)

Quand du JS/TS s'exécute, trois éléments distincts collaborent :

1. **Moteur JS** (V8, SpiderMonkey...) — exécute la spec ECMAScript pure. Ne connaît ni `document`, ni `fs`.
2. **Environnement d'exécution** (navigateur ou Node.js) — greffe des API autour du moteur (`setTimeout`, `fetch`, `fs`, DOM...).
3. **Ton code applicatif** — logique métier (Angular, Express, etc.).

```
Ton code JS/TS
     ↓
   Moteur JS (V8) — call stack, closures, exécution pure
     ↓
  Bindings natifs (C++)
     ↓
  libuv (Node) / thread réseau+rendu (navigateur) — event loop, thread pool, I/O système
     ↓
    OS
```

**Le point clé** : `setTimeout`, `fetch`, `fs.readFile` ne sont **pas du JS**. Ce sont des API fournies par l'environnement (navigateur ou Node), qui déléguent le travail lourd à des threads internes (thread pool libuv, threads réseau du navigateur), puis remettent un callback dans une queue que l'**event loop** exécute uniquement quand la **call stack JS est vide**.

C'est pour ça que `setTimeout(fn, 0)` ne s'exécute jamais "immédiatement" : le `0` fixe un délai minimum d'éligibilité, pas un moment d'exécution garanti — le code synchrone en cours passe toujours avant.

Deux queues distinctes existent en plus : la **microtask queue** (Promises, `queueMicrotask`) est vidée en priorité, avant chaque tour de la **macrotask queue** (`setTimeout`, `setInterval`, I/O).

## L'event loop, concrètement — c'est quoi et où elle vit

L'event loop n'est **pas une structure de données stockée quelque part** (pas un fichier, pas une variable inspectable) — c'est un **algorithme tournant en boucle infinie**, implémenté en code natif (C++), qui fait partie du processus en cours d'exécution.

### Où elle vit réellement

- **Dans le navigateur** : elle fait partie du moteur de rendu (Blink pour Chrome, Gecko pour Firefox), et tourne dans le processus du navigateur/de l'onglet, en RAM, pendant toute la durée de vie de la page.
- **Dans Node.js** : elle est implémentée dans **libuv**, une bibliothèque C compilée directement dans le binaire `node`. Elle démarre au lancement du process et tourne tant qu'il est vivant.

Concrètement : le PID visible via `ps aux | grep node` **est** le processus qui contient l'event loop active en mémoire — pas un fichier sur disque, une exécution en RAM.

### L'algorithme qu'elle exécute, en boucle

```
tant que le programme tourne :
    1. Si la call stack n'est pas vide → exécuter le prochain frame
    2. Si la call stack est vide :
        a. Vider ENTIÈREMENT la microtask queue (Promises, queueMicrotask)
        b. Prendre UNE tâche dans la macrotask queue (setTimeout, I/O, events) et l'exécuter
        c. Revenir en haut de la boucle
```

### Les structures qu'elle orchestre (elles, sont bien en mémoire)

| Structure | Rôle | Où elle est |
|---|---|---|
| **Call stack** | Pile des fonctions en cours d'exécution | Mémoire du processus (segment stack) |
| **Heap** | Objets, closures, données allouées dynamiquement | Mémoire du processus (segment heap) |
| **Microtask queue** | File des callbacks de Promises en attente | Mémoire du processus, gérée par le moteur JS (V8) |
| **Macrotask queue** | File des callbacks setTimeout/I/O/events en attente | Mémoire du processus, gérée par l'environnement (navigateur/libuv) |
| **Web APIs / Node APIs** | Timers actifs, requêtes réseau en cours | Threads séparés gérés par le navigateur ou le thread pool libuv |

L'event loop n'est donc rien d'autre que **la boucle qui fait le lien entre ces structures et la call stack** : elle vérifie en continu si la stack est libre, et si oui, ce qui attend dans les queues.

### Implication pratique

Comme l'event loop tourne dans le **même processus** que le code exécuté, un calcul synchrone trop long bloque tout : plus aucun timer, requête réseau ou clic utilisateur n'est traité tant que la call stack n'est pas libérée. C'est la cause du "freeze" d'une UI navigateur, ou d'un serveur Node qui ne répond plus pendant un traitement CPU-intensif bloquant.

---

### Angular `HttpClient`

`HttpClient` est une abstraction RxJS (`Observable`) au-dessus de `fetch` (backend par défaut depuis Angular 22, juin 2026) ou de `XMLHttpRequest` (via `withXhr()`, pour les cas comme le suivi de progression d'upload). Même mécanique que `setTimeout` : délégation au navigateur, retour via l'event loop. Différence avec une `Promise` : `Observable` est **annulable** (`unsubscribe()`) et peut émettre **plusieurs valeurs**. Détails complets dans `fetch-axios-httpclient-observable-promise.md`.

---

## Tableau de synthèse global

| Langage | Sync par défaut ? | Modèle async natif | Vrai multithreading (parallélisme CPU) | Lien direct avec le navigateur |
|---|---|---|---|---|
| **JavaScript** | Oui | Event loop native, callbacks/Promises/async-await | Non (single-thread) — Web Workers pour paralléliser | Exécuté nativement par le moteur du navigateur |
| **TypeScript** | Oui (hérite de JS) | Identique à JS après compilation | Identique à JS | Compilé en JS avant exécution |
| **Python** | Oui | `asyncio` (opt-in), event loop explicite | Non pour le code Python pur (GIL) — `multiprocessing` pour paralléliser | Aucun (sauf WASM via Pyodide) |
| **C** | Oui | Aucun natif — tout est manuel (threads OS, `select`/`epoll`) | Oui, natif (`pthread`) | Aucun direct — via WebAssembly compilé |
| **C++** | Oui | `std::async`, `std::future`, `std::thread` (depuis C++11) | Oui, natif | Aucun direct — via WebAssembly compilé |
| **Java** | Oui | Threads OS natifs, NIO non-bloquant, `CompletableFuture`, Virtual Threads (Java 21+) | Oui, natif | Aucun direct (sauf applets, obsolètes) |
| **PHP** | Oui (historiquement bloquant) | Async optionnel via extensions (Swoole, ReactPHP, Fibers natifs PHP 8.1+) | Non en PHP classique (process-based, pas de threads partagés) | Aucun — s'exécute côté serveur uniquement |

---

## C

### Modèle de base

C n'a **aucune notion d'asynchrone dans le langage lui-même**. Tout est synchrone et bloquant par défaut : un appel de fonction bloque le thread jusqu'à son retour.

```c
#include <stdio.h>
int main() {
    printf("1\n");
    sleep(2);       // bloque TOUT le thread pendant 2 secondes
    printf("2\n");
    return 0;
}
```

### Comment on simule de l'asynchrone en C

- **Threads OS natifs** (`pthread_create`) : on lance un thread séparé qui s'exécute en parallèle, avec un vrai parallélisme si plusieurs cœurs CPU sont disponibles.
- **I/O non-bloquant** : `select()`, `poll()`, `epoll()` (Linux) permettent de surveiller plusieurs descripteurs de fichiers/sockets sans bloquer — c'est **exactement le mécanisme bas niveau que libuv (donc Node.js) utilise en interne** sur Linux.
- Pas de garbage collector, pas d'event loop fournie par le langage : tout est géré manuellement par le développeur (mutex, sémaphores, condition variables pour la synchronisation entre threads).

### Relation avec le navigateur

Aucune, directement. Le C ne s'exécute jamais dans un navigateur nativement. Le seul pont possible est la compilation vers **WebAssembly** (via Emscripten par exemple), qui produit un binaire exécuté par le moteur JS du navigateur dans un environnement sandboxé — mais à ce moment-là, ce n'est plus "du C dans le navigateur", c'est du bytecode WASM piloté par JS.

---

## C++

### Modèle de base

Comme le C : synchrone et bloquant par défaut. Mais C++ moderne (C++11 et +) ajoute une **bibliothèque standard de concurrence** directement dans le langage.

```cpp
#include <future>
#include <iostream>

int calcul_long() {
    // simulation d'un traitement lourd
    return 42;
}

int main() {
    std::cout << "1\n";
    std::future<int> resultat = std::async(std::launch::async, calcul_long);
    std::cout << "3\n"; // s'exécute pendant que calcul_long tourne sur un autre thread
    std::cout << resultat.get() << "\n"; // bloque ici jusqu'à ce que le résultat soit prêt
}
```

### Outils clés

- `std::thread` : création de threads natifs OS.
- `std::async` / `std::future` / `std::promise` : équivalent conceptuel d'une `Promise` JS, mais avec du **vrai parallélisme** (threads OS réels, pas une simulation event-loop sur un seul thread).
- `std::mutex`, `std::condition_variable` : synchronisation entre threads.
- Bibliothèques tierces (Boost.Asio, libuv elle-même est écrite en C) pour de l'I/O asynchrone événementielle à la Node.js.

### Différence fondamentale avec JS

En C++, quand tu lances plusieurs threads, ils peuvent **réellement s'exécuter en même temps** sur des cœurs CPU différents (vrai parallélisme). En JS, l'asynchrone ne crée jamais de parallélisme d'exécution du code JS lui-même — un seul thread JS exécute une instruction à la fois, l'event loop ne fait qu'orchestrer l'ordre d'exécution des callbacks.

### Relation avec le navigateur

Identique au C : aucune exécution native, pont uniquement via **WebAssembly**. C'est d'ailleurs le langage historique le plus utilisé pour compiler vers WASM (moteurs de jeux, traitement d'image/vidéo dans le navigateur, Figma par exemple tourne en grande partie sur du C++ compilé en WASM).

---

## Java

### Modèle de base

Synchrone et bloquant par défaut, mais avec un **vrai support natif du multithreading** depuis toujours (contrairement à Python et son GIL).

```java
public class Main {
    public static void main(String[] args) {
        System.out.println("1");
        try { Thread.sleep(1000); } catch (Exception e) {}
        System.out.println("2");
    }
}
```

### Évolution du modèle async en Java

- **Threads classiques** (`Thread`, `Runnable`) : un thread OS par tâche — coûteux en mémoire (~1 Mo par thread), limite le nombre de threads simultanés (quelques milliers max).
- **`ExecutorService` / Thread pools** : réutilisation de threads pour éviter le coût de création.
- **NIO (`java.nio`)** : I/O non-bloquant, event-driven — base des frameworks réactifs comme Netty (utilisé par Spring WebFlux).
- **`CompletableFuture`** (Java 8+) : équivalent proche d'une `Promise` JS, avec chaînage (`.thenApply()`, `.thenCompose()`).
- **Virtual Threads / Project Loom** (Java 21+, LTS) : threads légers gérés par la JVM plutôt que l'OS — permet des **millions** de threads concurrents sans le coût mémoire des threads OS classiques. C'est la réponse de Java au modèle "un thread par requête" qui redevient viable même à très grande échelle, sans changer le style de code (pas besoin de callbacks/réactif).

### Différence avec Node.js/JS

Java peut faire du **vrai parallélisme multi-cœur** nativement (plusieurs threads OS exécutant du bytecode Java simultanément), ce que JS ne peut jamais faire sur son thread principal. Node.js compense via les **Worker Threads** (des threads V8 séparés, chacun avec sa propre stack et event loop, communication par passage de messages) — mais c'est un ajout de l'environnement Node, pas du langage JS lui-même.

### Relation avec le navigateur

Historiquement, Java pouvait tourner dans le navigateur via les **Applets Java** — techno abandonnée depuis longtemps (dépréciée par tous les navigateurs, dernier support retiré il y a des années pour raisons de sécurité). Aujourd'hui, **aucun lien direct** : Java tourne côté serveur (Spring Boot, etc.) ou via des passerelles comme **TeaVM** ou **GWT** qui transpilent du Java en JS/WASM, mais c'est marginal comparé à C++/Rust pour WASM.

---

## PHP

### Modèle de base

**Traditionnellement le plus "purement synchrone" de tous** : PHP a été conçu comme un langage **process-per-request** — chaque requête HTTP démarre un processus (ou un thread géré par PHP-FPM) qui exécute le script du début à la fin, de façon strictement bloquante, puis se termine. Pas d'event loop persistante entre les requêtes comme Node.

```php
<?php
echo "1\n";
sleep(2); // bloque tout le processus PHP en cours
echo "2\n";
```

### Le modèle PHP-FPM (comment ça scale quand même)

PHP ne gère pas la concurrence par de l'asynchrone dans le langage, mais par la **multiplication de processus/threads au niveau du serveur** :

```
Nginx/Apache
     ↓
PHP-FPM (process manager)
     ↓
   Pool de processus workers PHP
     ↓
  Chaque requête = 1 processus dédié, exécuté du début à la fin, synchrone
```

Chaque requête est indépendante et bloquante à l'intérieur d'elle-même, mais le serveur gère **plusieurs requêtes en parallèle** en ayant plusieurs processus PHP actifs simultanément (parallélisme au niveau OS, pas au niveau du langage).

### Async en PHP (les évolutions récentes)

- **Fibers** (PHP 8.1+, natif) : primitive bas niveau permettant de suspendre/reprendre l'exécution d'une fonction, base pour construire des bibliothèques async (comparable aux coroutines).
- **ReactPHP** : bibliothèque userland qui implémente une event loop façon Node.js, par-dessus PHP — permet du code non-bloquant explicite (`->then()`), mais **rien de natif au langage**, il faut l'installer et l'adopter volontairement.
- **Swoole / OpenSwoole** : extension C qui transforme PHP en serveur applicatif persistant avec event loop et coroutines natives, façon Node.js — remplace complètement le modèle PHP-FPM classique, très utilisé pour des APIs haute performance.

### Différence philosophique avec Node.js

Node.js a été conçu **dès le départ** autour d'une event loop unique et de l'asynchrone non-bloquant. PHP a été conçu autour du modèle **process-per-request synchrone**, et l'asynchrone y est une couche ajoutée après-coup (Fibers, extensions), pas le cœur du langage.

### Relation avec le navigateur

**Aucune, jamais.** PHP est un langage **exclusivement côté serveur** — il n'existe aucun moteur PHP embarqué dans un navigateur, ni de compilation vers WASM significative en usage réel. PHP génère du HTML/JSON envoyé au navigateur, mais ne s'exécute jamais dans le navigateur lui-même.

---

## Récapitulatif visuel : qui peut tourner où

| Langage | Navigateur (natif) | Navigateur (via WASM) | Serveur |
|---|---|---|---|
| JavaScript | ✅ Natif | — | ✅ (Node.js) |
| TypeScript | ✅ (compilé en JS) | — | ✅ (compilé en JS, via Node) |
| Python | ❌ | ⚠️ Marginal (Pyodide) | ✅ |
| C | ❌ | ✅ Courant (Emscripten) | ✅ |
| C++ | ❌ | ✅ Très courant | ✅ |
| Java | ❌ (applets mortes) | ⚠️ Marginal (TeaVM/GWT) | ✅ |
| PHP | ❌ | ❌ Quasi inexistant | ✅ Exclusivement |

---

## Point à retenir

Le seul langage qui a une **event loop et un modèle asynchrone imposés par le langage lui-même** (via ses environnements d'exécution officiels) est **JavaScript**. Tous les autres langages (C, C++, Java, PHP) proposent l'asynchrone comme une **option construite au-dessus** d'un modèle par défaut synchrone/bloquant — soit via de vrais threads OS avec parallélisme réel (C, C++, Java), soit via des extensions/bibliothèques tierces (PHP). Python se situe entre les deux : asynchrone disponible nativement dans la stdlib (`asyncio`) mais non actif par défaut, et bridé pour le vrai parallélisme CPU par le GIL.
