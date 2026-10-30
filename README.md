# Deployment Notes

🇬🇧 English · [🇫🇷 Français](fr/README.md)

Practical, copy-paste friendly guides to deploy JavaScript (MERN / Node.js) projects on a Linux VPS or on Vercel — from a fresh server to HTTPS, databases, automatic deployments and backups.

> Last reviewed: **October 2026** — targets **Ubuntu 24.04 LTS** and **Ubuntu 26.04 LTS**.

---

## Get a VPS

The guides work on any Ubuntu VPS. If you need one, here are Hostinger plans with a **20% discount**:

- [Hostinger VPS KVM1](https://www.hostinger.com/cart?product=vps%3Avps_kvm_1&period=12&referral_type=cart_link&REFERRALCODE=FULEBEHEMFY4&referral_id=01958194-af72-7176-905a-70836d0d6ab7)
- [Hostinger VPS KVM2](https://www.hostinger.com/cart?product=vps%3Avps_kvm_2&period=12&referral_type=cart_link&REFERRALCODE=FULEBEHEMFY4&referral_id=01958196-63c2-703b-9ce3-2149c8ba44c6)
- [Hostinger VPS KVM4](https://www.hostinger.com/cart?product=vps%3Avps_kvm_4&period=12&referral_type=cart_link&REFERRALCODE=FULEBEHEMFY4&referral_id=01958196-3bcc-717b-89a0-a86f7f605700)
- [Hostinger VPS KVM8](https://www.hostinger.com/cart?product=vps%3Avps_kvm_8&period=12&referral_type=cart_link&REFERRALCODE=FULEBEHEMFY4&referral_id=01958196-12eb-71dc-9f2a-3f506c4ebc7c)

When choosing the OS, pick a plain **Ubuntu 24.04** or **Ubuntu 26.04** template.

---

## Guides

### 1. Server

| Guide | What you get |
|---|---|
| [Initial server setup](server/initial-server-setup.md) | Non-root sudo user, SSH keys, hardened SSH, firewall, Fail2ban, automatic security updates, swap |
| [Nginx & SSL (Let's Encrypt)](server/nginx-and-ssl.md) | Domain/DNS, Nginx server blocks for SPAs and APIs, free HTTPS with Certbot, auto-renewal |
| [Caddy (alternative to Nginx)](server/caddy.md) | Automatic HTTPS, a whole MERN/Next.js config in ~10 lines |
| [Cloudflare in front of your VPS](server/cloudflare.md) | Proxy/CDN, Full (strict) SSL, real visitor IPs, origin lockdown, error codes |
| [Monitoring & alerts](server/monitoring.md) | Health checks, uptime monitoring (Uptime Kuma), disk/memory alerts, logs |
| [Several apps on one VPS](server/multiple-apps.md) | Ports, PM2, Nginx, databases per app, staging, isolation, sizing |

### 2. Deploy

| Guide | What you get |
|---|---|
| [MERN stack on a VPS](deploy/mern-on-vps.md) | Node.js, Express API with PM2, React (Vite) builds, Nginx, HTTPS, update workflow |
| [Next.js on a VPS](deploy/nextjs-on-vps.md) | `next start` or standalone build, zero-downtime releases & rollback, streaming |
| [Docker Compose on a VPS](deploy/docker-compose-on-vps.md) | Docker Engine, a Node API + database in containers behind Nginx |
| [Auto-deploy with GitHub Actions](deploy/github-actions-auto-deploy.md) | Deploy on every push to `main` over SSH |
| [MERN on Vercel](deploy/vercel-mern.md) | Zero-config Express backend, React Router rewrites, `vercel.json` reference |
| [Coolify & Dokploy](deploy/coolify-dokploy.md) | Self-hosted PaaS: push-to-deploy from a dashboard on your own VPS |

### 3. Databases

| Guide | Version covered |
|---|---|
| [MongoDB](databases/mongodb.md) | MongoDB 9.0 Community |
| [PostgreSQL](databases/postgresql.md) | PostgreSQL 16 / 18 |
| [MySQL](databases/mysql.md) | MySQL 8.0 / 8.4 LTS (9.7 LTS optional) |
| [MariaDB](databases/mariadb.md) | MariaDB 10.11 / 11.8 LTS (12.3 LTS optional) |
| [Redis](databases/redis.md) | Redis Open Source 8.x |
| [Database backups](databases/backups.md) | Automated daily dumps + restore for all of the above |

---

## Recommended order for a new VPS

1. [Initial server setup](server/initial-server-setup.md)
2. A database guide (e.g. [MongoDB](databases/mongodb.md))
3. [MERN stack](deploy/mern-on-vps.md) or [Next.js](deploy/nextjs-on-vps.md) — or [Docker Compose](deploy/docker-compose-on-vps.md)
4. [Nginx & SSL](server/nginx-and-ssl.md) (or [Caddy](server/caddy.md)), optionally behind [Cloudflare](server/cloudflare.md)
5. [Auto-deploy with GitHub Actions](deploy/github-actions-auto-deploy.md)
6. [Database backups](databases/backups.md)
7. [Monitoring & alerts](server/monitoring.md)

Prefer a dashboard to the command line? Start with [Coolify & Dokploy](deploy/coolify-dokploy.md) on a fresh VPS instead of steps 3–5.

---

## Versions used in these guides (October 2026)

| Software | Recommended | Notes |
|---|---|---|
| Ubuntu | 24.04 LTS or 26.04 LTS | 22.04 still works for most steps but reaches end of standard support in April 2027 |
| Node.js | **26 LTS** (or 24 LTS) | Node 26 became Active LTS on 28 Oct 2026; Node 24 is in maintenance until April 2028; Node 20 is end-of-life |
| Nginx | Ubuntu package | 1.24 on 24.04, 1.28 on 26.04 |
| Caddy | 2.11 | Official repo |
| Next.js | 16 | |
| Certbot | snap | Official EFF recommendation |
| MongoDB | 9.0 | Official `mongodb-org` repo |
| PostgreSQL | 18 | Default on 26.04; via the official PGDG repo on 24.04 |
| MySQL | 8.4 LTS | Default on 26.04 (8.0 on 24.04) |
| MariaDB | 11.8 LTS | Default on 26.04 (10.11 on 24.04) |
| Redis | 8.x | Official `packages.redis.io` repo |
| PM2 | 7.x | `npm install -g pm2` |
| Docker Engine | latest | Official repo |
| Coolify / Dokploy | 4.5 / 0.30 | |

---

## Conventions

- `yourdomain.com`, `api.yourdomain.com`, `your_vps_ip`, `deploy`, `app_db`, `app_user`… are **placeholders** — replace them with your own values.
- Commands are written for a **non-root user with `sudo`** (see [initial server setup](server/initial-server-setup.md)). If you are logged in as `root`, you can drop `sudo`.
- In `nano`: save with `Ctrl + O` → `Enter`, exit with `Ctrl + X`.
- Never commit `.env` files or passwords to Git.

## Contributing

Found an outdated command or a typo? [Open an issue](https://github.com/manuelebeh/deployment_notes/issues/new/choose) or a pull request — see [CONTRIBUTING.md](CONTRIBUTING.md).

## License

[MIT](LICENSE)
