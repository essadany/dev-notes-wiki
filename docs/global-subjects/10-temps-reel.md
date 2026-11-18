# 10. Temps réel

> 🎯 **Résumé.** Permet de refléter rapidement les changements côté client via polling, SSE ou WebSocket selon le besoin de fréquence et de bidirectionnalité. Le choix de la technique dépend du sens des échanges, de la latence acceptable et du coût opérationnel. Le piège principal est d'oublier la reconnexion avec backoff et le nettoyage des connexions au démontage des composants.

---

## 🧭 1. Contexte

| Technique | Quand |
|---|---|
| Polling (`refetchInterval`) | simple, faible fréquence |
| **SSE** (Server-Sent Events) | flux serveur → client unidirectionnel (notifications, progression) |
| **WebSocket** | bidirectionnel (chat, collaboration) |

Côté front : `EventSource` / `WebSocket` natifs, Socket.IO client, RxJS `webSocket` (Angular). Les messages reçus peuvent alimenter le cache : `queryClient.setQueryData` ou `invalidateQueries`.

> ⚠️ **Pièges.** reconnexion automatique avec backoff ; nettoyage des connexions au démontage du composant.

---

## 💻 2. Exemples techniques

Exemple React avec TanStack Query en polling simple pour un tableau de bord à faible fréquence.

```tsx
import { useQuery } from '@tanstack/react-query';

type Metrics = { activeUsers: number; errorRate: number };

export function DashboardMetrics() {
  const { data } = useQuery<Metrics>({
    queryKey: ['metrics'],
    queryFn: async () => {
      const res = await fetch('/api/metrics');
      if (!res.ok) throw new Error('Impossible de charger les métriques');
      return res.json();
    },
    refetchInterval: 15_000,
    refetchIntervalInBackground: false,
  });

  return (
    <section>
      <p>Utilisateurs actifs : {data?.activeUsers ?? '—'}</p>
      <p>Taux d'erreur : {data?.errorRate ?? '—'}</p>
    </section>
  );
}
```

Exemple Vue avec `EventSource` (SSE), reconnexion exponentielle avec backoff et nettoyage au démontage.

```vue
<script setup lang="ts">
import { onMounted, onUnmounted, ref } from 'vue'

const notifications = ref<string[]>([])
let eventSource: EventSource | null = null
let retryDelay = 1000
let reconnectTimer: ReturnType<typeof setTimeout> | null = null

function connect() {
  eventSource = new EventSource('/api/notifications/stream')
  eventSource.onopen = () => {
    retryDelay = 1000
  }
  eventSource.onmessage = (event) => {
    notifications.value.unshift(event.data)
  }
  eventSource.onerror = () => {
    eventSource?.close()
    reconnectTimer = setTimeout(connect, retryDelay)
    retryDelay = Math.min(retryDelay * 2, 30_000)
  }
}

onMounted(connect)
onUnmounted(() => {
  if (reconnectTimer) clearTimeout(reconnectTimer)
  eventSource?.close()
})
</script>

<template>
  <ul>
    <li v-for="(n, i) in notifications" :key="i">{{ n }}</li>
  </ul>
</template>
```

Exemple Angular avec RxJS `webSocket` pour un canal bidirectionnel et mise à jour du cache via `queryClient.setQueryData`.

```ts
import { Injectable, OnDestroy, inject } from '@angular/core';
import { webSocket, WebSocketSubject } from 'rxjs/webSocket';
import { retry, timer } from 'rxjs';
import { QueryClient } from '@tanstack/angular-query-experimental';

type ChatMessage = { id: string; roomId: string; author: string; body: string };

@Injectable({ providedIn: 'root' })
export class ChatRealtimeService implements OnDestroy {
  private readonly queryClient = inject(QueryClient);
  private socket$: WebSocketSubject<ChatMessage> | null = null;

  connect(roomId: string): void {
    this.socket$ = webSocket<ChatMessage>(`wss://api.example.com/ws/rooms/${roomId}`);
    this.socket$
      .pipe(retry({ delay: (_err, retryCount) => timer(Math.min(1000 * 2 ** retryCount, 30_000)) }))
      .subscribe((message) => {
        this.queryClient.setQueryData<ChatMessage[]>(['room', roomId, 'messages'], (old = []) => [
          ...old,
          message,
        ]);
      });
  }

  send(message: ChatMessage): void {
    this.socket$?.next(message);
  }

  ngOnDestroy(): void {
    this.socket$?.complete();
  }
}
```

---

## 🔗 3. Voir aussi

- [Sujet 1 — Cache des données serveur et pagination](01-cache-donnees-serveur-pagination.md)
- [Sujet 9 — Mises à jour optimistes](09-mises-a-jour-optimistes.md)
- [Sujet 22 — Jobs asynchrones et files de tâches](22-jobs-asynchrones-files.md)
