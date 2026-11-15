# 24. Object Storage - Tout Ce Qu'il Faut Savoir (Partie 2)

Après avoir vu les fondamentaux (immuabilité, buckets, clés, metadata, pre-signed URLs basiques), cette fiche s'attaque aux problèmes réels de **production à grande échelle** : l'upload de fichiers géants (plusieurs Go), la gestion du réseau, l'invalidation de cache sur CDN, les Range Requests HTTP, le streaming vidéo adaptatif et l'optimisation des coûts (Egress).

---

## 💥 1. Pourquoi les Uploadeurs PUT Simples Échouent sur les Gros Fichiers

Lorsqu'on essaie de pousser un fichier de plusieurs gigaoctets (vidéos HD, captures drone, dumps de bases de données, fichiers CAD) avec un unique `PUT` HTTP, on se heurte à **4 limites architecturales majeures** :

```
   ┌────────────────────────────────────────────────────────────────────────┐
   │                  LES 4 LIMITES D'UN PUT UNIQUE SIMPLISTE                │
   ├────────────────────────────────────────────────────────────────────────┤
   │ 1. HARD LIMIT S3     : Impossible de dépasser 5 Go en un seul PUT.      │
   │ 2. TOUT OU RIEN      : Une micro-coupure Wi-Fi/4G à 4.9 Go annule TOUT.│
   │ 3. BRIDE DE DÉBIT    : Une seule connexion TCP ne peut pas saturer le   │
   │                        lien réseau (limitation window/RTT & perte).    │
   │ 4. EXPÉRIENCE USER   : Pas de pause/reprise, progression imprécise.     │
   └────────────────────────────────────────────────────────────────────────┘
```

1. **La limite stricte des 5 Go** : Imposée par l'API S3 elle-même pour tout `PUT` monolithique.
2. **Le principe du Tout-ou-Rien (All-or-Nothing)** : Si un utilisateur en 4G upload un fichier de 5 Go et que son réseau saute à 4.8 Go, la requête est annulée. Il n'y a aucun bouton *Resume*, il faut repartir de zéro (0 Go).
3. **La sous-utilisation de la bande passante (TCP Bottleneck)** : Une seule connexion TCP est bridée par la formule `Window Size / RTT` et très sensible aux pertes de paquets (effondrement de la fenêtre de congestion). Résultat : sur une connexion 500 Mbps, un transfert monopasse plafonne souvent à 30 Mbps.
4. **Une UX déplorable** : Impossible d'offrir une pause/reprise ni un calcul stable de vitesse.

---

## 🧩 2. Le Protocole Multipart Upload (Le Standard Industriel)

Pour résoudre ces 4 problèmes, AWS S3 et les providers d'object storage utilisent le **Multipart Upload**. Le principe est de découper le fichier en petits morceaux (*chunks* ou *parts*), d'uploader ces morceaux indépendamment et en parallèle, puis de demander au service d'assembler le tout.

Le protocole repose sur **3 APIs principales + 1 API d'annulation** :

```
   BROWSER / CLIENT                    BACKEND API                        AWS S3 / R2
          │                                 │                                 │
          │ ─── 1. POST /uploads/init ────► │                                 │
          │                                 │ ─── CreateMultipartUpload ────► │
          │                                 │ ◄── Return UploadID (Staging) ─ │
          │ ◄── Return UploadID + URLs ──── │                                 │
          │                                 │                                 │
          │ ─── 2. PUT Part 1 (Parallel) ───────────────────────────────────► │ (Store in Staging)
          │ ─── 2. PUT Part 2 (Parallel) ───────────────────────────────────► │ (Returns ETag 1)
          │ ─── 2. PUT Part N (Parallel) ───────────────────────────────────► │ (Returns ETag 2)
          │                                 │                                 │
          │ ─── 3. POST /uploads/complete ─► │                                 │
          │       (UploadID + Parts/ETags)  │ ─── CompleteMultipartUpload ──► │ (Metadata Commit!)
          │                                 │ ◄── Object Created (200 OK) ─── │
          │ ◄── 200 OK (Upload Finished) ── │                                 │
```

