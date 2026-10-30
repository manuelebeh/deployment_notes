# MongoDB on a VPS (Community 9.0)

[🇫🇷 Lire en français](../fr/databases/mongodb.md)

Install MongoDB from the official repository, turn on authentication, and create a user for your app.

Supported: **Ubuntu 22.04 (jammy), 24.04 (noble), 26.04 (resolute)** — x86_64 and ARM64.

- [1. Install](#1-install)
- [2. Start the service](#2-start-the-service)
- [3. Enable authentication (do not skip)](#3-enable-authentication-do-not-skip)
- [4. Create a database user for your app](#4-create-a-database-user-for-your-app)
- [5. Connection strings](#5-connection-strings)
- [6. Remote access (only if you need it)](#6-remote-access-only-if-you-need-it)
- [Troubleshooting](#troubleshooting)

---

## 1. Install

Check the CPU supports AVX (required by MongoDB 5.0+ on x86_64 — most VPS do):

```bash
grep -q avx /proc/cpuinfo && echo "AVX OK" || echo "No AVX: use MongoDB Atlas or Docker on another host"
```

Remove Ubuntu's unofficial `mongodb` package if it's installed (it conflicts):

```bash
sudo apt remove -y mongodb mongodb-server 2>/dev/null || true
```

Import MongoDB's key:

```bash
sudo apt install -y gnupg curl
curl -fsSL https://pgp.mongodb.com/server-9.asc | \
  sudo gpg -o /usr/share/keyrings/mongodb-server-9.gpg --dearmor
```

Add the repository — this line picks your Ubuntu codename automatically:

```bash
echo "deb [ arch=amd64,arm64 signed-by=/usr/share/keyrings/mongodb-server-9.gpg ] https://repo.mongodb.org/apt/ubuntu $(. /etc/os-release && echo "$VERSION_CODENAME")/mongodb-org/9.0 multiverse" | \
  sudo tee /etc/apt/sources.list.d/mongodb-org-9.0.list
```

Install (includes the server `mongod`, the shell `mongosh` and the database tools):

```bash
sudo apt update
sudo apt install -y mongodb-org
```

<details>
<summary>Need MongoDB 8.0 instead? (e.g. to match an existing cluster)</summary>

Replace every `9` / `9.0` above with `8.0`:

```bash
curl -fsSL https://www.mongodb.org/static/pgp/server-8.0.asc | \
  sudo gpg -o /usr/share/keyrings/mongodb-server-8.0.gpg --dearmor
echo "deb [ arch=amd64,arm64 signed-by=/usr/share/keyrings/mongodb-server-8.0.gpg ] https://repo.mongodb.org/apt/ubuntu $(. /etc/os-release && echo "$VERSION_CODENAME")/mongodb-org/8.0 multiverse" | \
  sudo tee /etc/apt/sources.list.d/mongodb-org-8.0.list
```

Upgrading an existing server from 8.0 to 9.0? Back up first, then follow MongoDB's upgrade notes (the `featureCompatibilityVersion` must be `8.0` before switching binaries).
</details>

## 2. Start the service

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now mongod
sudo systemctl status mongod
```

Quick test:

```bash
mongosh --eval 'db.runCommand({ ping: 1 })'
```

At this point MongoDB listens on `127.0.0.1:27017` only, **without authentication**: any program on the server can read and write everything.

## 3. Enable authentication (do not skip)

Create an admin user:

```bash
mongosh
```

```js
use admin
db.createUser({
  user: "admin",
  pwd: passwordPrompt(),          // asks for the password, keeps it out of your shell history
  roles: [ { role: "root", db: "admin" } ]
})
exit
```

Turn on authorization:

```bash
sudo nano /etc/mongod.conf
```

```yaml
net:
  port: 27017
  bindIp: 127.0.0.1

security:
  authorization: enabled
```

> YAML is indentation-sensitive: use **2 spaces**, no tabs. `security:` is commented out (`#security:`) by default — remove the `#`.

```bash
sudo systemctl restart mongod
```

Log in as admin from now on:

```bash
mongosh -u admin --authenticationDatabase admin
```

## 4. Create a database user for your app

Never use the `admin`/`root` user in your application. Create one user per app with access to its own database only:

```js
use app_db
db.createUser({
  user: "app_user",
  pwd: passwordPrompt(),
  roles: [ { role: "readWrite", db: "app_db" } ]
})
exit
```

Test it:

```bash
mongosh "mongodb://app_user@127.0.0.1:27017/app_db?authSource=app_db"
```

## 5. Connection strings

```bash
# Without auth (local testing only)
mongodb://127.0.0.1:27017/app_db

# App user created in app_db (step 4)
mongodb://app_user:password@127.0.0.1:27017/app_db?authSource=app_db
```

- Use `127.0.0.1` rather than `localhost` (avoids IPv6 `::1` resolution issues with Node.js).
- If the password contains `@ : / ? # %`, URL-encode it: `node -e 'console.log(encodeURIComponent(process.argv[1]))' 'p@ss:word'`

## 6. Remote access (only if you need it)

Prefer an **SSH tunnel** — nothing to open on the server. From your computer:

```bash
ssh -N -L 27017:127.0.0.1:27017 deploy@your_vps_ip
```

Then connect MongoDB Compass to `mongodb://admin:password@127.0.0.1:27017/?authSource=admin`.

If you really need direct access (e.g. another server), only with auth enabled, and only from known IPs:

```yaml
# /etc/mongod.conf
net:
  bindIp: 127.0.0.1,your_vps_private_or_public_ip
```

```bash
sudo ufw allow from 203.0.113.10 to any port 27017 proto tcp
sudo systemctl restart mongod
```

> Never `ufw allow 27017` for everyone. Exposed MongoDB instances are scanned and ransomed within hours.

---

## Troubleshooting

| Problem | Fix |
|---|---|
| `Illegal instruction (core dumped)` | CPU without AVX — see step 1 |
| `mongod.service not found` | `sudo systemctl daemon-reload` |
| Service fails after editing `mongod.conf` | YAML indentation → `sudo journalctl -u mongod -n 50` |
| `Authentication failed` | Wrong `authSource` — it's the DB where the user was created |
| `ECONNREFUSED ::1:27017` in Node | Use `127.0.0.1` instead of `localhost` |
| Permission errors on `/var/lib/mongodb` | `sudo chown -R mongodb:mongodb /var/lib/mongodb /var/log/mongodb` |

Logs: `sudo tail -f /var/log/mongodb/mongod.log`

**Next:** [automatic backups](backups.md#mongodb).
