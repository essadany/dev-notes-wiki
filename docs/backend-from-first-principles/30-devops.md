# 30. DevOps & Infrastructure pour les Ingénieurs Backend

Jusqu'à présent, tout notre code backend (handlers HTTP, requêtes SQL, caches Redis, files d'attente) s'exécutait en local sur notre machine (`localhost`). Les utilisateurs réels n'y ont pas accès ! **Le rôle du DevOps est de répondre à la question essentielle : comment faire voyager notre code depuis un commit local jusqu'à un service vivant, sécurisé, hautement disponible et accessible sur Internet, sans aucune coupure de service pour les utilisateurs ?**


---

## 🧭 1. La Philosophie DevOps : Vitesse vs Stabilité (Un Mythe Brisé)

### ⚔️ Le Conflit Historique (Dev vs Ops)
Dans les structures traditionnelles, deux équipes aux objectifs contradictoires s'affrontaient :

```
┌─────────────────────────────────────────┐         ┌─────────────────────────────────────────┐
│              ÉQUIPE DEV                 │         │              ÉQUIPE OPS                 │
│  • Objectif : Livrer des features vite  │  VS     │  • Objectif : Maximiser la stabilité   │
│  • Mesure : Nombre de releases/features │         │  • Mesure : Zéro temps d'arrêt (uptime) │
│  👉 Poussent pour DEPLOYER souvent !    │         │  👉 Poussent pour BLOQUER les deploys ! │
└─────────────────────────────────────────┘         └─────────────────────────────────────────┘
```

