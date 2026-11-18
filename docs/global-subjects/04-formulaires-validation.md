# 4. Formulaires et validation

> 🎯 **Résumé.** Garantit une saisie correcte et une bonne expérience utilisateur en validant les données côté front tout en considérant le back comme source de vérité. L'usage d'un schéma partagé (par exemple Zod) aide à aligner les règles entre client et serveur. Le piège principal est d'oublier la revalidation serveur, de mal remonter les erreurs 422 par champ ou de permettre les doubles soumissions.

---

## 🧭 1. Contexte

**Principe** : valider côté front pour l'UX, **toujours** revalider côté back. Un schéma partagé (Zod/Yup) évite de dupliquer les règles.

| Stack | Solutions |
|---|---|
| React | React Hook Form + Zod, Formik |
| Vue | VeeValidate + Zod, FormKit |
| Angular | **Signal Forms** (Angular 21+, à privilégier pour un nouveau formulaire) ; Reactive Forms (`Validators`, validators custom async) pour les formulaires complexes ou existants ; Template-driven pour le très simple |
| Svelte | Superforms (SvelteKit) |

> ⚠️ **Pièges.** afficher les erreurs du back (422) champ par champ ; désactiver le bouton pendant l'envoi (double soumission) ; validation async avec debounce.

---

## 💻 2. Exemples techniques

Exemple React avec React Hook Form et Zod, incluant la remontée d'erreurs serveur 422 champ par champ.

```tsx
import { zodResolver } from '@hookform/resolvers/zod';
import { useForm } from 'react-hook-form';
import { z } from 'zod';

const schema = z.object({
  email: z.string().email('Email invalide'),
  name: z.string().min(2, 'Nom trop court'),
});

type FormValues = z.infer<typeof schema>;

export function SignupForm() {
  const {
    register,
    handleSubmit,
    setError,
    formState: { errors, isSubmitting },
  } = useForm<FormValues>({ resolver: zodResolver(schema) });

  const onSubmit = async (values: FormValues) => {
    const res = await fetch('/api/users', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(values),
    });

    if (res.status === 422) {
      const body = await res.json();
      Object.entries<string[]>(body.errors ?? {}).forEach(([field, messages]) => {
        setError(field as keyof FormValues, { message: messages[0] });
      });
      return;
    }

    if (!res.ok) throw new Error('Erreur serveur');
  };

  return (
    <form onSubmit={handleSubmit(onSubmit)}>
      <input {...register('email')} aria-invalid={!!errors.email} />
      {errors.email && <p role="alert">{errors.email.message}</p>}

      <input {...register('name')} aria-invalid={!!errors.name} />
      {errors.name && <p role="alert">{errors.name.message}</p>}

      <button type="submit" disabled={isSubmitting}>Créer le compte</button>
    </form>
  );
}
```

Exemple Vue avec VeeValidate et Zod pour une validation déclarative et un envoi protégé contre la double soumission.

```vue
<script setup lang="ts">
import { useForm, useField } from 'vee-validate'
import { toTypedSchema } from '@vee-validate/zod'
import { z } from 'zod'

const schema = toTypedSchema(
  z.object({
    email: z.string().email('Email invalide'),
    password: z.string().min(8, '8 caractères minimum'),
  }),
)

const { handleSubmit, isSubmitting, setFieldError } = useForm({ validationSchema: schema })
const { value: email, errorMessage: emailError } = useField<string>('email')
const { value: password, errorMessage: passwordError } = useField<string>('password')

const onSubmit = handleSubmit(async (values) => {
  const res = await fetch('/api/login', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify(values),
  })
  if (res.status === 422) {
    const body = await res.json()
    if (body.errors?.email?.[0]) setFieldError('email', body.errors.email[0])
    return
  }
})
</script>

<template>
  <form @submit="onSubmit">
    <input v-model="email" type="email" />
    <span v-if="emailError">{{ emailError }}</span>

    <input v-model="password" type="password" />
    <span v-if="passwordError">{{ passwordError }}</span>

    <button type="submit" :disabled="isSubmitting">Se connecter</button>
  </form>
</template>
```

Exemple Angular avec Reactive Forms et un validateur asynchrone debouncé pour vérifier l'unicité d'un email.

```ts
import { Component, inject } from '@angular/core';
import { AsyncValidatorFn, FormBuilder, ReactiveFormsModule, Validators } from '@angular/forms';
import { HttpClient } from '@angular/common/http';
import { catchError, debounceTime, first, map, of, switchMap } from 'rxjs';

@Component({
  selector: 'app-signup-form',
  standalone: true,
  imports: [ReactiveFormsModule],
  template: `
    <form [formGroup]="form" (ngSubmit)="submit()">
      <input formControlName="email" type="email" />
      @if (form.controls.email.hasError('emailTaken')) {
        <p>Email déjà utilisé</p>
      }
      <button type="submit" [disabled]="form.invalid || form.pending">Créer</button>
    </form>
  `,
})
export class SignupFormComponent {
  private readonly fb = inject(FormBuilder);
  private readonly http = inject(HttpClient);

  private readonly uniqueEmailValidator: AsyncValidatorFn = (control) =>
    of(control.value).pipe(
      debounceTime(300),
      switchMap((email) =>
        this.http.get<{ available: boolean }>('/api/users/check-email', { params: { email } }),
      ),
      map((result) => (result.available ? null : { emailTaken: true })),
      catchError(() => of(null)),
      first(),
    );

  readonly form = this.fb.nonNullable.group({
    email: ['', [Validators.required, Validators.email], [this.uniqueEmailValidator]],
  });

  submit(): void {
    if (this.form.invalid) return;
  }
}
```

---

## 🔗 3. Voir aussi

- [Sujet 8 — Gestion d'erreurs côté front](08-erreurs-front.md)
- [Sujet 12 — Sécurité front](12-securite-front.md)
- [Sujet 16 — Validation des entrées](16-validation-entrees.md)
