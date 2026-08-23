# Déploiement d'une app sur EC2 : DNS, ports, reverse proxy

## 1. Instance EC2 — c'est quoi

Un serveur virtuel loué chez AWS (le service de calcul de base).

- **AMI** : image de base pour démarrer l'instance (Ubuntu, Amazon Linux...)
- **Type d'instance** : taille matérielle (`t3.micro`, `t3.medium`...)
- **Facturation** : on-demand, réservé, ou spot
- **Stockage** : volumes **EBS** attachés à l'instance
- **Sécurité réseau** : gérée via les **security groups** (= firewall)

## 2. Plusieurs domaines sur une même instance

Oui, possible via un **reverse proxy** (Nginx) qui route selon le `server_name` (virtual hosting).

```nginx
server {
    server_name monsite1.com;
    location / { proxy_pass http://localhost:3000; }
}
server {
    server_name monsite2.com;
    location / { proxy_pass http://localhost:4000; }
}
```

Chaque domaine pointe (DNS type A) vers la même IP publique de l'instance. Avec Docker, on peut aussi utiliser **Traefik** ou **nginx-proxy** pour un routing automatique par labels.

## 3. Domaine vs déploiement : deux choses séparées

| Élément | Où ça se passe |
|---|---|
| Achat/gestion du nom de domaine | Chez un **registrar** (Route 53, OVH, Namecheap...) |
| Exécution du code | Sur l'**instance EC2** |
| Lien entre les deux | Un enregistrement **DNS type A** : domaine → IP publique EC2 |

Flow :
```
Domaine (registrar) → DNS record (A) → IP publique EC2 → Nginx → App (Docker)
```

### Étapes de déploiement
1. Créer l'instance EC2 (security group : ports 22, 80, 443)
2. Réserver une **Elastic IP** et l'associer (IP fixe, ne change pas au redémarrage)
3. Chez le registrar : enregistrement A `monsite.com` → Elastic IP
4. Installer Docker sur l'instance, déployer l'app (pull image GitLab Registry par ex.)
5. Installer Nginx en reverse proxy
6. Certbot pour le SSL : `certbot --nginx -d monsite.com`

## 4. Security group : port host, jamais le port container

Le security group filtre au niveau de **l'instance (host)**, pas à l'intérieur du réseau Docker.

```bash
docker run -p 8080:3000 monapp
# 3000 = port interne container (invisible depuis l'extérieur)
# 8080 = port exposé sur le host → c'est LUI qui doit être dans le security group
```

Bonne pratique : si Nginx fait le reverse proxy, bind le container en local uniquement :
`127.0.0.1:8080:3000` (pas `0.0.0.0:...`) → seul Nginx est exposé au monde.

**Inbound rules typiques à ouvrir** : 22 (SSH), 80 (HTTP), 443 (HTTPS). Rien d'autre.

## 5. Ports 80/443 vs ports d'app (3000, 8000...)

- **80/443** = ports standards du web, utilisés automatiquement par le navigateur (pas besoin de les préciser dans l'URL)
- **3000, 8000...** = conventions de dev (Angular, Django `runserver`...), aucune signification pour le web, juste pour éviter les conflits en local

**⚠️ Le backend ne doit JAMAIS écouter sur le port 80/443.**
- Un seul process peut occuper un port
- Nginx doit être le point d'entrée unique (gère HTTPS, routing, fichiers statiques)
- Les ports < 1024 nécessitent des privilèges root → mauvaise pratique pour un backend

### Flow correct
```
Internet → Port 443 (Nginx, SSL terminé ici) → proxy_pass en interne → Port 8000 (Gunicorn/Django, localhost uniquement)
```

```nginx
server {
    listen 443 ssl;
    server_name monsite.com;
    ssl_certificate     /etc/letsencrypt/live/monsite.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/monsite.com/privkey.pem;

    location /api/ {
        proxy_pass http://localhost:8000;  # backend Django/Gunicorn
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
    location / {
        root /var/www/monsite/frontend/dist;  # build Angular statique
        try_files $uri $uri/ /index.html;
    }
}

server {
    listen 80;
    server_name monsite.com;
    return 301 https://$host$request_uri;  # redirect HTTP → HTTPS
}
```

En prod, Angular n'est pas lancé en `ng serve` : on fait `ng build` → fichiers statiques servis directement par Nginx.

## 6. Résolution DNS — comment ça marche

```
1. Navigateur → resolver DNS (FAI)
2. Resolver → root servers → serveurs TLD .com
3. Serveurs .com → nameservers du registrar/Route 53
4. Nameservers → répondent avec l'IP (enregistrement A créé par nous)
5. Navigateur se connecte directement à l'IP:443
```

Résultat mis en cache selon le **TTL**.

### Ce qu'on configure nous-mêmes
Uniquement l'enregistrement DNS (type A) :

| Type | Nom | Valeur |
|---|---|---|
| A | monsite.com | Elastic IP |
| A | www.monsite.com | Elastic IP |

### Nameservers : qui les configure ?

Deux niveaux :
- **Quels nameservers pour le domaine** → configuré une fois chez le **registrar** (par défaut ceux du registrar, ou délégation vers Route 53)
- **Contenu des enregistrements DNS** (A, CNAME...) → géré là où sont les nameservers actifs

**Scénario simple (suffisant dans notre cas)** : garder les nameservers par défaut du registrar, ajouter l'enregistrement A directement dans son interface. Pas besoin de Route 53 sauf si on veut centraliser dans AWS ou utiliser CloudFront/ALB.

## 7. Pourquoi une Elastic IP

L'IP publique par défaut d'EC2 **change** à chaque redémarrage → casse le DNS. L'Elastic IP reste fixe tant qu'on ne la libère pas.

## Schéma récapitulatif complet

```
Registrar (achat domaine)
      ↓ (enregistrement A)
   Elastic IP (fixe) ─── associée à ───▶ Instance EC2
                                              ↓
                              Security Group (ports 22, 80, 443 ouverts)
                                              ↓
                                    Nginx (écoute 80 → redirige 443)
                                       ↓ SSL terminé ici (Certbot)
                            ┌──────────┴──────────┐
                            ↓                     ↓
                   /api/ → localhost:8000   / → fichiers statiques
                   (Django/Gunicorn)         (Angular build, ng build)
                   port interne, jamais
                   exposé directement
```
