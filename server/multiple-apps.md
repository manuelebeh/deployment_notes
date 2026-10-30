# Hosting Several Apps on One VPS

[🇫🇷 Lire en français](../fr/server/multiple-apps.md)

One VPS can comfortably run several projects (portfolio, client sites, APIs, staging…). The key is to stay organized so they don't step on each other.

- [1. Folder layout](#1-folder-layout)
- [2. One port per app](#2-one-port-per-app)
- [3. PM2: one process per app](#3-pm2-one-process-per-app)
- [4. Nginx: one file per domain](#4-nginx-one-file-per-domain)
- [5. Databases: one user per app](#5-databases-one-user-per-app)
- [6. Staging environment](#6-staging-environment)
- [7. Isolation (optional): one Linux user per app](#7-isolation-optional-one-linux-user-per-app)
- [8. Resources: when is the VPS full?](#8-resources-when-is-the-vps-full)

---

## 1. Folder layout

```text
/var/www/
├── portfolio/            ← static site / React build
├── shop-api/             ← Express API
├── shop-web/             ← Next.js
├── client-acme/
└── shop-api-staging/
```

One folder per app, one `.env` per app (`chmod 600`), and one `deploy.sh` per app.

## 2. One port per app

Each Node app listens on its own port, **on `127.0.0.1` only**. Keep a small registry so you never reuse a port by mistake — e.g. `/var/www/PORTS.md`:

| Port | App | Domain |
|---|---|---|
| 3000 | shop-web (Next.js) | shop.com |
| 3001 | uptime-kuma | status.shop.com |
| 4000 | shop-api | api.shop.com |
| 4001 | client-acme-api | api.acme.com |
| 4100 | shop-api-staging | api.staging.shop.com |

Check what's in use: `sudo ss -ltnp`

None of these ports is opened in the firewall — only Nginx (80/443) is public.

## 3. PM2: one process per app

Give every process a clear, unique name in its `ecosystem.config.cjs`, and a memory cap so one app can't starve the others:

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

## 4. Nginx: one file per domain

```text
/etc/nginx/sites-available/
├── portfolio.com.conf
├── shop.com.conf
├── api.shop.com.conf
└── staging.shop.com.conf
```

Enable/disable a site without deleting it:

```bash
sudo ln -s /etc/nginx/sites-available/shop.com.conf /etc/nginx/sites-enabled/   # enable
sudo rm /etc/nginx/sites-enabled/shop.com.conf                                 # disable
sudo nginx -t && sudo systemctl reload nginx
```

Certbot can add domains one app at a time: `sudo certbot --nginx -d shop.com -d www.shop.com`. Each command creates a separate certificate — that's fine.

## 5. Databases: one user per app

One database server, but **one database + one user per app**, each user only allowed on its own database:

- MongoDB: `readWrite` on `shop_db` only — [guide](../databases/mongodb.md#4-create-a-database-user-for-your-app)
- PostgreSQL: `CREATE DATABASE shop_db OWNER shop_user` — [guide](../databases/postgresql.md#2-create-a-user-and-a-database)
- MySQL/MariaDB: `GRANT ... ON shop_db.*` — [guide](../databases/mysql.md#3-create-a-database-and-a-user)
- Redis: one ACL user per app with a key prefix (`~shop:*`) — [guide](../databases/redis.md#2-secure-it)

A bug or a leaked password in one app then can't touch the others' data. Back up each database separately ([backups](../databases/backups.md)).

## 6. Staging environment

A copy of production on a subdomain (e.g. `staging.shop.com`) to test before releasing:

- Separate folder, port, PM2 name, `.env` and **database** (`shop_db_staging`) — never point staging at the production database.
- Deploy the `develop`/`staging` branch automatically ([GitHub Actions](../deploy/github-actions-auto-deploy.md)) with its own workflow.
- Hide it from the public and from Google with a password:

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

## 7. Isolation (optional): one Linux user per app

For client projects or apps you trust less, run each one under its own Linux user, so a compromised app can't read the other apps' files and `.env`:

```bash
sudo adduser --disabled-password --gecos "" shop
sudo chown -R shop:shop /var/www/shop-api
sudo chmod 750 /var/www/shop-api
sudo -iu shop                     # work as that user
pm2 start ecosystem.config.cjs    # PM2 runs per user
pm2 startup                       # run the printed sudo command from your deploy user
pm2 save
```

Nginx (`www-data`) must still be able to read static build folders: `sudo usermod -aG shop www-data` and make sure the build folder is group-readable.

## 8. Resources: when is the VPS full?

Rough memory usage per app:

| App | Typical RAM |
|---|---|
| Small Express API | 80–200 MB |
| Next.js app | 200–500 MB |
| MongoDB | 300 MB – 1 GB+ (uses up to ~50% of RAM for cache by default) |
| PostgreSQL / MySQL | 100–500 MB |
| Redis | 10–100 MB (depends on data) |
| Docker + Uptime Kuma | ~150 MB |

Signs you need a bigger plan (or a second VPS): RAM constantly > 85% with swap in use, frequent `max_memory_restart` restarts in `pm2 status`, builds killed, slow responses under normal traffic. Check with `free -h`, `pm2 monit`, `htop` — and set up [alerts](monitoring.md#3-disk--memory-alerts).

Common next step: move the database to its own server (or a managed service), keep apps on the first VPS.

> Upgrading to a bigger plan (e.g. Hostinger KVM1 → KVM2) is usually done from the provider's dashboard without reinstalling — take a snapshot first. See the [VPS links in the README](../README.md#get-a-vps).
