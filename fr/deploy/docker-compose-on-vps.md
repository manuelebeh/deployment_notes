# Déployer avec Docker Compose sur un VPS

[🇬🇧 Read in English](../../deploy/docker-compose-on-vps.md)

Faites tourner votre API Node.js et sa base de données dans des conteneurs, avec le Nginx de l'hôte devant pour le HTTPS. Même serveur, mais reproductible : `docker compose up -d` et c'est en ligne.

```text
Browser ──HTTPS──▶ Nginx (host) ──┬──▶ /var/www/.../frontend/dist (static)
                                  └──▶ 127.0.0.1:4000 ──▶ [api container] ──▶ [mongo container]
```

Prérequis : [configuration initiale du serveur](../server/initial-server-setup.md).

- [1. Installer Docker Engine](#1-installer-docker-engine)
- [Avertissement pare-feu](#avertissement-pare-feu)
- [2. Dockerfile de l'API](#2-dockerfile-de-lapi)
- [3. compose.yaml](#3-composeyaml)
- [4. Démarrer](#4-démarrer)
- [5. Nginx et HTTPS](#5-nginx-et-https)
- [6. Mises à jour et maintenance](#6-mises-à-jour-et-maintenance)

---

## 1. Installer Docker Engine

Dépôt officiel (compatible Ubuntu 22.04, 24.04, 26.04). Supprimez d'abord les anciens paquets non officiels :

```bash
for pkg in docker.io docker-doc docker-compose docker-compose-v2 podman-docker containerd runc; do sudo apt remove -y $pkg; done
```

Ajoutez le dépôt Docker :

```bash
sudo apt update
sudo apt install -y ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

sudo tee /etc/apt/sources.list.d/docker.sources <<EOF
Types: deb
URIs: https://download.docker.com/linux/ubuntu
Suites: $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}")
Components: stable
Architectures: $(dpkg --print-architecture)
Signed-By: /etc/apt/keyrings/docker.asc
EOF

sudo apt update
```

Installez :

```bash
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
sudo docker run hello-world
```

Utiliser Docker sans `sudo` (déconnectez-vous puis reconnectez-vous ensuite) :

```bash
sudo usermod -aG docker $USER
```

> Faire partie du groupe `docker` équivaut à un accès root. N'y ajoutez que des utilisateurs de confiance.

Limitez la taille des logs des conteneurs (sinon ils grossissent indéfiniment) :

```bash
sudo tee /etc/docker/daemon.json <<'EOF'
{
  "log-driver": "local",
  "log-opts": { "max-size": "10m", "max-file": "3" }
}
EOF
sudo systemctl restart docker
```

## Avertissement pare-feu

Docker écrit ses propres règles iptables : un port publié en `"4000:4000"` est **accessible depuis Internet même si UFW le bloque**.

Publiez toujours sur localhost uniquement, et laissez Nginx être le point d'entrée public :

```yaml
ports:
  - "127.0.0.1:4000:4000"   # ✅ local only
  # - "4000:4000"           # ❌ exposed to the whole internet
```

Les bases de données n'ont besoin d'aucun port publié — les conteneurs d'un même projet Compose se joignent par le nom du service (`mongo:27017`).

## 2. Dockerfile de l'API

`backend/Dockerfile` :

```dockerfile
FROM node:26-alpine

WORKDIR /app
ENV NODE_ENV=production

COPY package*.json ./
RUN npm ci --omit=dev && npm cache clean --force

COPY . .

USER node
EXPOSE 4000
CMD ["node", "server.js"]
```

`backend/.dockerignore` :

```text
node_modules
.env
.git
npm-debug.log
```

> Dans un conteneur, l'app doit écouter sur `0.0.0.0` (pas `127.0.0.1`), sinon Docker ne peut pas lui transmettre le trafic. C'est le `127.0.0.1:` dans `ports:` qui la garde privée.

## 3. compose.yaml

À la racine de votre dépôt :

```yaml
services:
  api:
    build: ./backend
    restart: unless-stopped
    env_file: ./backend/.env
    environment:
      MONGODB_URI: mongodb://${MONGO_USER}:${MONGO_PASSWORD}@mongo:27017/app_db?authSource=admin
    ports:
      - "127.0.0.1:4000:4000"
    depends_on:
      mongo:
        condition: service_healthy

  mongo:
    image: mongo:9.0
    restart: unless-stopped
    environment:
      MONGO_INITDB_ROOT_USERNAME: ${MONGO_USER}
      MONGO_INITDB_ROOT_PASSWORD: ${MONGO_PASSWORD}
    volumes:
      - mongo-data:/data/db
    healthcheck:
      test: ["CMD", "mongosh", "--quiet", "--eval", "db.adminCommand('ping')"]
      interval: 10s
      timeout: 5s
      retries: 5

volumes:
  mongo-data:
```

Créez un `.env` à côté de `compose.yaml` (Compose le lit automatiquement pour les variables `${…}`) :

```env
MONGO_USER=admin
MONGO_PASSWORD=change-me-long-random-password
```

```bash
chmod 600 .env backend/.env
```

> Pour garder l'exemple court, l'API se connecte avec l'utilisateur root de MongoDB. Pour un vrai projet, créez un utilisateur dédié en `readWrite` (`docker compose exec mongo mongosh -u admin -p`, puis l'étape 4 du [guide MongoDB](../databases/mongodb.md#4-créer-un-utilisateur-pour-votre-app)) et utilisez-le dans `MONGODB_URI`.

<details>
<summary>PostgreSQL à la place de MongoDB</summary>

```yaml
  postgres:
    image: postgres:18
    restart: unless-stopped
    environment:
      POSTGRES_USER: ${DB_USER}
      POSTGRES_PASSWORD: ${DB_PASSWORD}
      POSTGRES_DB: app_db
    volumes:
      - pg-data:/var/lib/postgresql
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${DB_USER} -d app_db"]
      interval: 10s
      retries: 5
```

Chaîne de connexion depuis l'API : `postgresql://${DB_USER}:${DB_PASSWORD}@postgres:5432/app_db`.
À noter : à partir de l'image `postgres:18`, le volume se monte sur `/var/lib/postgresql` (et non plus `/var/lib/postgresql/data` comme dans les anciennes versions).

</details>

## 4. Démarrer

```bash
cd /var/www/your-repo
docker compose up -d --build
docker compose ps
docker compose logs -f api
curl -i http://127.0.0.1:4000
```

## 5. Nginx et HTTPS

Exactement comme sans Docker : buildez votre frontend, faites pointer le `root` de Nginx vers `dist/`, `proxy_pass http://127.0.0.1:4000;` pour l'API, puis Certbot. Suivez [Nginx & SSL](../server/nginx-and-ssl.md).

## 6. Mises à jour et maintenance

Déployer une nouvelle version :

```bash
cd /var/www/your-repo
git pull --ff-only
docker compose up -d --build
docker image prune -f
```

Commandes utiles :

```bash
docker compose logs -f --tail=100 api    # logs
docker compose restart api               # restart one service
docker compose exec mongo mongosh -u admin -p   # open a DB shell
docker compose down                      # stop (volumes are kept)
docker system df                         # disk usage
```

> `docker compose down -v` **supprime les volumes — et donc votre base de données**. N'utilisez jamais `-v` en production.

Sauvegarder la base conteneurisée :

```bash
docker compose exec -T mongo sh -c 'mongodump --archive --gzip -u "$MONGO_INITDB_ROOT_USERNAME" -p "$MONGO_INITDB_ROOT_PASSWORD" --authenticationDatabase admin' > backup-$(date +%F).archive.gz
```

Voir [Sauvegardes des bases de données](../databases/backups.md) pour la planification et les copies hors serveur. Pour déployer à chaque push, réutilisez le [déploiement automatique avec GitHub Actions](github-actions-auto-deploy.md) avec les commandes ci-dessus dans `deploy.sh`.
