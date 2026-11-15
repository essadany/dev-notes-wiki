# 20. La Sécurité Backend (Everything You Need to Know)

La sécurité en backend n'est pas une simple fonctionnalité ou un bug à corriger : c'est un **état d'esprit permanent (*mindset*)**. En tant que développeur backend, une seule faille ignorée peut entraîner des fuites de données dévastatrices, des pertes financières énormes et la ruine de la réputation d'une entreprise. Voici mon carnet de notes complet rédigé avec mon ton de dev junior, mes schémas ASCII et mes antisèches sans fioritures !

---

## 🧠 1. Le Mindset de la Sécurité : "Penser comme un Attaquant"

La plupart des failles naissent d'une seule cause : **les hypothèses (*assumptions*) faites par le développeur**.

```
┌─────────────────────────────────────────────────────────────┐
│                    LE PIÈGE DES HYPOTHÈSES                  │
├─────────────────────────────────────────────────────────────┤
│  En dev / sous deadline, on pense au "Happy Path" :         │
│  • "L'utilisateur va remplir le formulaire correctement."   │
│  • "Personne ne va ouvrir le Network Tab du navigateur."    │
│  • "La requête provient forcément de notre propre Front."   │
├─────────────────────────────────────────────────────────────┤
│  L'attaquant cherche l'endroit exact où vous avez supposé  │
│  que les données reçues seraient propres et bienveillantes !│
└─────────────────────────────────────────────────────────────┘
```

> ⚠️ **Règle d'or** : Votre application backend parle plusieurs langues (SQL avec la BDD, Shell avec l'OS, HTML/JS avec le navigateur). Les vulnérabilités apparaissent lorsque **les données envoyées dans un langage traversent la frontière d'un autre langage et sont exécutées comme du code** !

---

## 💉 2. Les Attaques par Injection (Injection Attacks)

### A. SQL Injection (SQLi)

* **Le Problème** : La concaténation brute de chaînes de caractères pour construire des requêtes SQL.

```sql
-- Requête vulnérable par concaténation :
SELECT * FROM users WHERE email = 'USER_INPUT';

-- Si l'attaquant injecte : ' OR 1=1 --
SELECT * FROM users WHERE email = '' OR 1=1 --';
```

* **Pourquoi ça marche ?** : Le caractère `'` ferme prématurément la chaîne de données, `OR 1=1` crée une condition toujours vraie, et `--` commente la fin de la requête SQL. Résultat : l'attaquant extrait l'intégralité de la table `users` !
* **Attaque destructive** : L'injection de `; DROP TABLE users; --` peut supprimer définitivement la table si les permissions DB sont trop élevées.
* **La Solution : Les Requêtes Paramétrées (*Prepared Statements*)** :

```sql
-- Utilisation de slots de variables ($1, $2)
SELECT * FROM users WHERE email = $1;
```

> 💡 **Principe** : Le moteur SQL traite le contenu du paramètre `$1` exclusivement comme de la donnée brute (du texte), éliminant toute possibilité de confusion avec des instructions SQL exécutables.
* **NoSQL Injection** : Même avec MongoDB, passer directement un objet JSON utilisateur non filtré permet d'injecter des opérateurs comme `$ne` (Not Equal) ou `$gt` pour contourner l'authentification.

### B. Command Injection (Injection de Commandes OS)

