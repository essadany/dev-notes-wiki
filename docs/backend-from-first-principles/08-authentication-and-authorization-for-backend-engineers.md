# 8. L'authentification et l'autorisation pour les ingénieurs backend

**l'Authentification (AuthN) et l'Autorisation (AuthZ)**. C'est l'un des sujets les plus importants et sensibles pour un développeur. Si on se loupe ici, on ouvre la porte à des failles catastrophiques.

> *Objectif : Comprendre les mécanismes d'authentification et d'autorisation, savoir quand utiliser des sessions vs JWT, et maîtriser les concepts de RBAC et OAuth 2.0 pour sécuriser nos applications backend.*

---

## 🧭 1. Le Duel Sacré : AuthN vs AuthZ

C'est l'erreur classique de débutant : confondre ces deux termes. Pourtant, ils répondent à deux questions totalement différentes !

> 🔑 **Authentification (AuthN) : "Qui es-tu ?"**  
> C'est le mécanisme par lequel on associe une identité à un utilisateur (un sujet) dans un contexte donné.  
> *Exemple : Se connecter avec un email et un mot de passe ou via Google.*

> 🛡️ **Autorisation (AuthZ) : "Qu'as-tu le droit de faire ?"**  
> C'est le processus qui détermine les permissions et les capacités de cette identité une fois qu'elle est connectée.  
> *Exemple : Un utilisateur standard ne peut pas supprimer les données d'un autre utilisateur, mais un admin le peut.*

---

## 📜 2. La Petite Histoire de l'Authentification

Pour comprendre nos outils modernes, j'ai résumé l'évolution de l'authentification à travers les âges :

| Époque | Mécanisme | Principe de sécurité | Vulnérabilité / Problème |
| :--- | :--- | :--- | :--- |
| **Pré-industrielle** | Confiance humaine implicite (ex: un sage du village se porte garant). | Basé sur la reconnaissance humaine directe. | **Impossible à passer à l'échelle (No scalability).** |
| **Médiévale** | **Sceau en cire** sur les lettres (ancêtre du token). | **Quelque chose que l'on possède** (Physical token). | **Usurpation et falsification** (Premières attaques par contournement). |
| **Révolution Industrielle** | Passphrases et secrets partagés via **Télégraphe**. | **Quelque chose que l'on sait** (Ancêtre du mot de passe statique). | Interception physique des messages. |
| **Milieu 20ème (Mainframes)** | Introduction des mots de passe multi-utilisateurs (MIT, 1961). | Stockage en texte brut (Plain text). | **Impression accidentelle du fichier de mots de passe** (Origine de l'invention du **Hashing**). |
| **Années 1970** | Cryptographie asymétrique (Diffie-Hellman, PKI) et Kerberos. | Échange de secrets sur canal non sécurisé, tickets d'accès. | Complexité d'infrastructure. |
| **Années 1990 (Internet)** | Mots de passe + **MFA (Multi-Factor Auth)**. | Combinaison de : Quelque chose que tu sais + possèdes + es (biométrie). | Faux positifs/négatifs, sécurité des templates biométriques. |
| **21ème Siècle** | OAuth, OAuth 2.0, JWT, Zero Trust, Passwordless (WebAuthn). | Tokens cryptographiques, clés publiques/privées dans le hardware. | Vol de tokens. |
| **Le Futur** | Identité décentralisée (Blockchain), cryptographie post-quantique. | Protection contre la puissance de calcul des ordinateurs quantiques. | Technologie encore très jeune. |

---

## ⚙️ 3. Sessions, JWT et Cookies : Le Trio Technique

Avant de concevoir un système, je dois maîtriser ces trois outils indispensables :

### A. Les Sessions (Stateful)
Comme HTTP est **stateless** (sans mémoire), les sessions ont été inventées pour créer un contexte temporaire côté serveur.
1. Le client se connecte.
2. Le serveur crée un **Session ID** unique et stocke les infos utilisateur (rôle, panier, etc.) dans un magasin persistant (Database ou **Redis** en mémoire RAM pour la rapidité).
3. Le serveur renvoie ce Session ID au client.
4. À chaque requête, le client présente son Session ID et le serveur va chercher ses données en base.

```
┌──────────┐            1. POST /login (username/password)            ┌──────────┐
│          ├─────────────────────────────────────────────────────────>│          │
│          │            2. Crée SessionID + Sauvegarde en cache DB/Redis │          │
│  CLIENT  │<─────────────────────────────────────────────────────────┤ SERVEUR  │
│ (Browser)│            3. Réponse avec SessionID (dans un cookie)     │          │
│          │                                                          │          │
│          │            4. GET /dashboard (avec cookie SessionID)     │          │
│          ├─────────────────────────────────────────────────────────>│          │
└──────────┘                                                          └──────────┘
```

* **Évolution de la sauvegarde des sessions** :  
  `Fichiers locaux` (Zéro scalabilité) ──> `Base de Données` (Plus robuste mais lent) ──> `Stockage distribué en mémoire` (Redis / Memcached, ultra rapide).

---

### B. Les JSON Web Tokens (JWT) (Stateless)
Créés en 2015 pour résoudre les goulots d'étranglement des sessions (coût de stockage pour des millions d'utilisateurs et latence de synchronisation entre serveurs mondiaux).

