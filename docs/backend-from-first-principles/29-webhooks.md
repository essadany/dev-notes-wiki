# 29. Webhooks - Comment le Serveur Vous Appelle (Reverse APIs)

Jusqu'à présent dans ce cours, tout notre paradigme reposait sur une règle absolue : **c'est toujours le client qui initie la communication avec le serveur**. Même dans les systèmes temps réel (WebSockets, SSE), la connexion devait obligatoirement être démarrée par le client. 

Dans ce chapitre, on renverse complètement cette logique en étudiant la communication **Serveur à Serveur (Server-to-Server / Backend-to-Backend)** où un service externe prend l'initiative d'appeler notre propre backend.

---

## 🧭 1. Pourquoi avons-nous besoin des Webhooks ? (Le Cas d'Usage Stripe)

Imaginons notre application Taskboard avec une page de tarifs. Un utilisateur achète un abonnement Premium. Le paiement n'est pas géré sur nos serveurs, mais par un prestataire spécialisé comme **Stripe**.

```
   ┌──────────────┐         1. Inscription        ┌────────────────────┐
   │ Application  │──────────────────────────────►│    Stripe / SaaS   │
   │   Backend    │                               │ (Payment Provider) │
   └──────────────┘                               └─────────┬──────────┘
          ▲                                                 │
          │         2. Événement produit dans Stripe        │
          │         ("payment_intent.succeeded")            │
          │                                                 │
          └─────────────────────────────────────────────────┘
                3. Webhook HTTP POST "Push" (Reverse API)
                   /hooks/stripe  ──►  Action Backend
```

### Pourquoi ne pas compter sur le navigateur client ?
On pourrait penser qu'il suffit au navigateur du client de recevoir la confirmation de Stripe puis d'en informer notre backend. C'est une **erreur majeure en architecture backend** pour deux raisons :
1. **Fiabilité Réseau** : Le réseau du client est instable (fermeture d'onglet, perte de réseau 4G/Wifi, crash du navigateur). Le signal de paiement serait perdu.
2. **Sécurité** : Il est impensable de faire confiance au client pour valider une transaction financière. Notre serveur backend doit être la **seule source de vérité**.

### La Solution Naïve (Polling Backend ──► Stripe) et ses 3 défauts
Si notre serveur demande en boucle à Stripe toutes les secondes `GET /payments/123/status` :
* **Latence moyenne** : $\text{Intervalle} / 2 = 0,5 \text{ seconde}$ de retard.
* **Requêtes gâchées** : 99% des appels renvoient `"pending"` inutilement.
* **Explosion de la charge & Rate Limiting** : Si 10 000 marchands font du polling chaque seconde, Stripe subit 10 000 req/s. Stripe va donc appliquer un *Rate Limiting* très agressif, forçant notre polling à s'espacer (ex: 1 appel par minute), faisant chuter la latence moyenne à 30 secondes !

---

## 🪝 2. Qu'est-ce qu'un Webhook ? (Inversion de Communication & Vocabulaire)

Un **Webhook** est simplement une **requête HTTP POST classique** envoyée par un service tiers vers une URL spécifique que nous avons exposée sur notre serveur.

C'est une **"Reverse API"** : au lieu d'appeler l'API d'un service, c'est le service qui appelle notre propre API dès qu'un événement survient.

### Un peu d'histoire
* **2007** : Jeff Lindsay propose le concept pour transposer l'idée du *tuyau Unix (Unix Pipe)* au Web ("user-defined HTTP callbacks").
* **2009** : Des ingénieurs de Google développent le protocole *PubSubHubbub* (devenu WebSub) sur le même principe.

### Le Vocabulaire Essentiel
| Terme | Définition |
| :--- | :--- |
| **Provider / Sender** | Le système où l'événement se produit (Stripe, GitHub, Clerk). |
| **Consumer / Receiver** | Notre serveur backend qui écoute et traite l'événement. |
| **Endpoint / URL** | L'adresse HTTPS publique exposée par notre serveur. |
| **Event** | Le fait générateur (`payment_intent.succeeded`, `push`). |
| **Delivery** | Une tentative d'envoi de l'événement vers l'endpoint (*1 événement peut subir plusieurs deliveries en cas de retry !*). |
| **Subscription** | Notre inscription auprès du provider associant notre URL aux événements désirés. |

---

## 🛠️ 3. Enregistrement, Handshakes & Développement Local

### 1. Séparation stricte des Endpoints par Provider
Si notre backend s'interface avec GitHub et Stripe, il faut **toujours séparer les URLs** :
* ❌ `/hooks/callback` (Trop générique, à proscrire)
* ✅ `/hooks/github` et `/hooks/stripe`

Chaque provider possède ses propres en-têtes HTTP, son format de signature HMAC et ses règles de réessai.

### 2. Le Registration Handshake (Sécurité du Provider)
Certains services (Microsoft Graph, Discord, Amazon SNS) imposent une validation d'URL à l'inscription avant d'envoyer le moindre événement réel.
* **Pourquoi ?** Pour empêcher un attaquant d'inscrire l'URL d'une victime et d'utiliser le provider pour mener une attaque DoS par amplification.
* **Exemples** : Microsoft Graph exige le renvoi du token en texte brut ; Discord attend un message `PONG` en réponse à un `PING` ; Amazon SNS demande de visiter une URL de confirmation.

### 3. Le Défi du Développement Local (Tunnels)
Pendant le développement local, notre ordinateur portable est derrière un routeur familial sans IP publique et `localhost` n'a aucun sens pour Stripe ou GitHub.
* **Solution** : Utiliser un outil de tunneling (**ngrok**, **Cloudflare Tunnel**) qui génère une URL HTTPS publique temporaire redirigeant le trafic vers notre port local (ex: `8081`).

---

## 🔐 4. Sécurité & Authentification des Webhooks (HMAC & Prévention des Attaques)

Notre endpoint Webhook étant public, n'importe quel pirate peut deviner l'URL et envoyer un faux payload `{"payment": "success"}`.

### Les 5 Méthodes de Sécurisation
1. **Secret Token dans l'URL** (Faible : le token fuite dans les logs d'accès serveur).
2. **IP Allowlist** (Filtrage des IPs du provider au niveau du pare-feu - bon mais insuffisant seul).
3. **Mutual TLS (mTLS)** (Échange de certificats clients).
4. **Signature HMAC avec Clé Secrète Partagée** (La norme universelle utilisée par 80%+ des providers).
5. **Signature par Clé Publique / Privée** (Discord).

