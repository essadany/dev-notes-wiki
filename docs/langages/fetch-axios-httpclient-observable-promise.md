# Intercepteurs, clients HTTP, Observable vs Promise

> Référence de mastery — chaque concept est d'abord expliqué **indépendamment de tout framework, en TypeScript**, puis illustré avec **Angular 22** (stable, juin 2026), **React 19.2** (stable, `19.2.8`, juillet 2026) et **Vue 3.5** (stable, `3.5.42`, août 2026 — Vue 3.6/Vapor Mode encore en RC, non traité ici).

---

## 1. Le concept d'intercepteur (Interceptor / Middleware)

### C'est quoi, en général

Un **intercepteur** (ou *middleware* selon les écosystèmes) est un point d'insertion générique dans le trajet d'une requête ou d'un traitement, qui permet d'exécuter du code **avant** et/ou **après** l'opération réelle, sans modifier le code qui déclenche cette opération. Ce n'est pas un concept propre à Angular, ni même propre au HTTP — c'est un **pattern de conception** qu'on retrouve sous des noms différents dans presque tous les écosystèmes :

| Écosystème | Nom du concept |
|---|---|
| Angular | `HttpInterceptor` / `HttpInterceptorFn` |
| Express.js / Node | `middleware` |
| Axios | `interceptors.request` / `interceptors.response` |
| Django | `Middleware` |
| Spring (Java) | `Interceptor` / `Filter` |
| gRPC | `Interceptor` |
| ASP.NET | `Middleware` / `DelegatingHandler` |

### Le pattern sous-jacent : Chain of Responsibility

Peu importe le langage ou le framework, le principe est identique : chaque requête traverse une **chaîne** d'étapes, dans un ordre défini, avant d'atteindre le traitement final (souvent l'appel réseau réel ou l'exécution du handler métier).

```
Requête entrante
      ↓
  Étape 1 (ex: authentification)
      ↓
  Étape 2 (ex: logging)
      ↓
  Étape 3 (ex: validation)
      ↓
  Traitement réel
      ↓
[La réponse peut retraverser la chaîne dans l'ordre inverse]
```

Chaque étape a le pouvoir de :
- **laisser passer** la requête vers l'étape suivante (éventuellement modifiée) ;
- **court-circuiter** la chaîne (ex : renvoyer une erreur 401 sans jamais atteindre le serveur) ;
- **transformer** la réponse au retour.

### Pourquoi ce pattern existe (raison générale, pas Angular)

