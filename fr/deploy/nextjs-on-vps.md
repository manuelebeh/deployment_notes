# Déployer une app Next.js sur un VPS

[🇬🇧 Read in English](../../deploy/nextjs-on-vps.md)

Auto-hébergez une app Next.js (App Router, Server Components, routes d'API, `next/image`, ISR…) sur Ubuntu avec PM2 et Nginx. Tout fonctionne en auto-hébergement — Vercel n'est pas obligatoire.

> Écrit pour **Next.js 16** (octobre 2026). Nécessite Node.js 20.9+ ; ce guide utilise Node.js 26 LTS.

```text
Browser ──HTTPS──▶ Nginx ──▶ 127.0.0.1:3000  (Next.js server, run by PM2)
                     └──▶ /_next/static      (optional: served directly from disk)
```

- [0. Prérequis](#0-prérequis)
- [1. Deux façons de lancer Next.js](#1-deux-façons-de-lancer-nextjs)
- [2. Option A — `next start` (le plus simple)](#2-option-a--next-start-le-plus-simple)
- [3. Option B — build standalone (plus léger, sans coupure)](#3-option-b--build-standalone-plus-léger-sans-coupure)
- [4. Variables d'environnement](#4-variables-denvironnement)
- [5. Nginx](#5-nginx)
- [6. HTTPS](#6-https)
- [Dépannage](#dépannage)

---

## 0. Prérequis

- La [configuration initiale du serveur](../server/initial-server-setup.md) faite
- Node.js, Git et PM2 installés — voir le [guide MERN, étape 1](mern-on-vps.md#1-installer-nodejs-git-et-pm2)
- Votre code cloné dans `/var/www/your-next-app` — voir le [guide MERN, étape 3](mern-on-vps.md#3-récupérer-votre-code-sur-le-serveur)
- Au moins **2 Go de RAM** (ou du [swap](../server/initial-server-setup.md#8-swap-petit-vps)) : `next build` est gourmand en mémoire

## 1. Deux façons de lancer Next.js

| | Option A : `next start` | Option B : `output: 'standalone'` |
|---|---|---|
| Mise en place | La plus simple | Quelques étapes de plus |
| Disque / RAM | `node_modules` complet | Uniquement les fichiers réellement utilisés (bien plus léger) |
| Mises à jour | Courte coupure pendant le rebuild | Aucune coupure + rollback instantané |
| Idéal pour | Projets perso, premier déploiement | Production, petit VPS |

## 2. Option A — `next start` (le plus simple)

```bash
cd /var/www/your-next-app
npm ci
npm run build
```

Créez `ecosystem.config.cjs` :

```js
module.exports = {
  apps: [
    {
      name: 'next-app',
      script: 'node_modules/next/dist/bin/next',
      args: 'start -H 127.0.0.1 -p 3000',
      cwd: __dirname,
      max_memory_restart: '700M',
      env: { NODE_ENV: 'production' },
    },
  ],
};
```

```bash
pm2 start ecosystem.config.cjs
pm2 save
pm2 startup      # run the command it prints (once per server)
curl -I http://127.0.0.1:3000
```

**Mettre à jour :**

```bash
cd /var/www/your-next-app
git pull --ff-only
npm ci
npm run build
pm2 reload next-app
```

> `npm run build` réécrit le dossier `.next/` utilisé par le serveur en marche : le site peut renvoyer des erreurs pendant toute la durée du build. Si c'est gênant, utilisez l'option B.

## 3. Option B — build standalone (plus léger, sans coupure)

Activez la sortie standalone dans `next.config.js` (ou `.ts` / `.mjs`) :

```js
/** @type {import('next').NextConfig} */
const nextConfig = {
  output: 'standalone',
};

module.exports = nextConfig;
```

Le build crée `.next/standalone/` avec un `server.js` minimal et uniquement les `node_modules` nécessaires. `public/` et `.next/static/` ne sont **pas** copiés automatiquement — le script de déploiement ci-dessous s'en charge.

Chaque déploiement va dans son propre dossier `releases/<horodatage>` et un lien symbolique `current` pointe vers la version en ligne :

```text
/var/www/your-next-app/
├── repo/          ← git clone, where you build
├── releases/
│   ├── 20261030-101500/
│   └── 20261030-143000/
├── current -> releases/20261030-143000
└── shared/.env    ← runtime secrets, kept between releases
```

Mise en place (une seule fois) :

```bash
cd /var/www
mkdir -p your-next-app/{releases,shared}
git clone git@github.com:yourusername/your-next-app.git your-next-app/repo
nano your-next-app/shared/.env && chmod 600 your-next-app/shared/.env
```

`/var/www/your-next-app/deploy.sh` :

```bash
#!/usr/bin/env bash
set -euo pipefail

APP=/var/www/your-next-app
RELEASE=$APP/releases/$(date +%Y%m%d-%H%M%S)

# Make env vars (incl. NEXT_PUBLIC_*) available to the build
set -a; source $APP/shared/.env; set +a

cd $APP/repo
git pull --ff-only
npm ci
npm run build

# Assemble the release
mkdir -p "$RELEASE"
cp -r .next/standalone/. "$RELEASE"/
cp -r public "$RELEASE"/public 2>/dev/null || true
mkdir -p "$RELEASE"/.next && cp -r .next/static "$RELEASE"/.next/static

# Switch traffic to the new release
ln -sfn "$RELEASE" $APP/current
pm2 startOrReload $APP/ecosystem.config.cjs --update-env

# Keep the last 5 releases
ls -1dt $APP/releases/*/ | tail -n +6 | xargs -r rm -rf

echo "✅ Deployed $(git rev-parse --short HEAD) → $RELEASE"
```

`/var/www/your-next-app/ecosystem.config.cjs` :

```js
module.exports = {
  apps: [
    {
      name: 'next-app',
      cwd: '/var/www/your-next-app/current',
      script: 'server.js',
      node_args: '--env-file=/var/www/your-next-app/shared/.env',
      max_memory_restart: '700M',
      env: { NODE_ENV: 'production', PORT: 3000, HOSTNAME: '127.0.0.1' },
    },
  ],
};
```

```bash
chmod +x /var/www/your-next-app/deploy.sh
/var/www/your-next-app/deploy.sh
pm2 save && pm2 startup
```

L'ancienne version continue de servir le trafic pendant que la nouvelle se construit, et la bascule prend une seconde.

**Rollback** vers la version précédente :

```bash
cd /var/www/your-next-app
ln -sfn "$(ls -1dt releases/*/ | sed -n 2p)" current
pm2 reload next-app
```

> Le build échoue avec `Killed` sur un petit VPS ? Buildez plutôt dans GitHub Actions et envoyez le dossier de release assemblé — voir [Déploiement automatique avec GitHub Actions](github-actions-auto-deploy.md).

## 4. Variables d'environnement

| Type | Exemple | Quand elle est lue |
|---|---|---|
| `NEXT_PUBLIC_*` | `NEXT_PUBLIC_API_URL` | **Au build** — intégrée dans le JS du navigateur. Doit exister avant `npm run build`. Jamais de secrets ici. |
| Toutes les autres | `DATABASE_URL`, `AUTH_SECRET` | Côté serveur, à l'exécution (routes dynamiques) — et au build pour les pages générées statiquement |

- Option A : mettez-les dans `.env.production` (ou `.env`) dans le dossier du projet avant de builder.
- Option B : mettez-les dans `shared/.env`. `deploy.sh` les charge avant le build, et PM2 les transmet au serveur à l'exécution grâce à l'option `--env-file` de Node.

Générez les secrets avec `openssl rand -base64 32`.

## 5. Nginx

`/etc/nginx/sites-available/yourdomain.com.conf` :

```nginx
server {
    listen 80;
    listen [::]:80;
    server_name yourdomain.com www.yourdomain.com;

    client_max_body_size 20M;

    location / {
        proxy_pass http://127.0.0.1:3000;
        proxy_http_version 1.1;
        proxy_set_header Host              $host;
        proxy_set_header X-Real-IP         $remote_addr;
        proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header Upgrade           $http_upgrade;
        proxy_set_header Connection        $connection_upgrade;

        # Needed for streaming / Suspense / loading.js
        proxy_buffering off;
    }
}
```

`$connection_upgrade` doit être défini une fois — voir [Nginx & SSL, étape 4](../server/nginx-and-ssl.md#4-bloc-serveur--api-nodejs-reverse-proxy).

<details>
<summary>Facultatif : servir <code>/_next/static</code> directement depuis le disque (option B)</summary>

À ajouter dans le bloc `server` — Nginx sert alors les fichiers versionnés sans passer par Node :

```nginx
location /_next/static/ {
    alias /var/www/your-next-app/current/.next/static/;
    expires 1y;
    add_header Cache-Control "public, immutable";
    access_log off;
}
```

</details>

```bash
sudo ln -s /etc/nginx/sites-available/yourdomain.com.conf /etc/nginx/sites-enabled/
sudo nginx -t && sudo systemctl reload nginx
```

## 6. HTTPS

```bash
sudo certbot --nginx -d yourdomain.com -d www.yourdomain.com
```

Détails dans [Nginx & SSL](../server/nginx-and-ssl.md#7-https-avec-certbot).

---

## Dépannage

| Problème | Solution |
|---|---|
| `npm run build` → `Killed` / `JavaScript heap out of memory` | Ajoutez du swap, ou buildez en CI |
| Une valeur `NEXT_PUBLIC_*` vaut `undefined` dans le navigateur | Elle n'était pas définie **avant** le build — rebuildez |
| Standalone : CSS/JS en 404, page sans style | `.next/static` et `public/` n'ont pas été copiés dans la release |
| Standalone : `Cannot find module 'next'` | Vous avez lancé `server.js` depuis le dépôt au lieu de `.next/standalone` / `current` |
| `loading.js` / Suspense ne streame pas | Il manque `proxy_buffering off;` dans Nginx |
| `Failed to find Server Action` après un déploiement | Un onglet ouvert utilise l'ancien build — recharger la page suffit. Avec plusieurs instances, définissez `NEXT_SERVER_ACTIONS_ENCRYPTION_KEY` |
| `next/image` consomme beaucoup de RAM | Normal lors des premières requêtes (les images optimisées sont ensuite en cache dans `.next/cache/images`) ; baissez `max_memory_restart` ou mettez `images.unoptimized: true` |

**Références :** [Guide d'auto-hébergement Next.js](https://nextjs.org/docs/app/guides/self-hosting) · [`output: 'standalone'`](https://nextjs.org/docs/app/api-reference/config/next-config-js/output)
