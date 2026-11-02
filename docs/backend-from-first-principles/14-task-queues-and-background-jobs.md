# 14. Task Queues & Background Jobs (L'art de l'asynchronisme)

En tant que dev junior, comprendre les **Background Jobs** (tâches de fond) et les **Task Queues** (files de tâches) est une étape capitale : c'est la clé absolue pour créer des applications web ultra-réactives, scalables et capables de résister aux pannes de services tiers !

Voici mes notes de cours rédigées avec mon regard de dev, mes schémas d'architecture ASCII et mes antisèches sans aucune fioriture.

---

## 🧠 1. C'est quoi une Tâche de Fond (Background Task) ?

> 💡 **Définition simple** : Une tâche de fond est **n'importe quel morceau de code ou logique métier qui s'exécute en dehors du cycle de vie principal Requête-Réponse** (Request-Response Lifecycle).

```
┌──────────┐          ① Requête HTTP (ex: Sign-up)          ┌──────────┐
│          ├───────────────────────────────────────────────>│          │
│ CLIENT   │                                                │ SERVEUR  │
│ (Browser)│<───────────────────────────────────────────────┤  API     │
└──────────┘          ② Réponse immédiate (HTTP 201)        └────┬─────┘
                                                                 │
                                                                 │ ③ Offload (Asynchrone)
                                                                 ▼
                                                        ┌─────────────────┐
                                                        │ BACKGROUND JOB  │
                                                        │ (Processus séparé│
                                                        │  non-bloquant)  │
                                                        └─────────────────┘
```

