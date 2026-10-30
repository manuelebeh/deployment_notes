# Redis on a VPS

[🇫🇷 Lire en français](../fr/databases/redis.md)

Redis is an in-memory data store used for **caching, sessions, rate limiting and job queues** (BullMQ, etc.). This guide installs Redis Open Source 8.x from the official repository and locks it down.

- [1. Install](#1-install)
- [2. Secure it](#2-secure-it)
- [3. Memory & persistence](#3-memory--persistence)
- [4. Connect from Node.js](#4-connect-from-nodejs)
- [Useful commands](#useful-commands)

---

## 1. Install

Ubuntu's own `redis-server` package works but is older. For the latest Redis Open Source (8.x):

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

> **Valkey** is a community fork of Redis (BSD license, Linux Foundation) and a drop-in replacement — same protocol, same clients. Either works for the uses above.

## 2. Secure it

```bash
sudo nano /etc/redis/redis.conf
```

Check / set these lines:

```conf
# Local only (default)
bind 127.0.0.1 -::1
protected-mode yes
port 6379
# Generate with: openssl rand -hex 32
requirepass a-very-long-random-password
```

> `redis.conf` only accepts comments on their **own line** — a `# comment` at the end of a setting breaks the config.

```bash
sudo systemctl restart redis-server
redis-cli -a 'a-very-long-random-password' ping
```

<details>
<summary>Alternative: one ACL user per app (several apps on the same server)</summary>

Instead of `requirepass`, disable the shared `default` user and give each app its own user:

```conf
user default off
user app_user on >another-long-password ~app:* &* +@all
```

`~app:*` restricts this user to keys starting with `app:`. Connect with `redis-cli --user app_user --pass 'another-long-password'`.
</details>

> Never expose port 6379 to the internet. If another server needs Redis, use an SSH tunnel or a private network.

## 3. Memory & persistence

Cap memory so Redis can't take down your server, and choose what happens when it's full:

```conf
maxmemory 256mb
# Pure cache: evict least recently used keys
maxmemory-policy allkeys-lru
# Queues/sessions instead (BullMQ requires noeviction):
# maxmemory-policy noeviction
```

Persistence (keep data across restarts) — append-only file is the safest:

```conf
appendonly yes
appendfsync everysec
```

Recommended kernel setting (avoids failed background saves):

```bash
echo 'vm.overcommit_memory = 1' | sudo tee /etc/sysctl.d/99-redis.conf
sudo sysctl --system
sudo systemctl restart redis-server
```

## 4. Connect from Node.js

Connection URL:

```bash
redis://:password@127.0.0.1:6379/0              # with requirepass
redis://app_user:password@127.0.0.1:6379/0      # with an ACL user
```

With [`redis`](https://www.npmjs.com/package/redis) (node-redis):

```js
import { createClient } from 'redis';

const redis = createClient({ url: process.env.REDIS_URL });
redis.on('error', (err) => console.error('Redis error', err));
await redis.connect();

await redis.set('app:hello', 'world', { EX: 60 });   // expires in 60 s
```

With [`ioredis`](https://www.npmjs.com/package/ioredis) (used by BullMQ):

```js
import Redis from 'ioredis';
const redis = new Redis(process.env.REDIS_URL, { maxRetriesPerRequest: null });
```

---

## Useful commands

```bash
redis-cli -a 'password'          # open a shell (or: redis-cli, then AUTH password)
redis-cli -a 'password' info memory
redis-cli -a 'password' --scan --pattern 'app:*' | head
redis-cli -a 'password' monitor  # live commands (debug only)
sudo journalctl -u redis-server -n 50
```
