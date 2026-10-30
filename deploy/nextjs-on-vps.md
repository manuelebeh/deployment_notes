# Deploy a Next.js App on a VPS

[🇫🇷 Lire en français](../fr/deploy/nextjs-on-vps.md)

Self-host a Next.js app (App Router, Server Components, API routes, `next/image`, ISR…) on Ubuntu with PM2 and Nginx. Everything works self-hosted — no Vercel required.

> Written for **Next.js 16** (October 2026). Requires Node.js 20.9+; this guide uses Node.js 26 LTS.

```text
Browser ──HTTPS──▶ Nginx ──▶ 127.0.0.1:3000  (Next.js server, run by PM2)
                     └──▶ /_next/static      (optional: served directly from disk)
```

- [0. Prerequisites](#0-prerequisites)
- [1. Two ways to run Next.js](#1-two-ways-to-run-nextjs)
- [2. Option A — `next start` (simplest)](#2-option-a--next-start-simplest)
- [3. Option B — standalone build (lighter, zero-downtime)](#3-option-b--standalone-build-lighter-zero-downtime)
- [4. Environment variables](#4-environment-variables)
- [5. Nginx](#5-nginx)
- [6. HTTPS](#6-https)
- [Troubleshooting](#troubleshooting)

---

## 0. Prerequisites

- [Initial server setup](../server/initial-server-setup.md) done
- Node.js, Git and PM2 installed — see [MERN guide, step 1](mern-on-vps.md#1-install-nodejs-git--pm2)
- Your code cloned in `/var/www/your-next-app` — see [MERN guide, step 3](mern-on-vps.md#3-get-your-code-on-the-server)
- At least **2 GB RAM** (or [swap](../server/initial-server-setup.md#8-swap-small-vps)): `next build` is memory-hungry

## 1. Two ways to run Next.js

| | Option A: `next start` | Option B: `output: 'standalone'` |
|---|---|---|
| Setup | Easiest | A few extra steps |
| Disk / RAM | Full `node_modules` | Only the files actually used (much smaller) |
| Updates | Short downtime while rebuilding | Zero downtime + instant rollback |
| Best for | Side projects, first deploy | Production, small VPS |

## 2. Option A — `next start` (simplest)

```bash
cd /var/www/your-next-app
npm ci
npm run build
```

Create `ecosystem.config.cjs`:

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

**Updating:**

```bash
cd /var/www/your-next-app
git pull --ff-only
npm ci
npm run build
pm2 reload next-app
```

> `npm run build` rewrites the `.next/` folder that the running server uses, so the site can return errors for the duration of the build. If that's a problem, use Option B.

## 3. Option B — standalone build (lighter, zero-downtime)

Enable standalone output in `next.config.js` (or `.ts` / `.mjs`):

```js
/** @type {import('next').NextConfig} */
const nextConfig = {
  output: 'standalone',
};

module.exports = nextConfig;
```

The build creates `.next/standalone/` with a minimal `server.js` and only the `node_modules` it needs. `public/` and `.next/static/` are **not** copied automatically — the deploy script below does it.

Each deploy goes into its own `releases/<timestamp>` folder and a `current` symlink points to the live one:

```text
/var/www/your-next-app/
├── repo/          ← git clone, where you build
├── releases/
│   ├── 20261030-101500/
│   └── 20261030-143000/
├── current -> releases/20261030-143000
└── shared/.env    ← runtime secrets, kept between releases
```

Set it up once:

```bash
cd /var/www
mkdir -p your-next-app/{releases,shared}
git clone git@github.com:yourusername/your-next-app.git your-next-app/repo
nano your-next-app/shared/.env && chmod 600 your-next-app/shared/.env
```

`/var/www/your-next-app/deploy.sh`:

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

`/var/www/your-next-app/ecosystem.config.cjs`:

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

The old version keeps serving traffic while the new one builds, and the switch takes a second.

**Rollback** to the previous release:

```bash
cd /var/www/your-next-app
ln -sfn "$(ls -1dt releases/*/ | sed -n 2p)" current
pm2 reload next-app
```

> Build on a small VPS fails with `Killed`? Build in GitHub Actions instead and upload the assembled release folder — see [Auto-deploy with GitHub Actions](github-actions-auto-deploy.md).

## 4. Environment variables

| Kind | Example | When it's read |
|---|---|---|
| `NEXT_PUBLIC_*` | `NEXT_PUBLIC_API_URL` | **At build time** — inlined in the browser JS. Must exist before `npm run build`. Never put secrets here. |
| Everything else | `DATABASE_URL`, `AUTH_SECRET` | On the server, at runtime (dynamic routes) — and at build time for statically generated pages |

- Option A: put them in `.env.production` (or `.env`) in the project folder before building.
- Option B: put them in `shared/.env`. `deploy.sh` loads them before the build, and PM2 passes them to the server at runtime with Node's `--env-file`.

Generate secrets with `openssl rand -base64 32`.

## 5. Nginx

`/etc/nginx/sites-available/yourdomain.com.conf`:

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

`$connection_upgrade` must be defined once — see [Nginx & SSL, step 4](../server/nginx-and-ssl.md#4-server-block-nodejs-api-reverse-proxy).

<details>
<summary>Optional: serve <code>/_next/static</code> directly from disk (Option B)</summary>

Add inside the `server` block — Nginx then serves hashed assets without hitting Node:

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

Details in [Nginx & SSL](../server/nginx-and-ssl.md#7-https-with-certbot).

---

## Troubleshooting

| Problem | Fix |
|---|---|
| `npm run build` → `Killed` / `JavaScript heap out of memory` | Add swap, or build in CI |
| `NEXT_PUBLIC_*` value is `undefined` in the browser | It wasn't set **before** the build — rebuild |
| Standalone: CSS/JS 404, unstyled page | `.next/static` and `public/` weren't copied into the release |
| Standalone: `Cannot find module 'next'` | You started `server.js` from the repo instead of `.next/standalone` / `current` |
| `loading.js` / Suspense doesn't stream | `proxy_buffering off;` missing in Nginx |
| `Failed to find Server Action` after a deploy | Old tab using a previous build — reloading fixes it. For several instances, set `NEXT_SERVER_ACTIONS_ENCRYPTION_KEY` |
| `next/image` uses a lot of RAM | Normal on first requests (optimized images are then cached in `.next/cache/images`); lower `max_memory_restart` or set `images.unoptimized: true` |

**References:** [Next.js self-hosting guide](https://nextjs.org/docs/app/guides/self-hosting) · [`output: 'standalone'`](https://nextjs.org/docs/app/api-reference/config/next-config-js/output)