### Pourquoi c'est indispensable ?
* **Non-bloquant / Réactivité** : La tâche n'est pas mission-critique pour la réponse immédiate au client. On la décharge sur un processus séparé afin que l'API réponde en quelques millisecondes.
* **Tolérance aux pannes** : Si un service tiers (ex: envoi d'email, API de paiement) est au ralenti ou en panne, cela ne fait pas planter l'API principale ni l'expérience de l'utilisateur.

---

## 📧 2. Cas Concret : Le Flux d'Inscription (Sign-up & Email Verification)

Analysons la différence entre une approche **Synchrone** (mauvaise pratique) et une approche **Asynchrone avec Task Queue** (bonne pratique).

### A. L'Approche Synchrone (À éviter !)
L'utilisateur s'inscrit (Email, Mot de passe). Le serveur valide les données, enregistre l'utilisateur en BDD, puis appelle **synchroniquement** une API d'emailing tierce (ex: Resend, Brevo, Mailgun).

```
[ Client ] ──> (POST /signup) ──> [ Serveur API ] ──> (Insert BDD) ──> [ API Emailing Tierce ]
                                                                               │
   ❌ Si l'API d'emailing est lente (5s) ou en panne (500) :                   │
   • L'utilisateur attend inutilement devant un spinner.                       │
   • La requête /signup plante complètement et renvoie une 500 à l'utilisateur ! ◄┘
```

### B. L'Approche Asynchrone avec Task Queue (Best Practice)
Le serveur effectue les validations et la sauvegarde BDD, puis **pousse la tâche d'envoi dans une file d'attente (Task Queue)** et renvoie immédiatement un code `201 Created` au client avec le message *"Email de vérification envoyé !"*.

```
[ Client ] ──> (POST /signup) ──> [ Serveur API ] ──> (Insert BDD)
                                         │
                                         ├─ Enqueue (JSON) ──> [ TASK QUEUE ]
                                         │                          │
[ Client ] <── (201 Created 20ms) ───────┘                          │ Dequeue (Async)
                                                                    ▼
                                                            [ WORKER / CONSUMER ]
                                                                    │
                                                                    └──> [ API Emailing ] ──> ✉️ Email envoyé !
```

---

## 🔄 3. Le Mécanisme de Retry & Exponential Backoff

Que se passe-t-il si l'API d'emailing tombe en panne au moment où le Worker tente d'exécuter la tâche ?

```
[ WORKER ] ──> Tentative #1 d'envoi d'email ──> ❌ ÉCHEC (API Tierce Down)
    │
    ├──> Attente 1 min (Backoff)
    ├──> Tentative #2 d'envoi ──> ❌ ÉCHEC
    │
    ├──> Attente 2 min (Exponential Backoff)
    ├──> Tentative #3 d'envoi ──> ✅ SUCCÈS ! (L'API est revenue en ligne)
```

> ⚡ **Le pouvoir de l'Exponential Backoff** : Au lieu de réessayer frénétiquement chaque seconde, le Worker espace ses tentatives (ex: 1m, 2m, 4m, 8m). Comme les pannes de grands services d'infrastructure durent souvent quelques secondes, la tâche finit par réussir sans aucun échec visible pour l'utilisateur !

---

## 🏗️ 4. Architecture Interne d'une Task Queue (Producer, Broker, Consumer)

Le fonctionnement sous le capot d'une Task Queue repose sur trois composants fondamentaux :

```
┌────────────────────────┐         Enqueue (JSON)        ┌────────────────────────┐
│  PRODUCER / PUBLISHER  ├──────────────────────────────>│  BROKER / TASK QUEUE   │
│ (Code Application API) │                               │ (File d'attente/Redis) │
└────────────────────────┘                               └───────────┬────────────┘
                                                                     │
                                                                     │ Dequeue
                                                                     ▼
                                                         ┌────────────────────────┐
                                                         │   CONSUMER / WORKER    │
                                                         │ (Processus séparé)     │
                                                         └───────────┬────────────┘
                                                                     │
                                                                     └──> ACK (Acquittement)
```

### Détail des Composants :

| Composant | Rôle & Fonctionnement | Technologies associées |
| :--- | :--- | :--- |
| **Producer (Producteur)** | Le code applicatif de l'API. Il récolte les données nécessaires (`userId`, `email`), les **sérialise au format JSON** et pousse le job dans la queue (*Enqueue*). | Express, NestJS, FastAPI, Go |
| **Broker (Courtier)** | La structure de stockage temporaire (la "To-Do List" du backend). Il conserve les tâches jusqu'à ce qu'un worker soit disponible. | **Redis Pub/Sub** (BullMQ), **RabbitMQ**, **AWS SQS** |
| **Consumer / Worker** | Un processus ou thread complètement séparé. Il surveille la queue, dépile la tâche (*Dequeue*), la **désérialise en objet natif** et exécute la fonction associée. | BullMQ Workers, Celery, Asynq |

### Concepts Clés de Fiabilité :
* **ACK (Acknowledgement / Acquittement)** : Une fois le travail terminé, le Worker envoie un signal `ACK` au Broker pour que la tâche soit définitivement supprimée de la queue.
* **Visibility Timeout** : Durée pendant laquelle une tâche dépilée est masquée aux autres workers. Si le Worker plante brutalement avant d'envoyer l'ACK, le le timeout expire et le Broker rend la tâche à nouveau disponible pour un autre Worker. Aucune tâche ne s'interrompt ni ne se perd dans la nature !

---

## 🗂️ 5. Les 4 Types Majeurs de Tâches Background

```
                              ┌──────────────────────────────────┐
                              │    TYPES DE TÂCHES BACKGROUND    │
                              └────────────────┬─────────────────┘
                                               │
       ┌──────────────────────┬────────────────┴──────────────────┬──────────────────────┐
       ▼                      ▼                                   ▼                      ▼
┌──────────────┐       ┌──────────────┐                    ┌──────────────┐       ┌──────────────┐
│  1. ONE-OFF  │       │ 2. RECURRING │                    │  3. CHAINED  │       │   4. BATCH   │
│  (Ponctuel)  │       │ (Cron/Planifié)                   │  (Chaînées)  │       │  (Par lots)  │
└──────────────┘       └──────────────┘                    └──────────────┘       └──────────────┘
```

### 1️⃣ Tâches Ponctuelles (One-Off Tasks)
Déclenchées instantanément par un événement utilisateur précis.
* *Exemples* : Envoi d'un email de vérification, envoi d'un mot de passe oublié, notification push de réception d'un message direct.

### 2️⃣ Tâches Récurrentes (Recurring / Cron Jobs)
Exécutées à intervalles réguliers programmés.
* *Exemples* : Génération et envoi de rapports mensuels PDF, nettoyage nocturne des sessions utilisateur orphelines/expirées en BDD.

### 3️⃣ Tâches Chaînées (Chained / Parent-Child Tasks)
Relation de dépendance hiérarchique entre plusieurs tâches.
* *Exemple d'une plateforme vidéo (LMS type Udemy)* :
  ```
  [ Upload Vidéo ] ──> ❶ Encodage Vidéo (Parent)
                             │
                             ├──> ❷ Génération des Miniatures (Enfant 1)
                             └──> ❸ Transcription Audio / Sous-titres (Enfant 2)
  ```
  *Note* : La génération des miniatures et des sous-titres peut tourner en parallèle, mais toutes deux doivent attendre que l'encodage vidéo principal soit totalement terminé.

### 4️⃣ Tâches par Lots (Batch Tasks)
Déclenchement d'une cascade de sous-tâches à partir d'une seule action.
* *Exemple de Suppression de Compte (Delete Account)* :
  Au lieu de bloquer l'API pendant 30s pour tout nettoyer en BDD, l'API répond `200 OK` ("Compte en cours de suppression"). En tâche de fond, le job déclenche le nettoyage des projets, la suppression des fichiers S3, l'anonymisation BDD et l'envoi de l'email de confirmation.

---

## 🛠️ 6. Cas d'Usage Incontournables en SaaS

1. **Sending Emails & SMS** : Déchargement des appels vers Resend, Twilio, SendGrid.
2. **Traitement Médias (Images & Vidéos)** : Redimensionnement, compression et génération de formats multi-résolutions (adaptés aux mobiles vs écrans 4K).
3. **Génération de Rapports Lourds** : Création de fichiers PDF/Excel complexes sans sature la mémoire du serveur web.
4. **Push Notifications Mobiles** : Envoi de notifications via les services OS distants (Apple APNs, Google FCM).

---

## 🛡️ 7. Best Practices & Règles d'Or pour l'Ingénieur Backend

> 🔑 **1. Rendre les Tâches Idempotentes**
> Une tâche doit pouvoir être exécutée plusieurs fois de suite sans créer d'effets secondaires indésirables ou de doublons en BDD. Utilisez des transactions SQL et des rollbacks en cas de pépin au milieu d'un job.

> 🔑 **2. Garder les Tâches Petites et Atomiques**
> Évitez les tâches monolithiques "fourre-tout". Plus une tâche est courte et ciblée, plus elle est facile à surveiller, à isoler et à retenter automatiquement sans gaspiller de CPU.

> 🔑 **3. Mettre en place du Rate Limiting vers les APIs Externes**
> Si vos Workers envoient des milliers de requêtes vers des services tiers, veillez à ne pas dépasser leurs quotas de rate-limiting (au risque de vous faire bannir ou surfacturer).

> 🔑 **4. Monitoring & Observabilité de la Queue**
> Surveillez en permanence la longueur de la file d'attente (Queue Length), le taux d'échec des workers et le temps moyen de traitement à l'aide d'outils comme Prometheus et Grafana.

---