### Le Fonctionnement de l'HMAC SHA-256

```
 Provider (ex: Stripe)                      Consumer (Notre Backend)
 ┌──────────────────────┐                  ┌──────────────────────┐
 │ Secret = "sec_123"   │                  │ Secret = "sec_123"   │
 │ Body = '{"id": 42}'  │                  │ Body = '{"id": 42}'  │
 └──────────┬───────────┘                  └──────────┬───────────┘
            │                                         │
  HMAC_SHA256(Secret, Body)                 HMAC_SHA256(Secret, Body)
            │                                         │
            ▼                                         ▼
 Header: X-Signature ──────────────────────► Compare avec Constant-Time
                                                "Signature Matches!"
```

> ⚠️ **LA RÈGLE D'OR EN CODE** : Il faut **toujours vérifier la signature HMAC sur les OCTETS BRUTS (`Raw Bytes`)** de la requête HTTP avant tout parsing JSON !
> Si vous exécutez `express.json()` en premier puis ré-encodez l'objet en JSON string, la moindre modification d'espace ou de format d'encodage modifiera l'empreinte binaire et fera échouer la vérification !

```typescript
// ❌ MAUVAIS : Parser le JSON puis vérifier
app.post('/hooks/stripe', express.json(), (req, res) => {
    const rawString = JSON.stringify(req.body); // Les espaces ont changé !
    verifyHmac(rawString); // FAUX !
});

// ✅ BON : Vérifier sur les octets bruts d'abord !
app.post('/hooks/stripe', express.raw({ type: 'application/json' }), (req, res) => {
    const isValid = verifyHmac(req.body); // Octets d'origine exacts
    if (!isValid) return res.status(401).send("Unauthorized");
    const payload = JSON.parse(req.body);
});
```

### Attaques par Replay & Constant-Time Comparison
* **Replay Attack** : Un pirate intercepte une vraie requête signée et la rejoue 1 000 fois.
  * *Solution* : Inclure un timestamp Unix dans la chaîne signée (ex: header Stripe `t=...`) et rejeter tout message vieux de plus de 5 minutes.
* **Timing Attack** : Utiliser la comparaison de chaînes à temps constant (`crypto.timingSafeEqual` en Node.js, `hmac.Equal` en Go) pour empêcher l'extraction de la signature octet par octet en mesurant les nanosecondes de réponse.
* **SSRF (Server-Side Request Forgery)** : Pour le provider qui émet le Webhook, il faut obligatoirement résoudre l'URL et bloquer les plages d'IP privées (`127.0.0.1`, AWS Metadata `169.254.169.254`) pour éviter que des utilisateurs malveillants n'utilisent le Webhook pour scanner le réseau interne du provider.

---

## 🔄 5. Idempotence & Gestion des Retries (At-Least-Once Delivery)