Le JWT est un token **auto-contenu** encodé en Base64 qui transporte directement les données utilisateur (User ID, rôle). Il est composé de 3 parties :

```
┌──────────────────────────────────────────────────────────────────────────┐
│ HEADER      : Métadonnées (algorithme de signature utilisé, ex: HS256)   │
├──────────────────────────────────────────────────────────────────────────┤
│ PAYLOAD     : Les claims (données : "sub" pour l'ID, "iat" pour la date,│
│               "role" pour les permissions, etc.)                        │
├──────────────────────────────────────────────────────────────────────────┤
│ SIGNATURE   : Hash généré par le serveur avec sa "Clé Secrète" pour      │
│               garantir que le token n'a pas été modifié.                 │
└──────────────────────────────────────────────────────────────────────────┘
```

> ⚠️ **Attention** : Si un attaquant vole un JWT, il peut usurper l'identité de l'utilisateur. Et comme le serveur est *stateless*, il n'a aucun moyen simple d'invalider ce token individuel avant sa date d'expiration (sauf à changer la clé secrète globale, ce qui déconnecterait absolument tout le monde d'un coup !).
>
> 💡 *La solution hybride* : Garder une "Blacklist" des JWT révoqués dans Redis. Mais cela réintroduit une vérification en base... on perd donc une partie du côté purement stateless.

---

### C. Les Cookies
Ce n'est pas un protocole d'auth, c'est juste un **moyen de stockage dans le navigateur**, contrôlé par le serveur.
* Le navigateur envoie automatiquement le cookie à chaque requête vers le domaine qui l'a émis.
* **Sécurité max** : Toujours utiliser l'option **`HttpOnly`** sur les cookies pour empêcher le code JavaScript d'y accéder (gros rempart contre les failles XSS).

---

## 🛠️ 4. Les 4 Grands Types d'Authentification

```
                 ┌────────────────────────────────────────┐
                 │       QUEL SYSTEME D'AUTH CHOISIR ?    │
                 └───────────────────┬────────────────────┘
                                     │
         ┌───────────────────────────┼───────────────────────────┐
         ▼                           ▼                           ▼
┌──────────────────┐        ┌──────────────────┐        ┌──────────────────┐
│ 1. STATEFUL      │        │ 2. STATELESS     │        │ 3. API KEYS      │
│ (Sessions + BDD) │        │ (JWT auto-signé) │        │ (Clé machine)    │
└────────┬─────────┘        └────────┬─────────┘        └────────┬─────────┘
         │                           │                           │
 • Idéal pour les           • Idéal pour les           • Idéal pour la
   applications Web           architectures de           communication
   classiques (SaaS).         microservices et           de Machine à
                              systèmes distribués.       Machine (M2M).
 • Contrôle centralisé      • Zéro stockage côté       • Simple, statique
   total : révocation         serveur, facile à          et conçu pour un
   immédiate possible.        scale mondialement.        accès program-
                                                         matique (ex: API).
```

### 4. Oauth 2.0 & OpenID Connect (OIDC) : L'Authentification Tierce

#### Le Problème de la Délégation
Auparavant, si une application de voyage voulait scanner vos billets sur Gmail, ou si un réseau social voulait importer vos contacts Google, vous deviez leur **donner votre mot de passe Google**. C'était une catastrophe absolue pour la sécurité (accès total, révocation impossible).

#### La Révolution de la Délégation par Tokens (OAuth 2.0)
Créé en 2010, OAuth 2.0 permet de partager un **Token d'accès** avec des permissions très restreintes (uniquement lire les contacts, sans droit de suppression ni accès aux emails).

