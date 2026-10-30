# Redis sur un VPS

[🇬🇧 Read in English](../../databases/redis.md)

Redis est une base de données en mémoire utilisée pour le **cache, les sessions, la limitation de débit et les files de tâches** (BullMQ, etc.). Ce guide installe Redis Open Source 8.x depuis le dépôt officiel et le verrouille.

- [1. Installation](#1-installation)
- [2. Sécuriser](#2-sécuriser)
- [3. Mémoire et persistance](#3-mémoire-et-persistance)
- [4. Se connecter depuis Node.js](#4-se-connecter-depuis-nodejs)
- [Commandes utiles](#commandes-utiles)

---

## 1. Installation

Le paquet `redis-server` d'Ubuntu fonctionne mais il est plus ancien. Pour la dernière version de Redis Open Source (8.x) :

```bash
sudo apt install -y lsb-release curl gpg
curl -fsSL https://packages.redis.io/gpg | sudo gpg --dearmor -o /usr/share/keyrings/redis-archive-keyring.gpg
sudo chmod 644 /usr/share/keyrings/redis-archive-keyring.gpg
echo "deb [signed-by=/usr/share/keyrings/redis-archive-keyring.gpg] https://packages.redis.io/deb $(lsb_release -cs) main" | \
  sudo tee /etc/apt/sources.list.d/redis.list
sudo apt update
sudo apt install -y redis
```

```bash
sudo systemctl enable --now redis-server
redis-cli ping     # → PONG
```

> **Valkey** est un fork communautaire de Redis (licence BSD, Linux Foundation) et un remplaçant direct — même protocole, mêmes clients. Les deux conviennent pour les usages ci-dessus.

## 2. Sécuriser

```bash
sudo nano /etc/redis/redis.conf
```

Vérifiez / définissez ces lignes :

```conf
# Local only (default)
bind 127.0.0.1 -::1
protected-mode yes
port 6379
# Generate with: openssl rand -hex 32
requirepass a-very-long-random-password
```

> `redis.conf` n'accepte les commentaires que sur **leur propre ligne** — un `# commentaire` en fin de ligne casse la configuration.

```bash
sudo systemctl restart redis-server
redis-cli -a 'a-very-long-random-password' ping
```

<details>
<summary>Alternative : un utilisateur ACL par app (plusieurs apps sur le même serveur)</summary>

Au lieu de `requirepass`, désactivez l'utilisateur partagé `default` et donnez à chaque app son propre utilisateur :

```conf
user default off
user app_user on >another-long-password ~app:* &* +@all
```

`~app:*` limite cet utilisateur aux clés qui commencent par `app:`. Connexion avec `redis-cli --user app_user --pass 'another-long-password'`.

</details>

> N'exposez jamais le port 6379 sur Internet. Si un autre serveur a besoin de Redis, passez par un tunnel SSH ou un réseau privé.

## 3. Mémoire et persistance

Plafonnez la mémoire pour que Redis ne puisse pas faire tomber votre serveur, et choisissez ce qui se passe quand elle est pleine :

```conf
maxmemory 256mb
# Pure cache: evict least recently used keys
maxmemory-policy allkeys-lru
# Queues/sessions instead (BullMQ requires noeviction):
# maxmemory-policy noeviction
```

Persistance (garder les données après un redémarrage) — le fichier append-only est le plus sûr :

```conf
appendonly yes
appendfsync everysec
```

Réglage noyau recommandé (évite les échecs de sauvegarde en arrière-plan) :

```bash
echo 'vm.overcommit_memory = 1' | sudo tee /etc/sysctl.d/99-redis.conf
sudo sysctl --system
sudo systemctl restart redis-server
```

## 4. Se connecter depuis Node.js

URL de connexion :

```bash
redis://:password@127.0.0.1:6379/0              # with requirepass
redis://app_user:password@127.0.0.1:6379/0      # with an ACL user
```

Avec [`redis`](https://www.npmjs.com/package/redis) (node-redis) :

```js
import { createClient } from 'redis';

const redis = createClient({ url: process.env.REDIS_URL });
redis.on('error', (err) => console.error('Redis error', err));
await redis.connect();

await redis.set('app:hello', 'world', { EX: 60 });   // expires in 60 s
```

Avec [`ioredis`](https://www.npmjs.com/package/ioredis) (utilisé par BullMQ) :

```js
import Redis from 'ioredis';
const redis = new Redis(process.env.REDIS_URL, { maxRetriesPerRequest: null });
```

---

## Commandes utiles

```bash
redis-cli -a 'password'          # open a shell (or: redis-cli, then AUTH password)
redis-cli -a 'password' info memory
redis-cli -a 'password' --scan --pattern 'app:*' | head
redis-cli -a 'password' monitor  # live commands (debug only)
sudo journalctl -u redis-server -n 50
```
