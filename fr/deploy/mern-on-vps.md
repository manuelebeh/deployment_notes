# Déployer un projet MERN sur un VPS

[🇬🇧 Read in English](../../deploy/mern-on-vps.md)

Déployez une API Express/Node.js et un ou plusieurs frontends React sur un seul VPS Ubuntu, avec PM2, Nginx et le HTTPS gratuit.

> Vous utilisez Next.js ? Voir [Next.js sur un VPS](nextjs-on-vps.md). Plusieurs projets sur le même serveur ? Voir [Plusieurs applications sur un VPS](../server/multiple-apps.md).

```text
Browser ──HTTPS──▶ Nginx ──┬──▶ /var/www/.../frontend/dist   (React build, static files)
                           └──▶ 127.0.0.1:4000              (Express API, run by PM2)
                                        │
                                        └──▶ MongoDB 127.0.0.1:27017 (local only)
```

- [0. Prérequis](#0-prérequis)
- [1. Installer Node.js, Git et PM2](#1-installer-nodejs-git-et-pm2)
- [2. Mettre en place la base de données](#2-mettre-en-place-la-base-de-données)
- [3. Récupérer votre code sur le serveur](#3-récupérer-votre-code-sur-le-serveur)
- [4. Lancer le backend Express avec PM2](#4-lancer-le-backend-express-avec-pm2)
- [5. Builder les frontends React](#5-builder-les-frontends-react)
- [6. Nginx et HTTPS](#6-nginx-et-https)
- [7. Déployer les mises à jour](#7-déployer-les-mises-à-jour)
- [Dépannage](#dépannage)

---

## 0. Prérequis

### Obtenir votre VPS (20 % de réduction)

- [Hostinger VPS KVM1](https://www.hostinger.com/cart?product=vps%3Avps_kvm_1&period=12&referral_type=cart_link&REFERRALCODE=FULEBEHEMFY4&referral_id=01958194-af72-7176-905a-70836d0d6ab7)
- [Hostinger VPS KVM2](https://www.hostinger.com/cart?product=vps%3Avps_kvm_2&period=12&referral_type=cart_link&REFERRALCODE=FULEBEHEMFY4&referral_id=01958196-63c2-703b-9ce3-2149c8ba44c6)
- [Hostinger VPS KVM4](https://www.hostinger.com/cart?product=vps%3Avps_kvm_4&period=12&referral_type=cart_link&REFERRALCODE=FULEBEHEMFY4&referral_id=01958196-3bcc-717b-89a0-a86f7f605700)
- [Hostinger VPS KVM8](https://www.hostinger.com/cart?product=vps%3Avps_kvm_8&period=12&referral_type=cart_link&REFERRALCODE=FULEBEHEMFY4&referral_id=01958196-12eb-71dc-9f2a-3f506c4ebc7c)

### Avant de commencer

- Un VPS Ubuntu 24.04 ou 26.04
- La [configuration initiale du serveur](../server/initial-server-setup.md) faite (utilisateur sudo, clés SSH, pare-feu)
- Un nom de domaine dont vous pouvez modifier les DNS
- Votre projet sur GitHub/GitLab

Connectez-vous avec votre utilisateur sudo :

```bash
ssh deploy@your_vps_ip
```

## 1. Installer Node.js, Git et PM2

Installez la LTS actuelle depuis NodeSource. En octobre 2026, c'est **Node.js 26** (utilisez `setup_24.x` si votre projet n'est pas encore prêt pour la 26). Le paquet `nodejs` d'Ubuntu est beaucoup plus ancien — évitez-le.

```bash
sudo apt update
sudo apt install -y curl git build-essential
curl -fsSL https://deb.nodesource.com/setup_26.x | sudo -E bash -
sudo apt install -y nodejs
node -v && npm -v
```

<details>
<summary>Alternative : nvm (plusieurs versions de Node par utilisateur)</summary>

```bash
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.8/install.sh | bash
source ~/.bashrc
nvm install --lts
```

Avec nvm, Node n'est installé que pour votre utilisateur. Ça fonctionne avec PM2 — `pm2 startup` affiche une commande qui inclut le bon `PATH`.

</details>

Installez PM2 (gestionnaire de processus qui garde votre API en marche et la relance en cas de crash ou de redémarrage) :

```bash
sudo npm install -g pm2
```

## 2. Mettre en place la base de données

Choisissez-en une, suivez son guide, puis revenez ici :

- [MongoDB](../databases/mongodb.md) (le MERN classique)
- [PostgreSQL](../databases/postgresql.md) · [MySQL](../databases/mysql.md) · [MariaDB](../databases/mariadb.md)
- [Redis](../databases/redis.md) (cache, sessions, files d'attente)

> Vous utilisez plutôt une base managée (MongoDB Atlas, Neon, Supabase…) ? Sautez cette étape et mettez la chaîne de connexion dans votre `.env`. Si le service a une liste d'IP autorisées, ajoutez l'IP de votre VPS.

## 3. Récupérer votre code sur le serveur

Créez le dossier web et donnez-le à votre utilisateur :

```bash
sudo mkdir -p /var/www
sudo chown -R $USER:$USER /var/www
cd /var/www
```

**Dépôt public :**

```bash
git clone https://github.com/yourusername/your-repo.git
```

**Dépôt privé** — utilisez une clé de déploiement en lecture seule plutôt que vos identifiants personnels :

```bash
ssh-keygen -t ed25519 -C "deploy@your-vps" -f ~/.ssh/github_deploy -N ""
cat ~/.ssh/github_deploy.pub
```

Ajoutez cette clé dans GitHub → votre dépôt → *Settings* → *Deploy keys* (accès en lecture), puis :

```bash
cat >> ~/.ssh/config <<'EOF'
Host github.com
  IdentityFile ~/.ssh/github_deploy
  IdentitiesOnly yes
EOF
git clone git@github.com:yourusername/your-repo.git
```

## 4. Lancer le backend Express avec PM2

```bash
cd /var/www/your-repo/backend
npm ci --omit=dev        # uses package-lock.json; use "npm install" if you have no lock file
```

Créez le fichier d'environnement :

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

> Générez les secrets avec `openssl rand -hex 32`. Vérifiez que `.env` est bien dans `.gitignore`.

Dans le code serveur, n'écoutez qu'en local — c'est Nginx qui sera le point d'entrée public :

```js
app.set('trust proxy', 1);
app.listen(process.env.PORT || 4000, '127.0.0.1');
```

Créez un fichier de config PM2 à la racine du backend (commitez-le dans votre dépôt) :

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

Démarrez-le et faites-le survivre aux redémarrages :

```bash
pm2 start ecosystem.config.cjs
pm2 startup          # copy-paste and run the "sudo env PATH=..." command it prints
pm2 save
```

Faites tourner les logs pour qu'ils ne remplissent pas le disque :

```bash
pm2 install pm2-logrotate
```

Vérifiez qu'il répond en local :

```bash
pm2 status
curl -i http://127.0.0.1:4000
```

> **Ne lancez pas** `ufw allow 4000` : l'API ne doit être joignable qu'à travers Nginx (HTTPS). Les anciennes versions de ce guide ouvraient ce port — fermez-le avec `sudo ufw delete allow 4000` si vous l'aviez fait.

Commandes PM2 utiles :

```bash
pm2 logs project-backend      # live logs
pm2 restart project-backend
pm2 reload project-backend    # zero-downtime in cluster mode
pm2 monit
```

## 5. Builder les frontends React

```bash
cd /var/www/your-repo/frontend
nano .env.production
```

```env
VITE_API_URL=https://api.yourdomain.com
```

> Les variables d'environnement du frontend sont **intégrées dans le build** et visibles par tout le monde. N'y mettez jamais de secrets. Avec Vite, elles doivent commencer par `VITE_` (Create React App utilisait `REACT_APP_`).

```bash
npm ci
npm run build        # Vite outputs to dist/, Create React App to build/
```

Répétez pour chaque app React supplémentaire.

> **VPS avec peu de RAM ?** Si le build est tué, ajoutez du [swap](../server/initial-server-setup.md#8-swap-petit-vps), ou buildez sur votre ordinateur / en CI et envoyez seulement le dossier `dist/` :
> `rsync -avz --delete dist/ deploy@your_vps_ip:/var/www/your-repo/frontend/dist/`

> Create React App est déprécié (depuis début 2025). Les nouveaux projets doivent utiliser **Vite**.

## 6. Nginx et HTTPS

Suivez le [guide Nginx & SSL](../server/nginx-and-ssl.md). En résumé :

1. Faites pointer `yourdomain.com`, `www` et `api` vers l'IP de votre VPS (enregistrements DNS A).
2. Créez un bloc serveur par frontend avec `root /var/www/your-repo/frontend/dist;` et `try_files $uri $uri/ /index.html;`.
3. Créez un bloc serveur pour `api.yourdomain.com` avec `proxy_pass http://127.0.0.1:4000;`.
4. `sudo nginx -t && sudo systemctl reload nginx`
5. `sudo certbot --nginx -d yourdomain.com -d www.yourdomain.com -d api.yourdomain.com`

Votre app est maintenant en ligne sur `https://yourdomain.com` 🎉

N'oubliez pas d'autoriser l'origine de votre frontend dans la config CORS d'Express :

```js
import cors from 'cors';
app.use(cors({ origin: process.env.CLIENT_URL, credentials: true }));
```

## 7. Déployer les mises à jour

Enregistrez ce script sur le serveur sous `/var/www/your-repo/deploy.sh` (ou commitez-le) :

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

Nginx sert immédiatement le nouveau `dist/` — pas besoin de le recharger.

Pour le lancer automatiquement à chaque push, voir [Déploiement automatique avec GitHub Actions](github-actions-auto-deploy.md).

---

## Dépannage

| Problème | À vérifier |
|---|---|
| 502 Bad Gateway | `pm2 status` / `pm2 logs` — app plantée ou mauvais port |
| L'app répond sur `curl 127.0.0.1:4000` mais pas via le domaine | Config Nginx, DNS, `sudo nginx -t` |
| Erreurs CORS dans le navigateur | L'`origin` de `cors()` doit correspondre exactement à `https://yourdomain.com` (sans slash final) |
| Le frontend appelle encore `localhost` | `VITE_API_URL` n'était pas défini **avant** `npm run build` |
| `npm run build` → `Killed` | Manque de mémoire → ajoutez du swap ou buildez ailleurs |
| L'app ne redémarre pas après un reboot | Lancez la commande affichée par `pm2 startup`, puis `pm2 save` |
| `EADDRINUSE` | Port déjà utilisé → `sudo ss -ltnp 'sport = :4000'` |

**Ensuite :** [automatisez les déploiements](github-actions-auto-deploy.md), mettez en place les [sauvegardes de la base](../databases/backups.md) et le [monitoring](../server/monitoring.md).
