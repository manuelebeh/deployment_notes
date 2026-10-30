# Coolify & Dokploy : votre propre « Vercel » sur un VPS

[🇬🇧 Read in English](../../deploy/coolify-dokploy.md)

[Coolify](https://coolify.io) et [Dokploy](https://dokploy.com) sont des **PaaS auto-hébergés**, gratuits et open source. Vous les installez une fois sur votre VPS, puis vous déployez vos apps depuis un tableau de bord web : connectez un dépôt GitHub, définissez les variables d'environnement et un domaine, et chaque push redéploie — HTTPS, bases de données et sauvegardes inclus.

- [Quand les utiliser](#quand-les-utiliser)
- [Coolify ou Dokploy](#coolify-ou-dokploy)
- [Avant d'installer](#avant-dinstaller)
- [Installer Coolify](#installer-coolify)
- [Installer Dokploy](#installer-dokploy)
- [Déployer une app MERN / Next.js](#déployer-une-app-mern--nextjs)
- [Bonnes pratiques](#bonnes-pratiques)

---

## Quand les utiliser

| Prenez un PaaS (Coolify/Dokploy) si… | Faites-le à la main ([MERN](mern-on-vps.md), [Next.js](nextjs-on-vps.md)) si… |
|---|---|
| Vous déployez beaucoup d'apps et voulez un tableau de bord | Vous avez une ou deux apps |
| Vous voulez le déploiement par push, des previews, des bases en un clic | Vous voulez comprendre et maîtriser chaque élément |
| Ça ne vous dérange pas que tout tourne dans Docker | Vous avez un petit VPS (< 2 Go de RAM) |
| | Vous préférez moins de briques à maintenir |

## Coolify ou Dokploy

| | Coolify | Dokploy |
|---|---|---|
| Dernière version (oct. 2026) | v4.5.x | v0.30.x |
| Serveur minimum | 2 vCPU, 2 Go RAM, 20 Go+ de disque | 2 Go RAM, 30 Go de disque |
| Reverse proxy | Traefik (ou Caddy) | Traefik |
| Builds | Nixpacks, Dockerfile, Docker Compose, statique | Nixpacks, Railpack, Dockerfile, Buildpacks, Docker Compose |
| Services en un clic | Des centaines (bases de données, n8n, Uptime Kuma, Plausible…) | Modèles (bases de données, nombreuses apps) |
| Multi-serveurs | Oui | Oui (Docker Swarm) |
| Port du tableau de bord | 8000 | 3000 |
| Communauté | Très grande | En forte croissance |

Les deux sont bons. Coolify propose plus de services prêts à l'emploi ; Dokploy est plus léger et a une interface très épurée.

## Avant d'installer

- Utilisez un VPS **neuf** (Ubuntu 24.04 LTS recommandé). Ne l'installez pas sur un serveur qui fait déjà tourner Nginx/Caddy via les autres guides : Coolify/Dokploy ont besoin des ports **80 et 443** pour leur propre proxy.
- Faites d'abord la [configuration initiale du serveur](../server/initial-server-setup.md) (utilisateur, clés SSH, Fail2ban, mises à jour).
- **Docker contourne UFW** ([détails](docker-compose-on-vps.md#avertissement-pare-feu)) : les ports publiés par ces outils sont accessibles même si UFW dit le contraire. Utilisez le **pare-feu de votre hébergeur** (Hostinger : hPanel → VPS → *Pare-feu*) pour restreindre le port du tableau de bord.
- Faites pointer un domaine vers le VPS pour le tableau de bord, par ex. `panel.yourdomain.com`.

## Installer Coolify

```bash
curl -fsSL https://cdn.coollabs.io/coolify/install.sh -o coolify-install.sh
less coolify-install.sh            # read what you're about to run
sudo bash coolify-install.sh
```

Le script installe Docker si nécessaire (si Docker vient de **snap**, désinstallez-le d'abord).

Ensuite, **immédiatement** :

1. Ouvrez `http://your_vps_ip:8000` et créez le compte admin — la première personne qui arrive sur cette page devient admin.
2. Sauvegardez `/data/coolify/source/.env` en lieu sûr (il contient `APP_KEY`, nécessaire pour restaurer Coolify).
3. Dans les réglages de l'instance, définissez le domaine `https://panel.yourdomain.com` (Coolify génère le certificat).
4. Une fois le domaine fonctionnel, bloquez le port 8000 (ainsi que 6001/6002, utilisés pour le temps réel et le terminal) dans le pare-feu de l'hébergeur, en gardant 22, 80 et 443 ouverts.

Mises à jour : *Settings → Update* dans le tableau de bord (ou activez la mise à jour automatique).

## Installer Dokploy

```bash
curl -sSL https://dokploy.com/install.sh -o dokploy-install.sh
less dokploy-install.sh
sudo sh dokploy-install.sh
```

Les ports 80, 443 et 3000 doivent être libres avant l'installation.

Ensuite :

1. Ouvrez `http://your_vps_ip:3000` et créez le compte admin immédiatement.
2. Dans les réglages du serveur web, définissez le domaine du panel (`panel.yourdomain.com`) avec Let's Encrypt.
3. Une fois le domaine fonctionnel, arrêtez d'exposer le port 3000 :

   ```bash
   docker service update --publish-rm "published=3000,target=3000,mode=host" dokploy
   ```

## Déployer une app MERN / Next.js

Le déroulé est presque identique dans les deux outils :

1. **Connectez GitHub** (*Sources* / *Git*) — installez leur GitHub App pour accéder aux dépôts privés et activer le déploiement par push.
2. **Créez un projet** → ajoutez une **base de données** (MongoDB, PostgreSQL, MySQL, MariaDB, Redis) en un clic. Copiez son URL de connexion **interne** (accessible uniquement par vos apps, pas depuis Internet).
3. **Ajoutez le backend** → choisissez le dépôt, la branche `main`, le dossier de base `backend`, le build pack **Nixpacks** (détecte Node automatiquement) ou votre **Dockerfile**. Indiquez le port sur lequel votre app écoute (par ex. 4000) et faites-la écouter sur `0.0.0.0`.
4. **Variables d'environnement** → `MONGODB_URI` (l'URL interne de l'étape 2), `JWT_SECRET`, `CLIENT_URL`…
5. **Domaine** → `https://api.yourdomain.com` (le HTTPS est automatique).
6. **Ajoutez le frontend** :
   - React/Vite : dossier de base `frontend`, build *Static*, dossier de publication `dist`, et définissez `VITE_API_URL` comme variable **de build**.
   - Next.js : Nixpacks le détecte ; définissez les variables `NEXT_PUBLIC_*` comme variables de build.
7. **Déployez** — ensuite, chaque push sur `main` redéploie automatiquement.

Activez les **sauvegardes planifiées de la base vers S3** (Backblaze B2, Cloudflare R2, AWS…) dans les réglages de la base — c'est intégré.

## Bonnes pratiques

- Sauvegardez et mettez à jour le panel lui-même — il a un contrôle de niveau root sur votre serveur.
- Activez la 2FA sur le compte admin.
- Ne publiez pas les ports des bases de données sauf nécessité ; si vous le faites, restreignez-les dans le pare-feu de l'hébergeur.
- Surveillez l'espace disque : les images Docker et les caches de build grossissent vite. Les deux outils ont une option de nettoyage — planifiez-la. Voir aussi [Monitoring](../server/monitoring.md).
- Certains hébergeurs proposent des modèles d'OS avec Coolify/Dokploy préinstallé — regardez la liste des systèmes de votre hébergeur avant d'installer à la main.
