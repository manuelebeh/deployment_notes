# Deploy with Docker Compose on a VPS

Run your Node.js API and its database in containers, with the host's Nginx in front for HTTPS. Same server, but reproducible: `docker compose up -d` and you're done.

```
Browser ──HTTPS──▶ Nginx (host) ──┬──▶ /var/www/.../frontend/dist (static)
                                  └──▶ 127.0.0.1:4000 ──▶ [api container] ──▶ [mongo container]
```

Prerequisite: [initial server setup](../server/initial-server-setup.md).

- [1. Install Docker Engine](#1-install-docker-engine)
- [Firewall warning](#firewall-warning)
- [2. Dockerfile for the API](#2-dockerfile-for-the-api)
- [3. compose.yaml](#3-composeyaml)
- [4. Start it](#4-start-it)
- [5. Nginx & HTTPS](#5-nginx--https)
- [6. Updates & maintenance](#6-updates--maintenance)

---

## 1. Install Docker Engine

Official repository (supports Ubuntu 22.04, 24.04, 26.04). Remove old/unofficial packages first:

```bash
for pkg in docker.io docker-doc docker-compose docker-compose-v2 podman-docker containerd runc; do sudo apt remove -y $pkg; done
```

Add Docker's repository:

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

Install:

```bash
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
sudo docker run hello-world
```

Run Docker without `sudo` (log out and back in afterwards):

```bash
sudo usermod -aG docker $USER
```

> Being in the `docker` group is equivalent to root access. Only add trusted users.

Limit container log size (otherwise logs grow forever):

```bash
sudo tee /etc/docker/daemon.json <<'EOF'
{
  "log-driver": "local",
  "log-opts": { "max-size": "10m", "max-file": "3" }
}
EOF
sudo systemctl restart docker
```

## Firewall warning

Docker writes its own iptables rules: a port published as `"4000:4000"` is **reachable from the internet even if UFW blocks it**.

Always publish on localhost only, and let Nginx be the public entry point:

```yaml
ports:
  - "127.0.0.1:4000:4000"   # ✅ local only
  # - "4000:4000"           # ❌ exposed to the whole internet
```

Databases don't need published ports at all — containers in the same Compose project reach each other by service name (`mongo:27017`).

## 2. Dockerfile for the API

`backend/Dockerfile`:

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

`backend/.dockerignore`:

```
node_modules
.env
.git
npm-debug.log
```

> Inside a container the app must listen on `0.0.0.0` (not `127.0.0.1`), otherwise Docker can't forward traffic to it. The `127.0.0.1:` in `ports:` is what keeps it private.

## 3. compose.yaml

At the root of your repo:

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

Create a `.env` next to `compose.yaml` (Compose reads it automatically for `${…}` variables):

```env
MONGO_USER=admin
MONGO_PASSWORD=change-me-long-random-password
```

```bash
chmod 600 .env backend/.env
```

<details>
<summary>PostgreSQL instead of MongoDB</summary>

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

Connection string from the API: `postgresql://${DB_USER}:${DB_PASSWORD}@postgres:5432/app_db`.
Note: from the `postgres:18` image on, the volume is mounted at `/var/lib/postgresql` (not `/var/lib/postgresql/data` as in older versions).
</details>

## 4. Start it

```bash
cd /var/www/your-repo
docker compose up -d --build
docker compose ps
docker compose logs -f api
curl -i http://127.0.0.1:4000
```

## 5. Nginx & HTTPS

Exactly the same as without Docker: build your frontend, point Nginx's `root` to `dist/`, `proxy_pass http://127.0.0.1:4000;` for the API, then Certbot. Follow [Nginx & SSL](../server/nginx-and-ssl.md).

## 6. Updates & maintenance

Deploy a new version:

```bash
cd /var/www/your-repo
git pull --ff-only
docker compose up -d --build
docker image prune -f
```

Useful commands:

```bash
docker compose logs -f --tail=100 api    # logs
docker compose restart api               # restart one service
docker compose exec mongo mongosh -u admin -p   # open a DB shell
docker compose down                      # stop (volumes are kept)
docker system df                         # disk usage
```

> `docker compose down -v` **deletes the volumes — and your database**. Don't use `-v` in production.

Backup the containerized database:

```bash
docker compose exec -T mongo sh -c 'mongodump --archive --gzip -u "$MONGO_INITDB_ROOT_USERNAME" -p "$MONGO_INITDB_ROOT_PASSWORD" --authenticationDatabase admin' > backup-$(date +%F).archive.gz
```

See [Database backups](../databases/backups.md) for scheduling and off-site copies. To deploy on every push, reuse [GitHub Actions auto-deploy](github-actions-auto-deploy.md) with the commands above in `deploy.sh`.