Sans intercepteur, une préoccupation transverse (authentification, logging, gestion d'erreur, cache) doit être dupliquée dans **chaque appel individuel**. L'intercepteur centralise cette logique en un seul endroit, appliqué **automatiquement** à toutes les requêtes qui passent par le système — c'est une application du principe de **separation of concerns**.

**Cas d'usage universels**, valables dans n'importe quel langage/framework :
- Injection d'un token d'authentification dans chaque requête sortante.
- Logging centralisé des entrées/sorties.
- Gestion globale des erreurs (redirection, retry, normalisation du format d'erreur).
- Mesure de performance / tracing distribué.
- Cache de réponses.
- Transformation de format (adapter une API legacy à un format interne).

### Exemple générique en TypeScript (indépendant de tout framework)

Implémentation minimale du pattern Chain of Responsibility, en TypeScript pur — c'est exactement ce mécanisme qu'implémentent Express (`app.use()`), Axios (`interceptors`), et Angular (`HttpInterceptorFn`), chacun avec sa propre API, mais la même logique de composition en chaîne :

```typescript
interface RequeteHttp {
  url: string;
  headers: Record<string, string>;
  body?: unknown;
}

interface ReponseHttp {
  status: number;
  body: unknown;
}

// Un intercepteur reçoit la requête + la fonction "suivante" de la chaîne,
// et retourne une Promise de réponse
type Intercepteur = (
  req: RequeteHttp,
  next: (req: RequeteHttp) => Promise<ReponseHttp>
) => Promise<ReponseHttp>;

function composerChaine(
  intercepteurs: Intercepteur[],
  traitementFinal: (req: RequeteHttp) => Promise<ReponseHttp>
): (req: RequeteHttp) => Promise<ReponseHttp> {
  // On construit la chaîne à l'envers pour que chaque intercepteur
  // enveloppe le suivant
  return intercepteurs.reduceRight<(req: RequeteHttp) => Promise<ReponseHttp>>(
    (suivant, intercepteur) => (req) => intercepteur(req, suivant),
    traitementFinal
  );
}

// --- Exemple d'utilisation ---
const ajoutToken: Intercepteur = async (req, next) => {
  const token = "abc123";
  return next({ ...req, headers: { ...req.headers, Authorization: `Bearer ${token}` } });
};

const logging: Intercepteur = async (req, next) => {
  console.log("→ Requête envoyée :", req.url);
  const reponse = await next(req);
  console.log("← Réponse reçue :", reponse.status);
  return reponse;
};

const envoiReel = async (req: RequeteHttp): Promise<ReponseHttp> => {
  // ici on appellerait fetch(), axios, etc.
  return { status: 200, body: { ok: true } };
};

const chaine = composerChaine([ajoutToken, logging], envoiReel);
chaine({ url: "/api/users", headers: {} }).then((res) => console.log(res));
```

### Exemple Angular 22

Depuis Angular 15, la forme recommandée est **fonctionnelle** (la forme basée sur une classe + `HTTP_INTERCEPTORS` reste supportée mais est considérée comme historique) :

```typescript
// auth.interceptor.ts
import { HttpInterceptorFn } from '@angular/common/http';

export const authInterceptor: HttpInterceptorFn = (req, next) => {
  const token = localStorage.getItem('token');
  const cloned = req.clone({
    setHeaders: { Authorization: `Bearer ${token}` }
  });
  return next(cloned); // passe la requête modifiée à l'intercepteur suivant
};
```

```typescript
// error.interceptor.ts
import { HttpInterceptorFn } from '@angular/common/http';
import { catchError, throwError } from 'rxjs';
import { inject } from '@angular/core';
import { Router } from '@angular/router';

export const errorInterceptor: HttpInterceptorFn = (req, next) => {
  const router = inject(Router);
  return next(req).pipe(
    catchError((error) => {
      if (error.status === 401) {
        router.navigate(['/login']);
      }
      return throwError(() => error);
    })
  );
};
```

Enregistrement (Angular 22, standalone par défaut) :

```typescript
// app.config.ts
import { ApplicationConfig } from '@angular/core';
import { provideHttpClient, withInterceptors } from '@angular/common/http';

export const appConfig: ApplicationConfig = {
  providers: [
    provideHttpClient(withInterceptors([authInterceptor, errorInterceptor]))
  ]
};
```

**Point technique important** : un `HttpRequest` Angular est **immuable** — impossible de faire `req.headers.set(...)` directement. Il faut systématiquement passer par `req.clone({...})`, qui retourne une **nouvelle instance**. C'est volontaire, pour éviter les effets de bord si le même objet requête était partagé ailleurs.

**Ordre d'exécution** : l'ordre déclaré dans `withInterceptors([...])` définit l'ordre de traversée **à l'aller** ; au retour (traitement de la réponse), c'est l'**ordre inverse**. Avec l'exemple ci-dessus : `auth → error` à l'aller, `error → auth` au retour.

**Depuis Angular 22.1** : les intercepteurs sont désormais automatiquement `untracked` vis-à-vis des Signals — un `effect()` qui déclenche une requête HTTP ne se re-déclenche plus par erreur à cause d'un signal lu à l'intérieur d'un intercepteur.

### Exemple React 19.2 (via axios — React n'a pas de HttpClient intégré)

React est une bibliothèque UI, pas un framework HTTP — il n'existe aucun concept d'intercepteur natif. Le pattern s'implémente presque toujours via **axios**, dont l'API d'intercepteurs est strictement identique, peu importe le framework autour :

```typescript
// api/axiosInstance.ts
import axios from 'axios';

const api = axios.create({ baseURL: '/api' });

api.interceptors.request.use((config) => {
  const token = localStorage.getItem('token');
  if (token) config.headers.Authorization = `Bearer ${token}`;
  return config;
});

api.interceptors.response.use(
  (response) => response,
  (error) => {
    if (error.response?.status === 401) window.location.href = '/login';
    return Promise.reject(error);
  }
);

export default api;
```

```tsx
// useUsers.ts — hook custom consommant l'instance interceptée (React 19.2)
import { useEffect, useState } from 'react';
import api from './api/axiosInstance';

interface User {
  id: number;
  name: string;
}

function useUsers() {
  const [users, setUsers] = useState<User[]>([]);
  useEffect(() => {
    api.get<User[]>('/users').then((res) => setUsers(res.data));
  }, []);
  return users;
}
```

### Exemple Vue 3.5 (via axios, consommé en Composition API)

Même instance axios interceptée que React — c'est le même code, seule la façon de la consommer côté UI change (composable au lieu de hook) :

```typescript
// composables/useUsers.ts
import { ref, onMounted } from 'vue';
import api from '../api/axiosInstance'; // même fichier que l'exemple React

interface User {
  id: number;
  name: string;
}

export function useUsers() {
  const users = ref<User[]>([]);
  onMounted(async () => {
    const { data } = await api.get<User[]>('/users');
    users.value = data;
  });
  return { users };
}
```

**À retenir** : contrairement à Angular où `HttpInterceptor` est une brique du framework injectée via DI, React et Vue n'ont **aucun concept d'intercepteur natif** — ils héritent de celui d'axios (ou d'un wrapper `fetch` maison). C'est la bibliothèque HTTP choisie qui porte le concept, pas le framework UI.

---

## 2. Les clients HTTP : le concept général, puis fetch / axios / HttpClient

### C'est quoi, en général

Un **client HTTP** est simplement un composant logiciel qui construit et envoie une requête HTTP, puis expose la réponse au code appelant. Trois niveaux existent dans à peu près tous les langages :

1. **Primitive bas niveau du langage/plateforme** (souvent bloquante ou callback-based à l'origine) — ex : `HttpURLConnection` en Java, `requests`/`httpx` en Python, `curl`/sockets en C.
2. **API native moderne, généralement basée sur les Promises/futures** — ex : `fetch` en JS/TS, `HttpClient` (`java.net.http`) en Java 11+, `httpx.AsyncClient` en Python.
3. **Bibliothèques tierces** qui ajoutent des fonctionnalités par-dessus (retry, intercepteurs, timeout simple, parsing auto) — ex : `axios` en JS/TS, `OkHttp`/`Retrofit` en Java, `Guzzle` en PHP.

Angular `HttpClient` se situe dans une **4ᵉ catégorie propre à Angular** : une couche d'abstraction réactive (RxJS) construite **par-dessus** l'une des API natives du navigateur (1 ou 2), pas une alternative de même niveau.

### Comparatif `fetch` vs `axios` vs Angular `HttpClient`

| Aspect | `fetch` | `axios` | Angular `HttpClient` |
|---|---|---|---|
| **Nature** | Web API native du navigateur (+ native Node 18+) | Bibliothèque tierce (npm), wrapper XHR (navigateur) ou `http`/`https` (Node) | Module Angular, wrapper RxJS autour de `fetch` (défaut depuis Angular 22) ou XHR |
| **Retour** | `Promise<Response>` | `Promise<AxiosResponse>` | `Observable<T>` |
| **Parsing JSON** | Manuel : `await response.json()` (2 étapes) | Automatique : `response.data` déjà parsé | Automatique, typé génériquement (`http.get<User[]>(url)`) |
| **Gestion des erreurs HTTP** | Ne rejette **jamais** sur un 404/500 — il faut vérifier `response.ok` soi-même | Rejette automatiquement la Promise si status hors 2xx | Émet une erreur RxJS (`catchError`) automatiquement si status hors 2xx |
| **Annulation** | `AbortController` (verbeux) | `AbortController` | `unsubscribe()` — natif, souvent via `takeUntilDestroyed()` |
| **Intercepteurs** | Aucun natif — à coder à la main | Natif (`axios.interceptors`) | Natif (`HttpInterceptorFn`, chaînable via DI) |
| **Retry / debounce / combinaison** | Aucun — à coder à la main | Aucun natif (plugins tiers) | Natif via opérateurs RxJS (`retry()`, `debounceTime()`, `switchMap()`, `forkJoin()`) |
| **Progress upload** | Complexe (`ReadableStream`) | Natif (`onUploadProgress`) | `reportUploadProgress` — **non supporté par `FetchBackend`**, nécessite `withXhr()` |
| **Timeout** | Aucun natif | Natif (`timeout: 5000`) | Opérateur RxJS `timeout()` |
| **Usage typique** | Vanilla TS, petits scripts | React/Vue/Node — hors écosystème Angular | Exclusivement Angular |

### Piège classique universel avec `fetch`

```typescript
// fetch : PAS d'erreur levée sur un 404 !
const response = await fetch('/api/inexistant');
console.log(response.ok);     // false
console.log(response.status); // 404
// Sans vérification manuelle de response.ok, le code continue comme si de rien n'était
```

### Ce qui change avec Angular 22 : `FetchBackend` devient le défaut

Jusqu'à Angular 21, `HttpClient` utilisait `XMLHttpRequest` par défaut, et il fallait activer `withFetch()` explicitement pour passer sur `fetch`. **Depuis Angular 22 (juin 2026), c'est l'inverse** : `fetch` est le backend par défaut, et `withFetch()` est devenu inutile (déprécié).

```typescript
// app.config.ts — Angular 22, HttpClient utilise fetch par défaut
import { ApplicationConfig } from '@angular/core';
import { provideHttpClient } from '@angular/common/http';

export const appConfig: ApplicationConfig = {
  providers: [
    provideHttpClient() // FetchBackend en interne, aucune option nécessaire
  ]
};
```

Pour revenir à l'ancien comportement basé sur `XMLHttpRequest` (par exemple si besoin du `reportUploadProgress`, non supporté par `FetchBackend`) :

```typescript
import { provideHttpClient, withXhr } from '@angular/common/http';

export const appConfig: ApplicationConfig = {
  providers: [
    provideHttpClient(withXhr())
  ]
};
```

### Architecture interne de `HttpClient` (Angular 22)

```
Code applicatif (service Angular)
        ↓
   HttpClient (API publique, retourne des Observable)
        ↓
   HttpInterceptor(s) (chaîne d'intercepteurs)
        ↓
   HttpHandler (orchestre la chaîne)
        ↓
   HttpBackend
        ↓
   FetchBackend (fetch, PAR DÉFAUT) ou XhrFactory (XMLHttpRequest, via withXhr())
        ↓
   Web API du navigateur (hors thread JS)
```

- Avec `FetchBackend` (défaut) : la résolution passe par la **microtask queue**.
- Avec XHR (`withXhr()`) : les callbacks passent par la **macrotask queue**.
- L'argument historique pour `fetch` reste valable : le support **SSR**. `fetch` est nativement disponible dans Node depuis la version 18, alors que `XMLHttpRequest` n'existe pas côté Node et nécessitait un polyfill.

### Timeline complète d'une requête `HttpClient` en Angular 22

```
1. this.http.get(url).subscribe(...)     → code synchrone, retourne un Observable froid
2. subscribe() déclenche l'exécution réelle → fetch() appelé (backend par défaut)
3. Requête déléguée au thread réseau du navigateur → thread JS libéré immédiatement
4. Code Angular continue (autre logique)
5. Réponse réseau reçue par le navigateur (thread séparé)
6. Callback placé dans la microtask queue (résolution de la Promise fetch)
7. Call stack vide → event loop exécute le callback
8. FetchBackend transforme la réponse en émission RxJS
9. .subscribe(data => ...) reçoit la donnée → mise à jour du composant
```

Note sur la mise à jour de l'UI : historiquement, c'était **Zone.js** qui interceptait la fin des callbacks asynchrones pour déclencher la *change detection*. Angular 22 généralise l'architecture **Signal-first** : la stratégie `OnPush` devient le comportement par défaut pour tout composant qui ne précise rien, et un état porté par des `signal()` déclenche une mise à jour ciblée sans dépendre de Zone.js — cohérent avec le mode **zoneless**, de plus en plus la direction par défaut du framework.

### Exemple React 19.2 : pas de couche d'abstraction — `fetch` ou `axios` directement

React ne fournit **aucun client HTTP intégré** (contrairement à Angular) — l'appel réseau se fait directement avec `fetch`/`axios`, généralement dans un `useEffect` ou via une lib de data-fetching (React Query, SWR). En React 19.2, le hook `use()` (stable depuis la 19.0) permet aussi de consommer directement une Promise en lecture, à combiner avec `<Suspense>` :

```tsx
import { useEffect, useState } from 'react';

interface User {
  id: number;
  name: string;
}

function UserList() {
  const [users, setUsers] = useState<User[]>([]);

  useEffect(() => {
    const controller = new AbortController();
    fetch('/api/users', { signal: controller.signal })
      .then((res) => res.json() as Promise<User[]>)
      .then(setUsers);
    return () => controller.abort(); // annulation à la destruction du composant
  }, []);

  return <ul>{users.map((u) => <li key={u.id}>{u.name}</li>)}</ul>;
}
```

Avec **React Query** (le plus proche en confort de ce que `HttpClient`/`httpResource` offrent nativement à Angular — cache, retry, état de chargement) :

```tsx
import { useQuery } from '@tanstack/react-query';
import api from './api/axiosInstance';

interface User {
  id: number;
  name: string;
}

function UserList() {
  const { data: users, isLoading } = useQuery<User[]>({
    queryKey: ['users'],
    queryFn: () => api.get<User[]>('/users').then((r) => r.data),
  });
  if (isLoading) return <p>Chargement...</p>;
  return <ul>{users?.map((u) => <li key={u.id}>{u.name}</li>)}</ul>;
}
```

### Exemple Vue 3.5 : même logique, syntaxe Composition API

```typescript
// composables/useUsers.ts
import { ref, onMounted, onUnmounted } from 'vue';

interface User {
  id: number;
  name: string;
}

export function useUsers() {
  const users = ref<User[]>([]);
  const controller = new AbortController();

  onMounted(async () => {
    const res = await fetch('/api/users', { signal: controller.signal });
    users.value = (await res.json()) as User[];
  });
  onUnmounted(() => controller.abort()); // annulation au démontage du composant

  return { users };
}
```

Avec **VueUse** (`useFetch`, équivalent conceptuel de React Query pour Vue) :

```typescript
import { useFetch } from '@vueuse/core';

interface User {
  id: number;
  name: string;
}

const { data: users, isFetching } = useFetch('/api/users').json<User[]>();
```

**À retenir** : Angular est le seul des trois à imposer **une couche HTTP officielle et unifiée** (`HttpClient`) dans le framework lui-même. React et Vue laissent ce choix à l'écosystème (fetch natif, axios, React Query/SWR, VueUse) — plus de liberté, mais aussi plus de décisions à prendre soi-même (annulation, cache, retry ne sont pas fournis par défaut).

---

## 3. `Observable` vs `Promise` : le concept général, puis Angular/React/Vue

### C'est quoi, en général

Il s'agit de **deux modèles distincts pour représenter une valeur qui arrivera plus tard** (asynchrone) — mais qui répondent à des besoins différents et existent, sous des noms variés, dans la plupart des écosystèmes :

| Concept | Représente | Équivalents dans d'autres langages/écosystèmes |
|---|---|---|
| **Promise / Future** | **Une seule** valeur future, résolue une fois | `Future` (Java, Scala), `CompletableFuture` (Java), `Task` (C#), `asyncio.Future` (Python) |
| **Observable / Stream** | **Zéro, une, ou plusieurs** valeurs dans le temps | `Flux`/`Mono` (Reactor, Java), `Flow` (Kotlin), `IObservable` (Rx.NET), `Stream` (RxJava), Generators/AsyncIterators (JS/TS/Python) |

Le point commun conceptuel à toutes les implémentations "Observable/Stream" est le **pattern Observer** : un producteur (`Observable`) notifie un ou plusieurs consommateurs (`Observer`/`Subscriber`) à chaque nouvelle donnée disponible, jusqu'à ce que le flux se termine ou soit annulé.

### Les deux différences fondamentales, indépendamment du langage

1. **Cardinalité** : une Promise/Future résout **une fois**. Un Observable/Stream peut émettre **N fois** dans le temps (idéal pour des événements répétés : clics, messages WebSocket, valeurs de capteur, changements de formulaire).
2. **Eager vs Lazy** : une Promise démarre son exécution **dès sa création**, qu'on l'observe ou non (`eager`). Un Observable classique ne fait **rien** tant que personne ne s'y abonne (`lazy`) — chaque abonnement peut même redéclencher toute l'opération depuis le début (comportement "froid").

### Démonstration du eager vs lazy, en TypeScript

```typescript
import { Observable } from 'rxjs';

// Promise : eager — s'exécute IMMÉDIATEMENT à la création
const promise = new Promise<number>((resolve) => {
  console.log("Je m'exécute tout de suite !");
  resolve(42);
});
// Le message s'affiche même sans .then()

// Observable : lazy — ne s'exécute QUE si on s'abonne
const observable = new Observable<number>((subscriber) => {
  console.log("Je ne m'exécute que si on m'observe !");
  subscriber.next(42);
});
// Rien ne s'affiche tant qu'il n'y a pas de subscribe()
observable.subscribe((val) => console.log(val));
```

### Pourquoi ce choix compte, indépendamment du framework

- Une API qui représente un **événement DOM**, un **flux WebSocket**, ou une **valeur qui change dans le temps** (température, position GPS, texte d'un champ) a fondamentalement besoin d'un modèle **multi-émissions** — une Promise ne peut structurellement pas représenter ça (elle se résout et c'est terminé).
- Une API qui représente **une seule opération ponctuelle** (lire un fichier une fois, faire un calcul) est plus simplement représentée par une Promise/Future : plus légère conceptuellement, avec `async`/`await` nativement supporté dans la plupart des langages modernes.

### Exemple Angular 22 : pourquoi `HttpClient` utilise `Observable`

```typescript
import { Component, inject, DestroyRef, signal } from '@angular/core';
import { HttpClient } from '@angular/common/http';
import { takeUntilDestroyed } from '@angular/core/rxjs-interop';

interface User {
  id: number;
  name: string;
}

@Component({ selector: 'app-user-list', template: `...` })
export class UserListComponent {
  private http = inject(HttpClient);
  private destroyRef = inject(DestroyRef);
  users = signal<User[]>([]);

  loadUsers(): void {
    this.http.get<User[]>('/api/users')
      .pipe(takeUntilDestroyed(this.destroyRef)) // annule si le composant est détruit
      .subscribe((users) => this.users.set(users));
  }
}
```

```typescript
// switchMap : annule automatiquement la requête précédente à chaque nouvelle frappe
import { debounceTime, switchMap } from 'rxjs';

searchControl.valueChanges.pipe(
  debounceTime(300),
  switchMap((query) => this.http.get<Result[]>(`/api/search?q=${query}`))
).subscribe((results) => this.results.set(results));
```

Ce comportement (annulation, debounce, switch automatique) est **trivial en RxJS**, et laborieux à reproduire à la main avec des Promises seules — c'est la vraie justification du choix d'Angular.

### La nuance Angular 22 : la Resource API arrive en renfort, pas en remplacement

Angular 22 stabilise la **Resource API** (`resource()`, `rxResource()`, `httpResource()`) : une troisième façon de représenter une donnée asynchrone, pensée nativement pour les **Signals** — avec état de chargement, erreur, et rechargement intégrés dans un seul objet réactif.

```typescript
import { Component, signal } from '@angular/core';
import { httpResource } from '@angular/common/http';

interface User {
  id: number;
  name: string;
}

@Component({ selector: 'app-user-list', template: `...` })
export class UserListComponent {
  selectedPage = signal(1);
  usersResource = httpResource<User[]>(() => `/api/users?page=${this.selectedPage()}`);
  // usersResource.value(), usersResource.isLoading(), usersResource.error() — tout est un Signal
}
```

L'objectif n'est pas de remplacer RxJS partout, mais de simplifier le cas courant "charger une donnée liée à un Signal", sans avoir à écrire un `Observable` + `subscribe()` + gestion manuelle du unsubscribe pour un cas simple d'affichage.

### Convertir de l'un à l'autre (toujours valable en Angular 22)

```typescript
import { from, firstValueFrom } from 'rxjs';
import { toSignal } from '@angular/core/rxjs-interop';

// Promise → Observable
const obs$ = from(fetch('/api/data'));

// Observable → Promise (récupère la 1ère valeur émise, puis se désabonne)
const data = await firstValueFrom(this.http.get('/api/data'));

// Observable → Signal (Angular, natif depuis Angular 16+)
const usersSignal = toSignal(this.http.get<User[]>('/api/users'), { initialValue: [] as User[] });
```

### Exemple React 19.2 : `Observable` (RxJS) est utilisable, mais rarement idiomatique

RxJS n'est **pas propre à Angular** — c'est une bibliothèque TS/JS pure, utilisable dans n'importe quel contexte. En React, ça reste possible mais peu courant, car l'écosystème React privilégie plutôt les Promises + hooks :

```tsx
import { useEffect, useState } from 'react';
import { fromEvent } from 'rxjs';
import { debounceTime, switchMap } from 'rxjs/operators';

interface Result {
  id: number;
  label: string;
}

function SearchBox() {
  const [results, setResults] = useState<Result[]>([]);

  useEffect(() => {
    const input = document.getElementById('search') as HTMLInputElement;
    const sub = fromEvent<InputEvent>(input, 'input')
      .pipe(
        debounceTime(300),
        switchMap((e) =>
          fetch(`/api/search?q=${(e.target as HTMLInputElement).value}`).then(
            (r) => r.json() as Promise<Result[]>
          )
        )
      )
      .subscribe(setResults);
    return () => sub.unsubscribe(); // annulation propre au démontage
  }, []);

  return (
    <>
      <input id="search" />
      <ul>{results.map((r) => <li key={r.id}>{r.label}</li>)}</ul>
    </>
  );
}
```

L'idiome React natif pour ce même besoin (debounce + annulation de la requête précédente) repose sur des Promises + `AbortController`, sans RxJS — combiné en 19.2 avec `useEffectEvent` pour lire la dernière valeur de `query` sans la mettre dans le tableau de dépendances :

```tsx
import { useEffect, useState } from 'react';

function useDebouncedSearch(query: string, delay = 300) {
  const [results, setResults] = useState<Result[]>([]);

  useEffect(() => {
    const controller = new AbortController();
    const timeout = setTimeout(() => {
      fetch(`/api/search?q=${query}`, { signal: controller.signal })
        .then((r) => r.json() as Promise<Result[]>)
        .then(setResults);
    }, delay);
    return () => {
      clearTimeout(timeout);
      controller.abort();
    };
  }, [query, delay]);

  return results;
}
```

### Exemple Vue 3.5 : Composition API + VueUse plutôt que RxJS

Vue propose nativement un système réactif (`ref`, `watch`) qui couvre une bonne partie des cas où Angular utiliserait un `Observable`. Pour le même besoin (debounce + annulation) :

```typescript
import { ref, watch } from 'vue';
import { useDebounceFn } from '@vueuse/core';

interface Result {
  id: number;
  label: string;
}

export function useSearch() {
  const query = ref('');
  const results = ref<Result[]>([]);

  const search = useDebounceFn(async (q: string) => {
    const res = await fetch(`/api/search?q=${q}`);
    results.value = (await res.json()) as Result[];
  }, 300);

  watch(query, (q) => search(q));
  return { query, results };
}
```

**À retenir** : `Observable`/RxJS est **au cœur** d'Angular (utilisé partout : `HttpClient`, `Router`, `FormControl.valueChanges`), alors que React et Vue s'appuient par défaut sur des **Promises + primitives réactives propres au framework** (state/hooks pour React, `ref`/`watch`/Composition API pour Vue) pour couvrir les mêmes besoins de flux multi-valeurs — RxJS y reste une option externe, jamais un pilier du framework.

---

## Récapitulatif des trois concepts

| Concept | Le principe général | Angular 22 | React 19.2 | Vue 3.5 |
|---|---|---|---|---|
| Intercepteur | Chain of Responsibility appliquée aux requêtes/réponses | `HttpInterceptorFn` + `withInterceptors()`, natif au framework, `untracked` depuis 22.1 | Aucun natif — via `axios.interceptors` | Aucun natif — via `axios.interceptors` |
| Client HTTP | Construire/envoyer une requête, exposer la réponse | `HttpClient` (RxJS) au-dessus de `FetchBackend`, par défaut depuis Angular 22, imposé par le framework | Aucun officiel — `fetch`/`axios` + React Query/SWR | Aucun officiel — `fetch`/`axios` + VueUse `useFetch` |
| Promise vs Observable/Stream | Valeur unique vs flux de valeurs dans le temps, eager vs lazy | `Observable` central au framework ; `resource()`/`httpResource()` pour l'asynchrone Signal-first | RxJS possible mais marginal — Promises + hooks (`useEffect`, `useEffectEvent`) + `AbortController` privilégiés | RxJS possible mais marginal — réactivité native (`ref`/`watch`) + VueUse privilégiés |
