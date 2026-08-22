# 3. Qu'est-ce qu'un Backend, comment fonctionne-t-il et pourquoi en a-t-on besoin ?

> *Objectif : Comprendre comment fonctionne une architecture backend réelle (déployée sur AWS), et surtout pourquoi on ne peut pas tout faire tourner côté client (dans le navigateur).*

---

## 1. C'est quoi un Backend / un Serveur ? (En gros)
* **Un Backend** : C'est juste un ordinateur distant, allumé H24, qui écoute les requêtes réseau (HTTP, WebSocket, gRPC...) sur un port ouvert accessible sur Internet (genre le port 80 ou 443). Les clients s'y connectent pour envoyer ou récupérer des données.
* **Pourquoi on l'appelle "Serveur" ?** Parce qu'il *sert* du contenu. Ça peut être des fichiers statiques (HTML, CSS, JS, images) ou simplement des données structurées en JSON.

---

## 2. Le voyage d'une requête : Les étapes ("Hops") 🗺️
Quand un utilisateur tape `backend.demo.xyz/users` dans son navigateur, voici le chemin exact de la requête jusqu'à notre code :

```
[ 🖥️ MON NAVIGATEUR ] 
      │  1. Démarre la requête avec l'URL
      ▼
[ 🌐 SERVEUR DNS ]  ──> "OK, backend.demo.xyz correspond à l'IP publique d'AWS : 54.x.x.x"
      │                 • Enregistrement A : associe un domaine à une IP publique
      │                 • Enregistrement CNAME : redirige un domaine vers un autre
      ▼
[ 🛡️ FIREWALL AWS (Security Group) ] ──> Bloque TOUT par défaut sauf les ports spécifiés !
      │                                  • Port 80 (HTTP) & Port 443 (HTTPS) ouverts ? OK, ça passe.
      ▼
[ ☁️ INSTANCE AWS EC2 ] ──> C'est notre machine virtuelle Linux où tout tourne.
      │
      ▼
[ 🔀 REVERSE PROXY (Nginx) ] ──> Gère la redirection et le SSL (avec certbot/sbot) au même endroit.
      │                          • Écoute sur le port 80 (HTTP) et redirige vers le 443 (HTTPS)
      │                          • Redirige le domaine `backend.demo.xyz` vers `localhost:3001`
      ▼
[ 🟢 NODE.JS (Port 3001) ] ──> L'application finale qui tourne en tâche de fond (gérée par pm2/pn2).
```

---

## 3. Pourquoi on a BESOIN d'un Backend centralisé ?
* **Exemple concret : Le bouton "Like" d'Instagram**
    * Quand je clique sur "Like", mon appli envoie une requête HTTP au backend.
    * Le serveur reçoit l'info, regarde qui je suis (mon ID), et **sauvegarde** cette action (persistance) dans la base de données.
    * Ensuite, il cherche l'ID de mon pote (le créateur du post) et lui envoie une notification push sur son téléphone.
* **Le concept clé : La Centralisation**
    Mon application sur mon téléphone est personnalisée uniquement pour *moi*. L'application de mon pote est personnalisée pour *lui*. Il faut donc un ordinateur central (le backend) qui possède les données et l'état de **tout le monde** pour faire le lien et coordonner les actions de tous les utilisateurs.
* **En un mot : DONNÉES** (récupérer, stocker, sécuriser et traiter la donnée).

---

## 4. Pourquoi je ne peux pas mettre mon code Backend sur le Front-End ? 🚫
*Note pour mes futurs projets : Ne jamais essayer de connecter ma base de données directement depuis le navigateur ! Voici les 4 barrières physiques :*

| Barrière | Ce qui se passe dans le Navigateur (Front) | Pourquoi le Backend (Serveur) est indispensable |
| :--- | :--- | :--- |
| **1. Sécurité / Sandboxing** | Le navigateur est ultra-sécurisé (isolé de l'OS). C'est normal : on télécharge du code de serveurs tiers, donc le navigateur nous empêche d'accéder au système de fichiers ou d'écrire des logs locaux. | Le backend a un accès complet et natif à sa machine (OS) pour écrire des logs, lire des variables d'environnement secrètes, etc. |
| **2. Blocage CORS** | Le CORS est une sécurité du navigateur qui m'interdit d'appeler des API sur d'autres domaines (sauf si ces API renvoient les bons en-têtes HTTP de rechange). | Le serveur backend n'a pas de restriction CORS. Il peut se connecter et appeler n'importe quelle API externe librement. |
| **3. Accès Base de Données** | Impossible de gérer des pilotes natifs de BDD (comme `PG` pour Postgres ou MongoDB) qui demandent des connexions sockets persistantes et des données binaires. De plus, pas de gestion de **Connection Pooling** (le serveur crasherait si chaque utilisateur ouvrait sa propre connexion directe). | Le serveur gère un pool de connexions SQL ouvertes réutilisables pour tenir des milliers de requêtes/seconde sans surcharger la BDD. |
| **4. Puissance de calcul** | Le code s'exécute sur le device de l'utilisateur (qui peut être un vieux téléphone avec 256 Mo de RAM et un processeur lent). Ça ramerait ou crasherait direct sur de gros calculs. | Le serveur backend est centralisé. S'il y a trop de charge, on peut facilement augmenter sa RAM et son CPU (Scalability). |

---