Quand un incident survenait en production :
* Les **Devs** accusaient l'infrastructure (*« Ce n'est pas notre code, ce sont vos serveurs qui sautent ! »*).
* Les **Ops** accusaient le code (*« Ce n'est pas l'infra, c'est votre code codé avec les pieds ! »*).

### 📈 Le Mouvement DevOps & Les 4 Métriques DORA
Le mouvement **DevOps** réconcilie ces deux mondes en rendant les processus de déploiement **petits, fréquents, automatisés et ennuyeux (boring)**.

L'étude DORA (*DevOps Research and Assessment*) basée sur des dizaines de milliers d'équipes montre que **la vitesse et la stabilité ne sont pas un arbitrage (trade-off)** : les équipes qui déploient le plus souvent sont aussi celles dont les serveurs sont les plus stables !

| Métrique DORA | Définition | Performeurs "Top 20%" | Performeurs "Bas du tableau" |
| :--- | :--- | :--- | :--- |
| **Change Lead Time** *(Vitesse)* | Temps entre le commit local et la prod | < 1 heure / < 1 jour | 1 à 6 mois |
| **Deployment Frequency** *(Vitesse)* | Fréquence de déploiement en prod | À la demande (plusieurs/jour) | 1 fois tous les 1 à 6 mois |
| **Change Failure Rate** *(Stabilité)* | % de déploiements causant un bug | ~ 5% | ~ 40% |
| **Failed Deployment Recovery** *(Stabilité)* | Temps pour restaurer le service après panne | < 1 heure | Plus d'une semaine |

> 💡 **Pourquoi déployer souvent rend plus stable ?**  
> Si vous déployez 1 fois par mois (200 commits de 15 développeurs), trouver la ligne de code responsable d'un bug prend des jours. Si vous déployez **chaque commit individuellement**, le changement est minuscule, l'auteur a le code frais en tête, et l'isolation du problème prend 2 minutes.

---

## 🔀 2. Intégration Continue (CI) & Modèles de Branching

### 🔄 La Pratique de la CI (Continuous Integration)
La **CI** est une pratique avant d'être un serveur : chaque développeur fusionne son travail dans une branche partagée au moins **une fois par jour**, déclenchant un build et des tests automatisés.

```
   ┌─────────────────────────────────────────────────────────────┐
   │            MODÈLE CLASSIQUE (GitFlow / Multi-branches)      │
   │  Branche feature ouverte pendant 3 semaines                 │
   │  🚨 Résultat : "Merge Hell" & dérives massives de code !    │
   └──────────────────────────────┬──────────────────────────────┘
                                  │
                                  ▼
   ┌─────────────────────────────────────────────────────────────┐
   │            MODÈLE MODERNE (Trunk-Based Development)         │
   │  Branche 'main' unique. Petites branches d'un jour max.     │
   │  Features incomplètes cachées derrière des Feature Flags !  │
   └─────────────────────────────────────────────────────────────┘
```

> ⚠️ **GitFlow vs Trunk-Based** : GitFlow a été conçu pour les logiciels distribués sur desktop/mobile (qui doivent maintenir la v2 en parallèle de la v3). Pour le Web/Backend, le serveur n'a qu'une seule version en prod (la dernière) : le **Trunk-Based Development** est largement supérieur.

---

## 🏗️ 3. Builds, Pipelines CI/CD & Sécurité OIDC

### 📦 Qu'est-ce qu'un Build Idéal ?
Un **Build** transforme le code source en un artefact exécutable (binaire Go, package Node.js). 

Un build doit être **strictement reproductible (Byte-Identical)** :
* **Déclarer son compilateur exact** (version de Go, Node.js).
* **Verrouiller les dépendances exactes** via un lockfile (`go.sum`, `package-lock.json`, `Cargo.lock`) impérativement versionné dans Git.
* **Stamper le binaire** avec le **Commit Hash** Git (`git rev-parse HEAD`) pour savoir exactement quel code tourne en prod via l'endpoint `/health`.

### ⚙️ Pipelines GitHub Actions & Authentification OIDC
Un pipeline est une suite d'étapes (du plus rapide/moins cher au plus long/coûteux) configurées en YAML dans `.github/workflows/`.

```
┌─────────────────┐      ┌─────────────────┐      ┌─────────────────┐      ┌─────────────────┐
│  Compile & Unit │ ──►  │   Integration   │ ──►  │   Build Docker  │ ──►  │  Deployment to  │
│      Tests      │      │  Tests & DBs    │      │  Image & Push   │      │   Production    │
└─────────────────┘      └─────────────────┘      └─────────────────┘      └─────────────────┘
```

#### 🛡️ Sécurisation des Accès Cloud : OIDC vs Secrets Permanents
Au lieu de stocker des clés d'API Cloud permanentes (`AWS_SECRET_ACCESS_KEY`) dans les Secrets GitHub (qui risquent de fuiter), on utilise **OIDC (OpenID Connect)** :

```
 ┌──────────────┐   1. Demande de token signé   ┌───────────────────┐
 │ GitHub Action│ ────────────────────────────► │ GitHub Identity   │
 │   Runner     │ ◄──────────────────────────── │    Provider       │
 └──────┬───────┘   2. Short-lived JSON Token   └───────────────────┘
        │
        │ 3. Échange du token JWT
        ▼
 ┌──────────────┐   4. Vérification signature & Repo/Branch
 │  Cloud AWS   │ ───────────────────────────────────────────────────┐
 └──────┬───────┘                                                    │
        ▼                                                            │
 ┌───────────────────────────────────────────────────────────────────┐
 │ Renvoyé : Clés temporaires valides UNIQUEEMENT pour ce Job !      │
 └───────────────────────────────────────────────────────────────────┘
```

---

## 🐳 4. Conteneurs vs VMs : La Magie du Noyau Linux

### 🆚 Virtual Machine (VM) vs Conteneur
Un conteneur **N'EST PAS** une petite VM. C'est un simple processus ordinaire du système hôte, isolé par le Noyau (Kernel) Linux.

```
┌─────────────────────────────────────────┐  ┌─────────────────────────────────────────┐
│            VIRTUAL MACHINE              │  │                CONTENEUR                │
├─────────────────────────────────────────┤  ├─────────────────────────────────────────┤
│ App A               │ App B             │  │ App A               │ App B             │
│ Bin/Libs            │ Bin/Libs          │  │ Bin/Libs            │ Bin/Libs          │
│ OS Invité (Kernel A)│ OS Invité (Kernel B)│  │ (Pas d'OS ni de Kernel propre !)        │
├─────────────────────┴───────────────────┤  ├─────────────────────────────────────────┤
│ Hyperviseur (Type 1 / Type 2)           │  │ Moteur Conteneur (containerd / runC)    │
├─────────────────────────────────────────┤  ├─────────────────────────────────────────┤
│ Noyau Hôte & Matériel Physique (Host)   │  │ Noyau Hôte & Matériel Physique (Host)   │
└─────────────────────────────────────────┘  └─────────────────────────────────────────┘
```

### 🧬 Les 3 Briques du Noyau Linux créant l'illusion du Conteneur

#### 1. Namespaces (Ce que le processus peut VOIR)
Le Noyau Linux enveloppe les ressources globales pour donner au conteneur une vue isolée de la machine :
* **PID** : Le conteneur voit son propre processus comme le `PID 1`.
* **NET** : Ses propres interfaces réseau, sa table de routage et ses ports (deux conteneurs peuvent tous deux écouter sur le port `80`).
* **MOUNT** : Son propre système de fichiers racine (`/`).
* **UTS** : Son propre nom d'hôte (`hostname`).
* **IPC, USER, CGROUP, TIME**.

> 🛠️ **Les 3 appels système Unix fondamentaux** : `clone()` (crée le processus avec des flags comme `CLONE_NEWPID`, `CLONE_NEWNET`), `unshare()` et `setns()`.

#### 2. Control Groups / cgroups (Ce que le processus peut CONSOMMER)
Exposés dans `/sys/fs/cgroup/`, les cgroups limitent la RAM et le CPU :
* Si un conteneur dépasse sa limite `memory.max` (ex: 512 Mo), l'**OOM Killer** *(Out Of Memory Killer)* du Noyau Linux le tue immédiatement (`OOMKilled`).

#### 3. OverlayFS (Système de Fichiers en Couches)
Les images Docker sont des piles de couches en **lecture seule (Read-Only)** identifiées par le hash **SHA-256** de leurs octets.
* Lors du lancement, **OverlayFS** superpose ces couches en lecture seule et ajoute une fine couche supérieure **inscriptible (Writable Upper Layer)** éphémère.
* Si le conteneur s'arrête, la couche inscriptible est détruite. **Un conteneur est fondamentalement jetable (disposable).**

```
┌─────────────────────────────────────────────────────────────┐
│ 🟩 Upper Layer (Writable) : Modifications éphémères local   │
├─────────────────────────────────────────────────────────────┤
│ 🟦 Image Layer 3 (Read-Only) : Binaire Go / App (11 Mo)     │
├─────────────────────────────────────────────────────────────┤
│ 🟦 Image Layer 2 (Read-Only) : Certificats SSL / Ca-Cert    │
├─────────────────────────────────────────────────────────────┤
│ 🟦 Image Layer 1 (Read-Only) : Base Image (Alpine / Ubuntu) │
└─────────────────────────────────────────────────────────────┘
```

### 📦 Anatomie du Runtime : De Docker à runC (Standard OCI)
Quand vous tapez `docker run` :
1. `docker CLI` parle au démon `dockerd`.
2. `dockerd` délègue à `containerd` (qui gère le cycle de vie des images et le stockage).
3. `containerd` appelle `runC` (le binaire bas niveau conforme OCI) qui effectue les system calls `clone()` et écrit les fichiers `cgroups`.

### 🧹 Regles d'Or d'un bon Dockerfile
1. **Ordre des instructions & Cache** : Copier le fichier de dépendances (`package.json` ou `go.mod`) ET les installer *avant* de copier le reste du code source. Ainsi, modifier le code n'invalide pas le cache des dépendances !
2. **Multi-Stage Builds** : Séparer l'étape de compilation (avec compilateur lourd) de l'étape finale d'exécution (image minimale `scratch` ou `distroless` contenant uniquement le binaire).
3. **Non-Root & Pinned Versions** : Toujours utiliser un utilisateur non-privilégié et fixer les versions exactes (jamais `:latest`).

---

## 🌐 5. Du Serveur Unique (`systemd`) à l'Orchestration (`Kubernetes`)

### 🖥️ Niveau 1 : Le Serveur Unique
Sur un serveur Linux simple :
* **Process Manager** : `systemd` maintient le binaire actif, le relance en cas de crash, et envoie les logs à `journalctl`.
* **Reverse Proxy** : **Nginx** ou **Caddy** écoute sur les ports `80/443`, gère le HTTPS et redirige vers le port interne de l'application.
* **Certificats HTTPS** : Obtenus automatiquement sans intervention humaine via le protocole **ACME** (Let's Encrypt).

### 🚨 Le Piège du Serveur "Snowflake" vs L'Infrastructure Immuable
* ❌ **Snowflake Server** : Un serveur configuré à la main au fil des ans. Personne ne sait comment le reconstruire s'il plante.
* ✅ **Infrastructure Immuable** : On ne modifie JAMAIS un serveur existant (pas de SSH en prod !). On remplace l'instance complète par une nouvelle version.

---

## ☸️ 6. Orchestration avec Kubernetes (k8s)

Quand vous gérez 40 conteneurs sur 10 machines, vous avez besoin d'un **Orchestrateur**.

### 🤖 L'Approche Déclarative & Les Boucles de Contrôle
Dans Kubernetes, on ne donne pas d'ordres impératifs. On déclare **l'État Désiré (Desired State)** dans un fichier YAML. Des **Controllers** tournent en boucle pour aligner **l'État Actuel (Current State)** sur l'état désiré.

```
       ┌─────────────────────────────────────────────────────┐
       │             État Désiré (Fichier YAML)              │
       │             "Je veux 3 pods de mon API"             │
       └──────────────────────────┬──────────────────────────┘
                                  │
                                  ▼
       ┌─────────────────────────────────────────────────────┐
       │                 CONTROLLER LOOP                     │
       │   Compare : État Actuel (2)  vs  État Désiré (3)    │
       │   Action  : Démarre 1 nouveau Pod automatiquement !  │
       └─────────────────────────────────────────────────────┘
```

### 🏛️ L'Architecture de Kubernetes

```
┌────────────────────────────────────────────────────────────────────────────────────────┐
│                                   CONTROL PLANE                                        │
│  ┌──────────────┐      ┌──────────────┐      ┌────────────────────┐   ┌─────────────┐  │
│  │  API Server  │ ──►  │    etcd      │      │ Controller Manager │   │ Scheduler   │  │
│  │ (Front-end)  │      │ (Store KV)   │      │ (Boucles de ctrl)  │   │(Choix nœud) │  │
│  └──────────────┘      └──────────────┘      └────────────────────┘   └─────────────┘  │
└──────────────────────────────────────────┬─────────────────────────────────────────────┘
                                           │
                    ┌──────────────────────┴──────────────────────┐
                    ▼                                             ▼
┌─────────────────────────────────────────┐   ┌─────────────────────────────────────────┐
│               NODE 1                    │   │               NODE 2                    │
│  • kubelet (Agent local)                │   │  • kubelet (Agent local)                │
│  • kube-proxy (Réseau & Routage)        │   │  • kube-proxy (Réseau & Routage)        │
│  • Pod A (IP 10.244.0.5)                │   │  • Pod B (IP 10.244.1.8)                │
└─────────────────────────────────────────┘   └─────────────────────────────────────────┘
```

### 🧩 Les Objets Clés de Kubernetes
* **Pod** : La plus petite unité d'ordonnancement (contient 1 ou plusieurs conteneurs partageant la même IP et le même réseau).
* **ReplicaSet** : Maintient le nombre exact de Pods demandés.
* **Deployment** : Gère les mises à jour progressives (rollouts) et retours en arrière (rollbacks) des ReplicaSets.
* **Service** : Fournit un nom DNS et une IP virtuelle stable devant un groupe de Pods éphémères.
* **Ingress / Gateway API** : Ouvre le trafic HTTP/HTTPS depuis l'extérieur vers les Services internes.

### 🩺 Probes (Sondes de Santé)
* **Readiness Probe** : Vérifie si le Pod est prêt à recevoir du trafic réseau (si échec, le Pod est retiré du Load Balancer sans être tué).
* **Liveness Probe** : Vérifie si le processus est toujours en vie (si échec répété, `kubelet` tue et redémarre le conteneur).
  * ⚠️ **Attention au piège** : Ne mettez JAMAIS de vérification de base de données dans une Liveness Probe ! Si la BDD rame, la probe échouera et Kubernetes tuera tous vos Pods en boucle (*Cascading Failure*).

---

## 🚦 7. Stratégies de Déploiement Zero-Downtime

```
1. Recreate : Stop v1 ──► [Downtime] ──► Start v2  (Simple mais coupure de service)

2. Rolling Update : Remplace les Pods 1 par 1 (Par défaut k8s - Attention : v1 et v2 coexistent !)

3. Blue-Green : 
   [ Environnement Blue (v1) - Live 100% ]
   [ Environnement Green (v2) - Test 0%  ]  ──► Switch du Routeur en 1ms (Coût Infra x2)

4. Canary : 
   90% Trafic ──► 9 Pods (v1)
   10% Trafic ──► 1 Pod  (v2) ──► Validation Métriques ──► Montée progressive à 100%
```

### 🔄 GitOps (ArgoCD / Flux)
Au lieu que la CI pousse directement vers le cluster avec des identifiants admin :
1. La CI pousse la nouvelle image sur le Registry et met à jour le tag dans un repo Git de configuration.
2. Un agent **GitOps** interne au cluster (**ArgoCD**) détecte le changement Git et applique l'état dans le cluster. La CI n'a plus besoin d'accès au cluster !

---

## 🛠️ 8. Infrastructure as Code (IaC) & OpenTofu / Terraform

Pour éviter de configurer les serveurs Cloud à la main dans la console AWS (ce qui rend la reconstruction impossible en moins d'une heure) :

```
┌─────────────────────────────────────────────────────────────┐
│                    LES 3 OUTILS DE L'IaC                    │
├──────────────────────────┬──────────────────────────────────┤
│ 1. Provisioning          │ Terraform / OpenTofu             │
│    (Crée VMs, DBs, Net)  │                                  │
├──────────────────────────┼──────────────────────────────────┤
│ 2. Config Management     │ Ansible                          │
│    (Installe les app)    │                                  │
├──────────────────────────┼──────────────────────────────────┤
│ 3. Image Baking          │ Packer                           │
│    (Gère les AMIs/Images)│                                  │
└──────────────────────────┴──────────────────────────────────┘
```

### ⚠️ Le State File & Le Phénomène de "Drift"
* **State File (`.tfstate`)** : Fichier dans lequel Terraform cartographie les ressources Cloud réelles avec le code. Il doit être stocké à distance de manière chiffrée avec un verrou (S3 + DynamoDB).
* **Drift (Dérive)** : Si un ingénieur modifie un port de sécurité directement dans la console AWS, Terraform détectera la dérive lors du prochain `terraform plan` et remettra le Cloud en conformité avec le code.

---

## 📊 9. Opérations, SLI/SLO/SLA & Culture Postmortem

### 📐 Les 3 Définitions Essentielles
* **SLI (Service Level Indicator)** : La mesure réelle du service (ex: *99.92% des requêtes ont répondu en < 300ms*).
* **SLO (Service Level Objective)** : Cible interne sur une période (ex: *99.9% de succès sur 30 jours*).
* **SLA (Service Level Agreement)** : Le contrat commercial avec remboursement financier si le SLO n'est pas tenu.

### 💰 L'Error Budget (Budget d'Erreur)
Un SLO de 99.9% autorise **0.1% d'erreurs** (~43 minutes d'interruption par mois).
* Tant qu'il reste du budget d'erreur, l'équipe peut déployer rapidement et prendre des risques.
* Si le budget d'erreur est épuisé avant la fin du mois, **tous les déploiements sont gelés** et 100% de l'effort est dédié à la fiabilité !

### 🕊️ La Culture Blameless (Postmortem Sans Blâme)
Lors d'un incident majeur (Outage), après avoir stoppé le feu :
1. Rédiger un **Postmortem** minute par minute sans jamais nommer ni blâmer d'humain.
2. L'erreur humaine est toujours le symptôme d'un système mal conçu. **On blâme le système, jamais l'humain !**

---