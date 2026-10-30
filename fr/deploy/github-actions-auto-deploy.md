# Déploiement automatique sur un VPS avec GitHub Actions

[🇬🇧 Read in English](../../deploy/github-actions-auto-deploy.md)

Chaque push sur `main` → GitHub se connecte à votre VPS en SSH → lance votre script de déploiement.

Prérequis : votre app est déjà déployée à la main avec un `deploy.sh` fonctionnel (voir [MERN sur un VPS → Déployer les mises à jour](mern-on-vps.md#7-déployer-les-mises-à-jour)).

---

## 1. Créer une clé SSH dédiée à GitHub

Sur **votre ordinateur** (pas sur le serveur) :

```bash
ssh-keygen -t ed25519 -C "github-actions" -f ~/.ssh/gha_deploy -N ""
```

Autorisez la clé publique sur le VPS pour l'utilisateur `deploy` :

```bash
ssh-copy-id -i ~/.ssh/gha_deploy.pub deploy@your_vps_ip
```

Récupérez l'empreinte de la clé d'hôte du serveur (protège contre les attaques de l'homme du milieu) :

```bash
ssh-keyscan -t ed25519 your_vps_ip
```

## 2. Ajouter les secrets GitHub

Dépôt → *Settings* → *Secrets and variables* → *Actions* → *New repository secret* :

| Nom | Valeur |
|---|---|
| `VPS_HOST` | `your_vps_ip` |
| `VPS_USER` | `deploy` |
| `VPS_SSH_KEY` | contenu de `~/.ssh/gha_deploy` (la clé **privée**, lignes `BEGIN`/`END` comprises) |
| `VPS_FINGERPRINT` | empreinte SHA256, obtenue avec `ssh-keyscan -t ed25519 your_vps_ip \| ssh-keygen -lf -` (la partie `SHA256:…`) |

## 3. Ajouter le workflow

Créez `.github/workflows/deploy.yml` dans votre dépôt :

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

Pushez sur `main` et suivez l'onglet *Actions*.

> Si vous avez installé Node avec **nvm**, les sessions SSH non interactives ne le chargent pas. Ajoutez `source ~/.nvm/nvm.sh` en haut du bloc `script:` (ou de `deploy.sh`).

## 4. (Facultatif) Lancer les tests d'abord

Ajoutez un job qui doit réussir avant le déploiement :

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

## 5. (Facultatif) Builder le frontend en CI

Sur un petit VPS, builder React dans GitHub Actions et n'envoyer que `dist/` économise de la RAM et du temps :

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

Retirez ensuite la partie build du frontend de `deploy.sh`.

---

## Conseils de sécurité

- Utilisez une clé **réservée** à la CI, pour pouvoir la révoquer sans toucher à votre propre accès.
- Limitez ce que la clé peut faire en préfixant sa ligne dans `~/.ssh/authorized_keys` sur le serveur :
  `restrict,command="/var/www/your-repo/deploy.sh" ssh-ed25519 AAAA... github-actions`
  (le contenu de `script:` du workflow est alors ignoré — la clé ne peut lancer que le script de déploiement).
- Ajoutez des validateurs obligatoires à l'environnement `production` (*Settings → Environments*) si plusieurs personnes pushent sur `main`.
