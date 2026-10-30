# Coolify & Dokploy: Your Own "Vercel" on a VPS

[🇫🇷 Lire en français](../fr/deploy/coolify-dokploy.md)

[Coolify](https://coolify.io) and [Dokploy](https://dokploy.com) are free, open-source, **self-hosted PaaS**. You install them once on your VPS, then deploy apps from a web dashboard: connect a GitHub repo, set env vars and a domain, and every push redeploys — with HTTPS, databases and backups included.

- [When to use them](#when-to-use-them)
- [Coolify vs Dokploy](#coolify-vs-dokploy)
- [Before installing](#before-installing)
- [Install Coolify](#install-coolify)
- [Install Dokploy](#install-dokploy)
- [Deploy a MERN / Next.js app](#deploy-a-mern--nextjs-app)
- [Good practices](#good-practices)

---

## When to use them

| Use a PaaS (Coolify/Dokploy) if… | Do it by hand ([MERN](mern-on-vps.md), [Next.js](nextjs-on-vps.md)) if… |
|---|---|
| You deploy many apps and want a dashboard | You have one or two apps |
| You want push-to-deploy, preview deployments, one-click databases | You want to understand and control every piece |
| You're fine with everything running in Docker | You have a small VPS (< 2 GB RAM) |
| | You prefer fewer moving parts to maintain |

## Coolify vs Dokploy

| | Coolify | Dokploy |
|---|---|---|
| Latest version (Oct 2026) | v4.5.x | v0.30.x |
| Minimum server | 2 vCPU, 2 GB RAM, 20 GB+ disk | 2 GB RAM, 30 GB disk |
| Reverse proxy | Traefik (or Caddy) | Traefik |
| Builds | Nixpacks, Dockerfile, Docker Compose, static | Nixpacks, Railpack, Dockerfile, Buildpacks, Docker Compose |
| One-click services | Hundreds (databases, n8n, Uptime Kuma, Plausible…) | Templates (databases, many apps) |
| Multi-server | Yes | Yes (Docker Swarm) |
| Dashboard port | 8000 | 3000 |
| Community | Very large | Growing fast |

Both are good. Coolify has more ready-made services; Dokploy is lighter and has a very clean UI.

## Before installing

- Use a **fresh** VPS (Ubuntu 24.04 LTS recommended). Don't install it on a server already running Nginx/Caddy from the other guides: Coolify/Dokploy need ports **80 and 443** for their own proxy.
- Do the [initial server setup](../server/initial-server-setup.md) first (user, SSH keys, Fail2ban, updates).
- **Docker bypasses UFW** ([details](docker-compose-on-vps.md#firewall-warning)): ports published by these tools are reachable even if UFW says otherwise. Use your **provider's firewall** (Hostinger: hPanel → VPS → *Firewall*) to restrict the dashboard port.
- Point a domain for the dashboard to the VPS, e.g. `panel.yourdomain.com`.

## Install Coolify

```bash
curl -fsSL https://cdn.coollabs.io/coolify/install.sh -o coolify-install.sh
less coolify-install.sh            # read what you're about to run
sudo bash coolify-install.sh
```

The script installs Docker if needed (if Docker came from **snap**, remove it first).

Then, **right away**:

1. Open `http://your_vps_ip:8000` and create the admin account — the first person to reach this page becomes admin.
2. Back up `/data/coolify/source/.env` somewhere safe (contains `APP_KEY`, needed to restore Coolify).
3. In the instance settings, set the domain to `https://panel.yourdomain.com` (Coolify issues the certificate).
4. Once the domain works, block port 8000 (and 6001/6002, used for realtime/terminal) in the provider firewall, keeping 22, 80, 443 open.

Updates: *Settings → Update* in the dashboard (or enable auto-update).

## Install Dokploy

```bash
curl -sSL https://dokploy.com/install.sh -o dokploy-install.sh
less dokploy-install.sh
sudo sh dokploy-install.sh
```

Ports 80, 443 and 3000 must be free before installing.

Then:

1. Open `http://your_vps_ip:3000` and create the admin account immediately.
2. In the web server settings, set the panel domain (`panel.yourdomain.com`) with Let's Encrypt.
3. Once the domain works, stop exposing port 3000:

   ```bash
   docker service update --publish-rm "published=3000,target=3000,mode=host" dokploy
   ```

## Deploy a MERN / Next.js app

The flow is nearly the same in both tools:

1. **Connect GitHub** (*Sources* / *Git*) — install their GitHub App to access private repos and get push-to-deploy.
2. **Create a project** → add a **database** (MongoDB, PostgreSQL, MySQL, MariaDB, Redis) with one click. Copy its **internal** connection URL (it's reachable only by your apps, not from the internet).
3. **Add the backend** → choose the repo, branch `main`, base directory `backend`, build pack **Nixpacks** (auto-detects Node) or your **Dockerfile**. Set the port your app listens on (e.g. 4000) and make it listen on `0.0.0.0`.
4. **Environment variables** → `MONGODB_URI` (internal URL from step 2), `JWT_SECRET`, `CLIENT_URL`…
5. **Domain** → `https://api.yourdomain.com` (HTTPS is automatic).
6. **Add the frontend**:
   - React/Vite: base directory `frontend`, *Static* build, publish directory `dist`, and set `VITE_API_URL` as a **build-time** variable.
   - Next.js: Nixpacks detects it; set `NEXT_PUBLIC_*` variables as build-time.
7. **Deploy** — then every push to `main` redeploys automatically.

Enable **scheduled database backups to S3** (Backblaze B2, Cloudflare R2, AWS…) in the database settings — they're built in.

## Good practices

- Keep the panel itself backed up and updated — it has root-level control over your server.
- Enable 2FA on the admin account.
- Don't publish database ports publicly unless you need to; if you do, restrict them in the provider firewall.
- Monitor disk usage: Docker images and build caches grow fast. Both tools have a cleanup option — schedule it. See also [Monitoring](../server/monitoring.md).
- Some hosting providers offer ready-made Coolify/Dokploy OS templates — check your provider's OS list before installing by hand.