Si notre serveur traite le Webhook (insère en BDD) mais que la réponse HTTP `200 OK` est perdue à cause d'une micro-coupure réseau, le provider considère la livraison comme un échec et retente l'envoi.

### La Garantie "At-Least-Once"
Un Webhook est garanti d'arriver **au moins une fois**, mais il peut arriver **plusieurs fois**.

### Delivery ID & Transaction SQL Atomique
Chaque événement possède un identifiant de livraison unique (`X-GitHub-Delivery`, `event.id`).

```sql
-- Transaction SQL Atomique pour garantir l'Idempotence
BEGIN;

-- 1. Insérer l'ID de livraison dans la table d'idempotence
INSERT INTO processed_webhooks (delivery_id, processed_at)
VALUES ('del_abc123', NOW());

-- 2. Effectuer l'action métier (ex: Activer l'abonnement)
UPDATE users SET plan = 'premium' WHERE id = 'usr_456';

COMMIT;
```

> 💡 **Pourquoi la transaction unique est vitale ?**
> Si l'on effectue l'action métier d'abord puis que le serveur crashe avant d'enregistrer l'ID de livraison, le retry du provider exécutera l'action une deuxième fois. Si l'on enregistre l'ID d'abord mais que l'action échoue, le retry sera ignoré à tort. Les deux opérations doivent **réussir ou échouer ensemble**.

### Le Problème de l'Ordre des Événements (Out-of-Order Delivery)
Si un utilisateur crée puis supprime un objet instantanément, le Webhook `created` peut subir du réseau et arriver *après* le Webhook `deleted`.
* **Solutions** :
  1. Utiliser le Webhook comme un simple signal et refetch l'état courant via l'API du provider.
  2. Vérifier le champ `updated_at` / `version` et ignorer tout événement antérieur à l'état actuel en BDD.

---

## ⚡ 6. Architecture Haute Performance : Handler Fin & Background Workers

La plupart des providers imposent des timeouts stricts pour répondre au Webhook :
* **Slack** : 3 secondes max.
* **Shopify** : 5 secondes max.
* **GitHub** : 10 secondes max.

Si votre handler effectue des traitements lourds (envoi d'emails, appels d'APIs tierces, traitements d'images), il va dépasser le délai et déclencher des retries inutiles.

```
  Provider Webhook ──► [ Thin HTTP Handler ] ──► 1. Vérifie HMAC (Raw Bytes)
                              │                  2. Écrit dans la Queue (BullMQ/Redis)
                              │                  3. Répond HTTP 202 Accepted (< 50ms)
                              ▼
                     [ Task Queue (Redis) ]
                              │
                              ▼
                    [ Background Worker ] ──► Traitement Métier Lourd (Emails, PDF...)
```

---

## 📦 7. Côté Provider : Le Pattern "Transactional Outbox"

Si vous devez vous-même émettre des Webhooks vers vos clients, ne faites **jamais** l'appel HTTP directement dans votre controller métier !

### Le Pattern Transactional Outbox

```
 Controller HTTP (ex: Move Task)
   │
   ├──► BEGIN TRANSACTION
   │      ├── UPDATE tasks SET status = 'done';
   │      └── INSERT INTO outbox_events (event_name, payload, status) 
   │          VALUES ('task.moved', '{...}', 'pending');
   └──► COMMIT
          │
          ▼ (Asynchrone)
 [ Background Worker (Dispatcher) ] ──► Envoie HTTP POST vers l'URL du client
                                    ──► Gère Exponential Backoff + Jitter
```

* **Exponential Backoff & Jitter** : Espacer les tentatives (5s, 5m, 30m, 2h) et ajouter un délai aléatoire (Jitter) pour éviter que 1 000 retries ne frappent le client au même instant précis.
* **Circuit Breaker** : Après 10 échecs consécutifs, désactiver l'endpoint du client et envoyer un email d'avertissement.

---

## 🎯 8. Quand utiliser les Webhooks (et l'Anti-Pattern à éviter)

### ✅ Bons cas d'usage
* Notifications événementielles légères : *"Cet événement s'est produit, déclenche un pipeline CI/CD, poste un message Slack, ou débloque un accès"*.

### ❌ L'Anti-Pattern Récurrent : La Synchronisation de Base de Données
Essayer de dupliquer l'intégralité de la BDD d'un service SaaS (ex: Clerk, Stripe) dans sa propre BDD uniquement via des Webhooks est une source d'erreurs permanente (perte d'ordre, retards, bugs de synchronisation).

### 💡 La Bonne Solution de Synchronisation
1. Faire un polling régulier sur le journal d'événements ordonné du provider (`GET /v1/events?since=last_id`).
2. Utiliser le Webhook comme un simple **signal déclencheur** pour aller dépiler l'API `/events` du provider.

---