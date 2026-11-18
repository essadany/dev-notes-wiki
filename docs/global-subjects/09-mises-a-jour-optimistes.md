# 9. Mises à jour optimistes

> 🎯 **Résumé.** Rend l'interface plus réactive en appliquant immédiatement une modification côté client avant la confirmation serveur, puis en annulant en cas d'échec. Ce pattern repose sur un snapshot du cache, un rollback fiable et une réconciliation finale avec le serveur. Le piège principal est de l'utiliser pour des opérations risquées ou peu susceptibles de réussir, comme un paiement.

---

## 🧭 1. Contexte

**Principe** : mettre à jour l'UI avant la réponse du serveur, annuler si échec.

TanStack Query : `onMutate` (snapshot + modification du cache) → `onError` (rollback) → `onSettled` (`invalidateQueries`).
Angular/NgRx : mise à jour du store puis rollback sur l'action d'échec.

> ⚠️ **Pièges.** à réserver aux actions très probablement réussies (like, case à cocher), pas aux paiements.

---

## 💻 2. Exemples techniques

Exemple React avec TanStack Query pour un "like" optimiste avec snapshot, rollback et invalidation finale.

```tsx
import { useMutation, useQueryClient } from '@tanstack/react-query';

type Post = { id: string; title: string; liked: boolean };

async function toggleLike(postId: string): Promise<Post> {
  const res = await fetch(`/api/posts/${postId}/like`, { method: 'POST' });
  if (!res.ok) throw new Error('Échec du like');
  return res.json();
}

export function useToggleLike(postId: string) {
  const queryClient = useQueryClient();
  const queryKey = ['post', postId];

  return useMutation({
    mutationFn: () => toggleLike(postId),
    onMutate: async () => {
      await queryClient.cancelQueries({ queryKey });
      const previous = queryClient.getQueryData<Post>(queryKey);
      if (previous) {
        queryClient.setQueryData<Post>(queryKey, { ...previous, liked: !previous.liked });
      }
      return { previous };
    },
    onError: (_err, _vars, context) => {
      if (context?.previous) queryClient.setQueryData(queryKey, context.previous);
    },
    onSettled: () => {
      queryClient.invalidateQueries({ queryKey });
    },
  });
}
```

Exemple Vue avec TanStack Vue Query pour cocher une tâche immédiatement tout en garantissant le rollback.

```ts
import { useMutation, useQueryClient } from '@tanstack/vue-query'

type Todo = { id: string; label: string; done: boolean }

export function useToggleTodo(todoId: string) {
  const queryClient = useQueryClient()
  const queryKey = ['todo', todoId]

  return useMutation({
    mutationFn: async (done: boolean) => {
      const res = await fetch(`/api/todos/${todoId}`, {
        method: 'PATCH',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ done }),
      })
      if (!res.ok) throw new Error('Mise à jour impossible')
      return (await res.json()) as Todo
    },
    onMutate: async (done) => {
      await queryClient.cancelQueries({ queryKey })
      const previous = queryClient.getQueryData<Todo>(queryKey)
      if (previous) queryClient.setQueryData<Todo>(queryKey, { ...previous, done })
      return { previous }
    },
    onError: (_error, _done, context) => {
      if (context?.previous) queryClient.setQueryData(queryKey, context.previous)
    },
    onSettled: () => {
      queryClient.invalidateQueries({ queryKey })
    },
  })
}
```

Exemple Angular avec NgRx Store : la mise à jour optimiste part du composant/action et le rollback est déclenché sur l'action d'échec.

```ts
// posts.actions.ts
import { createActionGroup, props } from '@ngrx/store';
import { Post } from './posts.models';

export const PostsActions = createActionGroup({
  source: 'Posts',
  events: {
    'Toggle Like': props<{ post: Post }>(),
    'Toggle Like Success': props<{ post: Post }>(),
    'Toggle Like Failure': props<{ previous: Post; error: string }>(),
  },
});

// posts.reducer.ts (extrait)
on(PostsActions.toggleLike, (state, { post }) => ({
  ...state,
  entities: {
    ...state.entities,
    [post.id]: { ...state.entities[post.id], liked: !post.liked },
  },
})),
on(PostsActions.toggleLikeFailure, (state, { previous }) => ({
  ...state,
  entities: { ...state.entities, [previous.id]: previous },
})),
```

---

## 🔗 3. Voir aussi

- [Sujet 1 — Cache des données serveur et pagination](01-cache-donnees-serveur-pagination.md)
- [Sujet 8 — Gestion d'erreurs côté front](08-erreurs-front.md)
- [Sujet 10 — Temps réel](10-temps-reel.md)
