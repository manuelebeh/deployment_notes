# Deployment Notes

[🇬🇧 English](../README.md) · 🇫🇷 Français

Des guides pratiques, prêts à copier-coller, pour déployer des projets JavaScript (MERN / Node.js) sur un VPS Linux ou sur Vercel — d'un serveur tout neuf jusqu'au HTTPS, aux bases de données, aux déploiements automatiques et aux sauvegardes.

> Dernière mise à jour : **octobre 2026** — pour **Ubuntu 24.04 LTS** et **Ubuntu 26.04 LTS**.

---

## Obtenir un VPS

Les guides fonctionnent sur n'importe quel VPS Ubuntu. Si vous en cherchez un, voici les offres Hostinger avec **20 % de réduction** :

- [Hostinger VPS KVM1](https://www.hostinger.com/cart?product=vps%3Avps_kvm_1&period=12&referral_type=cart_link&REFERRALCODE=FULEBEHEMFY4&referral_id=01958194-af72-7176-905a-70836d0d6ab7)
- [Hostinger VPS KVM2](https://www.hostinger.com/cart?product=vps%3Avps_kvm_2&period=12&referral_type=cart_link&REFERRALCODE=FULEBEHEMFY4&referral_id=01958196-63c2-703b-9ce3-2149c8ba44c6)
- [Hostinger VPS KVM4](https://www.hostinger.com/cart?product=vps%3Avps_kvm_4&period=12&referral_type=cart_link&REFERRALCODE=FULEBEHEMFY4&referral_id=01958196-3bcc-717b-89a0-a86f7f605700)
- [Hostinger VPS KVM8](https://www.hostinger.com/cart?product=vps%3Avps_kvm_8&period=12&referral_type=cart_link&REFERRALCODE=FULEBEHEMFY4&referral_id=01958196-12eb-71dc-9f2a-3f506c4ebc7c)

Au moment de choisir le système, prenez un modèle **Ubuntu 24.04** ou **Ubuntu 26.04** simple (sans application préinstallée).

---

## Guides

### 1. Serveur

| Guide | Ce que vous obtenez |
|---|---|
| [Configuration initiale du serveur](server/initial-server-setup.md) | Utilisateur sudo, clés SSH, SSH durci, pare-feu, Fail2ban, mises à jour de sécurité automatiques, swap |
| [Nginx & SSL (Let's Encrypt)](server/nginx-and-ssl.md) | Domaine/DNS, blocs serveur Nginx pour SPA et API, HTTPS gratuit avec Certbot, renouvellement automatique |
| [Caddy (alternative à Nginx)](server/caddy.md) | HTTPS automatique, toute la config MERN/Next.js en ~10 lignes |
| [Cloudflare devant votre VPS](server/cloudflare.md) | Proxy/CDN, SSL Full (strict), vraie IP des visiteurs, verrouillage de l'origine, codes d'erreur |
| [Monitoring & alertes](server/monitoring.md) | Health checks, surveillance de disponibilité (Uptime Kuma), alertes disque/mémoire, logs |
| [Plusieurs applications sur un VPS](server/multiple-apps.md) | Ports, PM2, Nginx, bases de données par app, staging, isolation, dimensionnement |

### 2. Déploiement

| Guide | Ce que vous obtenez |
|---|---|
| [Stack MERN sur un VPS](deploy/mern-on-vps.md) | Node.js, API Express avec PM2, build React (Vite), Nginx, HTTPS, mise à jour |
| [Next.js sur un VPS](deploy/nextjs-on-vps.md) | `next start` ou build standalone, déploiements sans coupure et rollback, streaming |
| [Docker Compose sur un VPS](deploy/docker-compose-on-vps.md) | Docker Engine, une API Node + base de données en conteneurs derrière Nginx |
| [Déploiement automatique avec GitHub Actions](deploy/github-actions-auto-deploy.md) | Déploiement à chaque push sur `main` via SSH |
| [MERN sur Vercel](deploy/vercel-mern.md) | Backend Express sans configuration, réécritures React Router, référence `vercel.json` |
| [Coolify & Dokploy](deploy/coolify-dokploy.md) | PaaS auto-hébergé : déploiement par push depuis un tableau de bord sur votre VPS |

### 3. Bases de données

| Guide | Version couverte |
|---|---|
| [MongoDB](databases/mongodb.md) | MongoDB 9.0 Community |
| [PostgreSQL](databases/postgresql.md) | PostgreSQL 16 / 18 |
| [MySQL](databases/mysql.md) | MySQL 8.0 / 8.4 LTS (9.7 LTS en option) |
| [MariaDB](databases/mariadb.md) | MariaDB 10.11 / 11.8 LTS (12.3 LTS en option) |
| [Redis](databases/redis.md) | Redis Open Source 8.x |
| [Sauvegardes des bases de données](databases/backups.md) | Dumps quotidiens automatiques + restauration pour toutes les bases ci-dessus |

---

## Ordre conseillé pour un nouveau VPS

1. [Configuration initiale du serveur](server/initial-server-setup.md)
2. Un guide de base de données (par ex. [MongoDB](databases/mongodb.md))
3. [Stack MERN](deploy/mern-on-vps.md) ou [Next.js](deploy/nextjs-on-vps.md) — ou [Docker Compose](deploy/docker-compose-on-vps.md)
4. [Nginx & SSL](server/nginx-and-ssl.md) (ou [Caddy](server/caddy.md)), éventuellement derrière [Cloudflare](server/cloudflare.md)
5. [Déploiement automatique avec GitHub Actions](deploy/github-actions-auto-deploy.md)
6. [Sauvegardes des bases de données](databases/backups.md)
7. [Monitoring & alertes](server/monitoring.md)

Vous préférez un tableau de bord à la ligne de commande ? Commencez par [Coolify & Dokploy](deploy/coolify-dokploy.md) sur un VPS neuf, à la place des étapes 3 à 5.

---

## Versions utilisées dans ces guides (octobre 2026)

| Logiciel | Recommandé | Notes |
|---|---|---|
| Ubuntu | 24.04 LTS ou 26.04 LTS | 22.04 fonctionne encore pour la plupart des étapes mais arrive en fin de support standard en avril 2027 |
| Node.js | **26 LTS** (ou 24 LTS) | Node 26 est passé en Active LTS le 28 oct. 2026 ; Node 24 est en maintenance jusqu'en avril 2028 ; Node 20 est en fin de vie |
| Nginx | Paquet Ubuntu | 1.24 sur 24.04, 1.28 sur 26.04 |
| Caddy | 2.11 | Dépôt officiel |
| Next.js | 16 | |
| Certbot | snap | Recommandation officielle de l'EFF |
| MongoDB | 9.0 | Dépôt officiel `mongodb-org` |
| PostgreSQL | 18 | Par défaut sur 26.04 ; via le dépôt officiel PGDG sur 24.04 |
| MySQL | 8.4 LTS | Par défaut sur 26.04 (8.0 sur 24.04) |
| MariaDB | 11.8 LTS | Par défaut sur 26.04 (10.11 sur 24.04) |
| Redis | 8.x | Dépôt officiel `packages.redis.io` |
| PM2 | 7.x | `npm install -g pm2` |
| Docker Engine | dernière | Dépôt officiel |
| Coolify / Dokploy | 4.5 / 0.30 | |

---

## Conventions

- `yourdomain.com`, `api.yourdomain.com`, `your_vps_ip`, `deploy`, `app_db`, `app_user`… sont des **valeurs d'exemple** — remplacez-les par les vôtres.
- Les commandes sont écrites pour un **utilisateur non-root avec `sudo`** (voir la [configuration initiale](server/initial-server-setup.md)). Si vous êtes connecté en `root`, vous pouvez retirer `sudo`.
- Dans `nano` : enregistrer avec `Ctrl + O` → `Entrée`, quitter avec `Ctrl + X`.
- Ne commitez jamais de fichiers `.env` ni de mots de passe dans Git.

## Contribuer

Une commande obsolète ou une coquille ? [Ouvrez une issue](https://github.com/manuelebeh/deployment_notes/issues/new/choose) ou une pull request — voir [CONTRIBUTING.md](../CONTRIBUTING.md).

## Licence

[MIT](../LICENSE)