* **Les 4 acteurs d'OAuth 2.0** :
  1. **Resource Owner** : Vous (le propriétaire des données).
  2. **Client** : L'application tierce qui demande l'accès (ex: Facebook).
  3. **Resource Server** : Le serveur qui détient les données (ex: l'API Google Contacts).
  4. **Authorization Server** : Le serveur qui authentifie l'utilisateur et émet les tokens.

* **Les différents Flows d'OAuth 2.0 selon la plateforme** :
  * **Authorization Code Flow** : Pour les applications s'exécutant sur un serveur sécurisé.
  * **Implicit Flow** : Conçu pour les navigateurs (aujourd'hui **fortement déconseillé** pour raisons de sécurité).
  * **Client Credentials Flow** : Pour la communication directe de machine à machine (sans utilisateur humain).
  * **Device Code Flow** : Pour les appareils sans clavier/souris (ex: l'app de votre Smart TV qui affiche un code à valider sur votre téléphone).

> ⚠️ **Piège de terminologie** : OAuth 2.0 gère uniquement l'**Autorisation** (la délégation de pouvoir). Il ne dit pas "qui" vous êtes.  
> C'est pour cela qu'en 2014 est né **OpenID Connect (OIDC)**. OIDC ajoute une couche d'**Authentification** par-dessus OAuth 2.0 en introduisant un **ID Token** (un JWT contenant le nom, l'email et la photo de profil de l'utilisateur). C'est ce protocole qui fait tourner tous les boutons *"Se connecter avec Google"* du web.

---

## 🚦 5. Autorisation et RBAC (Role-Based Access Control)

Une fois l'utilisateur authentifié, le serveur doit vérifier ce qu'il a le droit de faire. La méthode standard de l'industrie est le **RBAC** (Contrôle d'accès basé sur les rôles).

```
   ┌──────────────────┐               ┌──────────────────┐
   │    UTILISATEUR   ├──────────────>│       RÔLE       │
   └──────────────────┘               └────────┬─────────┘
                                               │
                       ┌───────────────────────┼───────────────────────┐
                       ▼                       ▼                       ▼
              ┌──────────────────┐    ┌──────────────────┐    ┌──────────────────┐
              │  PERMISSION READ │    │  PERMISSION WRITE│    │ PERMISSION DELETE│
              └──────────────────┘    └──────────────────┘    └──────────────────┘
```

* **Comment ça marche dans le code ?**
  1. L'utilisateur se connecte.
  2. Le serveur identifie son rôle (ex: `Admin`, `Moderator`, `User`).
  3. Ce rôle est injecté dans le contexte de la requête (via un Middleware).
  4. Les fonctions protégées de l'API comparent le rôle de la requête avec les permissions requises. Si l'utilisateur n'a pas les droits, l'accès est bloqué avec un code **`403 Forbidden`**.

---

## 🛡️ 6. Guide de Sécurité Backend : Les Attaques d'Auth

En tant que développeur backend, voici deux règles de sécurité non négociables à appliquer directement sur nos routes d'authentification :

### Règle n°1 : Bannir les messages d'erreur trop précis !
* ❌ **Mauvaise pratique** : Renvoyer des erreurs amicales comme *"Cet email n'existe pas"* ou *"Mot de passe incorrect"*. Un hacker utilisera ces indices pour valider des listes d'emails d'utilisateurs réels et concentrer ses attaques par force brute uniquement sur les bons comptes.
* ✅ **Bonne pratique** : Renvoyer systématiquement un message d'erreur générique et flou : **`"Identifiants incorrects"`** ou **`"Échec de l'authentification"`**, que l'email n'existe pas ou que seul le mot de passe soit faux.

---

### Règle n°2 : Bloquer les attaques temporelles (Timing Attacks)
Une timing attack est une technique subtile où un attaquant mesure le temps exact mis par le serveur pour répondre à une tentative de connexion.

* **Pourquoi le temps varie-t-il ?**
  * Si l'email n'existe pas, votre code s'arrête dès la première étape (retour direct de l'erreur). Temps de réponse : **10 ms**.
  * Si l'email existe, le serveur doit exécuter l'étape suivante : **hasher le mot de passe reçu** pour le comparer avec le hash stocké en base. Le hashing cryptographique est volontairement lent et gourmand en CPU. Temps de réponse : **200 ms**.
  * Le pirate comprend immédiatement que si la réponse prend 200 ms, l'email existe !

```
Scénario A (Email inexistant)  : Vérifie l'email (Échec) ────────> Réponse (10ms)
Scénario B (Email existant)     : Vérifie l'email (OK) ──> Hashe le mot de passe ──> Réponse (200ms)
```

* **Comment contrer ça ?**
  1. **Comparaisons à temps constant** : Utiliser des fonctions cryptographiques spécifiques qui prennent toujours le même temps, peu importe si les chaînes comparées sont identiques ou non.
  2. **Simuler un délai artificiel** : Si l'email n'existe pas, on simule une attente équivalente au temps de hashage (ex: via un `setTimeout` ou un `sleep` calibré à 200 ms) avant de renvoyer la réponse. Ainsi, toutes les réponses d'erreur prennent le même temps et l'attaquant ne peut rien en déduire.
