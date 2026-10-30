# Auto-Deploy to a VPS with GitHub Actions

[🇫🇷 Lire en français](../fr/deploy/github-actions-auto-deploy.md)

Every push to `main` → GitHub connects to your VPS over SSH → runs your deploy script.

Prerequisite: your app is already deployed manually with a working `deploy.sh` (see [MERN on a VPS → Deploying updates](mern-on-vps.md#7-deploying-updates)).

---

## 1. Create a dedicated SSH key for GitHub

On **your computer** (not the server):

```bash
ssh-keygen -t ed25519 -C "github-actions" -f ~/.ssh/gha_deploy -N ""
```

Authorize the public key on the VPS for the `deploy` user:

```bash
ssh-copy-id -i ~/.ssh/gha_deploy.pub deploy@your_vps_ip
```

Get the server's host key fingerprint (protects against man-in-the-middle):

```bash
ssh-keyscan -t ed25519 your_vps_ip
```

## 2. Add GitHub secrets

Repo → *Settings* → *Secrets and variables* → *Actions* → *New repository secret*:

| Name | Value |
|---|---|
| `VPS_HOST` | `your_vps_ip` |
| `VPS_USER` | `deploy` |
| `VPS_SSH_KEY` | content of `~/.ssh/gha_deploy` (the **private** key, including the `BEGIN`/`END` lines) |
| `VPS_FINGERPRINT` | SHA256 fingerprint, from `ssh-keyscan -t ed25519 your_vps_ip \| ssh-keygen -lf -` (the `SHA256:…` part) |

## 3. Add the workflow

Create `.github/workflows/deploy.yml` in your repo:

```yaml
name: Deploy to VPS

on:
  push:
    branches: [main]
  workflow_dispatch:        # allows a manual "Run workflow" button

concurrency:
  group: production-deploy
  cancel-in-progress: false # never run two deploys at the same time

jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: production
    steps:
      - name: Deploy over SSH
        uses: appleboy/ssh-action@v1.2.5
        with:
          host: ${{ secrets.VPS_HOST }}
          username: ${{ secrets.VPS_USER }}
          key: ${{ secrets.VPS_SSH_KEY }}
          fingerprint: ${{ secrets.VPS_FINGERPRINT }}
          script: |
            set -e
            /var/www/your-repo/deploy.sh
```

Push to `main` and watch the *Actions* tab.

> If you installed Node with **nvm**, non-interactive SSH sessions don't load it. Add `source ~/.nvm/nvm.sh` at the top of the `script:` block (or of `deploy.sh`).

## 4. (Optional) Run tests first

Add a job that must pass before deploying:

```yaml
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v7
      - uses: actions/setup-node@v7
        with:
          node-version: 26
          cache: npm
          cache-dependency-path: backend/package-lock.json
      - run: npm ci
        working-directory: backend
      - run: npm test
        working-directory: backend

  deploy:
    needs: test
    # ... same as above
```

## 5. (Optional) Build the frontend in CI

On a small VPS, building React in GitHub Actions and uploading only `dist/` saves RAM and time:

```yaml
      - uses: actions/checkout@v7
      - uses: actions/setup-node@v7
        with: { node-version: 26, cache: npm, cache-dependency-path: frontend/package-lock.json }
      - run: npm ci && npm run build
        working-directory: frontend
        env:
          VITE_API_URL: https://api.yourdomain.com
      - name: Upload build
        uses: appleboy/scp-action@v1
        with:
          host: ${{ secrets.VPS_HOST }}
          username: ${{ secrets.VPS_USER }}
          key: ${{ secrets.VPS_SSH_KEY }}
          fingerprint: ${{ secrets.VPS_FINGERPRINT }}
          source: frontend/dist/*
          target: /var/www/your-repo/frontend/dist
          strip_components: 2
          rm: true
```

Then remove the frontend build part from `deploy.sh`.

---

## Security tips

- Use a key **only** for CI, so you can revoke it without touching your own access.
- Restrict what the key can do by prefixing its line in `~/.ssh/authorized_keys` on the server:
  `restrict,command="/var/www/your-repo/deploy.sh" ssh-ed25519 AAAA... github-actions`
  (then the workflow's `script:` content is ignored — the key can only run the deploy script).
- Add required reviewers to the `production` environment (*Settings → Environments*) if several people push to `main`.