### Étape 1 : `CreateMultipartUpload`
* Le backend signale au bucket l'intention d'uploader un fichier (Key, Content-Type, Bucket).
* S3 retourne un **`Upload ID`**.
* **Attention** : À cette étape, une zone de *Staging* (zone d'attente éphémère) est créée. L'objet n'existe pas encore ! Un `GET` sur la clé renverra toujours une `404 Not Found`.

### Étape 2 : `UploadPart` (Répété N fois)
* Le client envoie chaque morceau séparément avec l'Upload ID et un `PartNumber` (ex: part 1 à 125).
* **Propriété clé** : Les requêtes HTTP sont 100% indépendantes. Elles peuvent arriver **en parallèle** et **dans n'importe quel ordre** (la part 7 peut arriver avant la part 2).
* Si une part échoue à cause du réseau, **seule cette part est réessayée** !
* Chaque part envoyée avec succès renvoie un **ETag** spécifique.

### Étape 3 : `CompleteMultipartUpload`
* Le client ou le backend envoie l'Upload ID et le manifeste ordonné de toutes les parts.
* **Performances magiques** : Que l'objet final fasse 500 Mo ou 50 Go, cette requête de finalisation prend seulement quelques millisecondes !
* **Pourquoi ?** C'est une pure opération du **Metadata Plane**. S3 ne recopie pas physiquement 50 Go de données, il met juste à jour son index de métadonnées pour faire pointer la clé vers la liste des blocs enregistrés.

### Étape 4 : `AbortMultipartUpload`
* Permet d'annuler l'upload et de vider la zone de staging.

---

## 📏 3. Calcul Dynamique de la Taille des Parts (La Limite des 10 000 Parts)

Il existe une contrainte matérielle absolue sur S3 : **Un Multipart Upload ne peut pas dépasser 10 000 parts**.

```
                      ⚠️ LE PIÈGE DE LA TAILLE FIXE ⚠️
   Si vous fixez la taille d'une part à 5 Mo par défaut :
   Max File Size = 5 Mo × 10 000 parts = 50 Go.
   --> Si l'utilisateur tente d'uploader un fichier de 60 Go, l'upload CRASHERA
       à la part 10 001 après avoir déjà consommé 50 Go de transfert !
```

### La Formule pour une Taille de Part Dynamique

Pour éviter ce crash dramatique en cours de route, le backend doit calculer la taille des parts dynamiquement en fonction de la taille totale du fichier reçu :

`Part Size = Max(Default Floor (ex: 16 Mo), Total File Size / 10000)`

### Pourquoi 16 Mo à 32 Mo est le "Sweet Spot" ?
* **Taille de Part trop petite (ex: 5 Mo)** : Risque de dépasser la limite des 10 000 parts sur les très gros fichiers + surcharge de requêtes HTTP.
* **Taille de Part trop grande (ex: 500 Mo)** : Si un réseau mobile saute, la granularité de retry est de 500 Mo, ce qui rend les réessais très coûteux.
* **Sweet Spot (16 - 32 Mo)** : Permet de gérer des fichiers jusqu'à 160 Go - 320 Go sans calculs complexes tout en gardant un coût de réessai très faible en cas de micro-coupure.

---

## 🏷️ 4. Le Piège des ETags & L'Intégrité des Données

Sur un `PUT` simple monolithique, l'**ETag** renvoyé par S3 est le hash **MD5** brut du fichier. On peut comparer le MD5 local avec l'ETag pour vérifier l'intégrité.

```
   ┌────────────────────────────────────────────────────────────────────────┐
   │                       ATTENTION AUX ETAGS MULTIPART !                  │
   ├────────────────────────────────────────────────────────────────────────┤
   │ En Multipart Upload, l'ETag NE CORRESPOND PLUS AU HASH MD5 DU FICHIER ! │
   │ Format de l'ETag Multipart : [MD5 des hashes MD5 concaténés] - [N Parts]│
   │ Exemple : "c68709db1f201e523f260471b69f6880-42"                         │
   └────────────────────────────────────────────────────────────────────────┘
```

### Comment assurer une vraie vérification d'intégrité ?
1. **Checksums S3 natifs modernes** : Utiliser les nouveaux SDK S3 qui supportent `CRC32C` ou `SHA-256` par part et effectuent une vérification globale.
2. **User Metadata SHA-256 (Recommandé)** : Calculer le hash `SHA-256` complet du fichier côté client avant l'upload, le passer dans les métadonnées utilisateur (`x-amz-meta-sha256`), et le vérifier via un background job une fois l'upload terminé.

---

## 💸 5. La Fuite Financière Masquée : AbortIncompleteMultipartUploads

Voici l'un des pièges les plus coûteux et méconnus du cloud :

> **Le Problème** : Si un utilisateur lance un multipart upload, envoie 10 Go de données dans la zone de staging, puis ferme son navigateur sans que la requête `Complete` ou `Abort` ne soit appelée, **ces 10 Go restent stockés dans S3 indéfiniment**.

* Ces fichiers en staging **n'apparaissent pas dans la console S3** ni dans les requêtes `ListObjects`.
* Pourtant, AWS vous **facture ces gigaoctets chaque mois** !

### 🛡️ La Règle d'Or Obligatoire (Jour 1)
Sur **TOUS** vos buckets S3 en production, configurez immédiatement une règle de Lifecycle pour annuler automatiquement les multipart uploads incomplets au bout de 7 jours (`AbortIncompleteMultipartUpload`).

---

## 🏗️ 6. Architecture de Production Complète (Pre-signed POST + Multipart)

Voici le schéma architectural de référence utilisé par les grandes plateformes web pour gérer les uploads de gros fichiers directement depuis le navigateur :

```
┌─────────┐                ┌─────────┐                 ┌─────────┐              ┌───────────────┐
│ Browser │                │ API Dev │                 │ DB SQL  │              │ AWS S3 Bucket │
└────┬────┘                └────┬────┘                 └────┬────┘              └───────┬───────┘
     │                          │                           │                           │
     │ 1. POST /upload/init     │                           │                           │
     │   (name, size, type)     │                           │                           │
     ├─────────────────────────►│                           │                           │
     │                          │ 2. Compute Part Size      │                           │
     │                          │ 3. CreateMultipartUpload  │                           │
     │                          ├───────────────────────────┼──────────────────────────►│
     │                          │◄──────────────────────────┼───────────────────────────┤ (Upload ID)
     │                          │ 4. INSERT Row (pending)   │                           │
     │                          ├──────────────────────────►│                           │
     │                          │ 5. Generate Presigned     │                           │
     │                          │    URLs (Batch de 100)    │                           │
     │ 6. Return UploadID +     │                           │                           │
     │    Part URLs             │                           │                           │
     │◄─────────────────────────┤                           │                           │
     │                          │                           │                           │
     │ 7. PUT Part 1..N (Direct Parallel Upload - 4 à 6 streams)                        │
     ├─────────────────────────────────────────────────────────────────────────────────►│
     │◄─────────────────────────────────────────────────────────────────────────────────┤ (Returns ETags)
     │                          │                           │                           │
     │ 8. POST /upload/complete │                           │                           │
     │    (UploadID + ETags)    │                           │                           │
     ├─────────────────────────►│ 9. CompleteMultipartUpload│                           │
     │                          ├───────────────────────────┼──────────────────────────►│
     │                          │◄──────────────────────────┼───────────────────────────┤ (200 OK)
     │                          │ 10. HEAD Object Check     │                           │
     │                          ├───────────────────────────┼──────────────────────────►│
     │                          │ 11. UPDATE DB (ready)     │                           │
     │                          ├──────────────────────────►│                           │
     │ 12. 200 OK (Success!)    │ 12. Emit file.uploaded    │                           │
     │◄─────────────────────────┤     to Event Queue        │                           │
```

### Recommandations clés d'implémentation :
1. **Pre-sign par batches (pas 10 000 URLs d'un coup)** : Si le fichier nécessite 1000 parts, ne générez pas 1000 URLs dans la réponse JSON initiale (cela ferait un JSON de plusieurs Mo). Générez les URLs par lots de 100 au fur et à mesure de la progression du navigateur.
2. **Limiter la concurrence navigateur (4 à 6 streams)** : Uploader 4 à 6 parts en parallèle est le compromis idéal. Au-delà, les requêtes se battent pour la même bande passante et risquent de provoquer des timeouts.
3. **Exposer l'en-tête ETag dans la config CORS S3** : Configurer `ExposeHeader: ETag`.
   *Sans cette directive CORS, le JavaScript du navigateur ne pourra pas lire l'ETag dans la réponse de S3 et ne pourra jamais compléter l'upload !*

4. **Reprise sur panne (Resumable Uploads)** : Si l'utilisateur rafraîchit la page à 80% d'upload, le navigateur demande au backend : *"Ai-je un upload en cours pour ce fichier ?"*. Le backend interroge S3 via `ListParts(UploadID)`, identifie les parts déjà reçues en staging, et ne renvoie des URLs pré-signées que pour les parts manquantes !

---

## 📥 7. Architecture de Téléchargement & Stratégies CDN

Sur le web, les téléchargements et affichages de fichiers représentent **90% à 99% du trafic d'un object storage**.

```
   ❌ RÈGLE D'OR : NE JAMAIS PROXYFIER LES TÉLÉCHARGEMENTS PAR LE BACKEND !
   Faire `GET Object` sur S3 puis copier les octets vers la réponse client HTTP :
   - Bloque un worker/thread backend pendant tout le téléchargement.
   - Fait exploser les coûts d'Egress réseau de votre infrastructure cloud.
   - Plafonne la vitesse de téléchargement pour les utilisateurs éloignés.
```

### Les 3 Patrons de Téléchargement en Production

```
┌───────────────────────────────────────────────────────────────────────────────┐
│                             LES 3 PATRONS DE DOWNLOAD                         │
├───────────────────────────────────────────────────────────────────────────────┤
│ PATRON 1 : BUCKET PRIVÉ + CDN PUBLIC (Assets Publics)                         │
│ - Le bucket est privé, seul le CDN (Cloudflare/CloudFront) y a accès.          │
│ - Utilisation de clés basées sur le hash (Content-Addressed Keys).            │
│ - `Cache-Control: max-age=31536000, immutable`. Gratuit en Egress + Ultra-rapide│
├───────────────────────────────────────────────────────────────────────────────┤
│ PATRON 2 : PRE-SIGNED GET URLS (Fichiers Privés Ponctuels)                    │
│ - L'API vérifie les droits, puis génère un Pre-signed GET (valide 5-15 min).   │
│ - Astuce Cache CDN : Arrondir l'expiration à l'heure suivante (`top-of-hour`)│
│   pour que tous les users d'une même heure reçoivent la MÊME URL (Cache Hit!). │
├───────────────────────────────────────────────────────────────────────────────┤
│ PATRON 3 : CDN SIGNED COOKIES / EDGE AUTH (Streaming & Fichiers Multiples)   │
│ - L'autorisation se fait directement à l'Edge (Cloudflare Workers / Lambda). │
│ - Idéal pour le streaming vidéo (évite de signer 10 000 segments HLS).        │
└───────────────────────────────────────────────────────────────────────────────┘
```

---

## 🔍 8. Range Requests (HTTP 206 Partial Content)

Les **Range Requests** permettent à un client de demander seulement une portion d'un fichier (ex: *"donne-moi les octets du 1 millionième au 3 millionième"*).

* Entête Client : `Range: bytes=1000000-3000000`
* Réponse Serveur : `HTTP/1.1 206 Partial Content` avec `Content-Range: bytes 1000000-3000000/50000000`

### Les Cas d'Usage Incontournables
1. **Video Seeking (Saut dans une vidéo)** : Quand l'utilisateur déplace le curseur à la 30ème minute d'une vidéo, le lecteur calcule l'offset d'octets correspondant et lance un `Range Request`. Le navigateur ne télécharge pas les 30 premières minutes !
2. **Reprise de Téléchargement (Download Managers)** : Si un téléchargement s'interrompt à 400 Mo, la reprise demande la suite à partir de 400 Mo.
3. **Lecture d'en-têtes sans charger le fichier (Magic Numbers)** :
   * Exemple : Télécharger uniquement les 8 premiers octets d'un fichier pour valider le header PNG sans télécharger l'image complète.

---

## 🎬 9. Streaming Vidéo Adaptatif (HLS / DASH)

Pour les plateformes de vidéo (Netflix, YouTube), on ne sert pas un unique gros fichier `.mp4` via Range Requests. On utilise le **Streaming Adaptatif** :

```
                  ┌───────────────────────────────────────────┐
                  │          FICHIER SOURCE (4K .MP4)         │
                  └─────────────────────┬─────────────────────┘
                                        │ (Pipeline de Transcodage)
                                        ▼
      ┌─────────────────────────────────┼─────────────────────────────────┐
      ▼                                 ▼                                 ▼
┌──────────────┐                 ┌──────────────┐                 ┌──────────────┐
│  1080p HLS   │                 │   720p HLS   │                 │   480p HLS   │
│ (Segments .ts)                 │ (Segments .ts)                 │ (Segments .ts)
└──────┬───────┘                 └──────┬───────┘                 └──────┬───────┘
       │                                │                                │
       └────────────────────────────────┼────────────────────────────────┘
                                        │
                                        ▼
                         ┌─────────────────────────────┐
                         │   MANIFEST (.m3u8 / .mpd)   │
                         └─────────────────────────────┘
```

### Le Fonctionnement Technique
1. **Transcodage** : Le fichier source est converti en plusieurs résolutions (1080p, 720p, 480p).
2. **Segmentation** : Chaque version est découpée en petits fichiers de 2 à 6 secondes (segments `.ts` ou `.m4s`).
3. **Manifest (`.m3u8`)** : Un fichier texte dresse l'index complet des segments et des résolutions disponibles.
4. **Adaptive Bitrate (ABR)** : Le lecteur vidéo du navigateur télécharge le manifeste, mesure la vitesse réseau en continu, et passe de 1080p à 480p de façon transparente en cas de baisse de connexion !

> 💡 **Le rôle de l'Object Storage** : S3 sert simplement de stockage statique ultra-performant pour les millions de petits segments et les manifestes. La complexité réside dans les workers de transcodage (FFmpeg / AWS Elemental MediaConvert).

---

## 💰 10. Modèle Financier & Optimisation des Coûts (Egress)

Il existe **3 axes de facturation** en Object Storage :

| Axe de Coût | Description | Astuce d'Optimisation |
| :--- | :--- | :--- |
| **1. Stockage** | Facturé au Go / Mois (~0.023$/Go sur S3 Standard). | Utiliser les règles S3 Lifecycle pour basculer vers *Infrequent Access* ou *Glacier*. |
| **2. Opérations** | Facturé par paquet de 1 000 requêtes. **Les écritures (`PUT`) coûtent 10x plus cher que les lectures (`GET`)**. | Éviter les petites requêtes d'écriture répétitives. Batcher ou grouper. |
| **3. Egress (Sortie)** | Facturé au Go transféré du Cloud vers Internet (~0.09$/Go). **C'est le poste de dépense le plus élevé !** | Placer un CDN devant ou utiliser **Cloudflare R2** (0$ de frais d'Egress). |

---

## 📋 Checklist de Synthèse pour l'Ingénieur Backend

- [x] **Toujours utiliser le Multipart Upload** pour tout fichier supérieur à 100 Mo.
- [x] **Calculer la taille des parts dynamiquement** (`Total Size / 10000`) avec un plancher à 16-32 Mo.
- [x] **Configurer la règle Lifecycle `AbortIncompleteMultipartUploads`** à 7 jours dès le premier jour sur tous les buckets.
- [x] **Ne jamais proxyfier les téléchargements via le serveur backend** ; utiliser des Pre-signed GET URLs ou un CDN.
- [x] **Exposer l'en-tête CORS `ETag`** sur le bucket S3 pour débloquer les uploads JavaScript.
- [x] **Valider les fichiers post-upload avec `HEAD`** et utiliser des Pre-signed POST URLs restreintes.

---