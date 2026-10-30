# Plusieurs applications sur un VPS

[🇬🇧 Read in English](../../server/multiple-apps.md)

Un seul VPS peut faire tourner sans problème plusieurs projets (portfolio, sites clients, API, staging…). L'essentiel est de rester organisé pour qu'ils ne se marchent pas dessus.

- [1. Organisation des dossiers](#1-organisation-des-dossiers)
- [2. Un port par app](#2-un-port-par-app)
- [3. PM2 : un processus par app](#3-pm2--un-processus-par-app)
- [4. Nginx : un fichier par domaine](#4-nginx--un-fichier-par-domaine)
- [5. Bases de données : un utilisateur par app](#5-bases-de-données--un-utilisateur-par-app)
- [6. Environnement de staging](#6-environnement-de-staging)
- [7. Isolation (facultatif) : un utilisateur Linux par app](#7-isolation-facultatif--un-utilisateur-linux-par-app)
- [8. Ressources : quand le VPS est-il plein ?](#8-ressources--quand-le-vps-est-il-plein-)

---

## 1. Organisation des dossiers

```text
/var/www/
├── portfolio/            ← static site / React build
├── shop-api/             ← Express API
├── shop-web/             ← Next.js
├── client-acme/
└── shop-api-staging/
```

Un dossier par app, un `.env` par app (`chmod 600`) et un `deploy.sh` par app.

## 2. Un port par app

Chaque app Node écoute sur son propre port, **uniquement sur `127.0.0.1`**. Tenez un petit registre pour ne jamais réutiliser un port par erreur — par ex. `/var/www/PORTS.md` :

| Port | App | Domaine |
|---|---|---|
| 3000 | shop-web (Next.js) | shop.com |
| 3001 | uptime-kuma | status.shop.com |
| 4000 | shop-api | api.shop.com |
| 4001 | client-acme-api | api.acme.com |
| 4100 | shop-api-staging | api.staging.shop.com |

Voir les ports utilisés : `sudo ss -ltnp`

Aucun de ces ports n'est ouvert dans le pare-feu — seul Nginx (80/443) est public.

## 3. PM2 : un processus par app

Donnez à chaque processus un nom clair et unique dans son `ecosystem.config.cjs`, et une limite de mémoire pour qu'une app ne puisse pas affamer les autres :

```js
module.exports = {
  apps: [{
    name: 'shop-api',
    script: 'server.js',
    cwd: '/var/www/shop-api',
    max_memory_restart: '400M',
    env: { NODE_ENV: 'production', PORT: 4000 },
  }],
};
```

```bash
pm2 start /var/www/shop-api/ecosystem.config.cjs
pm2 start /var/www/shop-web/ecosystem.config.cjs
pm2 save
pm2 status          # all apps at a glance
pm2 logs shop-api   # one app
```

## 4. Nginx : un fichier par domaine

```text
/etc/nginx/sites-available/
├── portfolio.com.conf
├── shop.com.conf
├── api.shop.com.conf
└── staging.shop.com.conf
```

Activer/désactiver un site sans le supprimer :

```bash
sudo ln -s /etc/nginx/sites-available/shop.com.conf /etc/nginx/sites-enabled/   # enable
sudo rm /etc/nginx/sites-enabled/shop.com.conf                                 # disable
sudo nginx -t && sudo systemctl reload nginx
```

Certbot peut ajouter les domaines une app à la fois : `sudo certbot --nginx -d shop.com -d www.shop.com`. Chaque commande crée un certificat distinct — c'est normal.

## 5. Bases de données : un utilisateur par app

Un seul serveur de base de données, mais **une base + un utilisateur par app**, chaque utilisateur n'ayant accès qu'à sa propre base :

- MongoDB : `readWrite` sur `shop_db` uniquement — [guide](../databases/mongodb.md#4-créer-un-utilisateur-pour-votre-app)
- PostgreSQL : `CREATE DATABASE shop_db OWNER shop_user` — [guide](../databases/postgresql.md#2-créer-un-utilisateur-et-une-base)
- MySQL/MariaDB : `GRANT ... ON shop_db.*` — [guide](../databases/mysql.md#3-créer-une-base-et-un-utilisateur)
- Redis : un utilisateur ACL par app avec un préfixe de clés (`~shop:*`) — [guide](../databases/redis.md#2-sécuriser)

Un bug ou un mot de passe divulgué dans une app ne peut alors pas toucher aux données des autres. Sauvegardez chaque base séparément ([sauvegardes](../databases/backups.md)).

## 6. Environnement de staging

Une copie de la production sur un sous-domaine (par ex. `staging.shop.com`) pour tester avant de publier :

- Dossier, port, nom PM2, `.env` et **base de données** séparés (`shop_db_staging`) — ne faites jamais pointer le staging sur la base de production.
- Déployez la branche `develop`/`staging` automatiquement ([GitHub Actions](../deploy/github-actions-auto-deploy.md)) avec son propre workflow.
- Cachez-le du public et de Google avec un mot de passe :

```bash
sudo apt install -y apache2-utils
sudo htpasswd -c /etc/nginx/.htpasswd-staging admin
```

```nginx
server {
    server_name staging.shop.com;
    auth_basic "Staging";
    auth_basic_user_file /etc/nginx/.htpasswd-staging;
    # ... same location blocks as production, pointing to the staging port/folder
}
```

## 7. Isolation (facultatif) : un utilisateur Linux par app

Pour des projets clients ou des apps moins fiables, faites tourner chacune sous son propre utilisateur Linux : une app compromise ne pourra pas lire les fichiers et `.env` des autres :

```bash
sudo adduser --disabled-password --gecos "" shop
sudo chown -R shop:shop /var/www/shop-api
sudo chmod 750 /var/www/shop-api
sudo -iu shop                     # work as that user
pm2 start ecosystem.config.cjs    # PM2 runs per user
pm2 startup                       # run the printed sudo command from your deploy user
pm2 save
```

Nginx (`www-data`) doit quand même pouvoir lire les dossiers de build statiques : `sudo usermod -aG shop www-data`, et vérifiez que le dossier de build est lisible par le groupe.

## 8. Ressources : quand le VPS est-il plein ?

Consommation mémoire approximative par app :

| App | RAM typique |
|---|---|
| Petite API Express | 80–200 Mo |
| App Next.js | 200–500 Mo |
| MongoDB | 300 Mo – 1 Go+ (utilise par défaut jusqu'à ~50 % de la RAM pour son cache) |
| PostgreSQL / MySQL | 100–500 Mo |
| Redis | 10–100 Mo (selon les données) |
| Docker + Uptime Kuma | ~150 Mo |

Signes qu'il vous faut une offre plus grosse (ou un second VPS) : RAM constamment au-dessus de 85 % avec du swap utilisé, redémarrages fréquents dus à `max_memory_restart` dans `pm2 status`, builds tués, réponses lentes avec un trafic normal. Vérifiez avec `free -h`, `pm2 monit`, `htop` — et mettez en place des [alertes](monitoring.md#3-alertes-disque-et-mémoire).

Étape suivante courante : déplacer la base de données sur son propre serveur (ou un service managé) et garder les apps sur le premier VPS.

> Passer à une offre supérieure (par ex. Hostinger KVM1 → KVM2) se fait en général depuis le tableau de bord de l'hébergeur, sans réinstallation — faites d'abord un snapshot. Voir les [liens VPS dans le README](../README.md#obtenir-un-vps).
