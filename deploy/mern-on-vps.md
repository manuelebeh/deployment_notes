# Deploy a MERN Stack Project on a VPS

[🇫🇷 Lire en français](../fr/deploy/mern-on-vps.md)

Deploy an Express/Node.js API and one or more React frontends on a single Ubuntu VPS, with PM2, Nginx and free HTTPS.

> Using Next.js? See [Next.js on a VPS](nextjs-on-vps.md). Running several projects on the same server? See [Several apps on one VPS](../server/multiple-apps.md).

```text
Browser ──HTTPS──▶ Nginx ──┬──▶ /var/www/.../frontend/dist   (React build, static files)
                           └──▶ 127.0.0.1:4000              (Express API, run by PM2)
                                        │
                                        └──▶ MongoDB 127.0.0.1:27017 (local only)
```

- [0. Prerequisites](#0-prerequisites)
- [1. Install Node.js, Git & PM2](#1-install-nodejs-git--pm2)
- [2. Set up the database](#2-set-up-the-database)
- [3. Get your code on the server](#3-get-your-code-on-the-server)
- [4. Run the Express backend with PM2](#4-run-the-express-backend-with-pm2)
- [5. Build the React frontends](#5-build-the-react-frontends)
- [6. Nginx & HTTPS](#6-nginx--https)
- [7. Deploying updates](#7-deploying-updates)
- [Troubleshooting](#troubleshooting)

---

## 0. Prerequisites

### Get your VPS (20% discount)

- [Hostinger VPS KVM1](https://www.hostinger.com/cart?product=vps%3Avps_kvm_1&period=12&referral_type=cart_link&REFERRALCODE=FULEBEHEMFY4&referral_id=01958194-af72-7176-905a-70836d0d6ab7)
- [Hostinger VPS KVM2](https://www.hostinger.com/cart?product=vps%3Avps_kvm_2&period=12&referral_type=cart_link&REFERRALCODE=FULEBEHEMFY4&referral_id=01958196-63c2-703b-9ce3-2149c8ba44c6)
- [Hostinger VPS KVM4](https://www.hostinger.com/cart?product=vps%3Avps_kvm_4&period=12&referral_type=cart_link&REFERRALCODE=FULEBEHEMFY4&referral_id=01958196-3bcc-717b-89a0-a86f7f605700)
- [Hostinger VPS KVM8](https://www.hostinger.com/cart?product=vps%3Avps_kvm_8&period=12&referral_type=cart_link&REFERRALCODE=FULEBEHEMFY4&referral_id=01958196-12eb-71dc-9f2a-3f506c4ebc7c)

### Before you start

- Ubuntu 24.04 or 26.04 VPS
- [Initial server setup](../server/initial-server-setup.md) done (sudo user, SSH keys, firewall)
- A domain name you can edit DNS for
- Your project on GitHub/GitLab

Log in as your sudo user:

```bash
ssh deploy@your_vps_ip
```

## 1. Install Node.js, Git & PM2

Install the current LTS from NodeSource. In October 2026 that's **Node.js 26** (use `setup_24.x` if your project isn't ready for 26 yet). Ubuntu's own `nodejs` package is much older — avoid it.

```bash
sudo apt update
sudo apt install -y curl git build-essential
curl -fsSL https://deb.nodesource.com/setup_26.x | sudo -E bash -
sudo apt install -y nodejs
node -v && npm -v
```

<details>
<summary>Alternative: nvm (several Node versions per user)</summary>

```bash
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.8/install.sh | bash
source ~/.bashrc
nvm install --lts
```

With nvm, Node is installed for your user only. That's fine with PM2 — `pm2 startup` prints a command that includes the right `PATH`.
</details>

Install PM2 (process manager that keeps your API running and restarts it on crash/reboot):

```bash
sudo npm install -g pm2
```

## 2. Set up the database

Pick one and follow its guide, then come back:

- [MongoDB](../databases/mongodb.md) (classic MERN)
- [PostgreSQL](../databases/postgresql.md) · [MySQL](../databases/mysql.md) · [MariaDB](../databases/mariadb.md)
- [Redis](../databases/redis.md) (cache, sessions, queues)

> Using a managed database instead (MongoDB Atlas, Neon, Supabase…)? Skip this step and put the connection string in your `.env`. If the provider has an IP allowlist, add your VPS IP.

## 3. Get your code on the server

Create the web folder and give it to your user:

```bash
sudo mkdir -p /var/www
sudo chown -R $USER:$USER /var/www
cd /var/www
```

**Public repo:**

```bash
git clone https://github.com/yourusername/your-repo.git
```

**Private repo** — use a read-only deploy key instead of your personal credentials:

```bash
ssh-keygen -t ed25519 -C "deploy@your-vps" -f ~/.ssh/github_deploy -N ""
cat ~/.ssh/github_deploy.pub
```

Add that key in GitHub → your repo → *Settings* → *Deploy keys* (read access), then:

```bash
cat >> ~/.ssh/config <<'EOF'
Host github.com
  IdentityFile ~/.ssh/github_deploy
  IdentitiesOnly yes
EOF
git clone git@github.com:yourusername/your-repo.git
```

## 4. Run the Express backend with PM2

```bash
cd /var/www/your-repo/backend
npm ci --omit=dev        # uses package-lock.json; use "npm install" if you have no lock file
```

Create the environment file:

```bash
nano .env
```

```env
NODE_ENV=production
PORT=4000
MONGODB_URI=mongodb://app_user:password@127.0.0.1:27017/app_db?authSource=app_db
JWT_SECRET=change-me-to-a-long-random-string
CLIENT_URL=https://yourdomain.com
```

```bash
chmod 600 .env
```

> Generate secrets with `openssl rand -hex 32`. Make sure `.env` is in `.gitignore`.

In your server code, listen on localhost only — Nginx will be the public entry point:

```js
app.set('trust proxy', 1);
app.listen(process.env.PORT || 4000, '127.0.0.1');
```

Create a PM2 config file at the root of the backend (commit it to your repo):

```bash
nano ecosystem.config.cjs
```

```js
module.exports = {
  apps: [
    {
      name: 'project-backend',
      script: 'server.js',            // your entry file
      cwd: __dirname,
      instances: 1,                   // or 'max' for cluster mode (stateless apps only)
      exec_mode: 'fork',
      max_memory_restart: '500M',
      env: { NODE_ENV: 'production' },
    },
  ],
};
```

Start it and make it survive reboots:

```bash
pm2 start ecosystem.config.cjs
pm2 startup          # copy-paste and run the "sudo env PATH=..." command it prints
pm2 save
```

Rotate logs so they don't fill the disk:

```bash
pm2 install pm2-logrotate
```

Check it answers locally:

```bash
pm2 status
curl -i http://127.0.0.1:4000
```

> **Don't** run `ufw allow 4000`: the API must only be reachable through Nginx (HTTPS). Older versions of this guide opened that port — close it with `sudo ufw delete allow 4000` if you did.

Useful PM2 commands:

```bash
pm2 logs project-backend      # live logs
pm2 restart project-backend
pm2 reload project-backend    # zero-downtime in cluster mode
pm2 monit
```

## 5. Build the React frontends

```bash
cd /var/www/your-repo/frontend
nano .env.production
```

```env
VITE_API_URL=https://api.yourdomain.com
```

> Frontend env vars are **baked into the build** and visible to anyone. Never put secrets there. With Vite they must start with `VITE_` (Create React App used `REACT_APP_`).

```bash
npm ci
npm run build        # Vite outputs to dist/, Create React App to build/
```

Repeat for each additional React app.

> **Low-RAM VPS?** If the build gets killed, add [swap](../server/initial-server-setup.md#8-swap-small-vps), or build on your computer / in CI and upload the `dist/` folder:
> `rsync -avz --delete dist/ deploy@your_vps_ip:/var/www/your-repo/frontend/dist/`

> Create React App is deprecated (since early 2025). New projects should use **Vite**.

## 6. Nginx & HTTPS

Follow the [Nginx & SSL guide](../server/nginx-and-ssl.md). In short:

1. Point `yourdomain.com`, `www` and `api` to your VPS IP (DNS A records).
2. Create one server block per frontend with `root /var/www/your-repo/frontend/dist;` and `try_files $uri $uri/ /index.html;`.
3. Create a server block for `api.yourdomain.com` with `proxy_pass http://127.0.0.1:4000;`.
4. `sudo nginx -t && sudo systemctl reload nginx`
5. `sudo certbot --nginx -d yourdomain.com -d www.yourdomain.com -d api.yourdomain.com`

Your app is now live at `https://yourdomain.com` 🎉

Don't forget to allow your frontend origin in your Express CORS config:

```js
import cors from 'cors';
app.use(cors({ origin: process.env.CLIENT_URL, credentials: true }));
```

## 7. Deploying updates

Save this script on the server as `/var/www/your-repo/deploy.sh` (or commit it):

```bash
#!/usr/bin/env bash
set -euo pipefail

cd /var/www/your-repo
git pull --ff-only

cd backend
npm ci --omit=dev
pm2 reload ecosystem.config.cjs --update-env

cd ../frontend
npm ci
npm run build

echo "✅ Deployed $(git rev-parse --short HEAD)"
```

```bash
chmod +x /var/www/your-repo/deploy.sh
/var/www/your-repo/deploy.sh
```

Nginx serves the new `dist/` immediately — no reload needed.

To run it automatically on every push, see [Auto-deploy with GitHub Actions](github-actions-auto-deploy.md).

---

## Troubleshooting

| Problem | Check |
|---|---|
| 502 Bad Gateway | `pm2 status` / `pm2 logs` — app crashed or wrong port |
| App works on `curl 127.0.0.1:4000` but not via domain | Nginx config, DNS, `sudo nginx -t` |
| CORS errors in the browser | `origin` in `cors()` must exactly match `https://yourdomain.com` (no trailing slash) |
| Frontend still calls `localhost` | `VITE_API_URL` wasn't set **before** `npm run build` |
| `npm run build` → `Killed` | Out of memory → add swap or build elsewhere |
| App doesn't restart after reboot | Run the command printed by `pm2 startup`, then `pm2 save` |
| `EADDRINUSE` | Port already used → `sudo ss -ltnp 'sport = :4000'` |

**Next:** [automate deployments](github-actions-auto-deploy.md), set up [database backups](../databases/backups.md) and [monitoring](../server/monitoring.md).