* **Le Problème** : Exécuter une commande système (ex: `ffmpeg` pour redimensionner une image) en concaténant directement le nom du fichier fourni par l'utilisateur.
* **Exemple d'attaque** : Si l'utilisateur nomme son fichier `image.jpg; rm -rf /`, le shell exécute la première commande puis la commande destructive.
* **La Solution** : Utiliser les fonctions natives du langage (ex: `execFile` avec un tableau d'arguments `['height', '120', filename]`) qui passent les arguments directement au processus système sans passer par l'interpréteur de shell.

---

## 🔑 3. Authentification & Gestion des Mots de Passe

### A. Faut-il utiliser un Provider Auth (Clerk, Auth0) ou tout coder soi-même ?
* **Recommandation** : En production, privilégier un provider d'authentification tiers (Clerk, Auth0, Kinde). Cela fait gagner des semaines de dev et évite de mal gérer la révocation de session, le Multi-Factor Auth (MFA) ou le Single Sign-On (SSO).
* **Migration** : Ne réimplémenter son propre système d'auth in-house que lorsque l'entreprise a l'échelle et la rentabilité nécessaires pour couvrir les coûts d'équipe dédiés.

### B. L'Évolution du Stockage des Mots de Passe

```
┌─────────────────────────────────────────────────────────────┐
│ 1. Texte Brut (Plain Text) ❌                                │
│    En cas de breach BDD, tous les mots de passe sont fuités.│
├─────────────────────────────────────────────────────────────┤
│ 2. Hachage Simple (MD5 / SHA-256) ❌                        │
│    Vulnérable aux attaques par Rainbow Tables (lookup).     │
├─────────────────────────────────────────────────────────────┤
│ 3. Hachage + Sel (Salt) ❌ (si algo rapide)                 │
│    Ajout d'une chaîne aléatoire unique par user. Empêche    │
│    les Rainbow Tables mais vulnérable au Brute-Force GPU.  │
├─────────────────────────────────────────────────────────────┤
│ 4. Algorithmes Lents (Argon2id, BCrypt, SCrypt) ✅          │
│    Intègrent un "Cost Factor". Ralentissent volontairement  │
│    le calcul pour rendre le brute-forcing GPU impossible.   │
└─────────────────────────────────────────────────────────────┘
```

> 🛡️ **Standard Actuel** : Utilisez **Argon2id** (ou **BCrypt**) avec un coût réglé pour que le hachage prenne ~300 à 500 ms (imperceptible pour un utilisateur réel, catastrophique pour un attaquant offline).

### C. Sessions d'Authentification (Stateful vs Stateless JWT)

```
┌─────────────────────────────────────────────────────────────┐
│                SESSIONS STATEFUL (Recommandé)               │
├─────────────────────────────────────────────────────────────┤
│ • Le serveur génère un Session ID cryptographique aléatoire │
│   (128 à 256 bits).                                         │
│ • La session + métadonnées (User ID, IP, User-Agent) sont   │
│   stockés en BDD ou Redis.                                  │
│ • Révocation instantanée possible à tout moment.             │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│                 JWT STATELESS (À manipuler avec soin)       │
├─────────────────────────────────────────────────────────────┤
│ • Header + Payload (claims) + Signature HMAC/RSA.           │
│ • Le payload B64 est LISIBLE par tous (ne rien y cacher).    │
│ • Révocation difficile (nécessite Access Token court 5-10m  │
│   + Refresh Token + Blacklist Redis).                       │
└─────────────────────────────────────────────────────────────┘
```

### D. Sécuriser les Cookies HTTP
Lors du stockage d'un Session ID ou d'un JWT dans un Cookie, configurez **toujours** ces 3 flags :
1. `HttpOnly = true` : Interdit l'accès au cookie via JavaScript (bloque le vol de token via XSS).
2. `Secure = true` : Force l'envoi du cookie uniquement sur un canal chiffré HTTPS.
3. `SameSite = Lax` ou `Strict` : Empêche le navigateur d'envoyer le cookie lors de requêtes cross-origin malveillantes (bloque CSRF).

### E. Rate Limiting en 3 Couches
Pour contrer le Brute-Force et les pannes par déni de service :
* **Couche 1 (Per-IP)** : Limite le nombre de tentatives par IP (ex: 10/min). *(Invisibilisé par les VPN/Botnets)*.
* **Couche 2 (Per-Account)** : Bloque temporairement un compte après N échecs (ex: 5 échecs = verrouillage 15m).
* **Couche 3 (Global)** : Limite système globale sur l'endpoint de login pour déclencher une alerte en cas d'attaque massive distribuée.

---

## 🚫 4. Autorisation : Éviter les Failles d'Accès

### A. Broken Object Level Authorization (BOLA / IDOR)
* **La Faille** : Un utilisateur authentifié modifie l'ID dans la requête pour accéder à la ressource d'un autre utilisateur (`GET /invoices?id=102` au lieu de `101`).
* **La Solution** : Vérifier l'appartenance **au niveau de la requête BDD**, pas uniquement au niveau du router !

```sql
-- ❌ Incomplet (Vérifié au router mais pas en BDD)
SELECT * FROM invoices WHERE id = 102;

-- ✅ Correct (Vérification au point d'accès)
SELECT * FROM invoices WHERE id = 102 AND user_id = $current_user_id;
```

> 💡 **Astuce de Sécurité (Info Leakage)** : Si la requête ne renvoie rien, renvoyez une erreur **`404 Not Found`** au lieu d'un `403 Forbidden`. Cela empêche l'attaquant de tester la présence d'IDs valides dans votre système !

### B. Broken Function Level Authorization (BFLA)
* **La Faille** : Un utilisateur standard accède à une fonction d'administration (`POST /admin/invoices/purge`) car le serveur n'a pas vérifié son rôle.
* **Règle** : Ne comptez jamais sur "la sécurité par l'obscurité" (cacher l'URL de l'admin). Ajoutez un middleware de rôle strict (`RequireRole('admin')`).

### C. Modèle Mental d'Autorisation
* **Horizontal** (BOLA/IDOR) : Un utilisateur tente d'accéder aux données d'un autre utilisateur de même niveau.
* **Vertical** (BFLA) : Un utilisateur tente d'élever ses privilèges pour exécuter des fonctions réservées aux admins.
* **Principes clés** : Centraliser la logique AuthZ, appliquer le **Default Deny** (tout refuser par défaut), automatiser les tests d'autorisation dans le CI/CD et tenir des **Audit Logs**.

---

## 🧼 5. XSS, CSRF & Bonnes Pratiques de Production

### A. Cross-Site Scripting (XSS)
* **Principe** : L'attaquant réussit à injecter du code JavaScript malveillant qui s'exécute dans le navigateur d'autres utilisateurs.
* **Causes** : Stockage de HTML/Markdown non assaini (`Stored XSS`) ou affichage direct de paramètres d'URL (`Reflected XSS`).
* **Protection** :
  1. **Sanitisation systématique** côté backend des entrées utilisateurs.
  2. **Cookies `HttpOnly`** pour empêcher le vol de session.
  3. **Content Security Policy (CSP)** : Header HTTP indiquant au navigateur quelles sources de scripts sont autorisées.

### B. Cross-Site Request Forgery (CSRF)
* **Principe** : Piéger le navigateur d'un utilisateur connecté pour qu'il envoie à son insu une requête modifiant des données sur un autre site.
* **Protection** : Cookies configurés avec `SameSite=Lax` ou `Strict`, tokens anti-CSRF.

### C. La Checklist de Production
1. **Secrets Management** : Jamais de clés API ni de passwords en dur dans le code Git ! Utiliser des variables d'environnement (`.env`) ou AWS Parameter Store / HashiCorp Vault. En cas de commit accidentel, **révoquer et faire tourner le secret immédiatement**.
2. **Debug Mode en Prod** : Régler le niveau de log sur `INFO` en prod. Les stack traces et logs `DEBUG` exposent l'architecture interne et la structure SQL aux attaquants.
3. **Security Headers** : Activer les middlewares de sécurité standard (ex: `Helmet` en JS) pour injecter `X-Frame-Options` (anti-clickjacking), `CSP`, et `HSTS`.

---

## 📌 Résumé : La Stratégie de Défense en Profondeur (*Defense in Depth*)

```
┌─────────────────────────────────────────────────────────────┐
│ 1. Validation des Entrées (Strict input validation at entry)│
├─────────────────────────────────────────────────────────────┤
│ 2. Opérations Paramétrées (Prepared statements & OS arrays) │
├─────────────────────────────────────────────────────────────┤
│ 3. Contrôle d'Accès en BDD (Point-of-access AuthZ)          │
├─────────────────────────────────────────────────────────────┤
│ 4. En-têtes & Cookies Sécurisés (HttpOnly, SameSite, CSP)   │
├─────────────────────────────────────────────────────────────┤
│ 5. Audit Logs & Observabilité (Monitoring des échecs AuthZ) │
└─────────────────────────────────────────────────────────────┘
```

> 🔗 **Ressources recommandées par Sriniously pour aller plus loin** :
> * **PortSwigger Web Security Academy** (Gratuit, cours & labs d'injections, XSS, CSRF).
> * **OWASP Top 10 & OWASP Cheat Sheet Series** (La référence mondiale des vulnérabilités applicatives).
